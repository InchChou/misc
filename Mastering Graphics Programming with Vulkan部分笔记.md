## Chapter 2: Improving Resource Management

### Unlocking and implementing bindless rendering

使用 `VK_EXT_descriptor_indexing` 扩展，开启 `descriptorBindingPartiallyBound` 和 `runtimeDescriptorArray`。之后使用开启的功能设置 `PARTIALLY_BOUND_BIT` 和 `UPDATE_AFTER_BIND_BIT` 标志位来创建 DescriptorSetLayout。

在读取纹理数据，调用 `GpuDevice::create_texture` 创建纹理时，将纹理放入 `texture_to_update_bindless` 数组，在下一帧开始之前，使用这个数组来更新 descriptorSet。

这里的核心思想是将场景中所有的纹理都放入 `layout ( set = 1, binding = 10 ) uniform sampler2D global_textures[]` 数组中，而不再区分 baseColor、normalTexture 等纹理种类（区分这些种类就需要分别绑定）。将所有纹理打包放入数组后我们只需要绑定一次，在访问纹理时，只需要通过纹理的 index 将对应的纹理取出即可。纹理的 index 有多种保存方法，这一章中是将纹理 index 放入名为 Mesh 的 uniform buffer 中，这个 uniform buffer 实际上是 mesh 的额外数据。

### Automating pipeline layout generation

本章涉及到了对 SPIR-V 文件的反射的概念。通过解析 SPIR-V 文件，找到

```spirv
OpDecorate %__0 DescriptorSet 0
OpDecorate %__0 Binding 0

%__0 = OpVariable %_ptr_Uniform_LocalConstants Uniform
```

形式的变量，收集这些信息，使用它们创建 DescriptorLayout。

本章中写了一个小的解析器。也可以使用反射库：https://github.com/KhronosGroup/SPIRV-Reflect。

## Chapter 3: Unlocking Multi-Threading

描述**基于任务的并行化**（task-based parallism），使用 enkiTS 来实现它。

使用 enkiTS 中的 pinnedTask 固定占用一个线程，来进行**并行化 IO**，例如读取 texture。

使用 Secondary Command Buffer 来**并行录制绘制指令**。

> Primary command buffer: 可以执行任何命令(绘图、计算或复制命令)，但它们的粒度非常粗糙 - 必须使用至少一个renderPass，并且任何 pass 都不能进一步并行化。
>
> Secondary command buffer: 限制很多， 它们只能在 render pass 中执行绘制指令，但是可用于并行化包含许多绘制调用的渲染过程（例如 G-Buffer 渲染过程）。它不能直接提交到 queue 中，所以始终需要 Primary command buffer：它们必须被复制到 primary command buffer 中，并且只集成在 beginCommandBuffer 时所设置的 RenderPass 和 FrameBuffer。
>
> 示例中使用 Secondary command buffer 的步骤：
>
> 1. 使用 Primary command buffer，使用 `vkBeginCommandBuffer` 开始记录命令，准备好 renderpass 和 framebuffer
> 2. 在 `vkCmdBeginRenderPass` 中的 `VkRenderPassBeginInfo` 中设置 renderpass 和 framebuffer，同时 设置 `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS` 标志位
> 3. 创建 `VkCommandBufferInheritanceInfo`，使用刚才准备好的 renderpass 和 framebuffer，将 InheritanceInfo 传递给 的 Secondary command buffer 所使用的 `VkCommandBufferBeginInfo`。（这个 renderPass 实际上是跟 此 Secondary commandBuffer 关联起来了）
> 4. 将 `VkCommandBufferBeginInfo` 的标志位设置为 `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`，然后调用 `VkBeginCommandBuffer` 开始录制指令。
> 5. 使用 `vkCmdXXX` 将指令录制到 Secondary command buffer 中。
> 6. 录制完指令后，使用 `vkCmdExecuteCommands` 将 Secondary command buffer 复制到 Primary command buffer 中。
> 7. 注意，在执行 `vkCmdExecuteCommands` 之前必须有 `vkCmdBeginRenderPass`，`vkCmdBeginRenderPass` 可以在 `vkBeginCommandBuffer(primaryCB)` 到 `vkCmdExecuteCommands(primaryCB)` 之间的任意位置，它的 renderPass 是 被提交的 Secondary command buffer 关联的 renderPass 。



> 针对 Primary command buffer 和 Secondary command buffer，根据 [VkCommandPool(3) (khronos.org)](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkCommandPool.html) 中所述，不论在任何形式下，多个线程都不能同时对同一个 commandBuffer 进行指令录制。
>
> 它们的用法见 vulkan 的官方示例：[command_buffer_usage](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/command_buffer_usage) 和 [multithreading_render_passes](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/multithreading_render_passes)，官方示例写的比较清楚。



对此章中并行录制指令代码的简要总结：

1. 使用 enkiTS 创建了 n 个线程用于 task，enkiTS 会多创建一个线程（作用暂时不详，稍微看了下代码好像是标识主线程的），其中有 1 个线程专门用于并行化 IO，其他的用于并行录制指令。所以总共有 n+1 个线程
2. 设置同时渲染的帧的数量为 3（一般情况下为 swapchain 的 image 数量或者数量减 1，swapchainimage 数量在大多数设备上为 3）
3. 根据 Recording commands on multiple threads 中所述，在 Vulkan 中，任何类型的池都需要**由用户进行外部同步**；因此，最好的选择是**将线程与池关联起来**。The allocation strategy 小节所述，“每个线程不仅需要一个唯一的池来分配命令缓冲区和命令，而且还不能在 GPU 中处于运行状态”，所以这里使用了最简单的分配策略（可能会导致不平衡的命令生成，但是本章先用这种策略），根据线程的数量 `T=n+1` 和同时渲染帧数量 `F=3` 创建 `F*T` 个 commandPool，即每个线程和每个帧都有一个相关的 commandPool，这样的话可以**使用成对的 frame-thread ID 来获取对应的独立的 commandPool**。后续在使用 commandBuffer 时，只需要 reset commandPool 即可。在 raptor 中，使用 `CommandBufferManager` 来管理 commandPool 和 commandBuffer
4. 针对每个 commandPool，分配最多五个空命令缓冲区，2 个 Primary commandBuffer，3 个 Secondary commandBuffer。
5. 在读取 glTF 场景之后，准备渲染，调用 `glTFScene::submit_draw_task` 提交 drawTask 到 enkiTS 的 scheduler 中。
6. scheduler 调度任务，执行 drawTask，在执行 task 的 `ExecuteRange()` 时，会将 `enki::TaskSetPartition range` 和 当前执行线程的 index 传递给 drawTask，drawTask 执行如下：
   1. 此线程 index 被传递给 GpuDevice 来获取 primary commandBuffer：在 GpuDevice 中存储了当前渲染的 frame index，所以组成了 frame-thread ID，再从 `CommandBufferManager` 获取对应 frame-thread ID 的 primary commandBuffer
   2. 使用此 primary commandBuffer 录制指令，其中最重要也必须要录制的是 `vkCmdBeginRenderPass`
   3. 将 mesh 分成四组进行并行绘制，每组包装成一个 `SecondaryDrawTask` 并加入到 scheduler 中（由于 mesh 的数量不一定是 4 的倍数，所以还有未被分组的 mesh，所以在 drawTask 中又写了一遍 `SecondaryDrawTask` 的执行逻辑，是用于绘制未被分组的 mesh 和 ui），此时要传递的信息有：当前正在录制指令的 Primary commandBuffer（用于取回 renderPass 和 frameBuffer）、要绘制的 mesh 在 mesh 数组中的起始位置、渲染器
      1. 从使用当前执行线程 index 从渲染器的 GpuDevice 中获取 Secondary commandBuffer，逻辑和之前的步骤一样。
      2. 开始录制指令到 Secondary commandBuffer 中，此时填充 `VkCommandBufferInheritanceInfo` 结构体。
      3. 录制绘制指令
   4. 等待 `SecondaryDrawTask` 执行完成，将 `SecondaryDrawTask` 中的 Secondary commandBuffer 使用 `vkCmdExecuteCommands` 复制到 Primary commandBuffer 中。
   5. 提交 Primary commandBuffer 到 queue 中。

经过这些流程，就完成了多线程并行录制指令到 commandBuffer 中。

## Chapter 4: Implementing a Frame Graph

### Implementing the frame graph

Graph 中某个 Node 的 output 可能是另一个 Node 的 input，所以 `FrameGraphResource` 中的 `output_handle` 用于将此 input resource 链接到对应的 output 中（即 FrameGraph 中的某个 output resource）。把这些资源分开（指的是同一个资源既用于 output 也用于 input，但是这里把它看作是两个资源）是因为它们的类型可能不同：比如说一个 image 在用于 output 时是 attachment，在用于 input 时是 texture。这个细节可以用于自动放置 memory barrier。

使用 DFS 算法分析 Graph 的依赖关系，此章使用了基于 stack 的迭代形式算法来代替递归形式算法来实现**拓扑排序**，将叶子节点放在数组的前面，将父节点放在数组的后面，即以逆序存储节点到 `sorted_nodes` 中。后面再将 `sorted_nodes` 翻转过来存储到 `nodes` 数组中。

> 关于拓扑排序，可以参照《算法4》中的章节“4.2.4 环和有向无环图”。

**Memory aliasing**：由于 resource 的生命周期不一定跨越整个 Graph，所以我们可能可以重新使用不再需要的资源的内存，这就叫做内存别名。使用引用计数的方法，根据刚才得到的 `nodes` 数组，找到引用计数为 0 的资源，将此资源添加到空闲列表中，以备我们将要处理的下一个节点使用。这样就能达到别名的效果。

**使用 frame graph 驱动渲染**：对于每个 node 的 input 和 output，使用 barrier 将它们的 memory 布局转换到正确的布局，来确保它们的读写正确性。



## Chapter 5: Unlocking Async Compute

### Replacing multiple fences with a single timeline semaphore

使用 timeline semaphore 可以减少用于同步的 fence 和 binary semaphore 的数目。对于 fence 来说，每个缓冲帧都需要一个 fence。对于 binary semaphore 来说，每个缓冲帧至少需要两个，一个用于标识渲染结束，一个用于标识 present 结束。

可以使用 `vkWaitSemaphores` 和 `VkSemaphoreWaitInfo` 来在 CPU 上等待 timeline semaphore。

> 此章使用了 [VK_KHR_synchronization2](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_synchronization2.html) 扩展来简化 barrier 和 semaphore 代码的编写。

通过 timeline semaphore，我们现在可以在一个 submission 中等待 wait 并发送 signal 相同的 timeline semaphore，并且也可以不再使用 fence。

### Adding a separate queue for async compute

使用不同的 queue 来进行异步计算时，需要在不同的 queue 中使用不同的 commandBuffer。在此章中，在 compute queue 中需要等待 timeline semaphore，每次提交都要将 semaphore value 增加，同时需要注意等待的值的准确性。

在 graphics queue 中也要等待 compute queue 中的工作完成，也需要通过等待 timeline semaphore 来实现。

### Implementing cloth simulation using async compute

使用 GPU 进行通用计算，可以防止从 CPU 内存到 GPU 内存的拷贝，从而提升应用性能。

GPU 执行模型称为单指令多线程 (Single Instruction, Multiple Threads, SIMT)。它类似于现代 CPU 提供的单指令多数据 (SIMD)，可使用单个指令对多个数据条目进行操作。

[Compute Shader - OpenGL Wiki (khronos.org)](https://www.khronos.org/opengl/wiki/Compute_Shader)

[LearnOpenGL - Compute Shaders - Introduction](https://learnopengl.com/Guest-Articles/2022/Compute-Shaders/Introduction)

[OpenGL-Compute Shader的输入和输出 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/135144894) 与 [Compute Shaders - In Depth - OpenGL SuperBible: Comprehensive Tutorial and Reference, Sixth Edition (2013) (apprize.best)](https://apprize.best/programming/opengl_1/12.html)

| Type  | Built-in name           |                                                              |
| ----- | ----------------------- | ------------------------------------------------------------ |
| uvec3 | gl_NumWorkGroups        | number of work groups that have been dispatched set by glDispatchCompute() |
| uvec3 | gl_WorkGroupSize        | size of the work group (local size) operated on defined with layout |
| uvec3 | gl_WorkGroupID          | index of the work group currently being operated on          |
| uvec3 | gl_LocalInvocationID    | index of the current work item in the work group             |
| uvec3 | gl_GlobalInvocationID   | global index of the current work item  *`(gl_WorkGroupID * gl_WorkGroupSize + gl_LocalInvocationID)`* |
| uint  | gl_LocalInvocationIndex | 1d index representation of `gl_LocalInvocationID`  *`(gl_LocalInvocationID.z * gl_WorkGroupSize.x * gl_WorkGroupSize.y + gl_LocalInvocationID.y * gl_WorkGroupSize.x + gl_LocalInvocationID.x)`* |

> 其中 `gl_WorkGroupSize` 对应着 compute shader code 中的 `layout (local_size_x = 16, local_size_y = 16, local_size_z = 1) in` 的 `local_size_x/y/z` 的大小。比如这里会设置 `16*16*1` 大小的工作组(work group)。它代表的是你期望有多少个计算单元同时运行这个 compute shader，能够共享局部内存和进行同步。
>
> `gl_NumWorkGroups` 对应着 `glDispatchCompute()` 和 `vkCmdDispatch()` 中所设置的 `groupCountX/Y/Z`。它会开启 `groupCountX * Y * Z` 个工作组。虽然工作组内的各个着色器调用作为一个单元执行，但工作组完全独立且按不特定的顺序执行。
>
> 上面两个结合起来用就会是如下效果：开启 `groupCountX * Y * Z` 个工作组，每个工作组大小为 `16*16*1`。
>
> 一般来说，如果使用 compute shader 来处理一张 `imageSizeX * imageSizeY` 大小的图像，则可以将 `local_size` 设置为 `16*16*16`，然后启动 compute shader 时可以使用 `vkCmdDispatch(imageSizeX/16, imageSizeY/16, 1)`，这样的话，`gl_GlobalInvocationID` 就对应着每个像素的位置。

本章中的布料模拟使用了弹簧模型，见此文章 [GI95.dvi (stanford.edu)](http://graphics.stanford.edu/courses/cs468-02-winter/Papers/Rigidcloth.pdf)

## Chapter 6: GPU-Driven Rendering

此章中，会有：

- 将大的 mesh 分解为小的 meshlet
- 使用 task 和 mesh shader 处理 meshlet，实现背面和视锥体剔除
- 使用 compute shader 实现有效的遮挡剔除
- 使用 indirect drawing functions 在 GPU 上生成 draw commands

### Breaking down large meshes into meshlets

每个 mesh 通常被分为顶点组（每组 64 个顶点），即 meshlet。

> [漫谈网格着色器 mesh shader - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv31632564/)

会给每个 meshlet 添加一些额外数据：

- meshlet 的 bounding shpere，用于视锥体和遮挡剔除
- meshlet 锥（meshlet cone），用于背面剔除

本章中使用了 [zeux/meshoptimizer](https://github.com/zeux/meshoptimizer) 来生成 meshlet。每个 meshlet 除了顶点数据和 primitive 数据之外，还有一个 `meshopt_Bounds` 数据用于剔除，这个数据是此 meshlet 的圆锥简易表达（cone），包括中心 center、半径 radius、顶点 apex、轴 axis、cutoff 角。

### Understanding task an mesh shaders

在 task shader 中，可以通过 cone cull 来进行 meshlet 的背面剔除。然后通过 frustum 的六个面来进行视锥体剔除。然后将顶点索引输出给 mesh shader。

在 mesh shader 中，构建 meshlet 的各种数据，传递给光栅化器。

### GPU culling using compute

算法来源：[GPU-Driven Rendering Pipelines (realtimerendering.com)](https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pdf)

步骤：

1. 使用上一帧的深度缓冲，我们渲染场景中的可见对象并执行 mesh 和 meshlet 的视锥和遮挡剔除。这可能会导致误判（false negtives），例如，在此帧中可见但之前不可见的 mesh 或 meshlet 。我们存储这些对象的列表，以便可以在下一阶段解决任何误判。
2. 上一步直接在 compute shader 中生成绘制命令列表。此列表将用于使用间接绘制命令绘制可见对象。
3. 我们现在有一个更新后的深度缓冲区，并且我们也更新了深度金字塔。
4. 我们现在可以重新测试在第一阶段被剔除的对象并生成一个新的绘制列表以消除任何误报。
5. 我们绘制剩余的对象并生成最终的深度缓冲区。这将被用作下一帧的起点，然后重复该过程。

#### Depth pyramid generation

类似 mipmap，但是不使用双线性差值的算法去计算低一级别的值，如果用差值算法，我们会计算得到场景中不存在的深度。相反，我们读取想要减少的四个 fragment 并选取最大值。

使用 compute shader 实现这个过程。因为深度从 0（靠近相机）变为 1（远离相机）。下采样时，我们希望四个样本中最远的样本避免过度遮挡(over-occluding)。

#### Occlusion culling

这个剔除步骤完全在 compute shader 中完成。

首先加载当前的 mesh，然后在 view 空间下根据传入的 `mesh_bound` （此时是 bounding_sphere）计算当前 mesh 的 bouding sphere 的位置和半径。注意这是整个 mesh 的 bounding sphere，不是 meshlet 的，我们会对 meshlet 以同样的方式处理。

接下来是 frustum culling，与 tash shader 中的算法一样。

如果 mesh 通过了 frustum culling，我们对它进行 occlusion culling。首先计算透视投影球体的边界正方形。这一步是必要的，因为投影球体的形状可能是椭圆体。算法来源：[2D Polyhedral Bounds of a Clipped, Perspective-Projected 3D Sphere (JCGT)](https://jcgt.org/published/0002/02/05/) 和 [zeux/niagara](https://github.com/zeux/niagara/)。

然后检查球体是否完全位于近平面后面。如果是，则无需进一步处理。此处要求将球体坐标的 `z` 值取反，因为我们是看向 `-z` 的方向。

接下来计算 `x` 轴的最小点和最大点。我们只考虑 `xz` 平面，找到球体在这个平面上的投影，并计算这个投影的最小和最大 x 坐标。然后对 `y` 坐标重复相同的过程。计算出的点位于世界空间中，但我们需要它们在透视投影空间中的值，所以用算法将它们转化到投影空间中。得到 `aabb` 的值。

然后将这些值转换到 UV 空间中。UV 空间的坐标在 `[0, 1]` 之间，屏幕空间的坐标在 `[-1, 1]` 之间。我们对 y 使用取反，因为屏幕空间的原点在左下角，而 UV 空间的原点在左上角。

现在有了 mesh sphere 对应的 2D bounding box，现在可以检查它是否被遮蔽。首先根据 depth pyramid 的最高层级的大小缩放刚刚得到的 bounding box，取得到的宽高中最大的那个。根据此值计算 level。这个办法将 bounding box 简化为单个像素查找。请记住，在计算金字塔的 mipmap 时，低一级的 texture 会存储最远的深度值。借助这一点，我们可以安全地查找单个片段来确定 bounding box 是否被遮挡。使用 `depth = textureLod(depthTexture, position, level)` 来获取 depth mipmap 中对应位置和 level 的深度值。

然后计算 bounding sphere 的最近的深度值。

最后检查球体的深度与从金字塔读取的深度来确定球体是否被遮挡。

如果 mesh 通过了视锥体和遮挡剔除，我们将这个 command 加入到 command list 中。我们用这个 command list 来绘制可见的 mesh 的 meshlet，并更新 depth pyramid。

最后一步是重新运行在第一遍中丢弃的网格的剔除。使用更新后的深度金字塔，我们可以生成一个新的命令列表来绘制任何被错误剔除的网格。



## Chapter 7: Rendering Many Lights with Clustered Deferred Rendering

有以下内容：

- clustered lighting 简史
- G-buffer 的设置与实现
- 使用屏幕 tile 和 Z-binning 的 clustered lightling

### A brief history of clustered lightling

2000 年之前，实时渲染程序基本上都使用 forward rendering。前向渲染的光源数一般比较少，可能的值为 4 或 8。

1988 年提出了 deferred rendering 的概念，它只对同样的像素只着色一次。

另一个关键概念 G-buffer（geometric buffer）是通过文章 《Comprehensible Rendering of 3D Shapes》 引入的。

后面 defferred rendering 被绝大多数引擎所采用。

2012 年 AMD 发布了叫做 Leo 的 demo，带来了 Forward+。它为每个屏幕空间 tile 带来了光源列表。

这些年更加精细的细分算法得到了开发，其中 tile 变成 cluster，并从简单的 2D 屏幕空间图块转变为完全截头体形状的 3D 集群。

Cluster 是个很好的点子，但是在 3D 网格中会消耗很多内存。

目前最先进的聚类技术来自 Activision，这是此书选择的解决方案，将在本章的“Implementing light clusters”部分详细介绍它。

#### Differences between forward and deferred techniques

前向和后向渲染之间的最主要区别：**light assignment**。

前向渲染的主要优点：

- 在渲染材质时有完全的自由度
- 对于不透明和透明物体使用同一个 rendering path
- 支持 MSAA
- 在 GPU 上使用较低的内存带宽

其缺点为：

- 为了减少 fragment shade 数量，需要有 depth prepass。
- 场景着色的复杂性是对象数量 (N) 乘以灯光数量 (L)。
- 着色器复杂，需要执行大量操作，GPU register 压力大，会影响性能。



延迟渲染（有时称为延迟着色）的引入主要是为了将几何图形的渲染与光计算分离开来。在延迟渲染中，我们创建了多个渲染目标。通常，我们有一个用于反照率、法线、PBR 参数（粗糙度、金属度和遮挡）和深度的渲染目标。其主要优点为：

- 减少着色复杂度
- 不需要 depth prepass
- 着色器不那么复杂

缺点为：

- 高内存使用
- 法线精度丢失：法线通常为 16 位浮点数，为了减少内存使用，通常将它压缩到 8 位
- 透明物体需要单独的 pass 并且需要使用前向渲染
- 特殊材质需要将它们的参数打包进 G-buffer



共同的问题：在处理单个对象或片段时，我们必须遍历所有 light。有两种最常用的方法来解决：tiles 和 clusters。

- Light tiles：在屏幕空间中创建一个 grid，并确定哪些灯光会影响给定的 tile。渲染场景时，我们确定要着色的片段属于哪个图块，并且我们只迭代覆盖该图块的灯光。
- Light clusters：灯光簇将视锥体细分为 3D 网格。与图块一样，灯光被分配给每个 cell，并且在渲染时，我们仅迭代给定片段所属的灯光。大多数实现会为每个光构建轴对齐边界框 (AABB) 并将它们投影到 clip 空间中。

### Implementing a G-buffer

1. 在 Vulkan 中设置多个 render target 的第一步是创建 framebuffer（存储 G-buffer 数据） 和 renderpass。为了简化这些创建，使用了 `VK_KHR_dynamic_rendering` 扩展
2. 有了此扩展，我们不需要提前创建 renderpass 和 framebuffer。只需要在 `VkPipelineRenderingCreateInfoKHR` 中指定 attachment 的格式即可，再将它链接到 `VkGraphicsPipelineCreateInfo.pNext` 中。
3. 在渲染时，不使用 `vkCmdBeginRenderPass`，而是 `vkCmdBeginRenderingKHR`。使用 `VkRenderingAttachmentInfoKHR` 数组来指定 attachment。
4. 填充 attachment 数组
5. 对于 depth attachment 也填充同样的数据结构
6. 填充 `VkRenderingInfoKHR` 结构体，传入 `vkCmdBeginRenderingKHR` 中。完成渲染后使用 `vkCmdEndRenderingKHR` 代替 `vkCmdEndRenderPass`。

本章的 G-buffer 有四个渲染目标以加上一个深度缓冲区。使用各种压缩手段，将渲染目标压缩以减轻贷款压力。

### Implementing light clusters

算法基于 [Rendering of COD:IW (activision.com)](https://www.activision.com/cdn/research/2017_Sig_Improved_Culling_final.pdf)，有以下步骤：

1. 根据相机空间中的深度值对灯光进行排序。
2. 然后，我们将深度范围划分为大小相同的 bin。
3. 接下来，如果灯光的边界框在 bin 范围内，我们会将灯光分配给每个 bin。我们只存储给定 bin 的最小和最大灯光索引，因此每个 bin 只需要 16 位，除非您需要超过 65,535 个灯光
4. 我们将屏幕划分为图块（在我们的例子中为 8x8 像素），并确定哪些灯覆盖给定的图块。每个图块将存储活动光源的位域表示。
5. 给定我们想要着色的片段，我们确定片段的深度并读取 bin 索引。
6. 最后，我们从该 bin 中的最小光源 index 到最大光源 index 数进行迭代，并读取相应的图块以查看光是否可见，这次使用 x 和 y 坐标来检索图块。

该解决方案提供了一种非常有效的方法来循环遍历给定片段的活动光源。

#### CPU lights assignment

在每帧，我们有以下步骤：

1. 按照光源的深度值将它们排序
2. 为了避免对灯光列表进行排序，我们仅对灯光索引进行排序。这个优化手段使得我们只需要上传一次光源矩阵数据即可，然后只需要更新排序后的索引。
3. 进行图块分配，使用位域数组等计算并存储光源索引。
3. 将光源位置转换到相机空间，如果光源位于相机后面则不处理
3. 计算光源的AABB的角点投影到 clip 空间中的位置，形成一个四边形
3. 计算四边形在 screen space 中的大小，如果光源在 screen 中不可见，则处理下一个光源
3. 设置光源覆盖的 tiles 对应的位域数组

然后将光源 tiles 和 bin 数据上传到 GPU 中。

#### GPU light processing

使用刚才上传的数据进行光照计算：

1. 首先决定当前 fragment 属于哪个深度 bin。
2. 从 bin 的数据中，取出最小和最大灯光索引，将它们用于光源的计算循环。
3. 然后计算 tile 位域数组中的位置，检查在此深度 bin 中是否有光源存在
4. 如果 `min_light_id` 为 0，则说明此深度 bin 中无光源，所以没有光源会影响此 fragment。计算光源的 word id 和 bit id
5. 通过 word id 和 bit id 从对应 tile 的位域数组中取得对应光源的 flag（是否 active），来查看是否来自此 bin 中的光源是否同样影响了此 tile。如果影响了此 tile，则计算每个影响光源对此 fragment 的贡献。

## Chapter 8: Adding Shadows Using Mesh Shaders
### A brief history of shadow techniques

**Shadow volumes**

被定义为三角形的每个顶点沿光线方向向无穷远处的投影，从而创建一个体积。得到的阴影非常锐利。

其问题在于需要大量 geometry 计算，填充率高。

**Shadow mapping**

从光的角度渲染场景并保存每个像素的深度。之后，当从摄像机的角度渲染场景时，可以将像素位置转换到阴影坐标系，并根据阴影图中的相应像素进行测试，以查看当前像素是否处于阴影中。

**Raytraced shadows**

为屏幕上的每个像素追踪朝向影响像素的每个光源的一条光线，并计算对该像素的最终阴影贡献。

### Implementing shadow mapping using mesh shaders

第一步是剔除灯光下的 mesh 实例。这是在计算着色器中完成的，并将保存每个灯光的可见 mesh 实例列表。mesh 实例稍后用于检索相关网格，per-meshlet culling 稍后将使用任务着色器执行。

第二步是在 compute shader 中将 meshlet 参数写入 indirect draw 命令中，以便后续渲染 meshlet 到阴影贴图。

第三步是使用 indirect mesh shader 来绘制 meshlet 到阴影贴图。使用分层的 cubemap shadow texture，每一个 layer 对应一个光源。

最后第四步是在对场景进行光照时使用这个 shadow texture。

#### Cubemap shadows

#### A note about multiview rendering

Multiview Rendering：此扩展广泛用于虚拟现实应用程序，用于在立体投影的两个视图中渲染顶点，也可以与立方体贴图一起使用。

#### Per-light mesh instance culling

首先在 compute shader 中进行粗粒度的剔除，这里使用 mesh 和其 bounding volumes 作为链接到 meshlet 的更高层次结构。测试 light 的 bounding sphere 和 mesh 的 bounding sphere 是否相交，如果相交的话，就将此 mesh 相关的 meshlet 添加。

然后输出 per-light meshlet instances，定义为 mesh 实例和 meshlet 索引的组合。

#### Indirect draw commands generation

使用计算着色器为每个光源生成一个 indirect commands list。

会写入 6 个 command，每个 command 对应一个 cubemap 面。

#### Shadow cubemap face culling

在间接绘制 tash shader 中，根据立方体贴图剔除网格小块，以优化渲染。计算给定 meshlet 的 AABB 的哪个面将在立方体贴图中可见。算法为：使用立方体贴图面法线来计算中心和范围是否包含在用于定义六个立方体贴图面之一的四个平面中。

#### Meshlet shadow rendering – task shader

使用间接绘制并使用分层渲染在不同的立方体贴图上进行写入。

以 uint 打包 light index 和 face index，传递给 mesh shader，用以检索相应的立方体贴图视图投影矩阵并写入正确的 layer。

#### Meshlet shadow rendering – mesh shader

在 mesh shader 中，我们需要检索立方体贴图数组中要写入的 layer index，以及 light index 来读取正确的视图投影变换。

由于没有关联的片段着色器，在此 mesh shader 执行完后，阴影渲染就完成了。现在可以在照明阶段的着色器中读取生成的阴影纹理。

#### Shadow map sampling

使用硬阴影的 shadow map，我们计算世界空间坐标系下的点的坐标到光源的向量来采样 cubemap，采样时同时使用 3D 方向和 layer index（每个 layer 对应一个光源）。采样得到的结果即为在此方向上此光源能照射到的最近深度。

然后计算当前 fragment 的原始深度，从光矢量中获取主轴并将其转换为原始深度，用于跟刚才获取到的最近深度进行比较。

> 这里的算法与 [点阴影 - LearnOpenGL CN)](https://learnopengl-cn.github.io/05 Advanced Lighting/03 Shadows/02 Point Shadows/) 中一样

### Improving shadow memory with Vulkan’s sparse resources

我们目前为所有光源的每个立方体贴图分配了全部内存。根据光源的屏幕尺寸，我们可能会浪费内存，因为远处和小光源无法利用阴影贴图的高分辨率。

现在实现一项优化，能够根据摄像机位置动态确定每个立方体贴图的分辨率。有了这些信息，我们就可以管理稀疏纹理，并根据给定帧的要求在运行时重新分配其内存。

稀疏纹理（有时也称为虚拟纹理，virtual textures）在 vulkan 中原生支持。

vulkan 中寻常的资源必须绑定在单个内存分配上，无法绑定到其他分配上。vulkan 暴露了两种方法来让我们可以将资源绑定到内存中的不同地方：

- Sparse resource 允许我们将资源绑定到非连续的内存分配，但需要绑定完整资源。
- Sparse residency 允许我们将资源部分绑定到不同的内存分配。

在创建资源时需要添加 flag `VK_<resourceType>_CREATE_SPARSE_RESIDENCY_BIT` 和 `VK_<resourceType>_CREATE_SPARSE_BINDING_BIT` 。

使用 vma 分配 pages。

越远的光源对片段的影响越小，所以需要的立方体贴图的分辨率也越小。

在 CPU 端计算出每个光源对应的立方体贴图的最大分辨率，使用这个分辨率的值去绑定 cubemap 的内存。

获取每个 page 的信息后，使用 `VkSparseImageMemoryBind`、`VkSparseImageMemoryBindInfo&`、`VkBindSparseInfo` 和 `vkQueueBindSparse` 来绑定 page 内存和 image。

## Chapter 9: Implementing Variable Rate Shading
可变速率着色 (VRS) 是一种允许开发人员控制片段着色速率的技术。一般速率会选择 1x1，1x2，2x1，2x2。

一种技术为“注视点渲染（foveated rendering）”：其理念是以全速率渲染图像中心，同时降低中心以外的质量。

选择速率有多种方法，本章中已经实现的是在 lighting pass 之后通过亮度检测边缘。

这个想法是降低图像中亮度均匀区域的着色率，并在过渡区域使用全速率。这种方法之所以有效，是因为与亮度更均匀的区域相比，人眼更容易注意到这些区域的变化。

此章中使用 sobel 算子来检测边缘，在实现中将对 G 值大于 0.1 的片段使用完整的 1x1 速率。

Vulkan 中通过 `VK_KHR_fragment_shading_rate` 扩展来提供 VRS 功能。在此章中，使用 image attachment 来控制 shading rate。在生成 shading rate image attachment 之后，只需要在 `VkRenderingInfoKHR::pNext` 中设置 `VkRenderingFragmentShadingRateAttachmentInfoKHR` 即可。

**Taking advantage of specialization constants**

Specialization constants 是 Vulkan 的一项功能，允许开发人员在创建管道时定义常量值。它们可以在运行时动态控制，而无需重新编译着色器。

比如在本章中，我们希望能够根据我们正在运行的硬件控制计算着色器的工作组大小，以获得最佳性能。

实现的第一步是查看 shader 是否使用了 specialization constants。

然后在解析所有变量时，我们保存specialization constants 的详细信息，以便在编译使用此着色器的管道时使用它们：

有了 specConstant 的信息后，我们可以在创建管线时修改它的值，使用 `VkSpecializationInfo` 和 `VkSpecializationMapEntry`。

 ## Chapter 10: Adding Volumetric Fog

该技术可以实时实现，其可能性源于这样的观察：雾是一种低频效应；因此渲染可以采用比屏幕低得多的分辨率，从而提高实时使用的性能。

### Introducing Volumetric Fog Rendering

体积雾：体积渲染和雾现象的结合。

#### Volumetric Rendering

这种渲染技术描述了光线穿过参与介质时所发生的视觉效果。参与介质是包含密度或反照率局部变化的体积。

我们试图描述的是光在穿过参与介质，即雾量（或云或大气散射）时如何变化。主要有以下三种现象：

- **Absorption 吸收**：当光被困在介质内而无法射出时，就会发生这种情况。这是能量的净损失。
- **Out-scattering 向外散射**：是从介质中流出（因此可见）的能量损失。
- **In-Scattering 向内散射**：这是来自与介质相互作用的光的能量。

##### Phase function

相位函数：该函数描述光在不同方向上的散射。它取决于光入射方向和出射方向之间的角度。

如果使用现实方式表示，函数会比较复杂，所以一般使用 Henyey-Greenstein 函数，它也将各向异性考虑在内了：
$$
phase(\theta) = \frac{1}{4\pi} \frac{1 - g^{2}}{(1 + g^{2} - 2g \cos\theta)^{3/2}}
$$
$\theta$ 是观察方向和光入射方向的夹角

##### Extinction

消光是描述光散射量的量。我们将在算法的中间步骤中使用它，但要应用计算出的雾，我们需要透射率。

##### Transmittance

最后一个组成部分是透射率。透射率是光通过介质的一段时的消光，它使用 Beer-Lambert 定律计算：
$$
T(A \rarr B) = e^{-\int^{B}_{A} \beta e(x) dx}
$$

#### Volumetric Fog

Bart Wronski 最聪明的想法之一是使用视锥对齐体积纹理 Frustum Aligned Volume Texture。

使用体积纹理和与标准光栅化渲染相关的数学运算，我们可以在相机视锥体和纹理之间创建映射。

新颖之处在于将信息存储在体积纹理中以计算体积渲染。此纹理的每个元素通常称为锥体素，代表截头体素。

我们将使用分布函数：
$$
Zslice = Near_{z} * (Far_{z}/Near_{z})^{slice/numSlice}
$$
步骤如 Figure 10.7 所示。

##### Data injection

第一步是数据注入。此着色器将以颜色和密度的形式将一些彩色雾添加到仅包含数据的第一个 Frustum Aligned Texture 中。

##### Light scattering

在进行光散射时，我们计算来自场景中灯光的散射。使用在 light cluster 中相同的数据结构来计算光源对每个 froxel 的贡献。

同样采样 shadow maps 来实现更加真实的光照效果。

##### Spatial filtering

为了消除一些噪音，我们仅在视锥对齐纹理的 X 和 Y 轴上应用高斯滤波器，然后传递到最重要的滤波器，即时间滤波器。

##### Temporal filtering

此过滤器真正改善了视觉效果，因为它可以在算法的不同步骤中添加一些噪声以消除一些条带。它将读取前一帧的最终纹理（集成之前的纹理），并根据某个常数因子将当前光散射结果与前一个结果混合。

完成散射和消光后，我们可以进行光积分，从而准备场景将要采样的纹理。

##### Light integration

此步骤准备另一个 Frustum Aligned Volumetric Texture 来包含雾的集成。基本上，此着色器模拟低分辨率光线步进 Ray marching，以便场景可以采样此结果。

光线行进通常从相机开始，朝向场景的远平面。视锥对齐纹理与此积分的组合为每个锥素提供了光散射的缓存光线行进，以便场景轻松采样。在此步骤中，从之前纹理中保存的所有消光中，我们最终使用比尔-朗伯定律计算透射率，并使用该定律将雾合并到场景中。

这项技术以及时间过滤是解锁该算法实时可能性的一些重大创新。在更先进的解决方案中，例如在游戏《荒野大镖客 2》中，可以添加额外的光线行进来模拟更远距离的雾。

它还允许混合雾和体积云，使用纯光线行进方法，实现几乎无缝的过渡。

##### Scene application in Clustered Lighting

最后一步是使用世界位置读取光照着色器中的体积纹理。我们可以读取深度缓冲区，计算世界位置，计算锥素坐标并采样纹理。

### Implementing Volumetric Fog Rendering

#### Data injection

我们添加三种不同的雾效应：

- Constant Fog（恒定雾）：这种雾效应在整个场景中保持**均匀的密度**。无论观察者的位置如何，雾的浓度都是一致的。这种雾通常用于创建简单的视觉效果，给场景增加一种模糊感。
- Height Fog（高度雾）：高度雾通过模拟雾粒子在地面附近的聚集，使得雾的效果更加**物理真实**。在这种情况下，雾的厚度会根据**世界空间的Y轴坐标**而变化，通常在较低的高度雾更浓，而在较高的地方雾的浓度会逐渐减小。
- Fog in a Volume（体积雾）：体积雾是一种更复杂的雾效应，它在特定的**三维空间内**定义雾的密度和分布。这种雾可以与光源相互作用，产生更真实的光照效果，如光束穿透雾的效果。体积雾通常用于需要更高真实感的场景中。

对于每一种雾，我们需要计算散射和消光并累积它们。

`scattering_extinction += scattering_extinction_from_color_density(...)`

最后将 `scattering_extinction ` 存储到 3D texture 中。

#### Calculating the lighting contribution

光照阶段将使用已在一般光照函数中使用的 Clustered Lighting 数据结构来执行。

还是使用 compute shader，其中每个线程对应一个 froxel。

首先从 injection shader 的结果中读取 scattering 和 extinction。然后使用 clustered bin 来累加光源亮度。光源部分代码与第七章中的一样。

然后将 `phase_function` 添加到 lighting factor 中。使用 lighting factor 和 scattering 计算散射后的亮度值。

最后将 散射后的亮度值和 extinction 保存到 light_scattering texture 中。

#### Integrating scattering and extinction

在着色器中进行光线步进和中间计算。它还是会写入 frustum-aligned texture，但是每个 cell 都包含从此 cell 开始，累加后的 scattering 和 transmittance。

注意我们现在使用的是 transmittance 代替了 extinction。透射率是将消光率积分到特定空间的量。对于这个 compute shader 的 dispatch 仅在 frustum texture 的 X 和 Y 轴上，读取 light scaterring texture，因为我们将积分步骤放入了循环中。

最后可以得到一个包含光线行进散射和透射率值的体积纹理，可以从帧中的任何位置进行查询，以了解该点有多少雾以及雾的颜色。

#### Applying Volumetric Fog to the scene

使用屏幕空间坐标来计算纹理的采样坐标。此函数将在延迟和前向渲染路径的照明计算的末尾使用。

首先计算采样的坐标：将当前 fragment 的屏幕坐标（x, y）和深度 raw_depth 传入 `apply_volumetric_fog()` 中，计算出对应 froxel 的坐标。使用此坐标在 volumetric_fog_texture 3D纹理中进行采样，得到 `scaterring_transmittance` 后计算得到最终的颜色。

至此，完整实现体积雾渲染的必要步骤就结束了。但是，仍然存在一个大问题：带状（banding）。

低分辨率体积纹理会增加带状问题，但这对于实现实时性能是必要的。

#### Adding filters

为了提升视觉效果，添加两个过滤器：时间过滤和空间过滤。

时间滤波器才是真正起到作用的，因为它让我们能够在算法的不同部分添加噪声，从而消除带状。空间滤波器可以进一步消除雾气。

##### Spatial filtering

使用高斯滤波器来平滑 volumetric texture 的 X 轴和 Y 轴。它将读取光散射的结果并写入此时帧中未使用的锥素数据纹理，无需创建临时纹理。

##### Temporal filtering

此着色器将采用当前计算的 3D 光散射纹理并应用时间过滤器。为此，它将需要两个纹理，一个用于当前帧，一个用于前一帧。

我们需要计算前一帧的屏幕空间位置：计算世界空间位置，并使用前一帧的 view projection 矩阵，可以得到 UVW 坐标。使用此坐标读取前一帧的 light_scattering_texture 3D texture。

在读取了这两个 3D texture 之后，我们就将这两个值混合起来，混合比例可以自定义。然后将混合后的值写回到当前帧的 light_scattering_texture 3D texture。

至此就是 Volumetric Fog 完整算法的全部步骤了。

#### Volumetric noise generation

为了将雾密度稍微分解一下，使其更有趣，我们可以对体积噪声纹理进行采样，以稍微修改密度。我们可以添加一个单执行计算着色器，在 3D 纹理中创建和存储 Perlin 噪声，然后在对雾密度进行采样时读取它。

#### Blue Noise

作为用于算法中将不同区域的采样点略微偏移的附加噪声，我们使用蓝噪声，从纹理中读取它并向其中添加时间成分。

蓝噪声对视觉感知有很大的影响。书中没有详细阐述，只是从两个通道中读取噪声值并将它映射到 `-1` 到 `1` 的范围内。



## Chapter 11: Temporal Anti-Aliasing

提高图像质量的最常见方法之一是采样更多数据（超级采样）并将其过滤到所需的采样频率。

渲染中使用的主要技术是多重采样抗锯齿 (MSAA)。超级采样使用的**另一种技术**是时间超级采样或使用来自两个或更多帧的样本来重建更高质量的图像。

本章将了解如何使用时间抗锯齿 (TAA) 实现更好的图像质量。

近年来，越来越多的游戏开始在核心中使用延迟渲染，而 MSAA 很难应用于延迟渲染，因此这项技术（指 TAA）得到了广泛应用。

在 MSAA 只会，进入了 **Post-Process Anti-Aliasing** 时代，而 PPAA 的第一个广泛使用案例是 Morphological Anti-Aliasing（MLAA）。

后续开发了其他的继续 PPAA 的技术，这些技术都基于几何边缘识别和图像增强。

开始出现的另一个方面是重新使用前几帧的信息来进一步提高视觉质量，如 Sharp Morphological Anti-Aliasing（SMAA）。它开始添加时间分量来增强最终图像。

最广泛采用的抗锯齿技术是 TAA，它本身也存在一些挑战，但非常适合渲染管道，并允许其他技术（如体积雾）通过引入动画抖动减少带状来提高其视觉质量。

TAA 现在是大多数游戏引擎（无论是商业还是私人游戏引擎）的标准。它也有自己的挑战，例如处理透明物体和图像模糊。

### Overview

TAA 基于随时间收集样本，通过对相机投影矩阵应用小偏移并应用一些过滤器来生成最终图像.

有各种数值序列可用于偏移相机，移动相机称为**抖动（jittering）**，通过抖动相机，我们可以收集可用于增强图像的额外数据。

1. 计算想要读取 velocity 的坐标，记录为 velocity Coordinates。读取当前像素位置周围的 3x3 像素并使用当前帧的**深度纹理**找到最近的像素来实现的。
2. 使用刚才计算得到的坐标，在 Velocity Texture 中读取 velocity 值。
3. 从 History Texture 中读取颜色信息。它是上一帧的 TAA 输出。
4. 读取当前场景颜色。同时还会通过读取当前像素周围的像素来再次缓存信息，以约束之前读取的历史颜色并指导最终的解析阶段。
5. History constraint 历史约束。尝试将前一帧颜色限制在当前颜色的区域内，以拒绝来自遮挡或去遮挡的无效样本。如果不这样做，就会出现大量重影。
6. Resolve 解析。我们将当前颜色和约束历史颜色结合起来，通过应用一些附加过滤器来生成最终的像素颜色。

当前帧的 TAA 结果将是下一帧历史纹理，因此我们只需每帧切换纹理（历史和 TAA 结果），而无需复制结果。

### The simplest TAA implementation

首先将抖动添加到相机，以便我们可以渲染略有不同的场景视角并收集其他数据。

然后，我们将添加运动矢量 motion vector，以便能够在正确的位置读取前一帧的颜色信息。最后，我们将重新投影 reproject，或者简单地说，读取历史帧的颜色数据并将其与当前帧数据相结合。

#### Jittering the camera

此步骤的目标是将投影相机在 x 轴和 y 轴上稍微平移一下。在 host 端的 `GameCamera` 代码中添加 `apply_jittering` 方法。

首先重置投影矩阵。然后使用 x 和 y 的抖动值创建一个平移矩阵。

最后使用平移矩阵乘以投影矩阵，要注意乘法顺序。

有这些知识可以优化 `apply_jittering` 方法，仅需将 projection 矩阵的 `m[2][0]` 和 `m[2][1]` 加上 x y 即可。

#### Choosing jittering sequences

现在我们将构建一个 x 和 y 值序列来抖动相机。通常会使用不同的序列：

- Halton
- Hammersley
- Martin Robert's R2
- Interleaved gradients

每个序列都可以为图像提供略微不同的外观，因为它会随时间改变我们收集样本的方式。

另外一个基础的事情是缓存上一帧和当前帧的 jittering 值并将它们发送到 GPU 中，以便 motion vector 能够将所有的运动考虑在内。我们在 uniform buffer 中添加 `jitter_xy` 和 `previous_jitter_xy`。

#### Adding motion vectors

现在是时候添加运动矢量来正确读取前一帧的颜色数据了。运动源有两种：相机运动和动态物体运动。

（在计算着色器中计算并保存）添加一个 R16G16 格式的 velocity texutre（此章的shader代码把这个texture叫`motion_vectors`） 来保存逐像素的运动。对于相机运动，我们将计算当前和之前的屏幕空间位置，同时考虑抖动和运动矢量。

动态网格需要在顶点或网格着色器中写入额外的输出，并在相机运动着色器中完成类似的计算。

#### First implementation code

再次运行一个 compute shader 来计算 TAA。

1. 在当前 pixel 所在的位置对 motion vectors texture 进行采样，得到 velocity（motion vector）。
2. 在当前 pixel 所在的位置采样采样得到像素的颜色。
3. 使用 motion vector 计算前一帧对应像素的位置，将前一帧的像素颜色采样出来
4. 混合这两个颜色

可以检查它是否正常工作。您应该会看到一个更模糊的图像，并且存在一个大问题：移动相机或物体时出现重影。如果相机和场景是静态的，则应该没有像素移动。这对于了解抖动和重新投影是否正常工作至关重要。

接下来对这个简单的 TAA 实现进行改进。

### Improving TAA

有五个方面可以改进 TAA：重投影 reprojection、历史采样 history sampling、场景采样 scene sampling、历史约束 history constraint、解析 resolve。

每个方面都有不同的参数需要调整，以适应项目的渲染需求——TAA 并不精确或完美，因此需要从视觉角度考虑一些额外的问题。

#### Reprojection

首先要做的是改进重投影，从而计算坐标来读取速度来驱动历史采样 history sampling 部分。

要计算历史纹理像素坐标，最常见的解决方案是获取当前像素周围 3x3 正方形中的最近像素。在本章中，我们将读取深度纹理并使用深度值来确定最近像素，并缓存该像素的 x 和 y 位置。

仅通过使用找到的像素位置作为运动矢量的读取坐标，就能使重影变得不那么明显，边缘也将更加平滑。

还有其他读取 velocity 的方法，本章中使用的方法已经是最好的平衡了。

#### History sampling

在 sample 的时候，最简单的做法就是直接根据计算位置读取历史纹理 history texture。我们也可以应用过滤器来增强读取的视觉质量。

本文中使用 Catmull-Rom 过滤器来增强采样。

获得历史颜色后，我们将对当前场景颜色进行采样，并缓存历史约束和最终解决阶段所需的信息。

如果未经过进一步处理就直接使用历史颜色，将会导致重影。

#### Scene sampling

到现在，重影不太明显但依然存在。我们可以在当前像素周围搜索以计算颜色信息并对其应用过滤器。

基本上，我们将像素视为信号，而不是简单的颜色。（本章后会提供资源深入讨论）。在这一步中，我们将缓存用于限制来自前几帧的颜色的历史边界的信息。

我们需要知道的是，我们在当前像素周围采样另一个 3x3 区域并计算约束发生所需的信息。最有价值的信息是此区域中的最小和最大颜色，方差裁剪 Variance Clipping（我们稍后会介绍）还需要计算平均颜色和平方平均颜色（称为矩）以辅助历史约束。最后，我们还将对颜色采样应用一些过滤。

#### The history constraint

基于前面的步骤，我们创建了一系列我们认为有效的可能颜色值。如果我们将每个颜色通道视为一个值，我们基本上就创建了一个有效颜色区域，我们将根据该区域进行约束。

约束是一种接受或丢弃来自历史纹理的颜色信息的方法，可将重影减少到几乎为零。

我们添加了四个约束来测试：

- RGB clamp
- RGB clip
- Variance clip
- Variance clip with clamped RGB

其中使用 Variance clip with clamped RGB 的质量是最好的。

简而言之，我们尝试在色彩空间中构建一个 AABB，以将历史颜色限制在该范围内，以便最终颜色与当前颜色相比更加合理。

TAA 着色器的最后一步是解析，或者结合当前颜色和历史颜色并应用一些过滤器来生成最终的像素颜色。

#### Resolve

我们将应用一些额外的过滤器来决定前一个像素是否可用，以及可用程度。

第一个过滤器是时空过滤器，使用之前缓存的邻近像素的最小和最大值来计算权重，此权重决定该如何混合当前帧颜色和前一帧颜色。

接下来是另外两个连着的过滤器，它们都与亮度有关，其中一个用于抑制所谓的萤火虫（指很亮的亮点），或者在强光源下图像中可能存在的非常明亮的单个像素，而第二个使用亮度差异进一步将权重转向当前或以前的颜色。

最后我们使用新计算出来的权重来混合得到最终的结果。

关于 TAA 最常见的缺陷是图像的模糊。

### Sharpening the image

简要讨论三种不同的方法来改善图像的锐化。

#### Sharpness post-processing

提高图像清晰度的方法之一是在后期处理链中添加一个简单的锐化着色器。

#### Negative mip bias

减少模糊度的一个全局方法是将 `VkSamplerCreateInfo` 结构中的 `mipLodBias` 字段修改为负数，例如 `-0.25`，从而将纹理 **mip**（纹理的逐渐变小的图像金字塔）转移到更高的值。

#### Unjitter texture UVs

采样更清晰纹理的另一种可能的解决方案是按照相机没有任何抖动的方式计算 UV

TAA 与锐化相结合，大大改善了图像的边缘，同时保留了物体内部的细节。

### Improving banding

带状问题会影响帧渲染的各个步骤。例如，它会影响体积雾和照明计算。

添加时间重投影可以平滑添加的噪声，从而成为改善图像视觉质量的最佳方法之一。

## Chapter12: Getting Started with Ray Tracing

与传统渲染管道相比，光线追踪需要不同的设置。本书用了一整张来介绍如何设置光线追踪管道。将详细介绍如何设置着色器绑定表 Shader Binding Table，以告知 API 在给定光线的相交测试成功或失败时应调用哪些着色器。

### Introduction to ray tracing in Vulkan

引入加速结构（大多数情况下实现为 BVH），包括 BLAS 和 TLAS。

引入着色器绑定表 Shader Binding Table 和 各种 ray tracing 着色器。

### Building the BLAS and TLAS

填充 `VkAccelerationStructureGeometryKHR`、`VkAccelerationStructureBuildRangeInfoKHR`、`VkAccelerationStructureBuildGeometryInfoKHR` 等结构体。使用 `VkAccelerationStructureBuildSizesInfoKHR` 获取创建 AS 所需要内存大小并创建 buffer，之后填充 `VkAccelerationStructureCreateInfoKHR` 结构体，并调用 `vkCmdBuildAccelerationStructuresKHR()` 来创建 BLAS。

之后使用类似的方法创建 TLAS。

#### Defining and creating a ray tracing pipeline

与传统渲染管线不同，光线追踪着色器可以根据着色器绑定表设置调用其他着色器。

在着色器代码中，我们必须使用 `#extension GL_EXT_ray_tracing : enable` 来开启光追功能。使用 `rayPayloadEXT` 作为变量的修饰来指定变量可以在不同着色器中被访问和修改。使用 `accelerationStructureEXT` 来指定加速结构变量。其他的关键字不再赘述。

对于 ray tracing pipeline，我们需要填充 `VkRayTracingShaderGroupCreateInfoKHR` 结构体。

最后需要创建 Shader Binding Table。使用 `vkGetRayTracingShaderGroupHandlesKHR` 从创建好的 pipeline 中获取 shader group 的handle。获取 handle 后我们可以把它们组织成不同的独立 table，使用 buffer 存储这些 table，将这些 buffer 的 deviceAddress 传给 `VkStridedDeviceAddressRegionKHR`。在调用 `vkCmdTraceRaysKHR()` 执行光追管线时，指定这些代表着 shader binding table 的 DeviceAddressRegion。