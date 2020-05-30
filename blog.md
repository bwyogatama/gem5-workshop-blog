# Enabling Multi-GPU Support in gem5

## Introduction

In the past decade, GPUs have become an important resource for compute-intensive applications such as graphics processing, machine learning, big data analysis, and large-scale simulations. In the future, with the explosion of machine learning and big data, application demands will keep increasing, resulting in more data and computation being pushed to GPUs. However, due to the slowing of Moore's Law and rising manufacturing costs, it is becoming more and more challenging to add compute resources into a single GPU device to improve its throughput. As a result, using multiple GPUs on a single host is  popular in  data-centric and scientific applications. For example, Facebook uses 8 GPUs per server in their machine-learning platform.

However research infrastructure has not kept pace with this trend; most  GPU hardware simulators such as GPGPU-Sim and gem5 AMD APU support only a single GPU, making them impractical to run multi-GPU simulations. This precludes studying interference between GPUs, communication between GPUs, or work scheduling across GPUs.

*MS comment: introduce our work here, not the article. Something like: "Our research group has been working to address this shortcoming by adding multi-GPU support to gem5. We found changes were needed to the ..." -- make it more about introducing our work than introducing the article.*

In this work, we will discuss the changes required to add multi-GPU support in gem5, including the changes to the kernel fusion driver, GPU components, and other additions. Finally, to demonstrate the efficacy of our changes, we show some results for running various benchmarks in a multi-GPU environment.

## gem5 AMD APU

gem5 AMD APU extends gem5 with a GPU timing model that executes on top of ROCm (Radeon Open Compute Platform), a framework from AMD for GPU accelerated computing. Figure 1 shows the simulation flow when gem5 simulates a GPU. The application source is compiled by the HCC compiler, generating an application binary that is loaded into the simulated memory. HCC also produces HCC libraries which 
	
*MS: does it produc HCC libraries every time? Are these HCC libraries or HSA libraries?*

invoke the ROCr runtime library which calls the ROCt user space driver. This driver  makes ioctl() system calls to the kernel fusion driver (ROCk) which is simulated in gem5. Fortunately, the user space libraries and drivers already support multi-GPU, so we only had to update the support in gem5.

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/overview.png" alt="Figure 1" width="600"/>

## Multi-GPU Support in gem5

*MS: Write this more as a story of what you did, rather than an article: 'We found three categories of changes were needed ..."*

We categorize the changes required in gem5 to enable multi-GPU into 3 categories:
1. Replicating GPU components
2. Extending multi-GPU support in the emulated driver (ROCk)
3. Enabling writeback support in gem5 coherence protocol

## Replicating GPU Components

The first step of supporting multi-GPU is to replicate GPU hardware components for each simulated GPU. Figure 2 shows the GPU components that we replicate. Since we use a single-driver multi-GPU model, we will instantiate multiple GPU node on top of a single emulated driver (ROCk). Each GPU node will then have its own command processor, hardware scheduler, packet processor, dispatcher, and compute units. To ensure that each of the GPU components is distinguishable, we passed a unique GPU ID parameter to each component. For components that is operating on specific address range (e.g packet processor), we will also assign unique address range to avoid overlapping.
*MS: I'm confused: are these static configuration files, to which you can't pass anything, or is this code instantiation, where code runs to instantiate a device and you can provide a GPU? It isn't clear from the text*

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/replicate.png" alt="Figure 2" width="600"/>

## Adding Emulated Driver (ROCk) Support for Multi-GPU

Since the ROCk is designed to only support a single GPU, we need to add multi-GPU support to ROCk. We had to make two changes to add this support:
1. Managing software queues and sending works to multiple GPUs.
2. Mapping doorbell to multiple GPUs.

### Managing Software Queues and Sending Works to Multiple GPUs.
Application source typically communicates with the GPU using one or more software queues. These software queues maintain the application kernels that will be assigned to a GPU. Figure 3 shows how multiple applications interact with multiple GPUs through software queues: queues are private to an application, but an application may have more than one queue. gem5 emulates the ROCk kernel-space driver from Linux. This emulated driver creates and manages the software queues. To create a queue, the user space code makes an ioctl() system call to  ROCk with the request of creating a queue.  ROCk gets the GPU ID from  the ioctl() arguments and assigned the ker to the appropiate GPU as specified by the user. While the GPU ID is a parameter when creating a queue, it is not passed to routines that manipulate the queue. To further manage the queue,  ROCk has to know which GPU a queue serves so it can simulated GPU state for the queue. For example, when a program destroys a queue, ROCk must delete shared state in the GPU that points to the queue. Therefore, we added a hash table to ROCk to maintain a queue-to-GPU mapping to make it simple and efficient to identify the GPU associated with each queue.

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/queue.png" alt="Figure 3" width="400"/>

### Mapping Doorbell to Multiple GPUs.
Doorbells are a mechanism for user-space software to signal the GPU that there is  work to be done. The software places data in mutually agreed-upon memory locations, and "rings the doorbell" by writing to a doorbell region. This act of "ringing the doorbell" notifies the GPU that there are some tasks ready to be processed. Since we use multiple GPUs, we  also have multiple doorbell regions, one for each GPU. Software gets access to doorbell regions by mapping the the region into virtual memory with a  mmap() system call.

However, the GPU identity is not visible to the mmap() system call, so ROCk does not know which GPU to map to a doorbell region. We address this issue by encode the GPU ID  into the *offset* parameter passed to mmap(). The encoded offset is returned to user space from the create queue ioctl() ioctl call. Thus, the mmap() system call can decode the GPU ID from the offest parameter, and identify the associated GPU doorbell region used for mapping. This mechanism is illustrated in the Figure 4.

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/mmap.png" alt="Figure 4" width="600"/>

## Enabling Writeback Support in gem5 Coherence Protocol

Currently, the GPU coherence protocol in gem5 uses write through (WT) for both L1 and L2 caches. Not only this does not correctly model what real GPUs do, it  also puts bandwidth pressure on the directory and main memory. Therefore, we decided to enable the WB support in the GPU L2 cache in gem5. However, the current WB support in gem5 from AMD does not properly implement the flush operation that guarantees that written data is  accessible by all other cores and does not simulate flush timing properly.

Figure 5 shows how the writeback support is implemented in compares to writethrough. In the writethrough case, the written data is propagated through the cache hierarchy and therefore is visible to the other cores immediately. However, for writeback, the dirty data is held in the L2 without notifying the directory (step 3). In this situation, the dirty data is not visible nor accessible to the other cores and directory. To address this issue, we modified the coherence implementation to forward the flush instruction at the end of every kernel to the L2 cache(step 4 and 5), which flushes all dirty L2 entries (step 6). This ensures that all direty data is visible to the other cores and the directory at the end of every kernel.

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/writeback.png" alt="Figure 5" width="600"/>

## Evaluation

To test our changes, we ran a set of benchmarks with input sizes and GPU configuration as shown in the tables below:

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/methodology.png" alt="Table 1" width="600"/>

Figure 5 shows per GPU execution time for 1, 2, and 4 GPUs when running various benchmarks. It can be seen that the execution time decrease as we increase the total number of GPUs. We can also see that the speedup is not linear and could vary between workloads. This is most likely due to the serial portion of the workload that could be different between one another.

<img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/result.png" alt="Figure 6" width="600"/>

## Conclusion
