距学习 vulkan tutorial 过去了许久，很多概念都有点遗忘了，再次看文章的时候，有了新的疑惑和理解，可能会写一系列笔记，或者就这一篇。

其中第一个点是 RenderPass 和 Framebuffer，在 [Multisampling - Vulkan Tutorial (vulkan-tutorial.com)](https://vulkan-tutorial.com/Multisampling) 一章中，为了实现多重采样，修改了 RenderPass 和 Framebuffer。在 RenderPass 中添加了一个 `VkAttachmentDescription ` 和对应的 `VkAttachmentReference `，它们被称之为 `colorAttachmentResolve` 和 `colorAttachmentResolveRef`，与之前创建的 `colorAttachment` 对应。这里的 **Resolve** 指的是将之前的 Image 解析到目标 Image 中。在创建 Attachment 的时候，就如 `VkAttachmentDescription ` 和对应的 `VkAttachmentReference  `的名字 Description\Reference 所述，只指定了它的描述和引用，并没有直接指定渲染的资源（资源指 FrameBuffer 和其对应的各个 Image）。然后在 Framebuffer 的创建中，指定了作为 attachment 的 color、depth 和 swapchainImage。只有 swapchainImage 可以用来 Present，这里的 color 是用于多重采样的实际 attachment ，然后将 color 解析到 swapchainImage 中。

所以其实可以把 RenderPass 看作是**占位符**，描述了 RenderPass 的布局和资源的格式，后续将资源与这个占位符匹配就可以了。这个概念来源于 [理解Vulkan渲染通道(RenderPass) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/619295431?utm_id=0)。

> 这个作者有一系列文章：[Vulkan文章汇总 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/616082929)

这篇文章提到 "简单来说RenderPass是整个Render Pipeline的一次执行。RenderPass本质是定义一个完整的渲染流程以及使用的所有资源的描述，可以理解为是一份元数据或者占位符的概念其中不包含任何真正的数据。"

RenderPass 其实是通过 Framebuffer 中包含的 ImageView 拿到真正的数据(牢记 RenderPass 只是元数据)。并且之后的 RenderPass 中只要满足 Vulkan 定义的 Render Pass Compatibility 要求的 Framebuffer，Framebuffer 就能通过 RenderPass 渲染出相应的结果。

**使用 RenderPass**

开始 RenderPass：`vkCmdBeginRenderPass` 调用来创建一个 RenderPass 实例。并且如果一旦一个 RenderPass 要在 Command Buffer 中开始，提交给该 Command Buffer 的后续命令将在该 RednerPass 实例的第一个 SubPass 中执行。`vkCmdBeginRenderPass` 需要填充一个 `VkRenderPassBeginInfo` 结构用来指定要开始的 RenderPass 实例以及该实例使用的 Framebuffer。

切换 RenderPass：在一个 RenderPass 当中会有一个或者多个 SubPass，从上面的`vkCmdBeginRenderPass` 创建 RenderPass 实例，并且隐式的从第一个 SubPass 开始运行，但是假如 RenderPass 内有多个 SubPass，如何去切换到下一个 SubPass 呢。可以通过 `vkCmdNextSubpass` 来完成这个操作。RenderPass 的 SubPass 索引从记录 `vkCmdBeginRenderPass` 时的零开始，每次记录`vkCmdNextSubpass` 时都会递增。

结束 RenderPass：在全部命令提交到了 Command Buffer 之后，就可以结束 RenderPass。在这里通过 `vkCmdEndRenderPass` 来完成，结束一个 RenderPass 实例。