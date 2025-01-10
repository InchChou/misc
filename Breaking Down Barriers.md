[Breaking Down Barriers - Part 1: What's a Barrier? (therealmjp.github.io)](https://therealmjp.github.io/posts/breaking-down-barriers-part-1-whats-a-barrier/)

> 翻译在知乎上已经有文章了：
>
> [【译】拆解D3D12和Vulkan中的Barrier（1） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164491130)
>
> [【译】拆解D3D12和Vulkan中的Barrier（二） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164845095)
>
> [【译】拆解D3D12和Vulkan中的Barrier（3） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164872228)
>
> [【译】拆解D3D12和Vulkan中的Barrier（4） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164979258)
>
> [【译】拆解D3D12和Vulkan中的Barrier（5） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/164997399)
>
> [【译】拆解D3D12和Vulkan中的Barrier（6） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/165493688)

在某些上下文中，“barrier”是一个**同步点**，一旦一组线程到达它们正在运行的代码中的特定**点**，它们就必须全部停止。这种情况下 barrier 可以看作是一堵不能移动的墙。当您想知道一组线程何时全部完成执行任务（fork-join 模型中的“join”），或者当您有线程需要读取其他线程的结果时，这种事情很有用。作为一名程序员，您可以通过对通过原子操作更新的变量进行“旋转”（循环直到满足条件）来实现线程屏障，或者当您希望线程在等待时进入休眠状态时，通过使用信号量和条件变量来实现线程屏障。

另一种上下文种，"barrier"是 memory barrier，也可以称为 fence。在这些情况下，你通常要处理由编译器和/或处理器本身完成的内存操作的重新排序，当你有多个处理器通过共享内存进行通信时，这可能会给工作带来麻烦。内存屏障可以帮助你强制内存操作在屏障之前或之后完成，有效地将它们保持在栅栏的“一侧”。在 c++ 种有这种 api，比如 `std::atomic_thread_fence`。

“barrier”一词的这两个含义具有不同的具体含义，但它们也有一些共同点：它们主要用于一件事产生结果而另一件事需要读取该结果的情况。另一种说法是，一个任务依赖于另一个任务。



如果 GPU 的绘制/调度线程可以与其他线程重叠，则意味着 GPU 需要一种方法来防止在两个任务之间存在数据依赖性的情况下发生这种情况（指同时操作同一个数据）。可以像我们在 CPU 上所做的那样，插入类似于线程barrier的东西，以便让我们知道一组线程何时全部完成工作。实际上，GPU 倾向于以非常粗略的方式执行此操作，例如等待所有未完成的计算着色器线程完成后再启动下一个调度。这可以称为“flush”或“wait idle”。

GPU 为什么需要 barrier，由于 GPU 的架构，每个 CU、texture unit、color buffer 的 L1 和 L2 缓存可能没有连接起来。当您还有多个可能包含陈旧数据的缓存时，确保您的线程不重叠不足以解决读写依赖关系。您还必须使这些缓存无效或刷新，以使结果对需要读取数据的后续任务可见。

现代 GPU 为了节省带宽，在输出纹理时使用了压缩算法。虽然 ROP 可能知道如何处理压缩数据，但当着色器需要通过其纹理单元随机访问数据时，情况就不一定如此了。这意味着，根据硬件和纹理的使用方式，在纹理内容可供相关任务读取（或通过 ROP 以外的方式写入）之前，可能需要执行解压缩步骤。再一次，当我们谈论 GPU 和用于与它们对话的新显式 API 时，这属于“屏障”的范畴。

在阅读了我关于**线程同步(thread synchronization)、缓存一致性(cache coherency)和 GPU 压缩(GPU compression)**的胡言乱语后，希望您至少对典型 GPU 需要屏障才能执行我们期望的正常操作的 3 个潜在原因有一个非常基本的了解。但是，如果您查看 D3D12 或 Vulkan 中的实际屏障 API，您可能会注意到它们似乎与我们刚才讨论的内容并不直接对应。毕竟，ID3D12GraphicsCommandList 上并没有“WaitForDispatchedThreadsToFinish”或“FlushTextureCaches”函数。