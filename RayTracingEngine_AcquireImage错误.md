在将 vulkan sdk 从 1.3.268 升级到 275 后，运行引擎demo时出现了验证层错误。

```
(device_vk.cpp:98): VUID-vkAcquireNextImageKHR-semaphore-01779: Validation Error: [ VUID-vkAcquireNextImageKHR-semaphore-01779 ] Object 0: handle = 0xa43473000000002d, type = VK_OBJECT_TYPE_SEMAPHORE; | MessageID = 0x5717e75b | vkAcquireNextImageKHR():  Semaphore must not have any pending operations. The Vulkan spec states: If semaphore is not VK_NULL_HANDLE it must not have any uncompleted signal or wait operations pending (https://vulkan.lunarg.com/doc/view/1.3.283.0/windows/1.3-extensions/vkspec.html#VUID-vkAcquireNextImageKHR-semaphore-01779)
```

> 这里用了 283 版本的 SDK，但是从 275 开始，都会报这个错误

报错的提示为：在 `vkAcquireNextImageKHR()` 时，出现错误：对应的 semaphore 不得有任何未完成的 signal 或者是等待操作处理。

由于是 semaphore 出错，所以是同步问题。并且问题在升级 sdk 版本后出现，说明验证更严格了，之前对它的使用方法就不合理，现在把问题暴露出来了。

## 分析

首先找到调用 `vkAcquireNextImageKHR()` 的位置，在 `render_backend_vk.cpp` 中，可以看到，此时等待被 signal 的 semaphore 是来自 `SwapchainImagesVk` 结构体的信号量：

```c++
struct SwapchainImagesVk {
    vector<VkImage> images;
    vector<VkImageView> imageViews;

    uint32_t width{ 0 };
    uint32_t height{ 0 };

    VkSemaphore semaphores;
};
```

swapchain 中所有的 image 和对应的 imageView 都放在这里面（一般情况下是3个，后面将它们称呼为 SCimages），使用了一个 semaphore 来同步。

然后使用 vs 调试程序，在报错时 vs 会保存当时的调用堆栈，可以很方便的进行调试。

### 回顾 vulkan tutorial

在 vulkan tutorial 中使用了两个同步原语：semaphore 和 fence，前者用于 GPU queue 中的同步，后者用于 host 与 GPU 之间的同步。

在 semaphore 方面使用了 `imageAvailableSemaphores` 指明当前 SCimage **可用**，当 `FRAMES_IN_FLIGHT` 为1时实际上变为单个semaphore，并且在此版本的 sdk 中不报错。后续使用 `imageAvailableSemaphore` 来指代 `FRAMES_IN_FLIGHT` 为1时的 `imageAvailableSemaphores`。

在 fence 方面，由于每帧都要**重新录制 commandbuffer**，所以对当前帧，host 端需要等待 commandbuffer 在 GPU 中使用完毕，才能再次录制，素以使用了 inFlightFence 来指明这一帧中 commandbuffer 都使用完毕了。这里“commandbuffer 都使用完毕”隐式说明了其中的操作都已经完成。

`imageAvailableSemaphore` 主要用在 `vkAcquireNextImageKHR()` 和 `vkQueueSubmit()` 的 `VKSubmitInfo::pWaitSemaphores` 中。

在刚创建出 VkSemaphore 时，它是 unsignaled 的。

对于一张 SCimage 来说，其使用顺序应该如下：

- 从 swapchain 中获得 SCimage，此 image 可以用于绘制 ------ `vkAcquireNextImageKHR()`
- 执行绘制到获取的图像上的命令 ------ `vkQueueSubmit()`
- 将该图像呈现到屏幕上进行演示，并将其返回到交换链 ------ `vkQueuePresentKHR()`

**`vkAcquireNextImageKHR()`** 的伪代码形式为：`vkAcquireNextImageKHR(swapchain, signal: S, imageIndex)`。

它会返回 SCimage 对应的 index，并将 `VkSemaphore S` 置为 signaled 状态，此时 S 标志着此 SCimage 可用于绘制。

**`vkQueueSubmit()`** 的伪代码形式为：`vkQueueSubmit(work: A, signal: renderFinished, wait: S)`。这里它会等待 S 变成 signaled，然后执行渲染队列里的命令同时将 S 置为 unsignaled 状态，执行完后将 renderFinished 置为 signaled 状态。

**`vkQueuePresentKHR()`** 的伪代码形式为：`vkQueuePresentKHR(presentQueue, swapChain, wait: renderFinished)`。它会等待 renderFinished 变成 signaled，执行显示操作并把SCimage返回到交换链，同时将 renderFinished 置为 unsignaled。

> Acuqire 和 Present 部分，vulkan 中使用了 WSI，具体可以参照 [Slide 1 (nvidia.com)](https://developer.download.nvidia.com/gameworks/events/GDC2016/Vulkan_Essentials_GDC16_tlorach.pdf) 中的page 45。
>
> 还有一篇 slide 值得一看：[Slide 1 (nvidia.com)](https://developer.download.nvidia.com/gameworks/events/GDC2016/mschott_lbishop_gl_vulkan.pdf)

> `vkAcquireNextImageKHR()` 在无 image 可用，且未超时时，会阻塞，见[VK_KHR_swapchain(3) (khronos.org)](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_swapchain.html) Issues 7。无 image 可用指的是 swapchain 中所有 image 都不处于 available 状态。
>
> `vkQueueSubmit()` 不会阻塞。
>
> `vkQueuePresentKHR()` 在未超时时会阻塞，但是**必须**在有限事件内返回。并且**必须**使用 semaphore 来保证图像上的绘制操作已完成可以被显示。

回顾 [Frames in flight - Vulkan Tutorial (vulkan-tutorial.com)](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Frames_in_flight) 章节，它使用了 2 个飞行帧，每个飞行帧都需要有对应的 commandbuffer、semaphore、fence 等，所以它们都需要变成两个。

> 这里 commandbuffer 与帧不是强相关的，但是也需要两个，fence 是用于标志 commandbuffer 可更新。

```
               ┌─────────────┐                          
               │inFlightFrame│                          
               └──────┬──────┘                          
        require       │                                 
        ┌─────────────┴─────────────┐                   
        │                           │strong correlation 
 ┌──────┴───────┐  weak corr ┌──────┴──────┐            
 │swapchainImage├───────────►│commandBuffer│            
 └──────┬───────┘            └──────┬──────┘            
        │strong correlation         │ strong correlation
        │                           │                   
   ┌────▼────┐                   ┌──▼──┐                
   │semaphore│                   │fence│                
   └─────────┘                   └─────┘                
```



### 再分析程序

在引擎中，每个 RenderNode 都对应着一个 nodeContextData，所以它们数目一致（在这里是五个），nodeContextData 中有若干个 FrameCommandBuffer。只使用了一个 semaphore 对 Acquire 和 QueueSubmit 进行同步。使用了数目为 3 的 FrameCommandBuffer（其中包含 CommandBuffer 和对应的 Semaphore）。用了同样数目（3）的 fence 来进行 CPU 和 GPU 之间的同步，这个 3 意味着有 3 个飞行帧。提交一帧的命令之前，会等待对应的 fence，等待前一帧（实际上是前3帧）完成 present。

> 所以在一帧中有 5*3=15 个 commandbuffer，每一个 RenderNode 可以同时绘制 3 个飞行帧。
>
> “等待对应的 fence”时，有特殊情况，就是第一二三帧，fence 的初始化状态为 signaled，意味着不用等待。从第四帧开始才会真正的等待。

引擎中有计数器对帧进行计数，我们可以打断点，发现是在进行第二帧绘制的时候，在 `vkAcquireNextImageKHR()` 时报的错。

由于只使用了一个 semaphore（后文使用 SCIsemaphore 指代），意味着提交完第一帧的绘制命令后，在 acquire 用于第二帧的 SCimage 时有未完成的 signal 或者是等待操作处理。

继续调试，发现在 `vkQueueSubmit` 中，wait 的 semaphore 是 SCIsemaphore。在第一帧时，首先等待了fence（实际上并未等待），acquire image 后，SCIsemaphore 变为 signaled，所以此时能直接提交，在开始执行命令后，将 SCIsemaphore 由 signaled 变为 unsignaled。

在第二帧时，又等待了对应的 fence（实际上并未等待），acquire image 准备 signal 的 semaphore 依然是 SCIsemaphore，但是此时上一个 queueSubmit 中的命令可能还未开始执行，所以 SCIsemaphore 可能依然是 signaled 的状态，此时去 signal 一个 signaled 的信号量是不被支持的，所以验证层会报错。

简单来说，就是第二帧的 image 准备好了，和第一帧共用一个 semaphore，由于未等待 fence（为了等待第一帧的 present 完成，实际上第一帧的 image 很可能还未开始绘制），所以这个 semaphore 上还有未进行的操作（pending operation），造成了冲突。

如果是从第四帧开始，那么不会造成冲突。

> 上面说的 “wait 的 semaphore 是 SCIsemaphore” 就是验证层中的 wait operations pending，“命令可能还未开始执行，所以 SCIsemaphore 可能依然是 signaled 的状态” 就是验证层中的 uncompleted signal。
>
> 在 vulkan tutorial 中尝试在 `drawFrame()` 的不同位置再加一句冗余的 `vkAcquireNextImageKHR()`，可以看到如下的报错：
>
> 1. 在 `vkAcquireNextImageKHR()` 后面添加：报错 `Semaphore must not be currently signaled.`；
> 2. 在 `vkQueueSubmit` 后面添加：报错 `Semaphore must not have any pending operations.`。
>
> 所以在 `vkQueueSubmit` 后面添加时报错是一样的，这验证了文档中的说法。
>
> 另外还做了实验，将 `drawFrame()` 中开头的 `vkWaitForFences` 挪到 `vkAcquireNextImageKHR()` 后面，可以在第三帧和第四帧看到一样的报错。

**总结**：

实际上 QueueSubmit 中等待的 semaphore 与 commandBuffer 有隐式关联，它将 semaphore 变成 unsignaled，并且只有当 commandBuffer 中的指令都执行完毕，这个 semaphore 才能再次被 signal，即 semaphore 才是可用的。

`vkAcquireNextImageKHR` 会去 signal 上面那个 semaphore。所以实际上 `vkAcquireNextImageKHR` 是需要进行等待的（等待 commandBuffer 中的指令都执行完毕），但是它自己是立即执行的，这个时候就需要使用其他的同步机制，比如说 fence 来等待。但是在 wait fence 的时候有讲究，如果在 `vkAcquireNextImageKHR` 之前等待，等待的是此帧的 semaphore 可用；如果在 `vkAcquireNextImageKHR` 之前等待，等待的是上一帧的 semaphore 可用。

## 解决方法

在查看不少其他引擎和 vulkan sample 之后，可以看到他们都是先设定一个最大的同时绘制帧的数量（后续称为 MAX_CONCURRENT_FRAMES），即 vulkan tutorial 中的 inFlight 帧，这个数量会小于等于 swapchain image 的数量。然后使用 MAX_CONCURRENT_FRAMES 数目的 presentCompleteSemaphores、renderCompleteSemaphores 来标志 swapchain image 的可用情况，使用 MAX_CONCURRENT_FRAMES 数目的 Fence 来标志 commandbuffer 是否已经完成执行。

在我们的引擎中，使用了 3 个 fence 来进行 CPU 和 GPU 之间的同步，意味着有 3 个飞行帧。但是 SCIsemaphore 数目为 1，就会导致在 Acquire Image 时 SCIsemaphore 依然处于 signaled 状态或者是有未执行的操作。

最简单的解决方法就是将 SCIsemaphore 与飞行帧关联起来，即数目改成和飞行帧的数目一样，然后在 Acquire Image 和 Queue Submit 中分别 signal 和 wait 对应飞行帧的 SCIsemaphore，这样就能保证专帧专用。同时在 Acquire Image 前添加等待 fence。

## 总结

最好对于同时绘制的每个飞行帧，都创建对应的 semaphore 和 commandbuffer（还有用于 commandbuffer 的 fence），然后对于每一帧，都使用 waitFence 来保证 acquire 时commandbuffer 中所有命令都已完成，使用 semaphores 来在 GPU 端使 acquire/draw/submit/present 按顺序执行。

1. 在每帧开始时，先 waitFence，等待此帧对应的 commandbuffer 的 fence，再 acquire。
2. 在 acquire 中添加指示 image 可用的 imageAvailableSemaphore。
3. 在重新录制指令到此帧对应的 commandbuffer 之前，resetFence。
4. 在 queueSubmit 时，等待 imageAvailableSemaphore，同时将 signalFence 设置为此帧对应的 commandbuffer 的 fence，添加指示渲染完成的 renderFinishedSemaphore。
5. 在 present 中，等待 renderFinishedSemaphore。

> 也参考了部分 [Vulkan Tutorial中的同步问题 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/454825408) 和它的图
>
> ![](https://pic2.zhimg.com/80/v2-447ecd30478ab3dd11d9b258ea7d14d5_720w.webp)
>
> 或者按照 [Vulkan validation error with SDK 1.3.275 · Issue #7236 · ocornut/imgui (github.com)](https://github.com/ocornut/imgui/issues/7236) 中说所的，在 `vkAcquireNextImageKHR` 之后等待 fence，但是需要一个额外的 semaphore。

## Additional

根据 Vulkan validation doc 中的 [VkSemaphore / vvl::Semaphore](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/fine_grained_locking.md#vksemaphore--vvlsemaphore) 章节所述，与 `vkQueuePresentKHR()` 或 `vkAcquireNextImageKHR()` 一起使用的信号量必须特殊处理，因为状态跟踪器目前无法可靠地知道这些信号量何时改变状态。（这会造成很多误报）

所以当使用 `vkQueueSubmit(work: A, signal: renderFinished, wait: imageAvailableSemaphore, signal: inFlightFence)` 提交指令到 queue 中，等待 `imageAvailableSemaphore` 时，如果 queue 中的指令没有运行完，而又在接下来的 `vkAcquireNextImageKHR()`中使用了 `imageAvailableSemaphore` 时，验证层就直接报错，因为它无法判断此信号量何时改变状态。但是如果 `inFlightFence` 被 signal，就表明指令必然已经运行完了，此时 `imageAvailableSemaphore` 的信号量一定改变了状态，所以验证层不报错。
