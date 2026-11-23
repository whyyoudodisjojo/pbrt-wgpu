# Chapter 15: Wavefront Rendering on GPUs

One of the major changes in pbrt for this edition of the book is the addition of support for rendering on GPUs as well as on CPUs. Between the substantial computational capabilities that GPUs offer and the recent availability of custom hardware units for efficient ray intersection calculations, the GPU is a compelling target for ray tracing. For example, the image in Figure 15.1 takes 318.6 seconds to render with pbrt on a 2020-era high-end GPU at 1500× 1500 resolution with 2048 samples per pixel. On an 8-core CPU, it takes 11,983 seconds to render with the same settings—over 37 times longer. Even on a high-end 32-core CPU, it takes 2,669 seconds to render (still over 8 times longer).1

pbrt’s GPU rendering path offers only a single integration algorithm: volumetric path tracing, following the algorithms used in the CPU-based VolPathIntegrator described in Section 14.2.3. It otherwise supports all of pbrt’s functionality, using the same classes and functions that have been presented in the preceding 14 chapters. This chapter will therefore not introduce any new rendering algorithms but instead will focus on topics like parallelism and data layout in memory that are necessary to achieve good performance on GPUs.

The integrator described in this chapter, WavefrontPathIntegrator, is structured using a wavefront architecture—effectively, many rays are processed simultaneously, with rendering work organized in queues that collect related tasks to be processed together. (“Wavefront” in this context will be defined more precisely in Section 15.1.2.) Some of the code discussed in this chapter makes more extensive use of advanced C++ features than we have generally used in previous chapters. While we have tried not to use such features unnecessarily, we will see that in some cases they make it possible to generate highly specialized code that runs much more efficiently than if their capabilities are not used. We had previously sidestepped many low-level optimizations due to their comparatively small impact on CPUs. Such implementation-level decisions can, however, change rendering performance by orders of magnitude when targeting GPUs.

```text
    VolPathIntegrator 877
    WavefrontPathIntegrator 939
```

*Figure 15.1: Scene Used for CPU versus GPU Ray Tracing Performance Comparison. (Scene courtesy of Angelo Ferretti.)*

The WavefrontPathIntegrator imposes three requirements on a GPU platform:

```text
         1. It must support a unified address space, where the CPU and GPU can both access the
            GPU’s memory, using pointers that are consistent on both types of processor. This
            capability is integral to being able to parse the scene description and initialize the scene
            representation on the CPU, including initializing pointer-based data structures there,
            before the same data structures are then used in code that runs on the GPU.
         2. The GPU compilation infrastructure must be compatible with C++17, the language that
            the rest of pbrt is implemented in. This makes it possible to use the same class and
            function implementations on both types of processors.
```

```rust
           3. The GPU must have support for ray tracing, either in hardware or in vendor-supplied
              software. (pbrt’s existing acceleration structures would not be efficient on the GPU in
              their present form.)
```

```text
    WavefrontPathIntegrator 939
```

The attentive reader will note that CPUs themselves fulfill all of those requirements, the third potentially via pbrt’s acceleration structures from Chapter 7. Therefore, pbrt makes it possible to execute the WavefrontPathIntegrator on CPUs as well; it is used if the --wavefront command-line option is provided. However, the wavefront organization is usually not a good fit for CPUs and performance is almost always worse than if the VolPathIntegrator is used instead. Nonetheless, the CPU wavefront path is useful for debugging and testing the WavefrontPathIntegrator implementation on systems that do not have suitable GPUs.

At this writing, the only GPUs that provide all three of these capabilities are based on NVIDIA’s CUDA platform, so NVIDIA’s GPUs are the only ones that pbrt currently supports. We hope that it will be possible to support others in the future. Around two thousand lines of platform-specific code are required to handle low-level details like allocating unified memory, launching work on the GPU, and performing ray intersections on the GPU. As usual, we will not include platform-specific code in the book, but see the gpu/ directory in the pbrt source code distribution for its implementation.

## 15.1 MAPPING PATH TRACING TO THE GPU

Achieving good performance on GPUs requires some care in how computation is organized and how data is laid out in memory. We will start with an overview of how GPUs work and the ways in which they differ from CPUs. This foundation makes it possible to discuss the design space of GPU ray tracers. After summarizing some of the alternatives, we give an overview of the design of the wavefront path integrator, which subsequent sections will delve into more deeply.

### 15.1.1 BASIC GPU ARCHITECTURE

The performance difference between CPUs and GPUs stems from a fundamental difference in the computations that they are designed for. CPUs have long been designed to minimize latency—to run a single thread of computation as efficiently as possible, finishing its work as quickly as possible. (This has somewhat changed with the advent of multicore CPUs, though each core remains latency optimized.) In contrast, GPUs target throughput: they are designed to work on many computations in parallel and finish all of them quickly, which is a different thing than finishing any one of them as quickly as possible.

The focus on throughput allows GPUs to devote much less space on the chip for caches, branch prediction hardware, out-of-order execution units, and other features that have been invented to improve single-thread performance on CPUs. Thus, given a fixed amount of chip area, GPUs are able to provide many more of the arithmetic logic units (ALUs) that actually perform computation than a CPU provides. Yet, more ALUs do not necessarily deliver more performance: they must be kept occupied doing useful work.

An ALU cannot perform computation if the input values it requires are not available. On current processors, reading a value from memory consumes a few hundred processor cycles, and so it is important to avoid the situation where a processor remains idle while it waits for the completion of such read operations—substantial amounts of potential computation might be lost. CPUs and GPUs approach this problem quite differently. Understanding each helps illuminate their philosophical differences.

```text
    VolPathIntegrator 877
    WavefrontPathIntegrator 939
```

CPUs apply a barrage of techniques to this task. They feature a relatively large amount of onchip memory in the form of caches; if a memory request targets a location that is already present in a cache, the result can be returned much more quickly than reading from dynamic random access memory (DRAM). Cache memory access typically takes from a handful of cycles to at most a few tens of them. When it is necessary to wait for a memory read, CPUs also use out-of-order execution, continuing to execute the program’s instructions past the read instruction. Dependencies are carefully tracked during this process, and operations that are independent of pending computations can execute out of order. The CPU may also execute instructions speculatively, ready to roll back their effects if it turns out they should not have run. If all of that does not suffice, another thread may be available, ready to start executing—modern CPUs generally use hyperthreading to switch to another thread in the event of a stall. This thread switch can be performed without any overhead, which is much better than the thousands of processor cycles it takes for the operating system to perform a context switch.

GPUs instead focus on a single mechanism to address such latencies: much more aggressive thread switching, over many more threads than are used for hyperthreading on CPUs. If one thread reads from memory, a GPU will just switch to another, saving all of the complexity and chip area required for out-of-order execution logic. If that other thread issues a read, then yet another is scheduled. Given enough threads and computation between memory accesses, such an approach is sufficient to keep the ALUs fed with useful work while avoiding long periods of inactivity.

An implication of this design is that GPUs require much more parallelism than CPUs to run at their peak potential. While tens of threads—or at most a few hundred—suffice to fully utilize a modern multicore CPU, a GPU may require tens of thousands of them. Path tracing fortunately involves millions of independent calculations (one per pixel sample), which makes it a good fit for throughput-oriented architectures like GPUs.

Thread Execution GPUs contain an array of independent processors, numbered from the tens up to nearly a hundred at writing. We will not often need to consider these in the following, but will denote them as processors when we do.2 Each one typically executes 32 or 64 threads concurrently, with as many as a thousand threads available to be scheduled.

The execution model of GPUs centers around the concept of a kernel, which refers to a GPU-targeted function that is executed by a specified number of threads. Parameters passed to the kernel are forwarded to each thread; a thread index provides the information needed to distinguish one thread

from another. Launching a kernel refers to the operation that informs the GPU that a particular kernel function should be executed concurrently. This differs from an ordinary function call in the sense that the kernel will complete asynchronously at some later point. Frameworks like CUDA provide extensive API functionality to wait for the conclusion of such asynchronous computation, or to enforce ordering constraints between multiple separate kernel launches. Launching vast numbers of threads on the GPU is extremely efficient, so there is no need to amortize this cost using a thread pool, as is done in pbrt’s ThreadPool class for CPU parallelism.

Kernels may be launched both from the CPU and from the GPU, though pbrt only does the former. In contrast to an ordinary function call, a kernel launch cannot return any values to the caller. Kernels therefore must write their results to memory before exiting.

An important hardware simplification that distinguishes CPUs and GPUs is that GPUs bundle multiple threads into what we will refer to as a thread group.3 This group (32 threads on most current GPUs) executes instructions together, which means that a single instruction decoder can be shared by the group instead of requiring one for each executing thread. Consequently, silicon die area that would ordinarily be needed for instruction decoding can be dedicated to improving parallelism in the form of additional ALUs. Most GPU programming models further organize thread groups into larger aggregations—though these are not used in pbrt’s GPU implementation, so we will not discuss them further here.

While the hardware simplifications enabled by thread groups allow for additional parallelism, the elimination of per-thread instruction decoders also brings limitations that can have substantial performance implications. Efficient GPU implementation of algorithms requires a thorough understanding of them. Although the threads in a thread group are free to work independently, just as the threads on different CPU cores are, the more that they follow similar paths through the program, the better performance will be achieved. This is a different performance model than for CPUs and can be a subtle point to consider when optimizing code: performance is not just a consequence of the computation performed by an individual thread, but also how often that same computation is performed at the same time with other threads within the same group.

For example, consider this simple block of code:

```rust
         if (condition) a();
         else b();
```

Executed on a CPU, the processor will test the condition and then execute either a() or b() depending on the condition’s value. On a GPU, the situation is more complex: if all the threads in a thread group follow the same control flow path, then execution proceeds as it does on a CPU. However, if some threads need to evaluate a() and some b(), then the GPU will execute both functions’ instructions with a subset of the threads disabled for each one. These disabled threads represent a wasted opportunity to perform useful work.

In the worst case, a computation could be serialized all the way down to the level of individual threads, resulting in a 32× loss of performance that would largely negate the benefits of the GPU. Algorithms like path tracing are especially susceptible to this type of behavior, which is a consequence of the physical characteristics of light: when a beam of light interacts with an object, it will tend to

spread out and eventually reach every part of the environment with nonzero probability. Suppose that a bundle of rays is processed by a thread group: due to this property, an initially coherent computation could later encounter many different shapes and materials that are implemented in different parts of the system. Additional work is necessary to reorder computation into coherent groups to avoid such degenerate behavior.

The implementation of the FloatMixTexture::Evaluate() method from Section 10.3.3 can be better understood with thread groups in mind. Its body was: Float amt = amount.Evaluate(ctx); Float t1 = 0, t2 = 0; if (amt != 1) t1 = tex1.Evaluate(ctx); if (amt != 0) t2 = tex2.Evaluate(ctx); return (1 - amt) * t1 + amt * t2;

A more natural implementation might have been the following, which computes the same result in the end: Float amt = amount.Evaluate(ctx); if (amt == 0) return tex1.Evaluate(ctx); if (amt == 1) return tex2.Evaluate(ctx); return (1 - amt) * tex1.Evaluate(ctx) + amt * tex2.Evaluate(ctx);

```text
    FloatMixTexture::Evaluate() 659
```

Considered under the lens of GPU execution, we can see the benefit of the first implementation. If some of the threads have a value of 0 for amt, some have a value of 1, and the rest have a value in between, then the second implementation will execute the code for evaluating tex1 and tex2 twice, for a different subset of threads for each time.4 With the first implementation, all of the threads that need to evaluate tex1 do so together, and similarly for tex2. Table 15.1: Key Properties of a Representative Modern CPU and GPU. This CPU and GPU have approximately the same cost at time of writing but provide their computational capabilities using very different architectures. This table summarizes some of their most important characteristics.

```text
                                                          AMD 3970x CPU                   NVIDIA 3090 RTX GPU
   Processors                                                  32                                  82
   Peak single-precision TFLOPS                                                                    36
                                                       3.8
```

Peak memory bandwidth ~ 100 GiB/s 936 GiB/s

We will say that execution across a thread group is converged when all of the threads follow the same control flow path through the program, and that it has become divergent at points in the program execution where some threads follow one path and others follow another through the program code. Some divergence is inevitable, but the less there is the better. Convergence can be improved both by writing individual blocks of code to minimize the number of separate control paths and by sorting work so that all of the threads in a thread block do the same thing. This latter idea will come up repeatedly in Section 15.3 when we discuss the set of kernels that the WavefrontPathIntegrator executes.

One implication of thread groups is that techniques like Russian roulette may have a different performance impact on a CPU than on a GPU. With pbrt’s CPU integrators, if a ray path is terminated with Russian roulette, the CPU thread can immediately go on to start work on a new path. Depending on how the rendering computation is mapped to threads on a GPU, terminating one ray path may not have the same benefit if it just leads to an idle thread being carried along with an otherwise still-active thread group.

Memory Hierarchy Large differences in the memory system architectures of CPUs and GPUs further affect how a system should be structured to run efficiently on each type of processor. Table 15.1 summarizes some relevant quantities for a representative modern CPU and GPU that at the time of this writing have roughly the same cost. Two differences are immediately apparent: the peak memory bandwidth and number of TFLOPS (trillions of floating point operations per second) are both approximately ten times higher on the GPU. It is also clear that neither processor is able to reach its peak performance solely by operating on values stored in memory. For example, the 3.8 TFLOPS that the CPU is capable of would require 15.2 TB/s of memory bandwidth if each 4-byte floating-point value operated on was to be read from memory. Consequently, we can see that the performance of a computation such as iterating over a large array, reading each value, squaring it, and writing the

result back to memory would not be limited by the processor’s computational capabilities but would be limited by the memory bandwidth. We say that such computations are bandwidth limited. A useful measurement of a computation is its arithmetic intensity, which is usually measured as the number of floating-point operations performed per byte of memory accessed. Dividing peak TFLOPS by peak memory bandwidth gives a measurement of how much arithmetic intensity a processor requires to achieve its peak performance. For this CPU, we can see that it is roughly 38 floating-point operations (FLOPS) per byte, or 152 FLOPS for every 4-byte float read from memory. For the GPU, the values are 38.5 and 154, respectively—remarkably, almost exactly the same. Given such arithmetic intensity requirements, it is easy to become bandwidth limited. Table 15.2: Key Properties of the Example CPU and GPU Processors. This table summarizes a few relevant per-processor quantities for the CPU and GPU in Table 15.1. For the CPU, “maximum available threads” is the number that can be switched to without incurring the cost of an operating system thread switch. Furthermore, the number of CPU registers here is the total available for out-of-order execution, which is many more than are visible through the instruction set. The L2 cache on the GPU is shared across all processors and the L3 cache on the CPU is shared across four processors; here we report those cache sizes divided by the number of processors that share them.

AMD 3970x CPU NVIDIA 3090 RTX GPU Concurrently executing threads 1 32 Maximum available threads 2 1,024 float operations per cycle & thread 32 2 Registers 160 (float) 65,536 L1 cache 32 KiB (data) 128 KiB L2 cache 512 KiB ~ 75 KiB L3 cache 4 MiB none

Therefore, there must be some combination of reuse of each value read from memory and reuse of intermediate computed values that are stored in on-chip memory in order to reach peak floating-point performance. Both the processors’ register files and cache hierarchies are integral to keeping them working productively by making some values available from faster on-chip memory, though their effect is quite different on the two types of architecture. See Table 15.2, which presents additional quantities related to the individual processors on the example CPU and GPU.

To understand the differences, it is illuminating to compare the two in terms of their cache size with respect to the number of threads that they are running. (Space in the caches is not explicitly allocated to threads, though this is still a useful thought exercise.) This CPU runs one thread at a time on each core, with a second ready for hyperthreading, giving 16 KiB of L1 cache, 256 KiB of L2, and 2 MiB of L3 cache for each of the two threads. This is enough memory to give a fairly wide window for the reuse of previous values and is enough that, for example, we do not need to worry about how big the SurfaceInteraction structure is (it is just under 256 bytes); it fits comfortably in the caches close to the processor. These generous cache hierarchies can be a great help to programmers, leaving them with the task of making sure their programs have some locality in their memory accesses but often allowing them not to worry over it too much further. The GPU runs thread groups of 32 threads, with as many as 31 additional thread groups ready to switch to, for a total of 1,024 threads afoot. We are left with 128 bytes of L1 cache and 75 bytes of L2 per thread, meaning factors of 128× and 3500× less than the CPU, respectively. If the GPU threads are accessing independent memory locations, we are left with a very small window of potential data reuse that can be served by the caches. Thus, structuring GPU computation so that threads access the same locations in memory as much as possible can significantly improve performance by making the caches more effective.

```text
   SurfaceInteraction 138
```

GPUs partially make up for their small caches with large register files; for the one in this comparison there are 65,536 32-bit registers for each GPU processor, giving 64 or more for each thread. (Note that this register file actually has twice as much total storage as the processor’s L1 cache.) If a computation can be structured such that it fits into its allocation of registers and has limited memory traffic (especially reads that are different than other threads’), then its computation can achieve high performance on the GPU. The allocation of registers to a kernel must be determined at compile time; this presents a balance for the compiler to strike. On one hand, allocating more registers to a kernel gives it more on-chip storage and, in turn,

generally reduces the amount of memory bandwidth that it requires. However, the more registers that are allocated to a kernel, the fewer threads can be scheduled on a processor. For the example GPU, allocating 64 registers for each thread of a kernel means that 1,024 threads can run on a processor at once. 128 registers per thread means just 512 threads, and so forth. The fewer threads that are running, the more difficult it is to hide memory latency via thread switching, and performance may suffer when all threads are stalled waiting for memory reads. The effect of these constraints is that reducing the size of objects can significantly improve performance on the GPU: doing so reduces the amount of bandwidth consumed when reading them from memory (and so may improve performance if a computation is bandwidth limited) and can reduce the number of registers consumed to store them after they have been loaded, potentially allowing more threads and thus more effective latency hiding. This theme will come up repeatedly later in the chapter. The coherence of the memory locations accessed by the threads in a thread group affects performance as well. A reasonable model for thinking about this is in terms of the processor’s cache lines. A common GPU cache line size is 128 bytes. The cost of a memory access by the threads in a thread group is related to the total number of cache lines that the threads access. The best case is that they all access the same cache line, for a location that is already in the cache. (Thus with a 128-byte cache line size, 32 threads accessing successive cache line–aligned 4-byte values such as floats access a single cache line.) Performance remains reasonable if the locations accessed correspond to a small number of cache lines that are already in the cache. An entire cache line must be read for a cache miss. Here as well, the coherence of the locations accessed by the threads has a big impact on performance: if all locations are in a single cache line, then a single memory read can be performed. If all 32 threads access locations that lie in different cache lines, then 32 independent memory reads are required; not only is there a significant bandwidth cost to reading so much data, but there is much more memory latency—likely more than can be effectively hidden. Thus, another important theme in the following implementation will be

organizing data structures and computation in order to improve the coherence of memory locations accessed by the threads in a thread group. A final issue related to memory performance arises due to the various different types of memory that can be referenced by a computation. The GPU has its own device memory, distinct from the host memory used by the CPU. Each GPU processor offers a small high-performance shared memory that can be used by the threads running on it.5 It is best interpreted as a manually managed cache. Shared memory and L1 and L2 caches provide much higher bandwidth and lower latency than device memory, while host memory is the most expensive for the GPU to access: any read or write must be encapsulated into a transaction that is sent over the comparably slow PCI Express bus connecting the CPU and GPU. Optimally placing and, if necessary, moving data in memory during multiple phases of a computation requires expertise and extra engineering effort. pbrt sidesteps this issue using managed memory, which exists in a unified address space that can be accessed from both CPU and GPU. Its physical location is undecided and can migrate on demand to improve performance. This automatic migration comes at a small additional performance cost, but this is well worth the convenience of not having to micromanage memory allocations. In pbrt, the CPU initializes the scene in managed memory, and this migration cost is paid once when rendering on the GPU begins. There is then a small cost to read back the final image from the Film at the end. In the following implementation, as CPU code is launching kernels on the GPU, it is important that it does not inadvertently access GPU memory, which would harm performance.

### 15.1.2 STRUCTURING RENDERING COMPUTATION

With these factors that affect GPU performance in mind, we can consider various ways of structuring pbrt’s rendering computation so that it is suitable for the GPU. First, consider applying the same parallelism decomposition that is used in the ImageTileIntegrator: assigning each tile of the image to a thread that is then responsible for evaluating its pixel samples. Such an approach is hopeless from the start. Not only is it unlikely to provide a sufficient number of threads to fill the GPU, but the effect of the load imbalance among tiles is exacerbated when the threads execute in

groups. (Recall Figure 1.17, the histogram of time spent rendering image tiles in Figure 1.11.) Since a thread group continues executing until all of its threads have finished, performance is worse if the long-running threads are spread across many different thread groups versus all together in fewer. Another natural approach might be to assign each pixel sample to a thread, launching as many threads as there are pixel samples, and to have each thread execute the same code as runs on the CPU to evaluate a pixel sample. Each thread’s task then would be to generate a camera ray, find the closest intersection, evaluate the material and textures at the intersection point, and so forth. This is known as the megakernel approach, since a single large kernel is responsible for all rendering computation for a ray. This approach provides more than sufficient parallelism to the GPU, but suffers greatly from execution divergence. While the computation may remain converged until intersections are found, if different rays hit objects with different materials, or the same material but with different textures, their execution will diverge and performance will quickly deteriorate. Even if the camera rays all hit objects with the same material, coherence will generally be lost with tracing the first batch of indirect rays: some may find no intersection and leave the scene, others may hit objects with various materials, and yet others may end up scattering in participating media. Each different case leads to execution divergence. Even if all the threads end up sampling a light BVH, for instance, they may not do so at the same time and thus that code may be executed multiple times, just as was the case for the inferior approach of implementing the FloatMixTexture Evaluate() method. We can expect that over time all of the threads will fully diverge, leading to processing that is less efficient than it could be by a factor of the number of threads in a thread group. The performance of a megakernel ray tracer can be improved with the addition of work scheduling and reordering within the executing kernels. Such a megakernel ray tracer can be seen as what is effectively a thread group–wide state machine that successively chooses an operation to perform: “generate camera rays,” “find closest intersections,” “evaluate and sample diffuse BSDFs,” “sample the light BVH,” and so forth. It might choose among operations based on how many threads would like to perform the corresponding operation.

```text
   Film 244
   FloatMixTexture 659
   ImageTileIntegrator 24
```

This approach can greatly improve execution convergence. For example, if only a single thread is waiting to evaluate and sample diffuse BSDFs, that work can be deferred while other threads trace rays and do other rendering work. Perhaps some of those rays will intersect diffuse surfaces, adding themselves to the tally of threads that need to do that operation. When that operation is eventually selected, it can be done for the benefit of more threads, redundant executions saved. Direct implementation of the megakernel approach does have disadvantages. The megakernels themselves may be comprised of a large amount of code (effectively, everything that the renderer needs to do), which can lead to long compilation times depending on the sophistication of the ray tracer. They are further limited to finding shared work among the threads in a thread group or, at most, the threads running on a single GPU processor. It therefore may not be possible to achieve an optimal degree of thread convergence. Nevertheless, the approach is a good one, and is the most common one for real-time ray tracers today. The other main GPU ray tracer architecture is termed wavefront. A wavefront ray tracer separates the main operations into separate kernels, each one operating on many pixel samples in parallel: for example, there might be one kernel that generates camera rays, another that finds ray intersections, perhaps one to evaluate diffuse materials and another to evaluate dielectrics, and so forth. The kernels are organized in a dataflow architecture, where each one may enqueue additional work to be performed in one or more subsequent kernels. A significant advantage of the wavefront approach is that execution in each kernel can start out fully converged: the diffuse-material kernel is invoked for only the intersection points with a diffuse material, and so forth. While the execution may diverge within the kernel, regularly starting out with converged execution can greatly aid performance, especially for systems with a wide variety of materials, BSDFs, lights, and so forth.

Another advantage of the wavefront approach is that different numbers of registers can be allocated to different kernels. Thus, simple kernels can use fewer registers and reap benefits in more effective latency hiding, and it is only the more complex kernels that incur the costs of that trade-off. In contrast, the register allocation for a megakernel must be made according to the worst case across the entire set of rendering computation. However, wavefront ray tracers pay a cost in bandwidth. Because data does not persist onchip between kernel launches, each kernel must read all of its inputs from memory and then write its results back to it. In contrast, megakernels can often keep intermediate information on-chip. The performance of a wavefront ray tracer is more likely than a megakernel to be limited by the amount of memory bandwidth and not the GPU’s computational capabilities. This is an undesirable state of affairs since it is projected that bandwidth will continue to grow more slowly than computation in future processor architectures. The recent addition of hardware ray-tracing capabilities to GPUs has led to the development of graphics programming interfaces that allow the user to specify which kernels to run at various stages of ray tracing. This gives an alternative to the megakernel and wavefront approaches that avoids many of their respective disadvantages. With these APIs, the user not only provides single kernels for operations like generating initial rays, but can also specify multiple kernels to run at ray intersection points—where the kernel that runs at a given point might be determined based on an object’s material, for example. Scheduling computation and orchestrating the flow of data between stages is not the user’s concern, and the GPU’s hardware and software has the opportunity to schedule work in a way that is tuned to the hardware architecture. (The semantics of these APIs are discussed further in Section 15.3.6.)

### 15.1.3 SYSTEM OVERVIEW

This version of pbrt adopts the wavefront approach for its GPU rendering path, although some of its kernels fuse together multiple stages of the ray-tracing computation in order to reduce the amount of memory traffic that is used. We found this approach to be a good fit given the variety of materials, BxDFs, and light sources that pbrt supports. Further, rendering features like

volume scattering fit in nicely: we can skip the corresponding kernels entirely if the scene does not include those effects.

*Figure 15.2: Overview of Kernels Used in the Wavefront Integrator. Arrows correspond to queues on which kernels may enqueue work for subsequent kernels. After camera rays have been generated, the subsequent kernels execute one time for each ray depth up to the maximum depth.*

As with the CPU-targeted Integrators, a BasicScene parses the scene description and stores various entities that represent scene components. When the wavefront integrator is being used, the parsed scene is then passed to the RenderWavefront() function, which is defined in the file wavefront/wavefront.cpp. Beyond some housekeeping, its main task is to allocate an instance of the WavefrontPathIntegrator class and to call its Render() method. Since the housekeeping is not very interesting, we will not include its implementation here. The WavefrontPathIntegrator constructor converts many of the entities in the BasicScene to pbrt objects in the same way as is done for the CPU renderer: all the lights are converted to corresponding instances of the Light classes, and similarly for the camera, film, sampler, light sampler, media, materials, and textures. One important difference, however, is that the Allocator that is provided for them allocates managed memory so that it can be initialized on the CPU and accessed by the GPU. Another

difference is that only some shapes are handled with pbrt’s Shape implementations. Shapes like triangles that have native support from the GPU’s ray intersection hardware are instead handled using that hardware. Finally, image map textures use a specialized implementation that uses the GPU’s texturing hardware for texture lookups.

```text
   Allocator 40
   BasicScene 1134
   BxDF 538
   Shape 261
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::Render() 941
```

Once the scene representation is ready, a call to WavefrontPathIntegrator::Render() starts the rendering process. The details of the implementation of that method will be the subject of the following sections of this chapter, but Figure 15.2 gives an overview of the kernels that it launches.6 The sequence of computations is similar to that of the VolPathIntegrator::Li() method, though decomposed into kernels. Queues are used to buffer work between kernels: each kernel can push work onto one or more queues to be processed in subsequent kernels. Rendering starts with a kernel that generates camera rays for a number of pixel samples (typically, a million or so). Given camera rays, the loop up to the maximum ray depth can begin. Each time through the loop, the following kernels are executed: First, samples for use with the ray are generated using the Sampler and stored in memory for use in later kernels. The closest intersection of each ray is found with the scene geometry. The kernel that finds these intersections pushes work onto a variety of queues to be processed by other kernels, including a queue for rays that escape the scene, one for rays that hit emissive geometry, and another for rays that pass through participating media. Rays that intersect surfaces are pushed onto queues that partition work based on material types. Rays passing through participating media are then processed, possibly leading to a shadow ray and an indirect ray being enqueued if the ray scatters. Unscattered rays that were earlier

found to have intersected a surface are pushed on to the same variety of queues as are used in the intersection kernel for rays not passing through media. Rays that have intersected emissive geometry and rays that have left the scene are handled by two kernels that both update rays’ radiance estimates to incorporate the effect of emission found by such rays. Each material is then handled individually, with separate kernels specialized based on the material type. Textures are evaluated to initialize a BSDF and a light is sampled. This, too, leads to indirect and shadow rays being enqueued. A final kernel traces shadow rays and incorporates their contributions into the rays’ radiance estimates, including accounting for absorption along rays in participating media. Until the maximum ray depth, the loop then starts again with the queued indirect rays replacing the camera rays as the rays to trace next. 15.2 IMPLEMENTATION FOUNDATIONS

Before we discuss the implementation of the wavefront integrator and the details of its kernels, we will start by discussing some of the lower-level capabilities that it is built upon, as well as the abstractions that we use to hide the details of platform-specific APIs.

### 15.2.1 EXECUTION AND MEMORY SPACE SPECIFICATION

If you have perused the pbrt source code, you will have noticed that the signatures of many functions include a PBRT_CPU_GPU. We have elided all of these from the book text thus far in the interests of avoiding distraction. Now we must pay closer attention to them. When pbrt is compiled with GPU support, the compiler must know whether each function in the system is intended for CPU execution only, GPU execution only, or execution on both types of processor. In some cases, a function may only be able to run on one of the two—for example, if it uses low-level functionality that the other type of processor does not

support. In other cases, it may be possible for it to run on both, though it may not make sense to do so.

```text
    BSDF 544
    Sampler 469
    VolPathIntegrator::Li() 878
```

For example, an object constructor that does extensive serial processing would be unsuited to the GPU. PBRT_CPU_GPU hides the platform-specific details of how these characteristics are indicated. (With CUDA, it turns into a __host__ __device__ specifier.) There is also a PBRT_GPU macro, which signifies that a function can only be called from GPU code. These macros are all defined in the file pbrt.h. If no specifier is provided, a function can only be called from code that is running on the CPU. These specifiers can be used with variable declarations as well, to similar effect. pbrt only makes occasional use of them for that, mostly using managed memory for such purposes. (There is also a PBRT_CONST variable qualifier that is used to define large constant arrays in a way that makes them accessible from both CPU and GPU.) In addition to informing the compiler of which processors to compile functions for, having these specifiers in the signatures of functions allows the compiler to determine whether a function call is valid: a CPU-only function that directly calls a GPU-only function, or vice versa, leads to a compile time error.

### 15.2.2 LAUNCHING KERNELS ON THE GPU

pbrt also provides functions that abstract the platform-specific details of launching kernels on the GPU. These are all defined in the files gpu/util.h and gpu/util.cpp.

The most important of them is GPUParallelFor(), which launches a specified number of GPU threads and invokes the provided kernel function for each one, passing it an index ranging from zero up to the total number of threads. The index values passed always span a contiguous range for all of the threads in a thread group. This is an important property in that, for

example, indexing into a float array using the index leads to contiguous memory accesses. This function is the GPU analog to ParallelFor(), with the differences that it always starts iteration from zero, rather than taking a 1D range, and that it takes a description string that describes the kernel’s functionality. Its implementation includes code that tracks the total time each kernel spends executing on the GPU, which makes it possible to print a performance breakdown after rendering completes using the provided description string. 〈GPU Launch Function Declarations〉 ≡ template <typename F> void GPUParallelFor(const char *description, int nItems, F func);

Similar to ParallelFor(), all the work from one call to GPUParallelFor() finishes before any work from a subsequent call to GPUParallelFor() begins, and it, too, is also generally used with lambda functions. A simple example of its usage is below: the callback function func is invoked with a single integer argument, corresponding to the item index that it is responsible for. Note also that a PBRT_GPU specifier is necessary for the lambda function to indicate that it will be invoked from GPU code. float *array = /* allocate managed memory */; GPUParallelFor(“Initialize array“, 100, [=] PBRT_GPU (int i) { array[i] = i; });

We provide a macro for specifying lambda functions for GPUParallelFor() and related functions that adds the PBRT_GPU specifier and also hides some platform-system specific details (which are not included in the text here).

```text
   GPUParallelFor() 929
   ParallelFor() 1107
```

〈GPU Macro Definitions〉 ≡

#define PBRT_CPU_GPU_LAMBDA(…) [=] PBRT_CPU_GPU (__VA_ARGS__)

We do not provide a variant analogous to ParallelFor2D() for iterating over a 2D array, though it would be easy to do so; pbrt just has no need for such functionality.

*Figure 15.3: Comparison of Synchronous and Asynchronous CPU/GPU Execution. (a) If the two processors are synchronous, only one is ever executing. The CPU stalls after launching a kernel on the GPU, and then the GPU is idle after a kernel finishes until the CPU resumes and launches another one. (b) With the asynchronous model, the CPU continues execution after launching each kernel and is able to enqueue more of them while the GPU is working on earlier ones. In turn, the GPU does not need to wait for the next batch of work to do.*

Although execution is serialized between successive calls to GPUParallelFor(), it is important to be aware that the execution of the CPU and GPU are decoupled. When a call to GPUParallelFor() returns on the CPU, the corresponding threads on the GPU often will not even have been launched, let alone completed. Work on the GPU proceeds asynchronously. While this execution model thus requires explicit synchronization operations between the two processors, it is important for achieving high performance. Consider the two alternatives illustrated in Figure 15.3; automatically synchronizing execution of the two processors would mean that only one of them could be running at a given time.

An implication of this model is that the CPU must be careful about when it reads values from managed memory that were computed on the GPU. For example, if the implementation had code that tried to read the final image immediately after launching the last rendering kernel, it would almost certainly read an incomplete result. We therefore require a mechanism so that the CPU can wait for all the previous kernel launches to complete. This capability is provided by GPUWait(). Again, the implementation is not included here, as it is based on platform-specific functionality. 〈GPU Synchronization Function Declarations〉 ≡ void GPUWait();

### 15.2.3 STRUCTURE-OF-ARRAYS LAYOUT

```text
    ParallelFor2D() 1108
    PBRT_CPU_GPU 928
```

*Figure 15.4: Memory Layout of an Array of SimpleRay Structures. The elements of each ray are consecutive in memory.*

As discussed earlier, the thread group execution model of GPUs affects the pattern of memory accesses that program execution generates. A consequence is that the way in which data is laid out in memory can have a meaningful effect on performance. To understand the issue, consider the following definition of a ray structure: struct SimpleRay { Point3f o; Vector3f d; };

Now consider a kernel that takes a queue of SimpleRays as input. Such a kernel might be responsible for finding the closest intersection along each ray, for example. A natural representation of the queue would be an array of SimpleRays. (This layout is termed array of structures, and is sometimes abbreviated as AOS.) Such a queue would be laid out in memory as shown in Figure 15.4, where each SimpleRay occupies a contiguous region of memory with its elements stored in consecutive locations in memory. Now consider what happens when the threads in a thread group read their corresponding ray into memory. If the pointer to the array is rays and the index passed to each thread’s lambda function is i, then each thread might start out by reading its ray using code like SimpleRay r = rays[i];

The generated code would typically load each component of the origins and directions individually. Figure 15.5(a) illustrates the distribution of memory locations accessed when each thread in a thread group loads the x component of its ray origin. The locations span many cache lines, which in turn incurs a performance cost. An alternative layout is termed structure of arrays, or SOA. The idea behind this approach is to effectively transpose the layout of the object in memory, storing all the origins’ x components contiguously in an array of Floats, then all of its origins’ y components in another Float array, and so forth. We might declare a specialized structure to represent arrays of SimpleRays laid out like this: struct SimpleRayArray { Float *ox, *oy, *oz; Float *dx, *dy, *dz; };

In turn, if the threads in a thread group want to load the x component of their ray’s origin, the expression rays.ox[i] suffices to load that value. The locations accessed are contiguous in memory and span many fewer cache lines7 (Figure 15.5(b)).

*Figure 15.5: Effect of Memory Layout on Read Coherence. (a) When the threads in a thread group ti load the x component of their ray origin with an array of structures layout, each thread reads from a memory location that is offset by sizeof(SimpleRay) from the previous one; in this case, 24 bytes, assuming 4-byte Floats. Consequently, multiple cache lines must be accessed, with corresponding performance cost. (b) With structure of arrays layout, the threads in a thread group access contiguous locations to read the x component, corresponding to 1 or at most 2 cache-line accesses, depending on the alignment of the array. Performance is generally much better.*

However, this performance benefit comes at a cost of making the code more unwieldy. Loading an entire ray with this approach requires indexing into each of the constituent arrays—for example, SimpleRay r(Point3f(rays.ox[i], rays.oy[i], rays.oz[i]), Vector3f(rays.dx[i], rays.dy[i], rays.dz[i]));

Even more verbose manual indexing is required for the task of writing a ray out to memory in SOA format; the cleanliness of the AOS array syntax has been lost. In order to avoid such complexity, the code in the remainder of this chapter makes extensive use of SOA template types that make it possible to work with SOA data using syntax that is the same as array indexing with an array

of structures data layout. For any type that has an SOA template specialization (e.g., pbrt’s regular Ray class), we are able to write code like the following: SOA<Ray> rays(1024, Allocator()); int index = …; Ray r = rays[index]; Transform transform = …; rays[index] = transform(r);

Both loads from and stores to the array are expressed using regular C++ array indexing syntax. While it is straightforward programming to implement such SOA classes, doing so is rather tedious, especially if one has many types to lay out in SOA. Therefore, pbrt includes a small utility program, soac (structure of arrays compiler), that automates this process. Its source code is in the file cmd/soac.cpp in the pbrt distribution. There is not much to be proud of in its implementation, so we will not discuss that here, but we will describe its usage as well as the form of the SOA type definitions that it generates.

soac takes structure specifications in a custom format of the following form: flat Float; soa Point2f { Float x, y; };

Here, flat specifies that the following type should be stored directly in arrays, while soa specifies a structure. Although not shown here, soa structures can themselves hold other soa structure types. See the files pbrt.soa and wavefront/workitems.soa in the pbrt source code for more examples. This format is easy to parse and is sufficient to provide the information necessary to generate pbrt’s SOA type definitions. A more bulletproof solution (and one that would avoid the need for writing the code to parse a custom format) would be to modify a C++ compiler to optionally emit SOA type definitions for types it has already parsed. Of course, that is a more complex approach to implement than soac was.

When pbrt is compiled, soac is invoked to generate header files based on the *.soa specifications. For example, pbrt.soa is turned into pbrt_soa.h, which can be found in pbrt’s build directory after the system has been compiled. For each soa type defined in a specification file, soac generates an SOA template specialization. Here is the one for Point2f. (Here we have taken automatically generated code and brought it into the book text for dissection, which is the opposite flow from all the other code in the text.) 〈Point2f SOA Definition〉 ≡ template <> struct SOA<Point2f> { 〈Point2f SOA Types 934〉 〈Point2f SOA Public Methods 933〉 〈Point2f SOA Public Members 933〉 };

The constructor uses a provided allocator to allocate space for individual arrays for the class member variables. The use of the this-> syntax for initializing the member variables may seem gratuitous, though it ensures that if one of the member variables has the same name as one of the constructor parameters, it is still initialized correctly. 〈Point2f SOA Public Methods〉 ≡ 933

SOA(int n, Allocator alloc) : nAlloc(n) { this->x = alloc.allocate_object<Float>(n); this->y = alloc.allocate_object<Float>(n); }

```text
   Allocator 40
   Float 23
   PBRT_RESTRICT 933
   Point2f 92
   SOA 932
   std::pmr::polymorphic_allocator::allocate_object() 41
```

The SOA class’s members store the array size and the individual member pointers. The PBRT_ RESTRICT qualifier informs the compiler that no other pointer will point to these arrays, which can allow it to generate more efficient code.

〈Point2f SOA Public Members〉 ≡ 933

int nAlloc; Float * PBRT_RESTRICT x; Float * PBRT_RESTRICT y;

It is easy to generate a method that allows indexing into the SOA arrays to read values. Note that the generated code requires that the class has a default constructor and that it can directly initialize the class’s member variables. In the event that they are private, the class should use a friend declaration to make them available to its SOA specialization. 〈Point2f SOA Public Methods〉 +≡ 933

Point2f operator[](int i) const { Point2f r; r.x = this->x[i]; r.y = this->y[i]; return r; }

It is less obvious how to support assignment of values using the regular array indexing syntax. soac provides this capability through the indirection of an auxiliary class, GetSetIndirector. When operator[] is called with a non-const SOA object, it returns an instance of that class. It records not only a pointer to the SOA object but also the index value. 〈Point2f SOA Public Methods〉 +≡ 933

GetSetIndirector operator[](int i) { return GetSetIndirector{this, i}; }

Assignment is handled by the GetSetIndirector’s operator= method. Given a Point2f value, it is able to perform the appropriate assignments using the SOA * and the index.

〈Point2f SOA Types〉 ≡ 933

struct GetSetIndirector { void operator=(const Point2f &a) { soa->x[i] = a.x; soa->y[i] = a.y; } SOA *soa;

```rust
         int i;
   };
```

Variables of type GetSetIndirector should never be declared explicitly. Rather, the role of this structure is to cause an assignment of the form p[i] = Point2f(…) to correspond to the following code, where the initial parenthesized expression corresponds to invoking the SOA class’s operator[] to get a temporary GetSetIndirector and where the assignment is then handled by its operator= method.

```rust
   (p.operator[](i)).operator=(Point2f(…));
```

GetSetIndirector also provides an operator Point2f() conversion operator (not included here) that handles the case of an SOA array read with a non-const SOA object.

We conclude with a caveat: SOA layout is effective if access is coherent, but can be detrimental if it is not. If threads are accessing an array of some structure type in an incoherent fashion, then AOS may be a better choice: in that case, although each thread’s initial access to the structure may incur a cache miss, its subsequent accesses may be efficient if nearby values in the structure are still in the cache. Conversely, incoherent access to SOA data may incur a miss for each access to each structure member, polluting the cache by bringing in many unnecessary values that are not accessed by any other threads.

```text
    GetSetIndirector 934
    Point2f 92
    SOA 932
    SOA<Point2f>::x 933
    SOA<Point2f>::y 933
```

### 15.2.4 WORK QUEUES

Our last bit of groundwork before getting back into rendering will be to define two classes that manage the input and output of kernels in the ray-tracing pipeline. Both are defined in the file wavefront/workqueue.h.

First is WorkQueue, which represents a queue of objects of a specified type, WorkItem. The items in the queue are stored in SOA layout; this is achieved

by publicly inheriting from the SOA template class for the item type. This inheritance also allows the items in the work queue to be indexed using regular array indexing syntax, via the SOA public methods.

〈WorkQueue Definition〉 ≡ template <typename WorkItem> class WorkQueue : public SOA<WorkItem> { public: 〈WorkQueue Public Methods 935〉 protected: 〈WorkQueue Protected Methods 936〉 private: 〈WorkQueue Private Members 935〉 };

The constructor takes the maximum number of objects that can be stored in the queue as well as an allocator, but leaves the actual allocation to the SOA base class. 〈WorkQueue Public Methods〉 ≡ 935

WorkQueue(int n, Allocator alloc) : SOA<WorkItem>(n, alloc) {}

There is only a single private member variable: the number of objects stored in the queue. It is represented using a platform-specific atomic type. When WorkQueues are used on the CPU, a std::atomic is sufficient; that case is shown here. 〈WorkQueue Private Members〉 ≡ 935

std::atomic<int> size{0};

Simple methods return the size of the queue and reset it so that it stores no items. 〈WorkQueue Public Methods〉 +≡ 935

int Size() const { return size.load(std::memory_order_relaxed); } void Reset() { size.store(0, std::memory_order_relaxed);

```rust
   }
```

An item is added to the queue by finding a slot for it via a call to AllocateEntry(). We implement that functionality separately so that WorkQueue subclasses can implement their own methods for adding items to the queue if further processing is needed when doing so. In this case, all that needs to be done is to store the provided value in the specified spot using the SOA indexing operator.

```text
   Allocator 40
   SOA 932
   WorkQueue 935
   WorkQueue::size 935
```

〈WorkQueue Public Methods〉 +≡ 935

int Push(WorkItem w) { int index = AllocateEntry(); (*this)[index] = w; return index; }

When a slot is claimed for a new item in the queue, the size member variable is incremented using an atomic operation, and so it is fine if multiple threads are concurrently adding items to the queue without any further coordination. Returning to how threads access memory, we would do well to allocate consecutive slots for all of the threads in a thread group that are adding entries to a queue. Given the SOA layout of the queue, such an allocation leads to writes to consecutive memory locations, with corresponding performance benefits. Our use of fetch_add here does not provide that guarantee, since each thread calls it independently. However, a common way to implement atomic addition in the presence of thread groups is to aggregate the operation over all the active threads in the group and to perform a single addition for all of them, doling out corresponding individual results back to the individual threads. Our code here assumes such an implementation; on a platform where the assumption is incorrect, it would be worthwhile to use a different mechanism that did give that result. 〈WorkQueue Protected Methods〉 ≡ 935

```rust
   int AllocateEntry() {
          return size.fetch_add(1, std::memory_order_relaxed);
   }
```

ForAllQueued() makes it easy to apply a function to all of the items stored in a WorkQueue() in parallel. The provided callback function is passed the WorkItem that it is responsible for.

〈WorkQueue Inline Functions〉 ≡ template <typename F, typename WorkItem> void ForAllQueued(const char *desc, const WorkQueue<WorkItem> *q, int maxQueued, F &&func) { if (Options->useGPU) { 〈Launch GPU threads to process q using func 937〉 } else { 〈Process q using func with CPU threads 937〉 } }

If the GPU is being used, a thread is launched for the maximum number of items that could be stored in the queue, rather than only for the number that are actually stored there. This stems from the fact that kernels are launched from the CPU but the number of objects actually in the queue is stored in managed memory. Not only would it be inefficient to read the actual size of the queue back from GPU memory to the CPU for the call to GPUParallelFor(), but retrieving the correct value would require synchronization with the GPU, which would further harm performance. Therefore, a number of threads certain to be sufficient are launched and those that do not have an item to process exit immediately. Because thread creation is so inexpensive on GPUs, this approach does not introduce a meaningful amount of overhead. If it was a problem, it is also possible to launch kernels directly from the GPU, in which case exactly the correct number of threads could be launched. In this case, we adopt the greater simplicity of always launching kernels from the CPU.

```text
   BasicPBRTOptions::useGPU 1031
   GPUParallelFor() 929
```

```text
   Options 1032
   WorkQueue 935
   WorkQueue::AllocateEntry() 936
   WorkQueue::size 935
```

〈Launch GPU threads to process q using func〉 ≡ 936

GPUParallelFor(desc, maxQueued, [=] PBRT_GPU (int index) mutable { if (index >= q->Size()) return; func((*q)[index]); });

For CPU processing, there are no concerns about reading the size field, so precisely the right number of items to process can be passed to ParallelFor(). 〈Process q using func with CPU threads〉 ≡ 936

ParallelFor(0, q->Size(), [&](int index) { func((*q)[index]); });

We will also find it useful to have work queues that support multiple types of objects and maintain separate queues, one for each type. This functionality is provided by the MultiWorkQueue. 〈MultiWorkQueue Definition〉 ≡ template <typename T> class MultiWorkQueue;

The definition of this class is a variadic template specialization that takes all of the types Ts it is to manage using a TypePack.

〈MultiWorkQueue Definition〉 +≡ template <typename… Ts> class MultiWorkQueue<TypePack<Ts… >> { public: 〈MultiWorkQueue Public Methods 937〉 private: 〈MultiWorkQueue Private Members 937〉 };

The MultiWorkQueue internally expands to a set of per-type WorkQueue instances that are stored in a tuple.

〈MultiWorkQueue Private Members〉 ≡ 937

pstd::tuple<WorkQueue<Ts>…> queues;

Note the ellipsis (…) in the code fragment above, which is a C++ template pack expansion. Such an expansion is only valid when it contains a template pack (Ts in this case), and it simply inserts the preceding element once for each specified template argument while replacing Ts with the corresponding type. For example, a hypothetical MultiWorkQueue<TypePack<A, B>> would contain a tuple of the form pstd::tuple<WorkQueue<A>, WorkQueue<B>>. This and the following template-based transformations significantly reduce the amount of repetitive code that would otherwise be needed to implement equivalent functionality. The following template method returns a queue for a particular work item that must be one of the MultiWorkQueue template arguments. The search through tuple items is resolved at compile time and so incurs no additional runtime cost. 〈MultiWorkQueue Public Methods〉 ≡ 937

template <typename T> WorkQueue<T> *Get() { return &pstd::get<WorkQueue<T>>(queues); }

```text
   GPUParallelFor() 929
   MultiWorkQueue 937
   MultiWorkQueue::queues 937
   ParallelFor() 1107
   PBRT_GPU 929
   TypePack 1071
   WorkQueue 935
   WorkQueue::Size() 935
```

The MultiWorkQueue constructor takes the maximum number of items to store in the queue and an allocator. The third argument is a span of Boolean values that indicates whether each type is actually present. Saving memory for types that are not required in a given execution of the renderer is worthwhile when both the maximum number of items is large and the work item types themselves are large.

〈MultiWorkQueue Public Methods〉 +≡ 937

MultiWorkQueue(int n, Allocator alloc, pstd::span<const bool> haveType) { int index = 0; ((*Get<Ts>() = WorkQueue<Ts>(haveType[index++] ? n : 1, alloc)), …); }

Once more, note the use of the ellipsis in the above fragment, which is a more advanced case of a template pack expansion following the pattern ((expr), …). As before, this expansion will simply repeat the element (expr) preceding the ellipsis once for every Ts with appropriate substitutions. In contrast to the previous case, we are now expanding expressions rather than types. The actual values of these expressions are ignored because they will be joined by the comma operator that ignores the value of the preceding expression. Finally, the actual expression being repeated has side effects: it initializes each tuple entry with a suitable instance, and it also advances a counter index that is used to access corresponding elements of the haveType span. The Size() method returns the size of a queue, and Push() appends an element. Both methods require that the caller specify the concrete type T of work item, which allows it to directly call the corresponding method of the appropriate queue. 〈MultiWorkQueue Public Methods〉 +≡ 937

template <typename T> int Size() const { return Get<T>()->Size(); } template <typename T> int Push(const T &value) { return Get<T>()->Push(value); }

Finally, all queues are reset via a call to Reset(). Once again, template pack expansion generates calls to all the individual queues’ Reset() methods via the following terse code. 〈MultiWorkQueue Public Methods〉 +≡ 937

void Reset() { (Get<Ts>()->Reset(), …); }

## 15.3 PATH TRACER IMPLEMENTATION

With these utility classes in hand, we can turn to the WavefrontPathIntegrator class implementation. As mentioned earlier, its functionality matches that of the VolPathIntegrator, restructured to run with a wavefront architecture.

The WavefrontPathIntegrator class declaration is in the file wavefront/integrator.h and some of its implementation is in wavefront/integrator.cpp, though a number of its method implementations are distributed across separate source files in the wavefront/ directory. While this is a different organization than we have used elsewhere in pbrt (where all the non-inline method definitions for a class defined in a file named file.h are in file.cpp), distributing them in this way reduces the time necessary to compile pbrt, since many of the methods make use of C++ features that can lead to long compile times. Spreading them out across multiple files allows multiple CPU cores to compile their implementations in parallel.

```text
    Allocator 40
    MultiWorkQueue 937
    MultiWorkQueue::Get() 937
    VolPathIntegrator 877
    WorkQueue 935
    WorkQueue::Push() 936
    WorkQueue::Reset() 935
    WorkQueue::Size() 935
```

〈WavefrontPathIntegrator Definition〉 ≡ class WavefrontPathIntegrator { public: 〈WavefrontPathIntegrator Public Methods 939〉 〈WavefrontPathIntegrator Member Variables 939〉 };

The constructor converts the provided BasicScene into the objects that represent the scene for rendering. We will skip over the majority of its implementation here, however, as most of it just calls all the appropriate object Create() methods to allocate and initialize the corresponding objects (see Section C.3). All allocations are performed with Allocators that use the provided memory resource. When rendering on the GPU, this leads to the use of managed memory.

〈WavefrontPathIntegrator Public Methods〉 ≡ 939

WavefrontPathIntegrator(pstd::pmr::memory_resource *memoryResource, BasicScene &scene); 〈WavefrontPathIntegrator Member Variables〉 ≡ 939

pstd::pmr::memory_resource *memoryResource;

These are some of the key scene objects that are initialized in the constructor. As with the CPU-based Integrators, the infinite lights are stored independently so that they can be efficiently iterated over for rays that escape the scene.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

Filter filter; Film film; Sampler sampler; Camera camera; pstd::vector<Light> *infiniteLights; LightSampler lightSampler;

There are a few additional member variables to store what should now be familiar configuration options.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

int maxDepth, samplesPerPixel; bool regularize;

In order to limit memory use, this integrator places a limit on the number of pixel samples that it works on at once. All told, each active pixel sample requires roughly 1,000 bytes of additional storage for its state variables and to ensure that all the queues have sufficient space for the work to be done for the ray. (The actual amount of storage varies based on the scene, as the constructor is careful not to allocate work queues for the volume scattering kernels if there is no participating media in the scene, for example.) The following fragment from the WavefrontPathIntegrator constructor therefore sets a maximum number of active samples and then determines how many scanlines of pixels that corresponds to. That value in turn determines how many passes (of rendering that many scanlines) are necessary to cover the full image resolution. Finally, scanlinesPerPass is set with a new value that evens out the number of scanlines rendered in each pass. This can help with load balancing by avoiding having a small number of scanlines in the final pass.

```text
    Allocator 40
    BasicScene 1134
    Camera 206
    Film 244
    Filter 515
    Integrator 22
    Light 740
    LightSampler 781
    Sampler 469
    WavefrontPathIntegrator 939
```

〈Compute number of scanlines to render per pass〉 ≡ Vector2i resolution = film.PixelBounds().Diagonal(); int maxSamples = 1024 * 1024; scanlinesPerPass = std::max(1, maxSamples / resolution.x); int nPasses = (resolution.y + scanlinesPerPass - 1) / scanlinesPerPass; scanlinesPerPass = (resolution.y + nPasses - 1) / nPasses; maxQueueSize = resolution.x * scanlinesPerPass;

All the queues that are allocated to buffer work between kernels are also allocated to be able to store maxQueueSize individual work items. Thus, all queues are able to store one work item for each of the active pixel samples. This is a sufficient number, as none of the kernels in the current implementation ever push more than one item on a queue for each item processed. However, it may waste a substantial amount of memory. For example, there is a work queue for rays that hit emissive surfaces in the scene. It is rare that all the rays will hit emissive objects, yet there must be storage for all in case of that eventuality. The alternative, dynamically increasing the size of the queues when necessary, would be difficult to implement efficiently in the context of the massive parallelism on GPUs.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

int scanlinesPerPass, maxQueueSize;

The WavefrontPathIntegrator does not use only queues to provide values to kernels and to communicate results. While the queue model is elegant, it can be inefficient if some values are computed early and not used until much later. For example, consider the VisibleSurface structure that is provided to Film implementations like the GBufferFilm: it is initialized at the first intersection after the camera, but then its value is not used again until the full ray path has been traced and the Film::AddSample() method is called at the end. VisibleSurface is, further, a relatively large structure. In a purely queue-based model, a substantial amount of memory bandwidth would be consumed passing it along from kernel to kernel until the end.

Therefore, the PixelSampleState structure is used for storing all such values. Various member variables will be added to it in what follows.

〈PixelSampleState Definition〉 ≡ struct PixelSampleState { 〈PixelSampleState Public Members 945〉 };

The WavefrontPathIntegrator maintains an SOA-arranged array of PixelSampleStates, allocated to have maxQueueSize entries. Each sample’s index into this array is carried through the rendering computation so that it is easy to determine which entry corresponds to a pixel sample being processed.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

SOA<PixelSampleState> pixelSampleState;

### 15.3.1 WORK LAUNCH

Because the WavefrontPathIntegrator may be running either on the CPU or on the GPU, it provides methods for launching work that use the appropriate processor. Each selects the appropriate type of processor based on the renderer’s configuration.

The first, ParallelFor(), selects between the types of two parallel for loops.

```text
    Bounds2::Diagonal() 101
    Film 244
    Film::AddSample() 244
    Film::PixelBounds() 246
    GBufferFilm 253
    PixelSampleState 940
    SOA 932
    Vector2i 86
    VisibleSurface 245
    WavefrontPathIntegrator 939
```

〈WavefrontPathIntegrator Public Methods〉 +≡ 939

template <typename F> void ParallelFor(const char *description, int nItems, F &&func) { if (Options->useGPU) GPUParallelFor(description, nItems, func); else pbrt::ParallelFor(0, nItems, func); }

The Do() method executes the provided function in a single thread. On the CPU, it is no different than a regular function call; on the GPU, however, it executes the provided function using GPUParallelFor() with a single-item loop. For reasons that should be clear by now, this not a good way to do a meaningful amount of work on the GPU, but this capability is necessary for resetting counters, clearing queues, and the like.

〈WavefrontPathIntegrator Public Methods〉 +≡ 939

template <typename F> void Do(const char *description, F &&func) { if (Options->useGPU) GPUParallelFor(description, 1, [=] PBRT_GPU (int) mutable { func(); }); else func(); }

### 15.3.2 THE Render() METHOD

Rendering is initiated by a call to Render(). (Recall Figure 15.2 in Section 15.1.3, which summarizes the kernels that this method launches.) Similar to earlier integrators, it progressively takes more samples in all pixels until the requested number of samples have been taken. This method tracks how long rendering takes and returns the number of elapsed seconds; we have not included here the straightforward few lines of code that handle that.

```text
    BasicPBRTOptions::useGPU 1031
    Bounds2::Diagonal() 101
    Bounds2i 97
    Film 244
    Film::PixelBounds() 246
    Float 23
```

```text
    GPUParallelFor() 929
    Options 1032
    ParallelFor() 1107
    PBRT_GPU 929
    Sampler 469
    Sampler::SamplesPerPixel() 469
    Vector2i 86
    WavefrontPathIntegrator::film 939
```

One subtlety is that the initialization of the pixelBounds variable at the start is important for rendering performance. It will be necessary to have this value later on in the implementation of Render()—though because the Film is stored in managed memory, calling the PixelBounds() method after GPU kernels have been launched could incur the overhead of copying data back to the CPU.

〈WavefrontPathIntegrator Method Definitions〉 ≡ Float WavefrontPathIntegrator::Render() { Bounds2i pixelBounds = film.PixelBounds(); Vector2i resolution = pixelBounds.Diagonal(); 〈Loop over sample indices and evaluate pixel samples 942〉 }

By default, the number of samples taken in each pixel is the number determined by the Sampler; the value returned by its SamplesPerPixel() method is cached in a member variable for the same reason as pixelBounds is cached above. It is possible to limit rendering to a single specified sample index using the --debugstart command line option; the code to set firstSampleIndex and lastSampleIndex in that case is not included here.

〈Loop over sample indices and evaluate pixel samples〉 ≡ 941

```rust
   int firstSampleIndex = 0, lastSampleIndex = samplesPerPixel; for (int
   sampleIndex       =     firstSampleIndex;          sampleIndex < lastSampleIndex;
   ++sampleIndex) {
        〈Render image for sample sampleIndex 942〉
   }
```

Given a sample index, the next step is to loop over the one or more chunks of scanlines and to evaluate a sample in each of their pixels. This code is also similar to the corresponding code in the VolPathIntegrator: evaluating each sample starts with generating a camera ray and then following it through multiple intersections until it is terminated or the maximum depth is reached. The key difference is that, here, these tasks are being performed for as many as a million or so pixel samples at a time.

〈Render image for sample sampleIndex〉 ≡ 942

```rust
   for (int y0 = pixelBounds.pMin.y; y0                      <   pixelBounds.pMax.y;         y0   +=
   scanlinesPerPass) {
```

```rust
         〈Generate camera rays for current scanline range 942〉
         〈Trace rays and estimate radiance up to maximum ray depth 948〉
         UpdateFilm(); }
```

Before the rays are generated, it is necessary to reset the work queue that will store them. We will need to maintain more than one ray queue: one for the set of rays currently being traced and another for the indirect rays that have been spawned to be traced at the next depth. The CurrentRayQueue() method, defined shortly, returns the queue for the specified depth.

When the GPU is being used for rendering, it is critically important that the Reset() method is called from the GPU and not the CPU; Do() is thus used here. Not only could resetting it from the CPU be inefficient, as doing so would involve the CPU writing to managed memory, but—given the asynchronous execution of the GPU—it would almost certainly be incorrect, potentially resetting a queue that was still in use by the code that was executing on the GPU.

〈Generate camera rays for current scanline range〉 ≡ 942

```rust
   RayQueue *cameraRayQueue = CurrentRayQueue(0); Do(“Reset ray                             queue“,
   PBRT_CPU_GPU_LAMBDA () {
                                      cameraRayQueue->Reset();                                   });
                                      GenerateCameraRays(y0, sampleIndex);
```

The RayQueue class adds a few convenience methods to WorkQueue that we will discuss in the following pages. (RayWorkItem will also be defined in a few pages, closer to where it is first used.) 〈RayQueue Definition〉 ≡ class RayQueue : public WorkQueue<RayWorkItem> { public: 〈RayQueue Public Methods 947〉 };

The WavefrontPathIntegrator maintains a pair of ray queues, rather than allocating one for each ray depth up to the maximum. It manages them using double buffering: one stores the current active set of rays and should only be read from, while the other stores the rays enqueued to trace at the next depth and should only be written to. At each successive ray depth, the roles of the two queues are switched.

```text
    Bounds2::pMax 98
    Bounds2::pMin 98
    PBRT_CPU_GPU_LAMBDA 930
    RayQueue 942
    RayWorkItem 947
    VolPathIntegrator 877
    WavefrontPathIntegrator::CurrentRayQueue() 943
    WavefrontPathIntegrator::Do() 941
    WavefrontPathIntegrator::GenerateCameraRays() 943
```

```text
    WavefrontPathIntegrator::samplesPerPixel 939
    WavefrontPathIntegrator::scanlinesPerPass 940
    WavefrontPathIntegrator::UpdateFilm() 970
    WorkQueue 935
    WorkQueue::Reset() 935
```

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

RayQueue *rayQueues[2];

Two convenience methods return pointers to the RayQueues given a depth. The double buffering logic is effectively encapsulated in their implementations. 〈WavefrontPathIntegrator Public Methods〉 +≡ 939

RayQueue *CurrentRayQueue(int wavefrontDepth) { return rayQueues[wavefrontDepth & 1]; } RayQueue *NextRayQueue(int wavefrontDepth) { return rayQueues[(wavefrontDepth + 1) & 1]; }

### 15.3.3 GENERATING CAMERA RAYS

The two methods related to generating camera rays are implemented in the file wavefront/camera.cpp, not in wavefront/integrator.cpp, because we have used C++ templates to generate multiple specialized instances of the methods, each one specialized based on some of the object types involved. As mentioned earlier, techniques like this can cause lengthy compile times, so it is worthwhile to isolate the camera method implementations in their own source file. There are two GenerateCameraRays() methods. The first, which is called by WavefrontPath Integrator::Render(), determines the concrete type of the Sampler being used and then calls the second GenerateCameraRays() method, which is templated on the type of the Sampler. This idea is something that we will see again in other methods: CPU code making a runtime determination of the types of objects involved in a computation, which allows the execution of code that is specialized for those types. This is especially beneficial to performance when running on the GPU.

The first method defines a lambda function, generateRays, and then invokes the Tagged Pointer’s dynamic dispatch mechanism, which will end up calling the lambda function using the concrete type of the Sampler. (In this case, we use the DispatchCPU() variant, which must be used for code that can only execute on the CPU.) 〈WavefrontPathIntegrator Camera Ray Methods〉 ≡ void WavefrontPathIntegrator::GenerateCameraRays(int y0, int sampleIndex) { 〈Define generateRays lambda function 944〉 sampler.DispatchCPU(generateRays); }

Using auto in the parameter declaration for the following lambda function causes it to be parameterized by the type of sampler, like a template function. (C++20 provides a less obscure syntax for templated lambda functions, though this version of pbrt limits itself to C++17.) Thus, the type of sampler will be a concrete instance of one of the sampler types provided in the Sampler declaration in Section 8.3. There is a nit in that sampler is passed as a reference to a pointer to the specific Sampler type; a bit of work in the using declaration gives us the actual Sampler type. A second nit is that we must filter out the MLTSampler, which is only used by the MLTIntegrator, and DebugMLTSampler, a variant of it that is used for debugging. Those classes use vector methods like push_back() that are not supported in GPU code, and therefore we must make clear to the compiler what we know in any case: those samplers will not be used here.

```text
   RayQueue 942
   Sampler 469
   TaggedPointer 1073
   TaggedPointer::DispatchCPU() 1076
   WavefrontPathIntegrator::rayQueues 943
   WavefrontPathIntegrator::Render() 941
   WavefrontPathIntegrator::sampler 939
```

With the concrete sampler type in hand, the second GenerateCameraRays() method can be called, with the sampler type provided for the template specialization.

〈Define generateRays lambda function〉 ≡ 943

auto generateRays = [=](auto sampler) { using ConcreteSampler = std::remove_reference_t<decltype(*sampler)>; if constexpr (!std::is_same_v<ConcreteSampler, MLTSampler> && !std::is_same_v<ConcreteSampler, DebugMLTSampler>) GenerateCameraRays<ConcreteSampler>(y0, sampleIndex); };

We can now move on to the second GenerateCameraRays() method, which calls Wavefront PathIntegrator::ParallelFor() to generate all of the rays in parallel. 〈WavefrontPathIntegrator Camera Ray Methods〉 +≡ template <typename ConcreteSampler> void WavefrontPathIntegrator::GenerateCameraRays(int y0, int sampleIndex) { RayQueue *rayQueue = CurrentRayQueue(0); ParallelFor(“Generate camera rays“, maxQueueSize, PBRT_CPU_GPU_LAMBDA (int pixelIndex) { 〈Enqueue camera ray and set pixel state for sample 944〉 }); }

The sequence of operations performed for each camera ray again generally matches what we have seen before: after computing the pixel coordinates for the provided loop index, samples are generated, a set of wavelengths are sampled, and then the Camera generates the ray.

〈Enqueue camera ray and set pixel state for sample〉 ≡ 944

〈Compute pixel coordinates for pixelIndex 944〉 〈Test pixel coordinates against pixel bounds 945〉 〈Initialize Sampler for current pixel and sample 945〉 〈Sample wavelengths for ray path 945〉 〈Generate CameraSample and corresponding ray 946〉 〈Initialize remainder of PixelSampleState for ray 946〉 〈Enqueue camera ray for intersection tests 946〉

Given the pixel bounds of the film, the pixelIndex value is mapped to pixel coordinates, starting from the (x, y) lower bound given by the pixel bounds and y0 value, respectively. Threads are then assigned to pixels in

scanline order. Because the PBRT_GPU_LAMBDA macro includes *this in the lambda capture, various WavefrontPathIntegrator member variables such as film, which is used here, can be directly accessed in the lambda function. The pixel coordinates are then stored in PixelSampleState. Note that if we are just setting a single member variable of an SOA structure rather than assigning an entire structure value, then the member variable must be indexed and not the pixelSampleState structure itself; our automatically generated SOA classes are not able to provide the same syntax as would be used for an array of structures layout in that case. 〈Compute pixel coordinates for pixel Index〉 ≡ 944

Bounds2i pixelBounds = film.PixelBounds(); int xResolution = pixelBounds.pMax.x - pixelBounds.pMin.x; Point2i pPixel(pixelBounds.pMin.x + pixelIndex % xResolution, y0 + pixelIndex / xResolution); pixelSampleState.pPixel[pixelIndex] = pPixel;

```text
   Bounds2::pMax 98
   Bounds2::pMin 98
   Bounds2i 97
   Camera 206
   Film::PixelBounds() 246
   PBRT_CPU_GPU_LAMBDA 930
   PixelSampleState 940
   PixelSampleState::pPixel 945
   Point2i 92
   RayQueue 942
   Sampler 469
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::CurrentRayQueue() 943
   WavefrontPathIntegrator::film 939
   WavefrontPathIntegrator::maxQueueSize 940
   WavefrontPathIntegrator::ParallelFor() 941
   WavefrontPathIntegrator::pixelSampleState 940
```

The pixel coordinates corresponding to a sample are our first addition to the PixelSample State structure. 〈PixelSampleState Public Members〉 ≡ 940

Point2i pPixel;

If the image has been split into multiple spans of scanlines, then the number of parallel loop iterations in the final pass may be a few more than there are pixels left to be sampled. Rather than worry about this when specifying the number of threads to launch in the call to WavefrontPathIntegrator::ParallelFor(), we just check this condition in the kernel. Indices for out of bounds pixels return after initializing PixelSampleState::pPixel and do not enqueue any rays.

〈Test pixel coordinates against pixel bounds〉 ≡ 944

if (!InsideExclusive(pPixel, pixelBounds)) return;

Next, samples are needed to generate the camera ray. Here we can see the value of specializing this method based on the ConcreteSampler type. The following fragment is perhaps best understood by reading it with the ConcreteSampler in the following replaced with, say, HaltonSampler in your head. pixelSampler then is a stack-allocated HaltonSampler, and the TaggedPointer::Cast() call gives us a pointer of that type; dereferencing it lets each thread initialize its own sampler from the exemplar of the one that the WavefrontPathIntegrator stores. All threads access the same memory locations, so reading the Sampler from memory does not consume much bandwidth. The benefit from this approach comes from having the sampler allocated on the stack. On the GPU, its member variables can be stored directly in registers, giving high performance when executing its methods. As an additional bonus, there is no overhead for dynamic dispatch in the StartPixelSample() call: the sampler’s type is known, so the appropriate method can be called directly. It will usually be expanded inline at the call site. 〈Initialize Sampler for current pixel and sample〉 ≡ 944

```rust
   ConcreteSampler pixelSampler = *sampler.Cast<ConcreteSampler>();
   pixelSampler.StartPixelSample(pPixel, sampleIndex, 0);
```

Wavelengths are sampled in precisely the same way as they are in most of the other integrators. 〈Sample wavelengths for ray path〉 ≡ 944

```rust
   Float lu = pixelSampler.Get1D(); SampledWavelengths lambda =
   film.SampleWavelengths(lu);
```

The CameraSample can be initialized in the usual way and then it is on to call Camera::GenerateRay(). In this case, this method call is also resolved using pbrt’s usual dynamic dispatch system, just as it is in all the other integrators. An alternative implementation approach would be to further specialize the GenerateCameraRays() method based on the type of the Camera being used; after all, it is the same for all pixel samples and so it is somewhat wasteful for all threads to perform the same computations for dynamic dispatch. We have found that in practice that alternative gives a negligible performance benefit, and so our implementation here remains based on dynamic dispatch.

```text
   Bounds2::InsideExclusive() 100
   Camera::GenerateRay() 206
   CameraSample 206
   Film::SampleWavelengths() 246
   Float 23
   PixelSampleState 940
   PixelSampleState::pPixel 945
   Point2i 92
   Ray 95
   RayDifferential 96
   SampledWavelengths 173
   Sampler 469
   Sampler::Get1D() 470
   Sampler::StartPixelSample() 469
   TaggedPointer::Cast() 1074
   WavefrontPathIntegrator::ParallelFor() 941
   WavefrontPathIntegrator::sampler 939
```

One other thing to note is that the wavefront integrator path generates regular Rays here and not RayDifferentials. Approximate differentials for filtering will be computed later, trading off superior antialiasing quality in exchange for higher performance from reduced memory bandwidth due to having less information associated with each ray. An exercise at the end of the chapter revisits this choice. 〈Generate CameraSample and corresponding ray〉 ≡ 944

CameraSample cameraSample = GetCameraSample(pixelSampler, pPixel, filter); pstd::optional<CameraRay> cameraRay =

```rust
        camera.GenerateRay(cameraSample, lambda);
```

A few additional values that are stored in PixelSampleState can now be set. We avoid initializing the heavyweight VisibleSurface member if it is not going to be used by the Film; the initializeVisibleSurface member variable is set in the WavefrontPathIntegrator constructor to record whether it is needed. Saving this memory bandwidth when possible is worth this easy check. 〈Initialize remainder of PixelSampleState for ray〉 ≡ 944

```rust
   pixelSampleState.L[pixelIndex] = SampledSpectrum(0.f);
   pixelSampleState.lambda[pixelIndex] = lambda;
   pixelSampleState.filterWeight[pixelIndex] = cameraSample.filterWeight; if
   (initializeVisibleSurface) pixelSampleState.visibleSurface[pixelIndex] =
   VisibleSurface();
```

Of these member variables, we will see L most often in what follows. It stores the accumulated radiance estimate for the ray path. After the path terminates, its value will be provided to the Film. 〈PixelSampleState Public Members〉 +≡ 940

SampledSpectrum L; SampledWavelengths lambda; Float filterWeight; VisibleSurface visibleSurface;

If the Camera successfully generated a ray, it is pushed into the RayQueue along with its wavelengths and associated pixel index, which allows subsequent kernels to be able to index into WavefrontPathIntegrator::pixelSampleState to retrieve values there that are associated with this ray. 〈Enqueue camera ray for intersection tests〉 ≡ 944

if (cameraRay) { rayQueue->PushCameraRay(cameraRay->ray, lambda, pixelIndex); pixelSampleState.cameraRayWeight[pixelIndex] = cameraRay->weight; } else pixelSampleState.cameraRayWeight[pixelIndex] = SampledSpectrum(0); 〈PixelSampleState Public Members〉 +≡ 940

SampledSpectrum cameraRayWeight;

```text
   Camera 206
   Camera::GenerateRay() 206
   CameraRay 207
```

```text
   CameraRay::ray 207
   CameraRay::weight 207
   CameraSample 206
   CameraSample::filterWeight 206
   Film 244
   Float 23
   GetCameraSample() 516
   PixelSampleState::cameraRayWeight 946
   PixelSampleState::filterWeight 946
   PixelSampleState::L 946
   PixelSampleState::lambda 946
   PixelSampleState::visibleSurface 946
   RayQueue 942
   RayQueue::PushCameraRay() 947
   RayQueue::PushIndirectRay() 967
   RayWorkItem 947
   SampledSpectrum 171
   SampledWavelengths 173
   VisibleSurface 245
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::camera 939
   WavefrontPathIntegrator::pixelSampleState 940
```

Some of the work queues provide specialized methods for pushing work on to them; RayQueue is one of them. It provides a PushCameraRay() method for camera rays and Push IndirectRay() for scattered rays that sample indirect lighting. Doing so not only makes the code where work is pushed on to the queue slightly cleaner, but it also makes it possible to set some of the forthcoming RayWorkItem member variables to default values for camera rays without needing to specify default values here. (For example, RayWorkItem carries path sampling PDFs that are initialized to 1 for camera rays.) We will pass over the implementation of this method here, as it is all setting member variables, either from passed-in values or from defaults. 〈RayQueue Public Methods〉 ≡ 942

int PushCameraRay(const Ray &ray, const SampledWavelengths &lambda, int pixelIndex);

We will start the definition of the RayWorkItem structure here. 〈RayWorkItem Definition〉 ≡ struct RayWorkItem {

```rust
        〈RayWorkItem Public Members 947〉
   };
```

As would be expected from the parameters passed to PushCameraRay(), RayWorkItem stores a ray, its wavelengths, and an index into the WavefrontPathIntegrator’s pixelSampleState array from which additional data associated with the ray can be found. 〈RayWorkItem Public Members〉 ≡ 947

Ray ray; int depth; SampledWavelengths lambda; int pixelIndex;

The reader may have noted that lambda is both pushed to the ray queue and stored in the PixelSampleState, which is admittedly redundant, though there are good reasons for both. Its value must be stored in the PixelSampleState structure, as it is required for updating the film, which is not scheduled using work queues but via a loop over the PixelSampleState values; see further discussion of this design in Section 15.3.11. However, if other kernels along the way accessed lambda via PixelSampleState, performance would be poor, since the initial correlation between the value of pixelIndex and threads in a thread group quickly becomes shuffled up over the course of rendering. Thus, reads from PixelSampleState would be highly incoherent. Because it is used in just about every kernel, lambda is also passed along throughout the queues of the wavefront integrator. Since it is in work queues, the loads and stores to read and save its value are coherent across the thread group, giving good performance.

### 15.3.4 LOOP OVER RAY DEPTHS

With the camera having seeded the ray queue with an initial set of rays, we can now turn to the loop that executes once for each successive set of ray intersection tests and shading calculations up until the maximum ray path length. One subtlety is that this loop is over “wavefront depth,” which is different than the ray depth that has been tracked in other integrators. Here, the wavefront depth reflects the number of times that this loop has executed, tracing a batch of rays and processing their intersections. Each

ray tracks its own depth, which is usually the same as the wavefront depth, though the depth of rays that hit invisible surfaces that delineate the boundaries between volumetric media is not incremented at those intersections (as in other integrators).

```text
   PixelSampleState 940
   Ray 95
   SampledWavelengths 173
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::pixelSampleState 940
```

This loop always runs on the CPU; when the GPU is used for rendering, it is responsible for launching the appropriate kernels to perform the rendering computation. An implication of this design and the decoupling of the CPU and GPU is that the CPU has no visibility into the state of the ray-tracing calculations on the GPU. For example, if all rays terminate after the first intersection, the CPU will continue submitting kernel launches to execute the ray-tracing pipeline even though no work is passing through it. All the kernels would exit immediately, though there would be a cost from launching all of them and their determining that there is no work to be done. For many scenes this is not a problem, but it can harm this integrator’s performance with high maximum ray depths. An exercise at the end of the chapter outlines some design alternatives. 〈Trace rays and estimate radiance up to maximum ray depth〉 ≡ 942

for (int wavefrontDepth = 0; true; ++wavefrontDepth) { 〈Reset queues before tracing rays 948〉 〈Follow active ray paths and accumulate radiance estimates 948〉 }

All the work queues except for the RayQueue holding the current set of rays must be cleared at the start of each iteration. The following fragment clears the RayQueue for the next set of indirect rays and adds a fragment for the rest of the queues. We will add additional Reset() calls to this fragment for the various other queues along with the code that defines and uses them. 〈Reset queues before tracing rays〉 ≡ 948

RayQueue *nextQueue = NextRayQueue(wavefrontDepth); Do(“Reset queues before tracing rays“, PBRT_CPU_GPU_LAMBDA () { nextQueue->Reset(); 〈Reset queues before tracing next batch of rays 958〉

```rust
        });
```

Once the queues are cleared, rendering proceeds mostly following the same steps as the VolPathIntegrator. First, the Sampler computes sample values for all of the rays and stores them in memory. 〈Follow active ray paths and accumulate radiance estimates〉 ≡ 948

GenerateRaySamples(wavefrontDepth, sampleIndex);

The closest intersection with a surface, if any, is then found for each ray. 〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

〈Find closest intersections along active rays 952〉

Before surface scattering or emission is considered, the medium (if any) is sampled. If the medium scatters or absorbs the ray, then any surface intersection will be ignored. 〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

SampleMediumInteraction(wavefrontDepth);

Only after medium sampling are rays that left the scene taken care of. In this way, rays passing through participating media that do not interact with it and then leave the scene can be processed at the same time as rays that left the scene but did not pass through media. 〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

HandleEscapedRays();

The contribution of emissive surfaces to a sample’s radiance value is also only added after medium sampling, since it should not be included if the medium scattered or absorbed the ray. 〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

HandleEmissiveIntersection();

The loop over wavefront depth can only be terminated after emissive surfaces have been accounted for, since the MIS-based direct lighting calculation accounts for their contribution.

```text
   PBRT_CPU_GPU_LAMBDA 930
   RayQueue 942
```

```text
    Sampler 469
    VolPathIntegrator 877
    WavefrontPathIntegrator::Do() 941
    WavefrontPathIntegrator::GenerateRaySamples() 949
    WavefrontPathIntegrator::HandleEmissiveIntersection() 957
    WavefrontPathIntegrator::HandleEscapedRays() 955
    WavefrontPathIntegrator::NextRayQueue() 943
    WavefrontPathIntegrator::SampleMediumInteraction() 954
    WorkQueue::Reset() 935
```

〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

if (wavefrontDepth == maxDepth) break;

If the loop does not terminate, only now are surface intersections handled, with the Materials evaluating their textures and returning BSDFs, lights sampled, and indirect rays enqueued. 〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

EvaluateMaterialsAndBSDFs(wavefrontDepth);

Next, the shadow rays enqueued by the material evaluation kernel are traced; the radiance contributions of the unoccluded ones are accumulated in PixelSampleState.

〈Follow active ray paths and accumulate radiance estimates〉 +≡ 948

TraceShadowRays(wavefrontDepth);

The following sections go into these steps in more detail.

### 15.3.5 SAMPLE GENERATION

The first step in each loop iteration is to generate values for all the sample dimensions that may be required for sampling lighting if a scattering event is found along the ray, either in a participating medium or on a surface. Generating all of these samples ahead of time in a separate kernel allows for specializing that kernel based on the sampler type, giving the same performance benefits as were found in GenerateCameraRays().

The GenerateRaySamples() method is defined in the file wavefront/samples.cpp. Similar to camera rays, there is an initial GenerateRaySamples() method called in the main rendering loop that

determines the actual Sampler type and invokes the appropriate specialization. Because this dispatch method is nearly the same as the analog in GenerateCameraRays(), we omit it here and turn directly to the specialized method. 〈WavefrontPathIntegrator Sampler Methods〉 ≡ template <typename ConcreteSampler> void WavefrontPathIntegrator::GenerateRaySamples(int wavefrontDepth, int sampleIndex) { 〈Generate description string desc for ray sample generation〉 RayQueue *rayQueue = CurrentRayQueue(wavefrontDepth); ForAllQueued(desc.c_str(), rayQueue, maxQueueSize, PBRT_CPU_GPU_LAMBDA (const RayWorkItem w) { 〈Generate samples for ray segment at current sample index 950〉

}); }

Unlike the CPU-only integrators, where sample dimensions are allocated implicitly based on the runtime sequence of Get1D() and Get2D() method calls, here dimensions are explicitly allocated, ensuring that unique dimensions are assigned to different uses of samples. Doing so imposes a coupling between this kernel and the use of samples in following ones and also means that samples may be generated that are not actually used (e.g., if a perfect specular surface is intersected). These costs are worth paying in return for the performance benefits of generating samples in a specialized kernel.

```text
   BSDF 544
   ForAllQueued() 936
   Material 674
   PBRT_CPU_GPU_LAMBDA 930
   PixelSampleState 940
   RayQueue 942
   RayWorkItem 947
   Sampler 469
   WavefrontPathIntegrator::CurrentRayQueue() 943
   WavefrontPathIntegrator::EvaluateMaterialsAndBSDFs() 959
   WavefrontPathIntegrator::maxDepth 939
```

```text
   WavefrontPathIntegrator::TraceShadowRays() 969
```

〈Generate samples for ray segment at current sample index〉 ≡ 949

〈Find first sample dimension 950〉 〈Initialize Sampler for pixel, sample index, and dimension 950〉 〈Initialize RaySamples structure with sample values 950〉 〈Store RaySamples in pixel sample state 951〉

The first task is to find the first dimension to allocate for this ray’s samples. The first 5 sample dimensions are used to generate the camera ray, and then 1 is used to sample the wavelengths. At least 7 samples are needed for each ray depth: 3 for the call to BSDF::Sample_f(), 1 to sample a light source and then 2 to sample a position on it, and then 1 more for Russian roulette for the indirect ray. Two additional dimensions are consumed for sampling participating media. (The haveMedia WavefrontPathIntegrator member variable is set in its constructor based on the scene.) 〈Find first sample dimension〉 ≡ 950

int dimension = 6 + 7 * w.depth; if (haveMedia) dimension += 2 * w.depth;

This kernel uses the same trick to get a stack-allocated Sampler as was done for camera rays. In this case, figuring out which pixel the ray is associated with requires a read from the PixelSampleState. 〈Initialize Sampler for pixel, sample index, and dimension〉 ≡ 950

```rust
   ConcreteSampler pixelSampler = *sampler.Cast<ConcreteSampler>(); Point2i
   pPixel = pixelSampleState.pPixel[w.pixelIndex];
   pixelSampler.StartPixelSample(pPixel, sampleIndex, dimension);
```

The RaySamples structure bundles up the samples needed for a ray. A series of Get1D() and Get2D() calls initializes its member variables. We omit the fragment that initializes the indirect samples, as it is more of the same. We have carefully ordered the sample generation method calls below to match their use in the VolPathIntegrator so that the two integrators use the same sample values for their sampling tasks at each pixel sample. 〈Initialize RaySamples structure with sample values〉 ≡ 950

RaySamples rs; rs.direct.uc = pixelSampler.Get1D(); rs.direct.u = pixelSampler.Get2D(); 〈Initialize remaining samples in rs〉 〈RaySamples Definition〉 ≡ struct RaySamples {

〈RaySamples Public Members 950〉 };

```text
   BSDF::Sample_f() 545
   Float 23
   PixelSampleState 940
   PixelSampleState::pPixel 945
   Point2f 92
   Point2i 92
   RaySamples 950
   RaySamples::direct 950
   RaySamples::direct::u 950
   RaySamples::direct::uc 950
   RayWorkItem::depth 947
   RayWorkItem::pixelIndex 947
   Sampler 469
   Sampler::Get1D() 470
   Sampler::Get2D() 470
   Sampler::StartPixelSample() 469
   TaggedPointer::Cast() 1074
   VolPathIntegrator 877
   WavefrontPathIntegrator::pixelSampleState 940
```

The three sample dimensions for sampling the light source are available in the direct substructure. 〈RaySamples Public Members〉 ≡ 950

struct { Point2f u; Float uc; } direct;

Similarly, the dimensions for BSDF sampling and Russian roulette are available in indirect. 〈RaySamples Public Members〉 +≡ 950

struct { Float uc, rr; Point2f u; } indirect;

haveMedia indicates whether medium samples have been stored, which makes it possible to save bandwidth when they are unset. 〈RaySamples Public Members〉 +≡ 950

bool haveMedia; struct { Float uDist, uMode; } media;

Sample values are squirreled away in PixelSampleState rather than being passed along via work queues. This does mean that both storing sample values here and reading them in subsequent kernels is not done with coherent memory accesses, since a thread group’s pixelIndex values will not necessarily be contiguous after the initial camera rays. However, because they are not used in many kernels, passing them along through work queues would entail multiple instances of reading them from one queue just to write them to another, which would be a waste of bandwidth. We have found that the current approach gives marginally better performance in practice. 〈Store RaySamples in pixel sample state〉 ≡ 950

pixelSampleState.samples[w.pixelIndex] = rs; 〈PixelSampleState Public Members〉 +≡ 940

RaySamples samples;

One shortcoming of the approach implemented in this section is that samples are still generated for rays that do not intersect anything. A more optimized implementation might try to defer sample generation until the specific samples required were known, though the benefits are likely to be marginal: on the GPU, if the samples needed vary over the rays in a thread group, then—given the GPU’s thread group execution model—there may be no savings from skipping the work for some threads if it is still needed by others. Further, sample generation is normally just a few percent of overall rendering time, and so anything more sophisticated is not worth bothering with, at least for pbrt’s requirements.

### 15.3.6 INTERSECTION TESTING

Ray intersections are handled differently depending on whether the wavefront integrator is running on the CPU or the GPU. On the CPU, the ray queues are consumed using ParallelFor() calls and pbrt’s regular acceleration structures from Chapter 7 are used to find intersections. On the GPU, platform-specific functionality is used to do so. In order to abstract the differences between these approaches (and to make it easier to add support for additional GPU architectures), ray intersection work done by the WavefrontPathIntegrator is handled by an implementation of the

WavefrontAggregate class, which defines an interface that reflects the integrator’s needs. 〈WavefrontAggregate Definition〉 ≡ class WavefrontAggregate { public: 〈WavefrontAggregate Interface 952〉 };

```text
   Float 23
   ParallelFor() 1107
   PixelSampleState 940
   PixelSampleState::samples 951
   Point2f 92
   RaySamples 950
   RayWorkItem::pixelIndex 947
   WavefrontAggregate 951
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::pixelSampleState 940
```

The WavefrontPathIntegrator stores a WavefrontAggregate in its aggregate member variable. The CPU implementation, CPUAggregate, is found in the source files wavefront/ aggregate.h and wavefront/aggregate.cpp. The implementation for NVIDIA GPUs, OptiXAggregate, is found in gpu/aggregate.h and gpu/aggregate.cpp.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

WavefrontAggregate *aggregate = nullptr;

All WavefrontAggregates must provide a method that returns the bounds of the entire scene. 〈WavefrontAggregate Interface〉 ≡ 951

virtual Bounds3f Bounds() const = 0;

IntersectClosest() traces a set of rays and finds their closest surface intersections. Beyond the queue that provides the rays to be traced, a number of additional queues must be provided to it. Further work for a ray may be added to multiple queues depending on the specifics of its surface intersection.

〈WavefrontAggregate Interface〉 +≡ 951

virtual void IntersectClosest(int maxRays, const RayQueue *rayQ, EscapedRayQueue *escapedRayQ, HitAreaLightQueue *hitAreaLightQ, MaterialEvalQueue *basicMtlQ, MaterialEvalQueue *universalMtlQ, MediumSampleQueue *mediumSampleQ, RayQueue *nextRayQ) const = 0;

```text
   Bounds3f 97
   CPUAggregate 952
   EnqueueWorkAfterIntersection() 953
   EnqueueWorkAfterMiss() 953
   EscapedRayQueue 955
   HitAreaLightQueue 957
   MaterialEvalQueue 962
   ParallelFor() 1107
   Primitive::Intersect() 398
   Ray::medium 95
   RayQueue 942
   RayWorkItem 947
   RayWorkItem::ray 947
   ShapeIntersection 266
   ShapeIntersection::intr 266
   ShapeIntersection::tHit 266
   WavefrontAggregate 951
   WavefrontAggregate::IntersectClosest() 952
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::aggregate 952
   WavefrontPathIntegrator::basicEvalMaterialQueue 960
   WavefrontPathIntegrator::CurrentRayQueue() 943
   WavefrontPathIntegrator::escapedRayQueue 955
   WavefrontPathIntegrator::hitAreaLightQueue 957
   WavefrontPathIntegrator::maxQueueSize 940
   WavefrontPathIntegrator::mediumSampleQueue 954
   WavefrontPathIntegrator::NextRayQueue() 943
   WavefrontPathIntegrator::universalEvalMaterialQueue 960
   WorkQueue::Size() 935
```

The WavefrontPathIntegrator’s call to IntersectClosest() mostly passes the corresponding queues from its member variables, though calls to GetCurrentQueue() and NextRayQueue() are necessary to get the appropriate instances of those queues. 〈Find closest intersections along active rays〉 ≡ 948

aggregate->IntersectClosest(maxQueueSize, CurrentRayQueue(wavefrontDepth), escapedRayQueue, hitAreaLightQueue, basicEvalMaterialQueue,

```rust
   universalEvalMaterialQueue, mediumSampleQueue,
   NextRayQueue(wavefrontDepth));
```

In order to discuss the responsibilities of IntersectClosest() implementations, we will focus on its implementation in CPUAggregate. Rather than once again repeating the unwieldy list of arguments here, we will proceed directly to the method implementation, which starts with a parallel for loop over the items in the queue.

〈CPUAggregate::IntersectClosest() method implementation〉 ≡ ParallelFor(0, rayQueue->Size(), [=] (int index) { const RayWorkItem r = (*rayQueue)[index]; 〈Intersect r’s ray with the scene and enqueue resulting work 952〉 });

A regular aggregate stored in CPUAggregate handles the ray intersection test, with different cases afterward for rays that have an intersection with a surface and rays that do not. 〈Intersect r’s ray with the scene and enqueue resulting work〉 ≡ 952

```rust
   pstd::optional<ShapeIntersection> si = aggregate.Intersect(r.ray); if (!si)
   EnqueueWorkAfterMiss(r, mediumSampleQueue, escapedRayQueue); else
        EnqueueWorkAfterIntersection(r, r.ray.medium, si->tHit, si->intr,
        mediumSampleQueue, nextRayQueue, hitAreaLightQueue,
        basicEvalMaterialQueue, universalEvalMaterialQueue);
```

The details of enqueuing further work for rays that have no intersections are handled by the EnqueueWorkAfterMiss() function, which is defined in the file wavefront/intersect.h. That header file provides a number of functions that are used by both the CPU- and GPU-based ray intersection code. Gathering them there allows a single implementation to be used by both, which in turn reduces the complexity of those WavefrontAggregate implementations. Unlike many of the other functions in wavefront/intersect.h, EnqueueWorkAfterMiss() is simple enough that it barely merits its own function—if the ray is passing through participating media, it is enqueued for medium sampling, and otherwise it is enqueued for evaluating infinite light sources’ contribution to its radiance, if there are any in the scene.

〈Wavefront Ray Intersection Enqueuing Functions〉 ≡ void EnqueueWorkAfterMiss(RayWorkItem r, MediumSampleQueue *mediumSampleQueue, EscapedRayQueue *escapedRayQueue) { if (r.ray.medium) mediumSampleQueue->Push(r, Infinity); else if (escapedRayQueue) escapedRayQueue->Push(r); }

For rays that do intersect a surface, there is more to be done. The EnqueueWorkAfterInter section() function, which is not included in the text, handles all the following details. If the ray is passing through participating media, it is enqueued for medium sampling. Only if the ray is not absorbed or scattered in the medium sampling kernel is work then queued for the surface intersection to be processed. If a ray with an associated intersection is not scattered or absorbed by participating media and hits an emissive surface, it is added to a queue so that the surface’s scattered radiance will be added to the ray’s radiance estimate. Rays hitting surfaces are also sorted by the surfaces’ materials and the complexity of their textures into basicEvalMaterialQueue or universalEvalMaterialQueue, both of which are MultiWorkQueues. (Section 15.3.9 describes how the material queues are organized.) Finally, rays that hit surfaces with no materials that represent medium transitions are pushed on to the nextRayQueue to be continued in the next iteration, on the other side of the surface intersection with their medium member variable updated accordingly.

GPU Ray Intersections Following our custom of not including platform-specific code in the book text, we will not discuss the details of pbrt’s use of CUDA and the OptiX API for its GPU ray-tracing implementation, but we will summarize the abstractions currently used for GPU ray tracing. See the files gpu/optix.h and gpu/optix.cu for details, however.

Current GPU ray intersection APIs follow a different model than the CPU-focused accelerators in Chapter 7. Those accelerators provide a fully procedural model, where the user calls functions that take a single ray and

execute synchronously before returning their results. GPU ray tracing is based on a programmable pipeline, where some functionality is provided by the GPU vendor (either in hardware or in software), and some is provided by the user in the form of code that is executed at particular points in the pipeline. The user-supplied code is in the form of a series of shaders, each of which is a function that is invoked in specific cases. In practice, many instances of these shaders run concurrently, following the GPU’s thread group execution model. Ray tracing starts with a ray generation shader that is responsible for generating a ray and submitting it to the GPU ray-tracing function. In pbrt’s implementation, the ray generation shader retrieves the ray from the RayQueue.

```text
   EscapedRayQueue 955
   EscapedRayQueue::Push() 955
   Infinity 361
   MultiWorkQueue 937
   Ray::medium 95
   RayWorkItem 947
   RayWorkItem::ray 947
```

GPUs currently only have native support for intersecting rays with triangles. If a scene has no other types of shapes, then the intersection tests are handled entirely by the GPU’s ray-tracing implementation. For other types of shapes, custom intersection shaders can be provided by the user. The user specifies a shape’s axis-aligned bounding box and associates an intersection shader with it. When a ray intersects that box, the intersection shader is invoked to determine if there is an intersection. If there is, both the parametric t value along the ray and a handful of additional user-defined values can be returned to associate with the intersection. pbrt uses custom intersection shaders for all the quadric shapes as well as for the Bilinear Patch. All follow the same pattern. For example, for bilinear patches, the intersection shader calls the previously defined IntersectBilinearPatch() function. Recall that in the event of an intersection, it returns a BilinearIntersection, not a full SurfaceInteraction. This design is intentional, as a BilinearIntersection is much smaller—just 3 Floats. Returning a small

representation of the intersection is beneficial for performance, as it reduces how much information must be written to memory. Alpha testing adds an additional complication to intersection testing (recall the discussion of alpha textures in Section 7.1.1). An any hit shader is applied to any primitives with alpha textures, such as leaves with alpha masks. The any hit shader executes for all intersections with such primitives, before it is known if an intersection is the closest. Our implementation uses the any hit shader to evaluate the alpha texture and apply a stochastic test, just like the 〈Possibly ignore intersection based on stochastic alpha test〉 fragment in the GeometricPrimitive class used by the CPU. If the test fails, then the GPU is instructed to ignore the intersection completely. Once ray intersection testing has been completed, one of two shaders is invoked. For rays that have no intersections, a miss shader is called. Otherwise, a closest hit shader is invoked for processing at the intersection point. Now a full SurfaceInteraction is needed. All the shapes that are handled with custom intersection shaders have a method that converts their compact intersection representation to a SurfaceInteraction; for example, it is BilinearPatch::InteractionFromIntersection() for bilinear patches. The GPU reports the barycentric coordinates of triangle intersections, which are sufficient for the Triangle::InteractionFromIntersection() method to do its work.

Given final intersections (or the determination that a ray does not intersect anything), additional work is enqueued using functions from wavefront/intersection.h, as is the case for the wavefront CPU ray-tracing aggregate.

### 15.3.7 PARTICIPATING MEDIA

In the interest of space, we will not walk through the code for the kernels launched by the medium sampling method, SampleMediumInteraction(). Algorithmically, it matches the corresponding code in VolPathIntegrator, so we will just summarize the queues and types of work involved.

```text
    BilinearIntersection 335
    BilinearPatch 328
```

```text
    BilinearPatch::InteractionFromIntersection() 335
    GeometricPrimitive 399
    IntersectBilinearPatch() 332
    Medium 714
    RayQueue 942
    SampleT_maj() 859
    SurfaceInteraction 138
    Triangle::InteractionFromIntersection() 310
    VolPathIntegrator 877
```

For any ray with a non-nullptr Medium, the ray intersection code enqueues a MediumSample WorkItem in the mediumSampleQueue. A first medium-related kernel processes all the entries on this queue. Its task is to call the ray medium’s SampleT_maj() method, adding emission at each sampled point before sampling one of absorption, real scattering, or null scattering. Absorption causes path termination; real scattering causes work to be added to another queue, mediumScatterQueue, that holds MediumScatterWorkItems; and null scattering causes medium sampling to continue. The path throughput and path sampling PDFs are updated along the way. In the end, if the ray is neither scattered nor absorbed, then work is added to queues in the same manner as for rays that are not passing through media in the closest hit shader. A second kernel is then launched to process all the medium scattering events in the medium ScatterQueue. The usual sampling process ensues: a light and then a point on it are sampled, path sampling PDFs are computed, and a shadow ray is enqueued. Next, the phase function is sampled to generate an indirect ray direction. Work for that ray is then added to the next wavefront depth’s RayQueue via a call to PushIndirectRay().

### 15.3.8 RAY-FOUND EMISSION

Two kernels handle rays that may add emission to their radiance estimates due to their interaction with emissive entities. The first processes rays that have left the scene and the second handles rays that intersect emissive surfaces. escapedRayQueue is only allocated if the scene has one or more infinite area lights; there is otherwise no work to be done for such rays.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

EscapedRayQueue *escapedRayQueue = nullptr;

Note that HandleEscapedRays() returns immediately if there is no queue and thus no infinite area lights, saving the cost of an unnecessary kernel launch in that case. 〈WavefrontPathIntegrator Method Definitions〉 +≡ void WavefrontPathIntegrator::HandleEscapedRays() { if (!escapedRayQueue) return; ForAllQueued(“Handle escaped rays“, escapedRayQueue, maxQueueSize, PBRT_CPU_GPU_LAMBDA (const EscapedRayWorkItem w) { 〈Compute weighted radiance for escaped ray 956〉 〈Update pixel radiance if ray’s radiance is nonzero 956〉 }); }

```text
   EscapedRayQueue 955
   EscapedRayWorkItem 955
   ForAllQueued() 936
   PBRT_CPU_GPU_LAMBDA 930
   Point3f 92
   RayQueue 942
   RayQueue::PushIndirectRay() 967
   RayWorkItem 947
   SampledWavelengths 173
   Vector3f 86
   WavefrontPathIntegrator::escapedRayQueue 955
   WavefrontPathIntegrator::maxQueueSize 940
   WavefrontPathIntegrator::mediumScatterQueue 954
   WorkQueue 935
```

EscapedRayQueue, not included here, is a WorkQueue of EscapedRayWorkItems. It provides a Push() method that takes a RayWorkItem and copies the values from it that are needed in the kernel.

〈EscapedRayWorkItem Definition〉 ≡ struct EscapedRayWorkItem { 〈EscapedRayWorkItem Public Members 955〉

};

The work item for escaped rays stores the ray origin, direction, and depth as well as its wavelengths. The ray’s associated pixel index makes it possible to add any found emission to its radiance estimate. 〈EscapedRayWorkItem Public Members〉 ≡ 955

Point3f rayo; Vector3f rayd; int depth; SampledWavelengths lambda; int pixelIndex;

The kernel’s implementation parallels the 〈Accumulate contributions from infinite light sources〉 fragment in the VolPathIntegrator, just using the information about the ray from the EscapedRayWorkItem.

〈Compute weighted radiance for escaped ray〉 ≡ 955

SampledSpectrum L(0.f); for (const auto &light : *infiniteLights) { if (SampledSpectrum Le = light.Le(Ray(w.rayo, w.rayd), w.lambda); Le) { 〈Compute path radiance contribution from infinite light 956〉 } }

The final result is then added to the ray’s associated PixelSampleState::L value, so long as L is nonzero. If it is zero, skipping the unnecessary update may not lead to fewer instructions being executed, given the GPU’s execution model, but it will save memory bandwidth, which can be just as important to performance. 〈Update pixel radiance if ray’s radiance is nonzero〉 ≡ 955

if (L) { L += pixelSampleState.L[w.pixelIndex]; pixelSampleState.L[w.pixelIndex] = L; }

For infinite area lights that are directly visible or are encountered through specular reflection, MIS is performed using only the unidirectional path PDF, since that is the only way the path could have been sampled. 〈Compute path radiance contribution from infinite light〉 ≡ 956

if (w.depth == 0 || w.specularBounce) { L += w.beta * Le / w.r_u.Average(); } else { 〈Compute MIS-weighted radiance contribution from infinite light 956〉 }

The path throughput beta is needed to weight the light’s contribution. Further, whether or not the previous bounce was due to specular reflection must be tracked. Note that this value only requires a single bit; using a full 32-bit int is wasteful. A more optimized implementation might save some bandwidth by stealing one of the bits from pixelIndex, which does not need all 32 of them. 〈EscapedRayWorkItem Public Members〉 +≡ 955

SampledSpectrum beta; int specularBounce;

If other types of scattering preceded the ray’s escape, MIS weights are computed using both the light and unidirectional sampling PDFs, following the same approach as is implemented in the 〈Add infinite light contribution using both PDFs with MIS〉 fragment in the VolPathIntegrator.

〈Compute MIS-weighted radiance contribution from infinite light〉 ≡ 956

LightSampleContext ctx = w.prevIntrCtx; Float lightChoicePDF = lightSampler.PMF(ctx, light); SampledSpectrum r_l = w.r_l * lightChoicePDF * light.PDF_Li(ctx, w.rayd, true); L += w.beta * Le / (w.r_u + r_l).Average();

```text
   EscapedRayWorkItem::beta 956
   EscapedRayWorkItem::depth 955
   EscapedRayWorkItem::lambda 955
   EscapedRayWorkItem::pixelIndex 955
   EscapedRayWorkItem::prevIntrCtx 957
   EscapedRayWorkItem::rayd 955
   EscapedRayWorkItem::rayo 955
   EscapedRayWorkItem::r_l 957
   EscapedRayWorkItem::r_u 957
   EscapedRayWorkItem::specularBounce 956
   Float 23
   Light::Le() 743
   Light::PDF_Li() 743
   LightSampleContext 741
   LightSampler::PMF() 782
   PixelSampleState::L 946
   Ray 95
   SampledSpectrum 171
   SampledSpectrum::Average() 172
   VolPathIntegrator 877
   WavefrontPathIntegrator::infiniteLights 939
   WavefrontPathIntegrator::pixelSampleState 940
```

In order to compute MIS weights, the EscapedRayWorkItem must provide not only rescaled path probabilities but also geometric information about the previous path scattering vertex. 〈EscapedRayWorkItem Public Members〉 +≡ 955

SampledSpectrum r_u, r_l; LightSampleContext prevIntrCtx;

The second kernel handles rays that intersect emissive surfaces. 〈WavefrontPathIntegrator Method Definitions〉 +≡ void WavefrontPathIntegrator::HandleEmissiveIntersection() { ForAllQueued(“Handle emitters hit by indirect rays“, hitAreaLightQueue, maxQueueSize, PBRT_CPU_GPU_LAMBDA (const HitAreaLightWorkItem w) { 〈Find emitted radiance from surface that ray hit 957〉 〈Compute area light’s weighted radiance contribution to the path 958〉 〈Update L in PixelSampleState for area light’s radiance 958〉

}); }

The HitAreaLightQueue stores HitAreaLightWorkItems. 〈HitAreaLightQueue Definition〉 ≡ using HitAreaLightQueue = WorkQueue<HitAreaLightWorkItem>; 〈WavefrontPathIntegrator Member Variables〉 +≡ 939

HitAreaLightQueue *hitAreaLightQueue = nullptr; 〈HitAreaLightWorkItem Definition〉 ≡ struct HitAreaLightWorkItem { 〈HitAreaLightWorkItem Public Members 958〉 };

```text
   DiffuseAreaLight 759
   EscapedRayWorkItem 955
   ForAllQueued() 936
   HitAreaLightQueue 957
   HitAreaLightWorkItem 957
```

```text
   HitAreaLightWorkItem::areaLight 958
   HitAreaLightWorkItem::lambda 958
   HitAreaLightWorkItem::n 958
   HitAreaLightWorkItem::p 958
   HitAreaLightWorkItem::uv 958
   HitAreaLightWorkItem::wo 958
   Light 740
   Light::L() 743
   LightSampleContext 741
   PBRT_CPU_GPU_LAMBDA 930
   SampledSpectrum 171
   WavefrontPathIntegrator::hitAreaLightQueue 957
   WavefrontPathIntegrator::maxQueueSize 940
   WorkQueue 935
```

The first step in the kernel is to compute the emitted radiance at the ray’s intersection point. If there is none, the kernel can return immediately. Note that it thus could be beneficial if a Light method was added that did a quick conservative test for this case. Such a method could make it possible not to pay the cost of enqueuing work for cases such as a one-sided light source that was intersected on its non-emissive side. With the current implementation, that case is detected only here, costing both bandwidth for the queue work and execution divergence from the threads that return. (Such a method should be simple, deferring more complex tasks like performing texture lookups for surfaces that use image maps to modulate their emission, in order not to harm performance.) We also note that the call to Light::L() would be a potential source of execution divergence if pbrt had more than one Light implementation that could be used for emissive surfaces. Currently, there is only DiffuseAreaLight, so this is not a concern, but if there were more, it might be worthwhile to have a separate queue for each type of area light in order to avoid this divergence. 〈Find emitted radiance from surface that ray hit〉 ≡ 957

SampledSpectrum Le = w.areaLight.L(w.p, w.n, w.uv, w.wo, w.lambda); if (!Le) return;

The following HitAreaLightWorkItem member variables provide the information necessary to compute the emitted radiance from the intersection point back along the ray. 〈HitAreaLightWorkItem Public Members〉 ≡ 957

```rust
   Light areaLight; Point3f p; Normal3f n; Point2f uv; Vector3f wo;
   SampledWavelengths lambda;
```

The final path contribution is found similarly to how it is for escaped rays and infinite area lights. 〈Compute area light’s weighted radiance contribution to the path〉 ≡ 957

SampledSpectrum L(0.f); if (w.depth == 0 || w.specularBounce) { L = w.beta * Le / w.r_u.Average(); } else { 〈Compute MIS-weighted radiance contribution from area light〉 }

Once again, the specularBounce member could be packed in elsewhere in order to save storage and reduce bandwidth requirements. 〈HitAreaLightWorkItem Public Members〉 +≡ 957

int depth; SampledSpectrum beta, r_u, r_l; LightSampleContext prevIntrCtx; int specularBounce; int pixelIndex;

We will not include the 〈Compute MIS-weighted radiance contribution from area light〉 fragment here, which closely parallels the corresponding case in the VolPathIntegrator as well as the earlier fragment 〈Compute MIS-weighted radiance contribution from infinite light〉. The final weighted radiance value is then accumulated in the ray’s PixelSampleState.

〈Update L in PixelSampleState for area light’s radiance〉 ≡ 957

L += pixelSampleState.L[w.pixelIndex]; pixelSampleState.L[w.pixelIndex] = L;

The queues for both of these kernels need to be reset at the start of each ray depth iteration, so we will add the corresponding Reset() calls to the queue-resetting fragment defined earlier. 〈Reset queues before tracing next batch of rays〉 ≡ 948

if (escapedRayQueue) escapedRayQueue->Reset(); hitAreaLightQueue->Reset();

```text
   HitAreaLightWorkItem::beta 958
   HitAreaLightWorkItem::depth 958
   HitAreaLightWorkItem::pixelIndex 958
   HitAreaLightWorkItem::r_u 958
   HitAreaLightWorkItem::specularBounce 958
```

```text
    Light 740
    LightSampleContext 741
    Normal3f 94
    PixelSampleState 940
    Point2f 92
    Point3f 92
    SampledSpectrum 171
    SampledSpectrum::Average() 172
    SampledWavelengths 173
    Vector3f 86
    VolPathIntegrator 877
    WavefrontPathIntegrator::escapedRayQueue 955
    WavefrontPathIntegrator::hitAreaLightQueue 957
    WavefrontPathIntegrator::pixelSampleState 940
    WorkQueue::Reset() 935
```

Having seen these two kernels, it is fair to ask: why not handle these cases immediately in the ray intersection and medium sampling kernels, rather than incurring the bandwidth costs of queuing up work there and then consuming it here? This is yet another trade-off of bandwidth versus execution convergence. Doing this work in separate kernels for only the cases where it is required means that the kernels both start execution fully converged, with all threads doing useful work. In the intersection and medium scattering kernels, we would generally expect that only a subset of the rays would leave the scene or intersect emissive surfaces. In that case, we would have control divergence and all rays in the thread group that did not intersect an emissive surface would incur a performance cost if even one of the others did. The optimal trade-off depends on both the complexity of the computation to be done in those cases and the amount of bandwidth offered by the GPU.

### 15.3.9 SURFACE SCATTERING

With the WavefrontPathIntegrator, the majority of rendering time is usually spent in the kernels responsible for surface scattering. These kernels are specialized by the surfaces’ materials and, in turn, the type of BxDF that each material uses for the BSDF it returns. Starting from a ray–shape intersection, a surface-scattering kernel handles everything from normal and bump mapping to material evaluation, light and BSDF sampling, and queuing up shadow and indirect rays for later processing.

All of this starts with the Render() method calling EvaluateMaterialsAndBSDFs(), which is implemented in wavefront/surfscatter.cpp. With this method’s implementation, we encounter a new idiom that orchestrates the kernel launches: a call to ForEachType(). In its use here, that function iterates over all of the types that a Material could be and calls a callback function for each one of them. Thus, if pbrt is extended with an additional material, adding that one to the list of materials that Material passes to the TaggedPointer that it inherits from in its declaration in base/material.h automatically leads to a specialized kernel for that material being generated here. 〈WavefrontPathIntegrator Surface Scattering Methods〉 ≡ void WavefrontPathIntegrator::EvaluateMaterialsAndBSDFs( int wavefrontDepth) { ForEachType(EvaluateMaterialCallback{wavefrontDepth, this}, Material::Types()); }

A lambda function is not sufficient to pass to ForEachType(), as it does not pass a value of the given type to the callback but instead invokes its function call operator, passing the type for use in a template specialization. Therefore, we wrap the values needed for the material evaluation method call in a small structure. 〈EvaluateMaterialCallback Definition〉 ≡ struct EvaluateMaterialCallback { int wavefrontDepth; WavefrontPathIntegrator *integrator; 〈EvaluateMaterialCallback Public Methods 959〉 };

ForEachType() invokes the following method for each type of material. We skip over the MixMaterial here, since all instances of it are resolved to one of the other material types before being enqueued for the surface-shading kernels. (See Section 10.5.1 for a discussion of this detail of MixMaterial’s usage.)

〈EvaluateMaterialCallback Public Methods〉 ≡ 959

template <typename ConcreteMaterial> void operator()() {

```rust
        if constexpr (!std::is_same_v<ConcreteMaterial, MixMaterial>)
        integrator->EvaluateMaterialAndBSDF<ConcreteMaterial>(wavefrontDepth);
        }
```

```text
   BSDF 544
   BxDF 538
   EvaluateMaterialCallback 959
   ForEachType() 1073
   Material 674
   MixMaterial 681
   TaggedPointer 1073
   TaggedPointer::Types 1073
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::EvaluateMaterialAndBSDF() 960
```

The EvaluateMaterialAndBSDF() method does not yet bring us to the point of launching kernels—two considerations are handled beforehand. First, the implementation skips launching the specialized EvaluateMaterialAndBSDF() method for any material types that are not present in the scene. Such work queues will have no entries, so there is no reason to bother with them. This is handled using two arrays, haveBasicEvalMaterial and haveUniversalEvalMaterial, that are initialized in the WavefrontPathIntegrator constructor. (The two further distinguish the materials based on the complexity of their textures, which is a topic that will be discussed immediately after the following fragment.) Both arrays are indexed using the TaggedPointer TypeIndex() method, which returns an integer index for each representable type. 〈WavefrontPathIntegrator Surface Scattering Methods〉 +≡ template <typename ConcreteMaterial> void WavefrontPathIntegrator::EvaluateMaterialAndBSDF(int wavefrontDepth) { int index = Material::TypeIndex<ConcreteMaterial>(); if (haveBasicEvalMaterial[index]) EvaluateMaterialAndBSDF<ConcreteMaterial, BasicTextureEvaluator>( basicEvalMaterialQueue, wavefrontDepth); if (haveUniversalEvalMaterial[index

```rust
                                     ])
                                     EvaluateMaterialAndBSDF<Concrete
                                     Material,
                                     UniversalTextureEvaluator>(
                                     universalEvalMaterialQueue,
                                     wavefrontDepth); }
```

The WavefrontPathIntegrator maintains two work queues for materials, partitioning them based on the complexity of their textures. Each queue is a MultiWorkQueue with one entry for each material type. 〈WavefrontPathIntegrator Member Variables〉 +≡ 939

MaterialEvalQueue *basicEvalMaterialQueue = nullptr; MaterialEvalQueue *universalEvalMaterialQueue = nullptr;

These give two more queues to add to the ones that are reset at the start of tracing rays at each wavefront depth. 〈Reset queues before tracing next batch of rays〉 +≡ 948

basicEvalMaterialQueue->Reset(); universalEvalMaterialQueue->Reset();

Before continuing into the EvaluateMaterialAndBSDF() template specialization, we will detour to discuss texture evaluation in the wavefront integrator in more detail. Recall that when the Material interface was introduced in Section 10.5, it included the notion of a TextureEvaluator. Methods like GetBxDF() and GetBSSRDF() were templated on this type, took an instance of it as a parameter, and used it to evaluate textures rather than calling their Evaluate() methods directly.

There was no point in doing that for CPU rendering: there, a UniversalTextureEvaluator is always used. It immediately forwards texture evaluation requests on to the textures. pbrt’s full set of textures spans a wide range of complexity, however, ranging from trivial constant textures that return a fixed value to complex textures that evaluate noise functions to ones like SpectrumMixTexture that themselves recursively evaluate other textures.

```text
   BasicTextureEvaluator 961
   Material 674
```

```text
   Material::GetBSSRDF() 676
   Material::GetBxDF() 674
   MaterialEvalQueue 962
   MultiWorkQueue 937
   MultiWorkQueue::Reset() 938
   SpectrumMixTexture 659
   TaggedPointer 1073
   TaggedPointer::TypeIndex() 1074
   TextureEvaluator 676
   UniversalTextureEvaluator 677
   WavefrontPathIntegrator 939
   WavefrontPathIntegrator::basicEvalMaterialQueue 960
   WavefrontPathIntegrator::haveBasicEvalMaterial 960
   WavefrontPathIntegrator::haveUniversalEvalMaterial 960
   WavefrontPathIntegrator::universalEvalMaterialQueue 960
```

Not only do more complex textures require more registers on the GPU, but the potential for unbounded recursion from the mixture textures requires that the compiler allocate resources to be prepared for that case. In turn, the performance of evaluating the simpler textures can be harmed due to choices the compiler has made for the more complex ones. Because the simpler textures are common, it is thus worthwhile to separate materials according to the complexity of their textures and to have separate kernels for the materials that only use the simpler ones.8 Doing so can give further benefits from reducing control flow divergence. The following BasicTextureEvaluator class helps with this task.

〈BasicTextureEvaluator Definition〉 ≡ class BasicTextureEvaluator { public: 〈BasicTextureEvaluator Public Methods 961〉 };

We will categorize constant textures and plain image map textures as “basic.” Thus, the BasicTextureEvaluator’s CanEvaluate() method iterates over all provided textures and checks that each is one of those types. When material evaluation work is to be enqueued in the EnqueueWorkAfterIntersection() function, the material’s Material::CanEvaluate Textures() method is called with a BasicTextureEvaluator to determine whether the work is valid for the

basic material evaluation queues. If not, it goes on the appropriate universalEval MaterialQueue.

〈BasicTextureEvaluator Public Methods〉 ≡ 961

bool CanEvaluate(std::initializer_list<FloatTexture> ftex, std::initializer_list<SpectrumTexture> stex) const { 〈Return false if any FloatTextures cannot be evaluated 961〉 〈Return false if any SpectrumTextures cannot be evaluated〉 return true; }

The texture types are easily checked with the TaggedPointer::Is() method. (The scene initialization code creates instances of GPUFloatImageTexture in place of FloatImageTextures when GPU rendering is being used. That class uses platform-specific functionality to perform filtered texture look-ups more efficiently than executing Image methods would. GPUSpectrum ImageTexture follows equivalently.)

〈Return false if any FloatTextures cannot be evaluated〉 ≡ 961

for (FloatTexture f : ftex) if (f && !f.Is<FloatConstantTexture>() && !f.Is<FloatImageTexture>() && !f.Is<GPUFloatImageTexture>()) return false;

The corresponding fragment for SpectrumTextures is equivalent and is therefore not included here. The implementation of the TextureEvaluator evaluation method is key to the efficiency benefits provided by this approach. It would do no good to sort materials based on their textures but to then use pbrt’s regular dynamic dispatch mechanism for texture evaluation: in that case, the compiler would have no insight into the fact that the texture being passed to it must be one of the simple types accepted by CanEvaluate().

```text
   BasicTextureEvaluator 961
   EnqueueWorkAfterIntersection() 953
   FloatConstantTexture 656
   FloatImageTexture 661
   FloatTexture 656
   GPUFloatImageTexture 961
   GPUSpectrumImageTexture 961
   Image 1079
   Material::CanEvaluateTextures() 677
   SpectrumTexture 656
   TaggedPointer::Is() 1074
```

Therefore, the evaluation method instead tries casting the texture to each of the supported types until it finds the correct one. It then calls the corresponding evaluation method directly. In this way, the compiler can easily tell that the only texture evaluation methods that might be called are those for FloatConstantTexture, FloatImageTexture, and GPUFloatImageTexture, and it does not need to worry about resource allocation for evaluating the more complex texture types. 〈BasicTextureEvaluator Public Methods〉 +≡ 961

Float operator()(FloatTexture tex, TextureEvalContext ctx) { if (tex.Is<FloatConstantTexture>()) return tex.Cast<FloatConstantTexture>()->Evaluate(ctx); else if (tex.Is<FloatImageTexture>()) return tex.Cast<FloatImageTexture>()->Evaluate(ctx); else if (tex.Is<GPUFloatImageTexture>()) return tex.Cast<GPUFloatImageTexture>()->Evaluate(ctx); else return 0.f; }

The evaluation method for spectrum textures is similar and therefore not included here. With the motivation for the BasicTextureEvaluator explained, it is possible to better understand why ImageTextureBase in Section 10.4.1 offers scale and invert parameters even though scale and mix textures could be used to achieve the same results. With that capability directly available in textures that can be evaluated by the BasicTextureEvaluator, the UniversalTextureEvaluator can be invoked less often. pbrt actually has a texture rewriting pass that, for example, converts a scale texture with a constant scale applied to an image texture to just an image texture, configured to apply that scale itself. (See, for example, the SpectrumScaledTexture::Create() method implementation for details.) The following definition of the MaterialEvalQueue type is admittedly complex. However, it is another key to pbrt’s extensibility. For material and BSDF evaluation, we would like to have a MultiWorkQueue where there is a separate queue for each type of Material that pbrt supports, where the type of the work items is the template class MaterialEvalWorkItem, specialized with each material type. While we could enumerate all the currently supported materials in template parameters, to do so would mean that adding a new material to the system

would require editing an obscure part of the wavefront integrator implementation for it to be available there as well. Therefore, we go through some gymnastics in order to define the type of the MultiWorkQueue automatically. First, the TaggedPointer type that Material inherits from includes a type declaration, Types, that holds a TypePack of all types that the pointer can represent. We then use the MapType functionality from Section B.4.3 to wrap each material type inside the forthcoming MaterialEvalWorkItem template class. This gives us the final TypePack of types to provide to MultiWorkQueue.

〈MaterialEvalQueue Definition〉 ≡ using MaterialEvalQueue = MultiWorkQueue<typename MapType<MaterialEvalWorkItem, typename Material::Types>::type>;

〈MaterialEvalWorkItem Definition〉 ≡ template <typename ConcreteMaterial> struct MaterialEvalWorkItem { 〈MaterialEvalWorkItem Public Methods〉 〈MaterialEvalWorkItem Public Members 963〉 };

```text
   BasicTextureEvaluator 961
   Float 23
   FloatConstantTexture 656
   FloatConstantTexture::Evaluate() 656
   FloatImageTexture 661
   FloatTexture 656
   GPUFloatImageTexture 961
   ImageTextureBase 661
   MapType 1072
   Material 674
   MaterialEvalWorkItem 962
   MultiWorkQueue 937
   TaggedPointer 1073
   TaggedPointer::Cast() 1074
   TaggedPointer::Is() 1074
   TaggedPointer::Types 1073
   TextureEvalContext 650
```

```text
   TypePack 1071
   UniversalTextureEvaluator 677
```

It is crucial to have a pointer to the material in MaterialEvalWorkItem. Note that this can be a pointer to a concrete material type such as DiffuseMaterial due to MaterialEvalWorkItem being a template class on the ConcreteMaterial type. (We will introduce the rest of the member variables of the MaterialEvalWorkItem as they are used in code in the remainder of the section.) 〈MaterialEvalWorkItem Public Members〉 ≡ 962

const ConcreteMaterial *material;

With this context in hand, we can proceed to the implementation of the EvaluateMaterial AndBSDF() method. It is parameterized both by the concrete type of material that is being evaluated and by a TextureEvaluator, which allows us to generate specializations based not only on material but also on the two types of TextureEvaluator.

〈WavefrontPathIntegrator Surface Scattering Methods〉 +≡ template <typename ConcreteMaterial, typename TextureEvaluator> void WavefrontPathIntegrator::EvaluateMaterialAndBSDF( MaterialEvalQueue *evalQueue, int wavefrontDepth) { 〈Get BSDF for items in evalQueue and sample illumination 963〉 }

After some initialization work that includes getting a pointer to the WorkQueue for material evaluation of ConcreteMaterial, the work items in the queue are processed in parallel. 〈Get BSDF for items in evalQueue and sample illumination〉 ≡ 963

〈Construct desc for material/texture evaluation kernel〉 RayQueue *nextRayQueue = NextRayQueue(wavefrontDepth); auto queue = evalQueue->Get<MaterialEvalWorkItem<ConcreteMaterial>>(); ForAllQueued(desc.c_str(), queue, maxQueueSize, PBRT_CPU_GPU_LAMBDA (const MaterialEvalWorkItem<ConcreteMaterial> w) { 〈Evaluate material and BSDF for ray intersection 963〉 });

The structure of the material evaluation and sampling kernel parallels that of the surface-scattering–focused part of VolPathIntegrator::Li().

〈Evaluate material and BSDF for ray intersection〉 ≡ 963

TextureEvaluator texEval; 〈Compute differentials for position and (u, v) at intersection point 964〉 〈Compute shading normal if bump or normal mapping is being used 964〉 〈Get BSDF at intersection point 965〉 〈Regularize BSDF, if appropriate 966〉 〈Initialize VisibleSurface at first intersection if necessary〉 〈Sample BSDF and enqueue indirect ray at intersection point 966〉 〈Sample light and enqueue shadow ray at intersection point 968〉

One difference from the VolPathIntegrator is that the wavefront integrator does not carry ray differentials with the rays it traces. Therefore, in order to compute filter widths for texture filtering, the Camera’s Approximate_dp_dxy() method is used to compute approximate differentials at the intersection point. While the resulting filter width estimates are less accurate if there has been specular reflection or transmission from curved surfaces, they usually work well in practice. Note that by declaring local variables dpdu and dpdv at the end of this fragment that store the surface’s partial derivatives, we are able to reuse the earlier fragment 〈Estimate screen-space change in (u, v)〉 to compute the (u, v) texture derivatives.

```text
    DiffuseMaterial 678
    ForAllQueued() 936
    MaterialEvalQueue 962
    MaterialEvalWorkItem 962
    MultiWorkQueue::Get() 937
    PBRT_CPU_GPU_LAMBDA 930
    RayQueue 942
    TextureEvaluator 676
    VolPathIntegrator 877
    VolPathIntegrator::Li() 878
    WavefrontPathIntegrator 939
    WavefrontPathIntegrator::maxQueueSize 940
    WavefrontPathIntegrator::NextRayQueue() 943
    WorkQueue 935
```

〈Compute differentials for position and (u, v) at intersection point〉 ≡ 963

Vector3f dpdx, dpdy; Float dudx = 0, dudy = 0, dvdx = 0, dvdy = 0; camera.Approximate_dp_dxy(Point3f(w.pi), w.n, w.time, samplesPerPixel, &dpdx, &dpdy); Vector3f dpdu = w.dpdu, dpdv = w.dpdv; 〈Estimate screen-space change in (u, v) 641〉

MaterialEvalWorkItem also carries a variety of information about the local geometry of the ray–shape intersection that is copied from the SurfaceInteraction when material evaluation work is enqueued. 〈MaterialEvalWorkItem Public Members〉 +≡ 962

Point3fi pi; Normal3f n; Vector3f dpdu, dpdv; Float time; int depth;

Before any further work is done, the effect of normal or bump mapping on the local shading geometry is found, if appropriate. 〈Compute shading normal if bump or normal mapping is being used〉 ≡ 963

Normal3f ns = w.ns; Vector3f dpdus = w.dpdus; FloatTexture displacement = w.material->GetDisplacement(); const Image *normalMap = w.material->GetNormalMap(); if (normalMap) { 〈Call NormalMap() to find shading geometry 965〉 } else if (displacement) { 〈Call BumpMap() to find shading geometry〉 }

MaterialEvalWorkItem carries along information about the initial shading geometry from the intersected shape for a starting point for calls to NormalMap() or BumpMap() and for direct use if there is no normal or bump mapping at the intersection point. 〈MaterialEvalWorkItem Public Members〉 +≡ 962

Normal3f ns; Vector3f dpdus, dpdvs; Normal3f dndus, dndvs; Point2f uv;

As before, all work related to normal mapping is performed in the NormalMap() function and bump mapping is handled by BumpMap(). Note that here we are able to avoid dynamic dispatch in the calling of the material’s GetDisplacement() and GetNormalMap() methods thanks to ConcreteMaterial being a known specific material type. If a displacement texture or normal map is present, the NormalBumpEvalContext for the call to BumpMap() comes from a MaterialEvalWorkItem utility function that we do not include here; it just initializes a NormalBumpEvalContext using appropriate values from its member variables.

```text
   BumpMap() 687
   Camera::Approximate_dp_dxy() 638
   Float 23
   FloatTexture 656
   Image 1079
   Material::GetDisplacement() 678
   Material::GetNormalMap() 677
   MaterialEvalWorkItem 962
   MaterialEvalWorkItem::dpdu 964
   MaterialEvalWorkItem::dpdus 964
   MaterialEvalWorkItem::dpdv 964
   MaterialEvalWorkItem::material 963
   MaterialEvalWorkItem::n 964
   MaterialEvalWorkItem::ns 964
   MaterialEvalWorkItem::pi 964
   MaterialEvalWorkItem::time 964
   Normal3f 94
   NormalBumpEvalContext 685
   NormalMap() 685
   Point2f 92
   Point3f 92
   Point3fi 1061
   SurfaceInteraction 138
   Vector3f 86
   WavefrontPathIntegrator::camera 939
   WavefrontPathIntegrator::samplesPerPixel 939
```

On the GPU, there are a few ways that there may be execution divergence here. First, if some of the threads in the thread group have normal maps or displacement textures and some do not, then all of them will pay the cost of executing the NormalMap() and BumpMap() functions. Second, some threads may have displacement textures and others may have normal maps, which will lead to execution divergence. Finally, the types of the FloatTextures used for defining displacements may vary across the threads. Each of these factors could be used to sort work more finely, though we have not found this divergence to be too much of a problem in practice. 〈Call NormalMap() to find shading geometry〉 ≡ 964

NormalBumpEvalContext bctx = w.GetNormalBumpEvalContext(dudx, dudy, dvdx, dvdy); Vector3f dpdvs; NormalMap(*normalMap, bctx, &dpdus, &dpdvs); ns = Normal3f(Normalize(Cross(dpdus, dpdvs))); ns = FaceForward(ns, w.n);

```text
   BSDF 544
```

```text
   BumpMap() 687
   BxDF 538
   Cross() 91
   FaceForward() 94
   FloatTexture 656
   Material 674
   Material::GetBSDF() 675
   Material::GetBxDF() 674
   MaterialEvalContext 676
   MaterialEvalContext::dpdus 676
   MaterialEvalContext::ns 676
   MaterialEvalWorkItem::GetMaterialEvalContext() 965
   MaterialEvalWorkItem::GetNormalBumpEvalContext() 964
   MaterialEvalWorkItem::lambda 965
   MaterialEvalWorkItem::material 963
   MaterialEvalWorkItem::n 964
   MaterialEvalWorkItem::pixelIndex 966
   Normal3f 94
   NormalBumpEvalContext 685
   Normalize() 88
   NormalMap() 685
   PixelSampleState 940
   PixelSampleState::lambda 946
   SampledWavelengths 173
   SampledWavelengths::SecondaryTerminated() 174
   Sampler 469
   ScratchBuffer 1078
   Vector3f 86
   VolPathIntegrator 877
   WavefrontPathIntegrator::pixelSampleState 940
```

The fragment that calls BumpMap() for bump mapping parallels the one for normal mapping, so it is not included here. The BSDF is initialized here in a different way than in the VolPathIntegrator. Because the EvaluateMaterialAndBSDF() method is specialized on the material type, and because Materials in pbrt must provide a type definition for their associated BxDF, it is possible to stack-allocate the BxDF here rather than use an instance of the ScratchBuffer class to allocate space for it. It is then possible to call the material’s GetBxDF() method directly rather than using dynamic dispatch through a call to Material::GetBSDF() to do so.

The benefit from avoiding use of the ScratchBuffer for BxDFs is significant: just as stack-allocating the concrete Sampler type in the camera ray generation and sample generation kernels made it possible for the Sampler’s member variables to be stored in GPU registers, this approach does the same for the BxDF, giving a substantial performance benefit compared to storing them in device memory. The MaterialEvalWorkItem GetMaterialEvalContext() method, which is also not included here, initializes a MaterialEvalContext from its corresponding member variables. 〈Get BSDF at intersection point〉 ≡ 963

SampledWavelengths lambda = w.lambda; MaterialEvalContext ctx = w.GetMaterialEvalContext(dudx, dudy, dvdx, dvdy, ns, dpdus); using ConcreteBxDF = typename ConcreteMaterial::BxDF; ConcreteBxDF bxdf = w.material->GetBxDF(texEval, ctx, lambda); BSDF bsdf(ctx.ns, ctx.dpdus, &bxdf); 〈Handle terminated secondary wavelengths after BSDF creation 965〉 〈MaterialEvalWorkItem Public Members〉 +≡ 962

SampledWavelengths lambda;

The call to GetBxDF() above may cause secondary wavelengths to be terminated—for example, in case of dispersion. It is therefore important that the stack-allocated lambda value both be used here in this kernel and also be passed along to subsequent rendering kernels, rather than the initial SampledWavelengths value from the MaterialEvalWorkItem. If the secondary wavelengths in lambda have been terminated, the copy of the SampledWavelengths for the path in PixelSampleState must be updated. Note that the implementation here may end up writing lambda multiple times redundantly at subsequent intersections in that case. 〈Handle terminated secondary wavelengths after BSDF creation〉 ≡ 965

if (lambda.SecondaryTerminated()) pixelSampleState.lambda[w.pixelIndex] = lambda;

〈MaterialEvalWorkItem Public Members〉 +≡ 962

int pixelIndex;

BSDF regularization proceeds as before, only happening if the option has been enabled and a ray has undergone a non-specular scattering event. 〈Regularize BSDF, if appropriate〉 ≡ 963

```rust
   if (regularize && w.anyNonSpecularBounces) bsdf.Regularize();
```

The anyNonSpecularBounces member variable is yet another single-bit quantity taking up 32 bits. A more bandwidth-efficient implementation would pack this value into a free bit elsewhere in the MaterialEvalWorkItem.

〈MaterialEvalWorkItem Public Members〉 +≡ 962

int anyNonSpecularBounces;

We will omit the fragment that initializes the VisibleSurface at the first intersection, as it is not especially interesting or different than the corresponding code in the CPU-based rendering path. There are two new things to see in the fragment that samples the BSDF. First, the sample values used come from the RaySamples object rather than from Sampler method calls. Second, because the ConcreteBxDF type is known here at compile time, it is possible to call the templated BSDF::Sample_f() variant that takes the BxDF type and thus avoids dynamic dispatch, calling the appropriate BxDF method implementation directly. 〈Sample BSDF and enqueue indirect ray at intersection point〉 ≡ 963

Vector3f wo = w.wo; RaySamples raySamples = pixelSampleState.samples[w.pixelIndex]; pstd::optional<BSDFSample> bsdfSample = bsdf.Sample_f<ConcreteBxDF>(wo, raySamples.indirect.uc, raySamples.indirect.u); if (bsdfSample) { 〈Compute updated path throughput and PDFs and enqueue indirect ray 966〉 } 〈MaterialEvalWorkItem Public Members〉 +≡ 962

Vector3f wo;

The path throughput and rescaled path probabilities are updated in the same manner as in the VolPathIntegrator 〈Update volumetric integrator path state after surface scattering〉 fragment. 〈Compute updated path throughput and PDFs and enqueue indirect ray〉 ≡ 966

Vector3f wi = bsdfSample->wi; SampledSpectrum beta = w.beta * bsdfSample->f * AbsDot(wi, ns)/bsdfSample->pdf; SampledSpectrum r_u = w.r_u, r_l; 〈Update r_u based on BSDF sample PDF 967〉 〈Update etaScale accounting for BSDF scattering 967〉

```rust
   〈Apply Russian roulette to indirect ray based on weighted path throughput〉
   if (beta) {
       〈Initialize spawned ray and enqueue for next ray depth 967〉
   }
```

Only the unidirectional rescaled path probability needs to be passed in to this kernel, since the light path probability is initialized during its execution.

```text
   AbsDot() 90
   BSDF::Regularize() 842
   BSDF::Sample_f() 545
   BSDFSample 541
   BSDFSample::f 541
   BSDFSample::pdf 541
   BSDFSample::wi 541
   BxDF 538
   MaterialEvalWorkItem 962
   MaterialEvalWorkItem::anyNonSpecularBounces 966
   MaterialEvalWorkItem::beta 967
   MaterialEvalWorkItem::pixelIndex 966
   MaterialEvalWorkItem::r_u 967
   MaterialEvalWorkItem::wo 966
   PixelSampleState::samples 951
   RaySamples 950
   RaySamples::indirect::u 951
   RaySamples::indirect::uc 951
   SampledSpectrum 171
   Sampler 469
   Vector3f 86
   VolPathIntegrator 877
   WavefrontPathIntegrator::pixelSampleState 940
   WavefrontPathIntegrator::regularize 939
```

〈MaterialEvalWorkItem Public Members〉 +≡ 962

SampledSpectrum beta, r_u;

The logic for updating the unidirectional rescaled path probability also follows that of the VolPathIntegrator, including how proportional BSDF samples such as those from the LayeredBxDF are handled. The call to the BSDF::PDF() method presents another opportunity to avoid dynamic dispatch given a compile-time known BxDF, however. 〈Update r_u based on BSDF sample PDF〉 ≡ 966

```rust
   if (bsdfSample->pdfIsProportional) r_l = r_u / bsdf.PDF<ConcreteBxDF>(wo,
   bsdfSample->wi); else
       r_l = r_u / bsdfSample->pdf;
```

The etaScale factor used in Russian roulette is also updated in the same manner as before.

```text
   BSDF::PDF() 546
   BSDFSample::eta 541
   BSDFSample::IsSpecular() 541
   BSDFSample::IsTransmission() 541
   BSDFSample::pdf 541
   BSDFSample::pdfIsProportional 541
   Dot() 89
   Float 23
   LayeredBxDF 895
   LightSampleContext 741
   MaterialEvalWorkItem::anyNonSpecularBounces 966
   MaterialEvalWorkItem::depth 964
   MaterialEvalWorkItem::etaScale 967
   MaterialEvalWorkItem::mediumInterface 967
   MaterialEvalWorkItem::n 964
   MaterialEvalWorkItem::pi 964
   MaterialEvalWorkItem::pixelIndex 966
   MaterialEvalWorkItem::time 964
   MediumInterface 715
   Ray 95
   Ray::d 95
   Ray::medium 95
   RayQueue 942
   RayQueue::PushIndirectRay() 967
   RaySamples::indirect::rr 951
   SampledSpectrum 171
   SpawnRay() 382
   Sqr() 1034
   VolPathIntegrator 877
```

〈Update etaScale accounting for BSDF scattering〉 ≡ 966

Float etaScale = w.etaScale; if (bsdfSample->IsTransmission()) etaScale *= Sqr(bsdfSample->eta); 〈MaterialEvalWorkItem Public Members〉 +≡ 962

Float etaScale;

We will omit the 〈Apply Russian roulette to indirect ray based on weighted path throughput〉 fragment, which is almost exactly the same as the

VolPathIntegrator’s 〈Possibly terminate volumetric path with Russian roulette〉, other than the fact that in this case the sample value for the Russian roulette test comes from RaySamples.indirect.rr. For indirect rays, a ray is initialized and pushed on to the RayQueue for the next level of the ray tree. (We omit the implementation of the RayQueue::PushIndirectRay() method, which stores the provided values in the corresponding member variables.) 〈Initialize spawned ray and enqueue for next ray depth〉 ≡ 966

```rust
   Ray ray = SpawnRay(w.pi, w.n, w.time, wi); 〈Initialize ray medium if media are
   present 967〉
   bool anyNonSpecularBounces = !bsdfSample->IsSpecular() ||
                                         w.anyNonSpecularBounces; LightSampleContext
                                         ctx(w.pi, w.n, ns); nextRayQueue-
                                         >PushIndirectRay(
         ray, w.depth + 1, ctx, beta, r_u, r_l, lambda, etaScale, bsdfSample-
         >IsSpecular(), anyNonSpecularBounces, w.pixelIndex);
```

The medium member of the Ray must be set manually based on which side of the surface the ray starts in. 〈Initialize ray medium if media are present〉 ≡ 967,

```rust
   if (haveMedia) ray.medium = Dot(ray.d, w.n) > 0 ?                                   968
```

```rust
   w.mediumInterface.outside : w.mediumInterface.inside;
```

The MediumInterface at the intersection point is thus needed in MaterialEvalWorkItem.

〈MaterialEvalWorkItem Public Members〉 +≡ 962

MediumInterface mediumInterface;

Indirect rays require a few additions to the RayWorkItem class including the current path throughput β, rescaled path probabilities, and the information necessary to compute MIS weights if the ray encounters emission. 〈RayWorkItem Public Members〉 +≡ 947

SampledSpectrum beta, r_u, r_l; LightSampleContext prevIntrCtx; Float etaScale; int specularBounce; int anyNonSpecularBounces;

The direct lighting calculation follows a similar path as in the VolPathIntegrator.

〈Sample light and enqueue shadow ray at intersection point〉 ≡ 963

BxDFFlags flags = bsdf.Flags(); if (IsNonSpecular(flags)) { 〈Choose a light source using the LightSampler〉 〈Sample light source and evaluate BSDF for direct lighting〉 〈Compute path throughput and path PDFs for light sample 968〉 〈Enqueue shadow ray with tentative radiance contribution 968〉 }

We will omit the 〈Choose a light source using the LightSampler〉 and 〈Sample light source and evaluate BSDF for direct lighting〉 fragments, as they are essentially the same as corresponding fragments that we have seen in earlier integrators. The first fragment gives us the sampledLight variable that stores a SampledLight and the second calls Light::SampleLi() with that light, giving an ls variable that stores the LightLiSample.

The path throughput and path sampling weights are computed in the same way as they are in the VolPathIntegrator::SampleLd() method, though here it is again possible to use the templated BSDF::PDF() method that avoids the dynamic dispatch overhead to compute the BSDF’s PDF. 〈Compute path throughput and path PDFs for light sample〉 ≡ 968

SampledSpectrum beta = w.beta * f * AbsDot(wi, ns); Float lightPDF = ls->pdf * sampledLight->p; Float bsdfPDF = IsDeltaLight(light.Type()) ? 0.f : bsdf.PDF<ConcreteBxDF>(wo, wi); SampledSpectrum r_u = w.r_u * bsdfPDF; SampledSpectrum r_l = w.r_u * lightPDF;

We go ahead and compute Ld, which is the final weighted contribution that the shadow ray would give to the pixel sample’s radiance estimate if it is unoccluded. If the ray is unoccluded and there are participating media in the scene, Ld may still be reduced due to the effect of extinction along the shadow ray; that factor is handled when the shadow ray is traced. 〈Enqueue shadow ray with tentative radiance contribution〉 ≡ 968

SampledSpectrum Ld = beta * ls->L; Ray ray = SpawnRayTo(w.pi, w.n, w.time, ls->pLight.pi, ls->pLight.n); 〈Initialize ray medium if media are present 967〉 shadowRayQueue->Push(ShadowRayWorkItem{ray, 1 - ShadowEpsilon, lambda, Ld, r_u, r_l, w.pixelIndex});

```text
   AbsDot() 90
   BSDF::Flags() 544
```

```text
    BSDF::PDF() 546
    BxDFFlags 539
    BxDFFlags::IsNonSpecular() 539
    Float 23
    Interaction::n 137
    Interaction::pi 137
    IsDeltaLight() 741
    Light::SampleLi() 741
    Light::Type() 740
    LightLiSample 743
    LightLiSample::L 743
    LightLiSample::pdf 743
    LightLiSample::pLight 743
    LightSampleContext 741
    MaterialEvalWorkItem::beta 967
    MaterialEvalWorkItem::n 964
    MaterialEvalWorkItem::pi 964
    MaterialEvalWorkItem::pixelIndex 966
    MaterialEvalWorkItem::r_u 967
    MaterialEvalWorkItem::time 964
    Ray 95
    RayWorkItem 947
    SampledLight 782
    SampledLight::p 782
    SampledSpectrum 171
    ShadowEpsilon 383
    ShadowRayWorkItem 969
    SpawnRayTo() 382
    VolPathIntegrator 877
    VolPathIntegrator::SampleLd() 886
    WavefrontPathIntegrator::shadowRayQueue 969
    WorkQueue::Push() 936
```

### 15.3.10 SHADOW RAYS

The set of shadow rays to be traced after material evaluation and the direct lighting calculation in the EvaluateMaterialAndBSDF() method are stored in a ShadowRayQueue. 〈ShadowRayQueue Definition〉 ≡ using ShadowRayQueue = WorkQueue<ShadowRayWorkItem>;

A single ShadowRayQueue is maintained by the WavefrontPathIntegrator.

〈WavefrontPathIntegrator Member Variables〉 +≡ 939

ShadowRayQueue *shadowRayQueue = nullptr;

The work items in the queue store values that include the ray to be traced, its wavelengths, and tentative contribution Ld, as well as the rescaled path probabilities up to the ray’s origin. 〈ShadowRayWorkItem Definition〉 ≡ struct ShadowRayWorkItem { Ray ray; Float tMax; SampledWavelengths lambda; SampledSpectrum Ld, r_u, r_l; int pixelIndex; };

The WavefrontAggregate interface includes two methods for tracing shadow rays: Intersect Shadow(), which is called for scenes that have no participating media where only a binary visibility test is needed, and IntersectShadowTr() for scenes that do have media and where transmittance must be computed. A call to the WavefrontPathIntegrator::TraceShadowRays() method leads to a call of the appropriate method; the shadow ray queue is reset immediately afterward. 〈WavefrontAggregate Interface〉 +≡ 951

virtual void IntersectShadow(int maxRays, ShadowRayQueue *shadowRayQueue, SOA<PixelSampleState> *pixelSampleState) const = 0; virtual void IntersectShadowTr(int maxRays, ShadowRayQueue *shadowRayQueue, SOA<PixelSampleState> *pixelSampleState) const = 0;

The CPU implementation of IntersectShadow() processes items from the queue in parallel and calls its aggregate’s IntersectP() method to determine if each ray is occluded. A call to RecordShadowRayResult(), another utility function shared by both CPU and GPU aggregates that is defined in wavefront/intersect.h, takes care of the additional work to be done after the intersection test has been resolved. 〈CPUAggregate Method Definitions〉 ≡ void CPUAggregate::IntersectShadow( int maxRays, ShadowRayQueue *shadowRayQueue, SOA<PixelSampleState> *pixelSampleState) const { 〈Intersect shadow rays from shadowRayQueue in parallel 970〉

}

```text
   Float 23
   PixelSampleState 940
   Ray 95
   SampledSpectrum 171
   SampledWavelengths 173
   ShadowRayQueue 969
   ShadowRayWorkItem 969
   SOA 932
   WavefrontAggregate 951
   WavefrontPathIntegrator 939
   WorkQueue 935
```

〈Intersect shadow rays from shadowRayQueue in parallel〉 ≡ 969

```rust
   ParallelFor(0, shadowRayQueue->Size(), [=] (int index) {
                        const ShadowRayWorkItem w = (*shadowRayQueue)[index]; bool
                        hit = aggregate.IntersectP(w.ray, w.tMax);
                        RecordShadowRayResult(w, pixelSampleState, hit); });
```

If the ray was occluded, then no further work is necessary. Otherwise, the final MIS-weighted radiance is found by dividing by the two rescaled path probabilities and is then added to the running sum of the radiance estimate in the PixelSampleState for the ray. 〈Wavefront Ray Intersection Enqueuing Functions〉 +≡ void RecordShadowRayResult(const ShadowRayWorkItem w, SOA<PixelSampleState> *pixelSampleState, bool foundIntersection) { if (foundIntersection) return; SampledSpectrum Ld = w.Ld / (w.r_u + w.r_l).Average(); SampledSpectrum Lpixel = pixelSampleState->L[w.pixelIndex]; pixelSampleState->L[w.pixelIndex] = Lpixel + Ld; }

In the presence of participating media, there is more work to do for shadow rays. The computation proceeds just as in the latter two thirds of the VolPathIntegrator::SampleLd() method: the ray is successively intersected against the scene geometry, terminating if it hits an opaque surface. Otherwise a call to SampleT_maj() samples the medium, so that

transmittance can be estimated using ratio tracking just as in 〈Update transmittance for current ray segment〉 in that method. Given the close similarities, we will not include that code for the wavefront integrator here.

### 15.3.11 UPDATING THE FILM

Each pixel sample’s value is provided to the Film only after all of them are finished. One might wonder: why not push samples on to a queue whenever their ray paths terminate and add film samples sooner by periodically running a kernel that processes the items on the queue? There are two costs to such an approach: the first is that samples would be added to the film in an arbitrary order, so the accesses to values like PixelSampleState::L would be irregular across threads in thread groups, which could harm performance. The second cost is the unnecessary bandwidth that would be used to push work onto the queue and to process it. We know that every pixel sample taken at the start of tracing each path will eventually be added to the film, so there is no need to separately enumerate them. For both of these reasons, it is more efficient to use a plain GPUParallelFor() loop and process the PixelSampleState structures in order.

〈WavefrontPathIntegrator Film Methods〉 ≡ void WavefrontPathIntegrator::UpdateFilm() { ParallelFor(“Update film“, maxQueueSize, PBRT_CPU_GPU_LAMBDA (int pixelIndex) { 〈Check pixel against film bounds 971〉 〈Compute final weighted radiance value 971〉 〈Provide sample radiance value to film 971〉 }); }

```text
    Film 244
    GPUParallelFor() 929
    ParallelFor() 1107
    PBRT_CPU_GPU_LAMBDA 930
    PixelSampleState 940
    PixelSampleState::L 946
```

```text
   Primitive::IntersectP() 398
   RecordShadowRayResult() 970
   SampledSpectrum 171
   SampledSpectrum::Average() 172
   SampleT_maj() 859
   ShadowRayWorkItem 969
   ShadowRayWorkItem::Ld 969
   ShadowRayWorkItem::pixelIndex 969
   ShadowRayWorkItem::ray 969
   ShadowRayWorkItem::r_l 969
   ShadowRayWorkItem::r_u 969
   ShadowRayWorkItem::tMax 969
   SOA 932
   VolPathIntegrator::SampleLd() 886
   WavefrontPathIntegrator::maxQueueSize 940
   WavefrontPathIntegrator::ParallelFor() 941
   WorkQueue::Size() 935
```

Recall that the GenerateCameraRays() method sets the PixelSampleState::pPixel coordinates for all the threads, but that it then exits immediately for threads corresponding to extra scanlines beyond the end of the image. Therefore, we must perform the same check of the pixel coordinates against the film bounds here, and not do any further processing for such pixels. 〈Check pixel against film bounds〉 ≡ 970

Point2i pPixel = pixelSampleState.pPixel[pixelIndex]; if (!InsideExclusive(pPixel, film.PixelBounds())) return;

The final radiance value is scaled by the camera ray weight returned by the Camera before it is provided to the Film.9

〈Compute final weighted radiance value〉 ≡ 970

SampledSpectrum Lw = SampledSpectrum(pixelSampleState.L[pixelIndex]) * pixelSampleState.cameraRayWeight[pixelIndex];

A few more values read from pixelSampleState and we have everything that we need to call the Film’s AddSample() method. The implementation uses the initializeVisibleSurface member variable, set in the constructor, to distinguish between Film implementations that make use of the VisibleSurface and those that do not in order to save the memory bandwidth of reading it if it will not be used.

〈Provide sample radiance value to film〉 ≡ 970

SampledWavelengths lambda = pixelSampleState.lambda[pixelIndex]; Float filterWeight = pixelSampleState.filterWeight[pixelIndex]; if (initializeVisibleSurface) { 〈Call Film::AddSample() with VisibleSurface for pixel sample 971〉 } else film.AddSample(pPixel, Lw, lambda, nullptr, filterWeight);

One interesting detail is that the VisibleSurface must be loaded from SOA format into the regular VisibleSurface structure layout so that a pointer to it can be passed to the AddSample() method; a pointer to its current in-memory representation does not correspond to the layout that this method expects. 〈Call Film::AddSample() with VisibleSurface for pixel sample〉 ≡ 971

VisibleSurface visibleSurface = pixelSampleState.visibleSurface[pixelIndex]; film.AddSample(pPixel, Lw, lambda, &visibleSurface, filterWeight);

```text
   Bounds2::InsideExclusive() 100
   Camera 206
   Film 244
   Film::AddSample() 244
   Film::PixelBounds() 246
   Float 23
   GetSetIndirector 934
   PixelSampleState::cameraRayWeight 946
   PixelSampleState::filterWeight 946
   PixelSampleState::L 946
   PixelSampleState::lambda 946
   PixelSampleState::pPixel 945
   PixelSampleState::visibleSurface 946
   Point2i 92
   SampledSpectrum 171
   SampledWavelengths 173
   VisibleSurface 245
   WavefrontPathIntegrator::film 939
   WavefrontPathIntegrator::pixelSampleState 940
```

FURTHER READING

Purcell et al. (2002, 2003) and Carr, Hall, and Hart (2002) were the first to map general-purpose ray tracers to graphics processors.

A classic paper by Aila and Laine (2009) carefully analyzed the performance of ray tracing on contemporary GPUs and developed improved traversal algorithms based on their insights. Follow-on work by Laine et al. (2013) discussed the benefits of the wavefront architecture for rendering systems that support a wide variety of materials, textures, and lights. (The use of a wavefront approach for the path tracer described in this chapter is motivated by Laine et al.’s insights.) Most work in performance optimization for GPU ray tracers analyzes the balance between improving thread execution and memory convergence versus the cost of reordering work to do so.10 Influential early work includes Hoberock et al. (2009), who re-sorted a large number of intersection points to create coherent collections of work before executing their surface shaders. Novák et al. (2010) introduced path regeneration to start tracing new ray paths in threads that are otherwise idle due to ray termination. Wald (2011) and van Antwerpen (2011) both applied compaction, densely packing the active threads in thread groups. Lier et al. (2018b) considered the unconventional approach of distributing the work for a single ray across multiple GPU threads and showed performance benefits for incoherent rays. (This approach parallels how computation is often mapped to CPU SIMD units for high-performance ray tracing.) Reordering the rays to be traced can also improve performance by improving the coherence of memory accesses performed during intersection tests. Early work in this area was done by Garanzha and Loop (2010) and Costa et al. (2015). Meister et al. (2020) have recently examined ray reordering in the context of a GPU with hardware-accelerated intersection testing and found benefits from using it. An alternative to taking an arbitrary set of rays and finding structure in them is to generate rays that are inherently coherent in the first place. Examples include the algorithms of Szirmay-Kalos and Purgathofer (1998) and Hachisuka (2005), which select a single direction for all indirect rays at each level, allowing the use of a rasterizer with parallel projection to trace them. More generally, adding structure to the sample values used for importance sampling can lead to coherence in the rays that are traced. Keller and Heidrich (2001) developed interleaved sampling patterns that reuse sample values at separated pixels in order to trade off sample coherence and variation, and Sadeghi et al. (2009) investigated the

combination of interleaved sampling and using the same pseudo-random sequence at nearby pixels to increase ray coherence. Dufay et al. (2016) randomized samples using small random offsets so that nearby pixels still have similar sample values. Efficient GPU-based construction of acceleration structures is challenging due to the degree of parallelism required; there has been much research on this topic. See Zhou et al. (2008), Lauterbach et al. (2009), Pantaleoni and Luebke (2010), Garanzha et al. (2011), Karras and Aila (2013), Domingues and Pedrini (2015), and Vinkler et al. (2016) for techniques for building kd-trees and BVHs on GPUs. See also the “Further Reading” section in Chapter 7 for additional discussion of algorithms for constructing and traversing acceleration structures on the GPU. The relatively limited amount of on-chip memory that GPUs have can make it challenging to efficiently implement light transport algorithms that require more than a small amount of storage for each ray. (For example, even storing all the vertices of a pair of subpaths for a bidirectional path-tracing algorithm is much more than a thread could ask to keep onchip.) The paper by Davidovič et al. (2014) gives a thorough overview of these issues and previous work and includes a discussion of implementations of a number of sophisticated light transport algorithms on the GPU. Zellmann and Lang used compile time polymorphism in C++ to improve the performance of a GPU ray tracer (Zellmann and Lang 2017); our implementation in this chapter is based on similar ideas. Zhang et al. (2021) compared a number of approaches for dynamic function dispatch on GPUs and evaluated their performance. Fewer papers have been written about the design of full ray-tracing–based rendering systems on the GPU than on the CPU. Notable papers in this area include Pantaleoni et al.’s (2010) description of PantaRay, which was used to compute occlusion and lighting by Weta Digital, and Keller et al.’s (2017) discussion of the architecture of the Iray rendering system. Bikker and van Schijndel (2013) described Brigade, which targets path-traced games, balancing work between the CPU and GPU and adapting the workload to maintain the desired frame rate.

Ray-Tracing Hardware While all the stages of ray-tracing calculations—construction of the acceleration hierarchy, traversal of the hierarchy, and ray–primitive intersections, as well as shading, lighting, and integration calculations—can be implemented in software on GPUs, there has long been interest in designing specialized hardware for ray–primitive intersection tests and construction and traversal of the acceleration hierarchy for better performance. Deng et al.’s survey article has thorough coverage of hardware acceleration of ray tracing through 2017 (Deng et al. 2017); here, we will focus on early work and more recent developments. Early published work in this area includes a paper by Woop et al. (2005), who described the design of a “ray processing unit” (RPU). Aila and Karras (2010) described general architectural issues related to handling incoherent rays, as are common with global illumination algorithms. More recently, Shkurko et al. (2017) and Vasiou et al. (2019) have described a hardware architecture that is based on reordering ray intersection computation so that it exhibits predictable streaming memory accesses. Doyle et al. (2013) did early work on SAH BVH construction using specialized hardware. Viitanen et al. (2017, 2018) have done additional work in this area, designing architectures for efficient HLBVH construction for animated scenes and for high-quality SAH-based BVH construction. Imagination Technologies announced a mobile GPU that would use a ray-tracing architecture from Caustic (McCombe 2013), though it never shipped in volume. The NVIDIA Turing architecture (NVIDIA 2018) is the first GPU with hardware-accelerated ray tracing that has seen widespread adoption. The details of its ray-tracing hardware architecture are not publicly documented, though Sanzharov et al. (2020) have applied targeted benchmarks to measure its performance characteristics in order to develop hypotheses about its implementation. EXERCISES

```text
   SampledWavelengths 173
   WavefrontPathIntegrator 939
```

➊ 15.1 Modify soac so that the code it generates leaves objects in AOS layout in memory and recompile pbrt. (You will need to manually update a few places in the WavefrontPathIntegrator that only access a single field of a structure, as well.) How is performance affected by this change? ➋ 15.2 pbrt’s SampledWavelengths class stores two Floats for each wavelength: one for the wavelength value and one for its PDF. This class is passed along between almost all kernels. Render a scene on the GPU and work out an estimate of the amount of bandwidth consumed in communicating these values between kernels. (You may need to make some assumptions to do so.) Then, implement an alternative SOA representation for SampledWavelengths that stores only two values: the Float sample used to originally sample the wavelengths and a Boolean value that indicates whether the secondary wavelengths have been terminated. You might use the sign bit to encode the Boolean value, or you might even try a 16-bit encoding, with the [0, 1) sample value quantized to 15 bits and the 16th used to indicate termination. Write code to encode SampledWavelengths to this representation when they are pushed to a queue and to decode this representation back to SampledWavelengths when work is read from the queue via a call to Film::SampleWavelengths() and then possibly a call to SampledWavelengths::TerminateSecondary(). Estimate how much bandwidth your improved representation saves. How is runtime performance affected? Can you draw any conclusions about whether your GPU is memory or bandwidth limited when running these kernels? ➋ 15.3 The direct lighting code in the EvaluateMaterialsAndBSDFs() kernel may suffer from divergence in the Light::SampleLi() call if the scene has a variety of types of light source. Construct such a scene and then experiment with moving light sampling into a separate kernel, using a work queue to supply work to it and where the light samples are pushed on to a queue for the rest of the direct lighting computation. What is the effect on performance for your test scene? Is performance negatively impacted for scenes with just a single type of light? ➌ 15.4 Add support for ray differentials to the WavefrontPathIntegrator, including both generating them for camera rays and computing updated differentials for reflected and refracted rays. (You will likely want to repurpose the code in the implementation of the SurfaceInteraction SpawnRay() method in Section 10.1.3.) After ensuring that texture filtering results match pbrt running on the CPU, measure the performance impact of your changes. How much performance is lost from the bandwidth used in passing ray differentials between kernels? Do any kernels have better performance? If so, can you explain why?

Next, implement one of the more space-efficient techniques for representing derivative information with rays that are described by Akenine-Möller et al. (2019). How do performance and filtering quality compare to ray differentials? ➌ 15.5 The WavefrontPathIntegrator’s performance can suffer from scenes with very high maximum ray depths when there are few active rays remaining at high depths and, in turn, insufficient parallelism for the GPU to reach its peak capabilities. One approach to address this problem is path regeneration, which was described by Novák et al. (2010).

```text
             Film 244
             Film::SampleWavelengths() 246
             Light::SampleLi() 741
             RayQueue 942
             SampledWavelengths 173
             SampledWavelengths::TerminateSecondary() 174
             SurfaceInteraction 138
             SurfaceInteraction::SpawnRay() 645
             WavefrontPathIntegrator 939
             WavefrontPathIntegrator:: EvaluateMaterialsAndBSDFs() 959
```

```text
         Following this approach, modify pbrt so that each ray traced handles its
         termination individually when it reaches the maximum depth. Execute a
         modified camera ray generation kernel each time through the main rendering
         loop so that additional pixel samples are taken and camera rays are generated
         until the current RayQueue is filled or there are no more samples to take. Note
         that you will have to handle Film updates in a different way than the current
         implementation—for example, via a work queue when rays terminate. You may
         also have to handle the case of multiple threads updating the same pixel sample.
         Finally, implement a mechanism for the GPU to notify the CPU when all rays
         have terminated so that it knows when to stop launching kernels.
```

With all that taken care of, measure pbrt’s performance for a scene with a high maximum ray depth. (Scenes that include volumetric scattering with media with very high albedos are a good choice for this measurement.) How much is performance improved with your approach? How is performance affected for easier scenes with lower maximum depths that do not suffer from this problem? ➌ 15.6 In pbrt’s current implementation, the wavefront path tracer is usually slower than the VolPathIntegrator when running on the CPU. Render a few scenes using both approaches and benchmark pbrt’s performance. Are any opportunities to improve the performance of the wavefront approach on the CPU evident?

Next, measure how performance changes as you increase or decrease the queue sizes (and consequently, the number of pixel samples that are evaluated in parallel). Performance may be suboptimal with the current value of WavefrontPathIntegrator::maxQueueSize, which leads to queues much larger than can fit in the on-chip caches. However, too small a queue size may offer insufficient parallelism or may lead to too little work being done in each ParallelFor() call, which may also hurt performance. Are there better default queue sizes for the CPU than the ones used currently? ➌ 15.7 When the WavefrontPathIntegrator runs on the CPU, there is currently minimal performance benefit from organizing work in queues. However, the queues offer the possibility of making it easier to use SIMD instructions on the CPU: kernels might remove 8 work items at a time, for example, processing them together using the 8 elements of a 256-bit SIMD register. Implement this approach and investigate pbrt’s performance. (You may want to consider using a language such as ispc (Pharr and Mark 2012) to avoid the challenges of manually writing code using SIMD intrinsics.)

➌ 15.8 Implement a GPU ray tracer that is based on pbrt’s class implementations from previous chapters but uses the GPU’s ray-tracing API for scheduling rendering work instead of the wavefront-based architecture used in this chapter. (You may want to start by supporting only a subset of the full functionality of the WavefrontPath Integrator.) Measure the performance of the two implementations and discuss their differences. You may find it illuminating to use a profiler to measure the bandwidth consumed by each implementation. Can you find cases where the wavefront integrator’s performance is limited by available memory bandwidth but yours is not?

```text
     ParallelFor() 1107
     VolPathIntegrator 877
     WavefrontPathIntegrator::maxQueueSize 940
```

_________________ 1 For these measurements, the GPU was an NVIDIA RTX 3090. The 8-core CPU was a 3.6GHz Intel Core i9, and the 32-core CPU was a 3.7GHz AMD Ryzen Threadripper 3970X. 2 These correspond to compute units on AMD GPUs, execution units on Intel GPUs, and streaming multiprocessors on NVIDIA GPUs. 3 This term corresponds to a subgroup in OpenCL and Vulkan, a warp in CUDA’s model, and a wavefront on AMD GPUs.

```text
     ThreadPool 1102
```

4 This assumes that the compiler is unable to automatically restructure the code in the way that we have done manually. It might, but it might not; our experience has been that it is best not to expect too much of compilers in such ways, lest they disappoint.

```text
     WavefrontPathIntegrator 939
```

5 Shared memory corresponds to local memory in OpenCL, thread group shared memory in DirectX Compute, and shared memory in CUDA. 6 In describing the WavefrontPathIntegrator in the remainder of this chapter, we will frequently use the terminology of “launching kernels” to describe its operation, even though when it is running on the CPU, “launch” is just a function call, and a kernel is a regular class method. 7 Another alternative, array of structures of arrays, or AOSOA, offers a middle ground between AOS and SOA by repeatedly applying SOA with fixed-size (e.g., 32-element) arrays, collecting those in structures. This provides many of SOA’s benefits while further improving memory access locality. 8 We have found that this partitioning of texture evaluation work can give as much as a 20% improvement in rendering performance on current GPUs. 9 The explicit SampledSpectrum cast of the L value read from pixelSampleState is an unfortunate artifact of the GetSetIndirector used in the SOA classes. If the value read from the array is not immediately converted to a SampledSpectrum, then the compiler is left with what seems to be an attempt to multiply a GetSetIndirector type with a SampledSpectrum, leading to a syntax error. 10 Similar issues apply with packet or stream tracing on the CPU; see Section 16.2.3 for discussion of work on that topic.
