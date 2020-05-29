# Enabling Multi-GPU Support in gem5

## Introduction

In the past decade, GPUs have become an important resource for compute-intensive applications such as graphics processing, machine learning, big data analysis, and large-scale simulations. In the future, with the explosion of machine learning and big data, application demands will keep increasing, resulting in more data and computation being pushed to GPUs. However, due to the slowing of Moore's Law and increasing manufacturing costs, it is becoming more and more challenging to add compute resources into a single GPU device to improve its throughput. As a result, using multiple GPUs on a single host is becoming popular in  data-centric and scientific applications. For example, Facebook uses 8 GPUs and two CPU sockets in their machine-learning platform.

However, most  GPU hardware simulators such as GPGPU-Sim and gem5 AMD APU support only a single GPU, making them impractical to run multi-GPU simulations. This precludes studying interference between GPUs, communication between GPUs, or work scheduling across GPUs. In this work, we will discuss the changes required to add multi-GPU support in gem5, including the changes to the kernel fusion driver, GPU components, and other additions. Finally, to demonstrate the efficacy of our changes, we show some results for running various benchmarks in a multi-GPU environment.

## gem5 AMD APU

gem5 AMD APU extends gem5 with a GPU timing model that executes on top of ROCm (Radeon Open Compute Platform), a framework for GPU accelerated computing. Figure below shows the simulation flow when gem5 simulates a GPU. The application source is compiled by the HCC generating an application binary that is loaded into the memory. At the same time, HCC compiler will produce HCC libraries which will invoke the ROCr runtime library which will call the ROCt user space driver. This driver will then make an ioctl() system call to the kernel fusion driver (ROCk) which is simulated in gem5. Fortunately, the user space is already capable of supporting multi-GPU, and therefore we only need to update the support in gem5.

## Multi-GPU Support in gem5

We categorize the changes required in gem5 to enable multi-GPU into 3 categories:
1. Replicating GPU components
2. Extending multi-GPU support in the emulated driver (ROCk)
3. Enabling writeback support in gem5 coherence protocol

## Replicating GPU Components

Figure 1 shows the GPU components that we replicate. Since we use a single-driver multi-GPU model, we will instantiate multiple GPU node on top of a single emulated driver (ROCk). Each GPU node will then have its own command processor, hardware scheduler, packet processor, dispatcher, and compute units. To ensure that each of the GPU components is distinguishable, we passed a unique GPU ID parameter to each component. For components that is operating on specific address range (e.g packet processor), we will also assign unique address range to avoid overlapping.

![Figure 1. Replicating GPU Components](/image/replicate.png)

## Adding Emulated Driver (ROCk) Support for Multi-GPU

Since the ROCk is designed to only support a single GPU, we need to add multi-GPU support to ROCk. There are two changes required to add this support:
1. Managing software queues and sending works to multiple GPUs.
2. Mapping doorbell to multiple GPUs.

### Managing Software Queues and Sending Works to Multiple GPUs.
Application source typically communicates with the GPU using software queues. Software queues is a data structure that maintains the kernels in the application that will be assigned to the GPU. Figure ... shows how several applications interacting with multiple GPUs through software queues. Emulated driver (ROCk) is the component which creates and manages the software queues. To create a queue, the user space will make an ioctl() system call to the ROCk with the request of creating the queue. The ROCk will then get the GPU ID from one of the ioctl() arguments and assigned the kernels to the appropiate GPU as specified by the user. To further manages the queue, the ROCk has to know which GPU to communicate with to manage the simulated state for the GPU serving the queue. For example, when a program destroy a queue, ROCk must delete shared state in the GPU that points to the queue. Therefore, the queue to GPU mapping will have to be maintained in the ROCk layer. We use hash table as a data structure to maintain this mapping.

### Mapping Doorbell to Multiple GPUs.
Doorbell is a mechanism whereby a software system can signal or notify the hardware/GPU that there is some work to be done. The software system will place data in some well-known and mutually agreed upon memory locations, and "ring the doorbell" by writing to a doorbell region. This act of "ringing the doorbell" notifies the GPU that there are some tasks ready to be processed. Since we use multiple-GPUs, we will also have multiple doorbell regions, one for each GPU. Each of this doorbell region need to be mapped to the physical memory by invoking the mmap() system calls.

However, the GPU information is not visible to the mmap() system call and therefore the system calls has no information on which doorbell region to map into. To address this issue, we encode the GPU ID information into the offset parameter passed to mmap. The encoded offset was returned to userspace from an ioctl call during queue creation. Therefore, during the mmap system calls, we could decode the GPU ID information from the offset to figure out which associated GPU doorbell region to map into. This mechanism is illustrated in the figure below.

## Enabling Writeback Support in gem5 Coherence Protocol

Currently, the GPU coherence protocol in gem5 uses WT for its L1 and L2 cache. Not only this does not correctly models what real GPU does, it could also put pressure on the directory and main memory. Therefore, we decide to enable the WB support in the GPU L2 cache in gem5. The issue with the current WB support that AMD provided is it does not guarantee the written data to be accessible by all other cores and it does not simulate the timing of the flush properly.

Figure below shows how the writeback support is implemented in compares to writethrough. In the writethrough case, the written data should be propagated through the cache hierarchy and therefore should be visible to the other cores immediately. However, for writeback case, the up to date data will be held in the L2 without notifying the directory (step 3). In this situation, the up to date data is not visible nor accessible to the other cores and directory. To address this issue, we propagate the flush instruction at the end of every kernel to the L2 (step 4 and 5) to flush all the L2 entries that hold an updated data (step 6). By doing this, we ensure that all written data is visible to the other cores and the directory at the end of every kernel.

## Evaluation

To test our changes, we run a set of benchmarks with input sizes and GPU configuration as shown in the tables below:

Figure ... shows per GPU execution time for 1, 2, and 4 GPUs when running various benchmarks. It can be seen that the execution time decrease as we increase the total number of GPUs. We can also see that the speedup is not linear and could vary between workloads. This is most likely due to the serial portion of the workload that could be different between one another. 
