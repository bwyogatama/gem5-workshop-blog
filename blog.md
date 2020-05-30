# Enabling Multi-GPU Support in gem5

## Introduction

In the past decade, GPUs have become an important resource for compute-intensive, general-purpose GPU applications such as machine learning, big data analysis, and large-scale simulations. In the future, with the explosion of machine learning and big data, application demands will keep increasing, resulting in more data and computation being pushed to GPUs. However, due to the slowing of Moore's Law and rising manufacturing costs, it is becoming more and more challenging to add compute resources into a single GPU device to improve its throughput. As a result, having programs spread their work across multiple GPUs is popular in data-centric and scientific applications. For example, Facebook uses 8 GPUs per server in a recent machine learning platform.

However research infrastructure has not kept pace with this trend; most GPU hardware simulators, including gem5, only support a single GPU. Thus, it is not possible to study interference between GPUs, communication between GPUs, or work scheduling across GPUs. Our research group has been working to address this shortcoming by adding multi-GPU support to gem5. We found changes were needed to the emulated driver, GPU components, and coherence protocol. Finally, to demonstrate the efficacy of our changes, we show some results for running various benchmarks in a multi-GPU environment.

## gem5 AMD APU

The recent gem5 AMD APU model extends gem5 with an accurate, high fidelity GPU timing model that executes on top of ROCm (Radeon Open Compute Platform), AMD's framework for GPU accelerated computing. Figure 1 shows the simulation flow when gem5 simulates a GPU. The application source is compiled by the HCC compiler, generating an application binary that is loaded into the simulated memory.  The compiled program invokes the ROCr runtime library which calls the ROCt user-space driver. This driver makes ioctl() system calls to the kernel fusion driver (ROCk), which is simulated in gem5 since the current support uses syscall emulation (SE) mode. However, since the user-space libraries and drivers already support multi-GPU, we only had to update gem5's multi-GPU support.

<figure>
    <img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/overview.png" alt="Figure 1" width="600"/>
    <em>Figure 1. gem5 Simulation Flow</em>
</figure>

## Multi-GPU Support in gem5

We found three categories of changes were needed to enable multi-GPU support in gem5:
1. Replicating GPU components
2. Adding Emulated Driver (ROCk) Support for Multi-GPU
3. Enabling writeback support in gem5 coherence protocol

## Replicating GPU Components

The first step of supporting multi-GPU is to replicate GPU hardware components for each simulated GPU. Figure 2 shows the GPU components that we replicate. Since we use a single-driver multi-GPU model, we will instantiate multiple GPU node on top of a single emulated driver (ROCk). Each GPU node will then have its own command processor, hardware scheduler, packet processor, dispatcher, and compute units. To ensure that each of the GPU components is distinguishable, we passed a unique GPU ID parameter to each component. For components that is operating on specific address range (e.g packet processor), we will also assign unique address range to avoid overlapping.
*MS: I'm confused: are these static configuration files, to which you can't pass anything, or is this code instantiation, where code runs to instantiate a device and you can provide a GPU? It isn't clear from the text*

<figure>
    <img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/replicate.png" alt="Figure 2" width="600"/>
    <em>Figure 2. Replicated GPU Components</em>
</figure>

## Adding Emulated Driver (ROCk) Support for Multi-GPU

Since the ROCk is designed to only support a single GPU, we need to add multi-GPU support to ROCk. We had to make two changes to add this support:
1. Managing software queues and sending works to multiple GPUs.
2. Mapping doorbell to multiple GPUs.

### Managing Software Queues and Sending Works to Multiple GPUs.
Application source typically communicates with the GPU using one or more software queues. These software queues maintain the application kernels that will be assigned to a GPU. Figure 3 shows how multiple applications interact with multiple GPUs through software queues: queues are private to an application, but an application may have more than one queue. gem5 emulates the ROCk kernel-space driver from Linux. This emulated driver creates and manages the software queues. To create a queue, the user space code makes an ioctl() system call to  ROCk with the request of creating a queue.  ROCk gets the GPU ID from  the ioctl() arguments and assigned the kernel to the appropriate GPU as specified by the user. While the GPU ID is a parameter when creating a queue, it is not passed to routines that manipulate the queue. To further manage the queue,  ROCk has to know which GPU a queue serves so it can simulate GPU state for the queue. For example, when a program destroys a queue, ROCk must delete shared state in the GPU that points to the queue. Therefore, we added a hash table to ROCk to maintain a queue-to-GPU mapping to make it simple and efficient to identify the GPU associated with each queue.

<figure>
    <img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/queue.png" alt="Figure 3" width="400"/>
    <em>Figure 3. Applicatiosn to GPU Communication</em>
</figure>


### Mapping Doorbell to Multiple GPUs.
Doorbells are a mechanism for user-space software to signal the GPU that there is  work to be done. The software places data in mutually agreed-upon memory locations, and "rings the doorbell" by writing to a doorbell region. This act of "ringing the doorbell" notifies the GPU that there are some tasks ready to be processed. Since we use multiple GPUs, we also have multiple doorbell regions, one for each GPU. Software gets access to doorbell regions by mapping the the region into virtual memory with a mmap() system call.

However, the GPU identity is not visible to the mmap() system call, so ROCk does not know which GPU to map to a doorbell region. We address this issue by encoding the GPU ID into the *offset* parameter passed to mmap(). The encoded offset is returned to user space from the create queue ioctl() ioctl call. Thus, the mmap() system call can decode the GPU ID from the offset parameter, and identify the associated GPU doorbell region used for mapping. This mechanism is illustrated in the Figure 4.

<figure>
    <img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/mmap.png" alt="Figure 4" width="600"/>
    <em>Figure 4. Mapping Doorbell Region for Multiple GPUs</em>
</figure>


## Enabling Writeback Support in gem5 Coherence Protocol

Currently, the GPU coherence protocol in gem5 uses write-through (WT) for both L1 and L2 caches. Although this is a valid implementation, in multi-GPU systems it leads to significant bandwidth pressure on the directory and main memory.  Moreover, modern AMD and NVIDIA GPUs generally have the GPUs last-level cache be writeback (WB) caches. Therefore, we decided to change the GPU's L2 cache in gem5 to be WB instead of WT.  Although partial support already existed in gem5 for WB GPU L2 caches, it did not work correctly because the flush operations necessary for correct ordering were not being sent to the L2.

Figure 5 shows how our added GPU WB support compares to the current WT approach. In the WT version, as expected all written data is propagated through the cache hierarchy and therefore is visible to the other cores shortly after the write occurs. However, our WB support holds dirty data in the GPU's L2 without notifying the directory (step 3). In this situation, the dirty data is not visible nor accessible to the other cores and directory. This is safe because the GPU memory consistency model assumes data race freedom.  However, to ensure this dirty data is made visible to other cores by the next synchronization point (e.g., a store release or the end of the kernel), we modified the coherence implementation to forward the flush instructions, generated on store releases and the end of kernels, to the L2 cache (steps 4 and 5), which flushes all dirty L2 entries (step 6). This ensures that all dirty data is visible to the other cores and the directory at the end of every kernel, while also providing additional reuse opportunities and reducing pressure on the directory.

<figure>
    <img src="https://github.com/bwyogatama/gem5-workshop-blog/blob/master/image/writeback.png" alt="Figure 5" width="500"/>
    <em>Figure 5. Writethrough vs Writeback Implementation</em>
</figure>


## Conclusion

In recent few years, multi-GPU systems have become increasingly common as more data and computation being pushed into GPU. However, up until now, gem5 only simulates a single GPU, which makes it difficult to study issues arising in multi-GPU system. Our research group attempts to address this issue by extending multi-GPU support in gem5 by (1) Replicating the GPU components, (2) Modifying the emulated driver, and (3) Enabling writeback support in the coherence protocol. To test our changes, we also run multi-GPU experiment on various benchmarks and observe the GPU execution time for various number of GPUs. For details about our changes and our experiment, check our workshop presentation [here](https://www.youtube.com/watch?v=TSULdaGw0V8&list=PL_hVbFs_loVQ8FDTRCmRvmkPFzf6swgZh&index=3&t=2s) !!!
