距学习 vulkan tutorial 过去了许久，很多概念都有点遗忘了，再次看文章的时候，有了新的疑惑和理解，可能会写一系列笔记，或者就这一篇。

其中第一个点是 RenderPass 和 Framebuffer，在 [Multisampling - Vulkan Tutorial (vulkan-tutorial.com)](https://vulkan-tutorial.com/Multisampling) 一章中，为了实现多重采样，修改了 RenderPass 和 Framebuffer。在 RenderPass 中添加了一个 `VkAttachmentDescription ` 和对应的 `VkAttachmentReference `，它们被称之为 `colorAttachmentResolve` 和 `colorAttachmentResolveRef`，与之前创建的 `colorAttachment` 对应。这里的 **Resolve** 指的是将之前的 Image 解析到目标 Image 中。在创建 Attachment 的时候，就如 `VkAttachmentDescription ` 和对应的 `VkAttachmentReference  `的名字 Description\Reference 所述，只指定了它的描述和引用，并没有直接指定渲染的资源（资源指 FrameBuffer 和其对应的各个 Image）。然后在 Framebuffer 的创建中，指定了作为 attachment 的 color、depth 和 swapchainImage。只有 swapchainImage 可以用来 Present，这里的 color 是用于多重采样的实际 attachment ，然后将 color 解析到 swapchainImage 中。

所以其实可以把 RenderPass 看作是**占位符**，描述了 RenderPass 的布局和资源的格式，后续将资源与这个占位符匹配就可以了。这个概念来源于 [理解Vulkan渲染通道(RenderPass) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/619295431?utm_id=0)。

> 这个作者有一系列文章：[Vulkan文章汇总 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/616082929)

这篇文章提到 "简单来说RenderPass是整个Render Pipeline的一次执行。RenderPass本质是定义一个完整的渲染流程以及使用的所有资源的描述，可以理解为是一份元数据或者占位符的概念其中不包含任何真正的数据。"