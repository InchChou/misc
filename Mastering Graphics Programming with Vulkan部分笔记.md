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

## Chapter 13: Revisiting Shadows with Ray Tracing
### Implementing simple ray-traced shadows

传统技术的主要问题是，它们基于从每个光源的角度捕获深度缓冲区。这对于靠近光源和相机的物体很有效，但随着我们离得越来越远，深度不连续性会导致最终结果中出现伪影。

解决此问题的方法包括过滤结果，例如使用百分比接近过滤 (Percentage Closer Filtering, PCF) 或级联阴影贴图 (Cascade Shadow Maps, CSM)。此技术需要捕获多个深度切片 - 级联在我们远离光线时保持足够的分辨率。这通常仅用于阳光，因为它需要大量内存和时间来多次重新渲染场景。在级联之间的边界上获得良好的结果也相当困难。

阴影贴图的另一个主要问题是，由于深度缓冲区的分辨率及其引入的不连续性，很难获得硬阴影（感觉这里应该是软阴影）。我们可以通过光线追踪缓解这些问题。离线渲染多年来一直使用光线和路径追踪来实现逼真的效果，包括阴影。

我们引入了光线查询 ray query，它允许我们遍历从片段和计算着色器设置的加速结构。

在使用 ray query 之后，我们不需要从每个光源视点计算立方体贴图，而是简单地从片段世界位置向每个光源投射一条射线。

在初始化光线查询相关信息之后，调用 `rayQueryProceedEXT()` 执行光线遍历，它将在发现命中或射线终止时返回。使用射线查询时，我们不必指定着色器绑定表。在本章中，只需要确定光线是否命中了任何几何体。

如果没有命中几何体，则意味着我们可以从该片段中看到我们正在处理的光源，并且可以在最终计算中考虑该光源的贡献。我们对每个光源重复此操作以获得整体阴影项。

这个实现非常简单，但它主要适用于点光源。对于其他类型的光源（例如区域光源），我们需要投射多条射线来确定可见性。随着光源数量的增加，使用这种简单的技术可能会变得过于昂贵。

### Improving ray-traced shadows

上节基于可见性实现了简单的光追阴影算法，但是对于大数量的光源表现不好。本节中实现了一个不同的算法，基于《Ray Tracing Gems》书中 “Ray Traced Shadows” 一文。

结果可能依然有噪声，我们会像 TAA 中一样，使用空间和时间滤波器来过滤噪声。

这项技术通过 3 个 pass 实现，也会利用到 motion vector。

#### Motion vectors

运动矢量用来确定给定片段上的对象在帧之间移动了多远。我们需要此信息来确定在计算中要保留哪些信息以及要丢弃哪些信息。这有助于我们避免在最终图像中出现重影伪影。

在本章中，我们计算 Motion Vector 的方式与 TAA 中的稍有不同。我们首先计算两帧之间的深度比例差异（proportional difference of depth）。

接下来，我们计算一个 epsilon 值，该值将用于确定可接受的深度变化。

最后，我们使用这两个值来判断重新投影是否成功。如果成功的话，就将 `visibility_motion` 设置为当前 ndc 坐标的 xy 值与前一帧 ndc 坐标 xy 值的差值，否则设为 `(-1, -1)`。

我们将把这个值存储在纹理中以供以后使用。下一步是计算过去四帧的可见性变化。

#### Computing visibility variance

该技术使用过去四帧的数据来确定每个片段的每个光源需要多少个样本。我们将 `visibility` 值存储在 3D RGBA16 纹理中，其中每个通道都是前几帧的 `visibility` 值。每个层存储单个光源的可见性历史记录。

在这个 pass 中，我们只需计算过去四帧中最小值和最大值之间的差异 `delta`。

我们将这个 `delta` 值存储到另一个 3D texture 中以便下一个 pass 使用。

#### Computing visibility

这个 pass 负责根据过去四帧的差异来计算每个光源要发射多少条射线。

此 pass 需要从不同的纹理读取大量数据。我们将使用本地数据存储 (Local data storage, LDS) 在着色器调用中的所有线程中缓存值

如我们在第 9 章中所述，我们需要小心同步这些写入，方法是在访问 `local_image_data` 中的数据之前放置一个 `barrier()` 调用。

接下来，我们将过滤这些数据，使其在时间上更加稳定。第一步是计算 5x5 区域中的最大值，并将结果存储在另一个 LDS 矩阵中：

计算出最大值后，我们将它们传递给 13x13 tent 滤波器（三角形分布的滤波器）。

这样做是为了消除相邻片段之间的差异，同时仍然给予我们正在处理的片段更多的权重。然后我们将这个值与时间数据（即上一帧的值）结合起来。然后更新 variation cache，将 x、y、z、w 分量依次向右移。

接下来，我们利用刚刚获得的数据来计算可见性项。首先，我们需要确定样本数量。如果上一次的重新投影失败，我们只需使用最大样本数量。如果重新投影成功，我们将获得最后一帧的样本数，并确定过去四帧的样本数是否稳定。

然后，我们将此信息与之前计算的过滤值相结合，以确定此帧的样本数。

如果过滤后的值超过给定的阈值，我们将增加样本数量。这意味着我们在过去四帧中发现了一个高方差值，我们需要更多的样本才能收敛到更好的结果。另一方面，如果样本数在过去四帧中一直保持稳定，我们就会减少样本数。

虽然这在实践中效果很好，但如果场景稳定（例如，当相机没有移动时），则样本数可能会达到 0。这会导致场景不亮。因此，如果过去四帧的样本数也为 0，我们会将样本数强制为 1。

现在我们知道需要多少个样本，我们可以继续计算可见性值。使用 `get_light_visibility`，这个函数会在场景中发射光线，其实现如下所述：

- 首先计算一些参数如： fragment 到光源的向量 `position_to_light` 及其归一化向量 `l`、法线 `normal` 与 `l` 的点乘 `NoL`、fragment 到光源的距离 `d`。
- 接下来，只有当光线足够接近并且不在该片段的几何体后面时，我们才会追踪穿过场景的光线。
- 对于每一个样本，追踪一条光线。为了确保结果随时间收敛，我们使用预先计算的泊松圆盘来计算射线方向。
- 有了射线方向后，就可以调用 `rayQueryEXT`、`rayQueryInitializeEXT()`、`rayQueryProceedEXT()` 来进行光线遍历。
- 累加 `visibility` 结果，然后使用 `sample_count` 求平均值。
- 现在我们有了此帧的可见性值，我们需要更新可见性历史缓存。如果重新投影成功，我们只需添加新值到 `last_visibility_values`；如果投影失败，我们使用新值覆盖旧值。
- 最后一步是更新 `sample_count` cache。

#### Computing filtered visibility

如果直接用上一节计算出来的 `visibility` 值，输出中会有很多噪声。对于每一帧，我们可能有不同的样本数和样本位置，尤其是在相机或物体移动的情况下。因此，我们需要在使用结果之前对其进行清理。一种常见的方法是使用降噪器 denoiser。

在本章中，我们将使用一个简单的时间和空间过滤器来减少此技术所需的时间。

我们首先读取此片段中给定灯光的历史可见性数据，然后简单计算平均值，这是时间过滤器。

对于空间过滤，我们将使用高斯核。

在计算得到每个光源对应的 `visibility` 值后，我们将会在 lighting pass 中使用它来进行光照计算。

#### Using the filtered visibility

我们仅需要将 `visibility` 读取，然后在计算 `attenuation` 后乘上此项即可。

可以将传统的阴影贴图与我们在此处描述的光线追踪实现相结合。这完全取决于该技术的帧预算、场景中的灯光类型以及所需的质量。

## Chapter 14: Adding Dynamic Diffuse Global Illumination with Ray Tracing

在本章中，我们将通过添加**间接照明**来增强照明，在视频游戏中通常称为全局照明。

### Introduction to indirect lighting

从渲染的角度来看，我们使用 G-buffer 信息来计算与我们视点可见的表面之间的第一次光交互，但是我们对视野之外的情况却几乎没有数据。

对于间接照明，仅依靠相机的视点是不够的，因为我们需要计算其他光源和几何形状如何发挥作用并仍然影响场景的可见部分但在视图之外以及可见表面。

就此而言，光线追踪是最好的工具：它是一种空间查询场景的方法，因为我们可以用它来计算不同光线的反射如何影响给定片段的最终值。

在下一节中，我们将讨论我们选择的实现：**动态漫反射全局照明（Dynamic Diffuse Global Illumination, DDGI）**，它主要由 Nvidia 的研究人员开发，但正在迅速成为 AAA 游戏中使用最广泛的解决方案之一。

### Introduction to Dynamic Diffuse Global Illumination (DDGI)

DDGI 基于两个主要工具：光探测器和辐照度体积： 

- **光照探针（Light probes）**是空间中的点，表示为球体，用于编码光信息。
- **辐照度体积(Irradiance volumes)**定义为包含光探测器三维网格的空间，这些网格之间具有固定间距。

探针使用八面体映射进行编码，这是一种将正方形映射到球体的便捷方法。

DDGI 背后的核心理念是使用光线追踪动态更新探测器：对于每个探针，我们将投射一些光线并计算三角形交叉点处的辐射度。辐射度是使用引擎中存在的动态光源计算的，可实时响应任何光源或几何形状的变化。

由于网格的分辨率与屏幕上的像素相比较低，因此唯一可能的照明现象是漫射照明。

该算法的简要步骤为：

1. 对每个探针进行光线追踪，并计算辐射度和距离。
2. 使用应用一些滞后时计算出的辐射度来更新所有探头的辐射度。
3. 使用光线追踪过程中计算出的距离更新所有探测器的可见性数据，同样带有一些滞后。
4. （可选）使用光线追踪距离计算每个探头的偏移位置。
5. 通过读取更新的辐照度、可见度和探针头偏移来计算间接照明。

#### Ray tracing for each probe

这是算法的第一步。对于每个需要更新的探测器的每条射线，我们必须使用动态照明对场景进行射线追踪。

在 hit shader 中，我们计算命中三角形的世界位置和法线，并执行简化的漫反射光照计算。 可选地，但更昂贵的是，我们可以读取其他辐照度探测器，为光照计算添加无限数量的反射，使其看起来更加逼真。

这里特别重要的是纹理布局：每行代表单个探测器的光线。因此，如果每个探测器有 128 条光线，我们将有一行 128 个纹素，而每列代表一个探测器。

因此，具有 128 条射线和 24 个探针的设置将产生 128x24 的纹理尺寸。我们将照明计算作为辐射度存储在纹理的 RGB 通道中，并将命中距离存储在 Alpha 通道中。

命中距离将用于帮助解决漏光问题和计算探头偏移。

#### Probes offsetting

探针偏移是将辐照体积加载到世界中或其属性发生更改（例如间距或位置）时执行的一个步骤。使用来自光线追踪步骤的命中距离，我们可以计算探测器是否直接放置在表面上，然后为其创建偏移。

偏移量不能大于与其他探测器距离的一半，以便网格仍然保持网格索引和其位置之间的某种一致性。此步骤仅执行几次（通常，大约五次是合适的次数），因为连续运行将无限移动探测器，从而导致灯光闪烁。

几何图形内部的探测器不仅不会对采样产生照明贡献，而且还会产生视觉伪影。

#### Probes irradiance and visibility updates

现在，我们已经获得了应用动态照明后每个探测器跟踪的每条射线的结果。我们如何对这些信息进行编码？如 "Introduction to Dynamic Diffuse Global Illumination" 部分所述，其中一种方法是使用八面体映射，将球体展开为矩形。

鉴于我们将每个探测器的辐射度存储为 3D 体积，我们需要一个包含每个探测器矩形的纹理。我们将选择创建一个单独的纹理，其中一行包含**一层**探测器，大小为 `M*N`，而高度包含其他层。

例如，如果我们有一个 3x2x4 探针网格，则每行将包含 6 个探针（3x2），最终的纹理将有 4 行。我们将执行此步骤两次，一次从辐射度更新辐照度，另一次从每个探针的距离更新可见性。

visibility 对于最大限度地减少漏光至关重要，irradiance 和 visibility 存储在不同的纹理中，并且可以具有不同的大小。

需要注意的一点是，为了添加对双线性过滤的支持，我们需要在每个矩形周围存储一个额外的 1 像素边框.

着色器将读取新计算的 radiance 辐射度和距离以及前一帧的 irradiance 辐照度和 visibility 可见性纹理来混合值以避免闪烁，就像体积雾使用简单的滞后进行时间重新投影一样。

如果照明条件发生剧烈变化，可以动态改变滞后，以抵消使用滞后带来的缓慢更新。结果通常对光线移动的反应较慢，但这是避免闪烁所需的缺点。

着色器的最后一部分涉及更新双线性过滤的边框。双线性过滤要求按特定顺序读取样本，如图 14.6 所示，最外层的网格以一定顺序拷贝内侧矩形中的像素值。

我们将运行两个不同的着色器 - 一个用于更新探针辐照度，另一个用于更新探针可见性。

#### Probes sampling

此步骤涉及读取 irradiance 辐照度探针并计算间接照明贡献。我们将从主摄像头的角度进行渲染，并根据世界位置和方向对八个最近的探针进行采样。visibility 可见性纹理用于最大限度地减少光泄漏并柔化照明效果。

考虑到漫反射间接分量的柔和照明特性，为了获得更好的性能，我们选择以四分之一分辨率进行采样，因此我们需要格外注意采样的位置，以避免像素不准确。

探测光线追踪、辐照度更新、可见性更新、探针偏移和探针采样，我们描述了实现 DDGI 所需的所有基本步骤。

### Implementing DDGI

首先定义光线载荷，也就是执行射线追踪查询后缓存的信息：

```glsl
struct RayPayload {
    vec3 radiance;
    float distance;
};
```

#### Ray-generation shader

它使用球面斐波那契数列，在球体上使用随机方向从探测器位置生成射线。

使用随机方向和时间累积（发生在探测更新着色器中）可以让我们获得更多有关场景的信息，从而增强视觉效果。

#### Ray-hit shader

这里是需要做的工作是最多的。

首先，我们必须声明有效载荷和重心坐标来计算正确的三角形数据。

然后，如果相交的三角形是背向的，则不需要进行光照计算，仅存储距离。如果是正向，则需要光照计算，进行下列操作。

- 读取 mesh instace data 和 index buffer，然后读取 vertices 并计算它们在世界空间的位置。
- 以类似的方式，读取 UV 坐标，然后根据 glsl 光追扩展提供的的重心坐标，重心插值计算出命中点实际的 uv 坐标
- 使用刚才计算的 uv 坐标，从 albedo texture 中读取命中点反照率（这一点书中有误）。
- 读取三个顶点的法线值，然后也利用重心插值计算出命中点的法线值。
- 再将法线值转换到世界空间中。使用命中点坐标 `world_position`、命中点反照率 `albedo`，命中点法线 `normal`、光源位置 `light.world_position`，计算出直接光照的漫反射项 `diffuse`。
- 最后存储 `diffuse` 项为 `attenuation` 项，存储相机到命中点的距离 `distance`。同时将它们放入光追载荷中。

#### Ray-miss shader

在这个 shader 中，我们仅返回天空或者天空盒的颜色。

#### Updating probes irradiance and visibility shaders

这个 shader 是 compute shader，将读取前一帧的辐照度/可见度和当前帧的辐射度/距离，并更新每个探针的八面体表示。该着色器将执行两次 - 一次用于更新辐照度，一次用于更新可见度。它还将更新边框以添加对双线性过滤的支持。

首先，我们必须检查当前像素是否是边框。如果是，我们必须特殊处理。

对于非边界像素，根据射线方向和八面体坐标编码的球体方向计算权重，并将辐照度计算为辐照度的总权重。

添加每条射线的贡献，读取此射线的距离，如果背面太多则提前退出。此时，根据我们是否更新辐照度或可见性，我们执行不同的计算：

- 对于 irradiance，我们根据该条射线的采样位置，从 radiance output texture 中读取 radiance，计算能量衰减。
- 对于 visibility，从 radiance output texture 中读取 distance，然后限制它到 `probe_max_ray_distance` 范围内，根据 distance 和权重计算可见性值。

最后我们将计算结果与权重结合起来。

现在我们可以读取前一帧的辐照度或可见性，并使用滞后将其混合。

此时，我们结束非边界像素的着色器。我们将等待 local group 完成并将像素复制到边界。

接下来对边界像素进行操作。鉴于我们正在处理与每个方块一样大的本地线程组，当一个线程组完成时，我们可以复制带有当前更新数据的边框像素。这是一个优化过程，可帮助我们避免调度另外两个着色器并添加障碍以等待更新完成。

最终的辐照度/可见度存储在纹理中，因此我们可以复制边界像素以添加双线性采样支持。如图 14.6 所示，我们需要按特定顺序读取像素以确保双线性过滤正常工作。首先计算要拷贝的源像素的位置，然后将其拷贝并保存到 texture 的对应位置。

#### Indirect lighting sampling

该计算着色器负责读取间接辐照度，以便照明可以使用。它使用名为 `sample_irradiance` 的辅助函数，该函数也在光线命中着色器中用于模拟无限反射。

这里使用四分之一分辨率，循环遍历 2x2 像素的邻域并获取最接近的深度，然后保存像素索引。根据保存的像素索引同样读取法线。

根据 `screen_uv` 和最接近的深度 `raw_depth` 计算得到当前像素的世界空间坐标 `pixel_world_position`。再使用 `pixel_world_position`、`normal` 和 `camera_position` 去采样并计算得到最终的 irradiance，此处是调用 `sample_irradiance()` 函数。这个函数完成实际的重载工作。

它首先计算一个偏置向量来移动采样，使其稍微位于几何体的前面，以帮助解决阴影泄漏问题。然后计算采样点在空间网格中的位置（用 grid index 表示），有了这个位置之后就知道该采样哪些探针了。通过探针的位置和采样点的位置，计算出对应的 per-axis 值（以 `alpha` 表示）。有了这些信息，我们就可以从临近的 8 个探针中采样了。

对于每个探针，我们从它们的索引中算出对应的世界坐标。然后根据 grid cell 顶点计算三线性插值的权重，以便能实现探针之间的平滑过渡。

接下来就能看到 visibility texture 的使用，它存储了深度信息，有助于缓解光泄露。

首先计算从光探针位置到偏移后的片段位置的向量 `probe_to_biased_point_direction`，将此向量归一化。然后通过此向量、探针索引信息、visibility texture 的大小信息，计算出探针的 UV。通过此 UV 在 visibility texture中采样，得到 visibility 值。

有了 visibility 值，就可以检查探针是否位于 "阴影" 中，并计算切比雪夫权重。

利用计算出的该探针的权重，我们可以应用三线性偏移，读取辐照度，并计算其贡献。

对所有探测器进行采样后，最终辐照度将进行相应缩放并返回。

有了最终辐照度，我们就可以计算间接漫反射了，需要修改 shader 中的 `calculate_lighting` 函数。

#### Modifications to the `calculate_lighting` method

在这个函数中添加一些代码，读取 indirect lighting texture，然后按照 PBR 的算法计算出最终的间接光照分量。最后将间接光照分量与 G-buffer 中的 albedo 结合起来，得到 `final_colors`，这个值就是最终渲染出来的片段值。

#### Probe offsets shader

该计算着色器巧妙地利用来自光线追踪过程的每条光线的距离来根据背面和正面计数计算偏移量。每个 shader 调用代表着处理一个探针。

首先，我们必须检查无效的探测索引，以避免写入错误的内存。即 invocation id 大于探针总数的时候直接 return，不做处理。

然后，根据计算出的光线追踪距离来寻找正面和背面命中。对于此探针的每条射线，读取距离并计算它是正面还是背面。我们在命中着色器中存储背面的负距离。

现在知道了此探测器的正面和背面索引和距离。假设我们逐步移动探测器，读取前一帧的偏移量（通过直接读取 probe_offset_texture 来实现，因为此时这个 texture 中保存的就是上一帧的偏移量）。

现在，我们必须检查探针是否可以被视为几何体内部（当 1/4 数目的光线命中的是背面的话，视为在内部），并计算远离该方向但在探针间距限制内（我们可以称之为 cell）的偏移量。计算偏移量时，先计算最近背面的方向，我们向着此方向的相反方向把探针往外偏移。

然后找到在 cell 内部移动探针的最大偏移量。

如果我们没有击中背面，我们必须稍微移动探头以将其置于静止位置。

仅当偏移量在间距内或单元格限制内时才更新偏移量。然后将该值存储在适当的纹理中（即 probe_offset_texture 中）。

这个 shader 巧妙地使用已有的信息（在本例中为每个射线探测距离）将探针移出相交几何体。

## Adding Reflections with Ray Tracing

在光追出来之前，反射一般通过屏幕空间来实现，但是有个问题是，是这种实现只能使用屏幕空间信息，如果有一条光线命中了屏幕空间中没有的几何体，它就没法利用起来了。由于上述限制，这种算法的缺点在于它的一致性不好，需要依赖于相机位置。

有了光追，我们就能利用屏幕空间以外的信息，但消耗较高。

为了降低这种技术的成本，开发人员通常以一半的分辨率跟踪反射，或者仅在屏幕空间反射失败时才使用光线跟踪。另一种方法是在光线跟踪路径中使用较低分辨率的几何图形来降低光线遍历的成本。在本章中，我们将实现仅使用光线跟踪的解决方案，因为这可以提供最佳质量的结果。然后，在此基础上实现前面提到的优化将变得很容易。

### How screen-space reflections work

最常见的方法之一是在 G-buffer 区数据可用后对场景进行光线追踪。表面是否会产生反射取决于材料的粗糙度。只有粗糙度较低的材料才会发出反射。这也有助于降低该技术的成本，因为通常只有少量表面才能满足此要求。

Ray marching 在第十章被介绍过了，是一种将光线沿一个方向步进的技术。为了获得最好的质量，我们需要大量的迭代和较小的步长，但这会使该技术过于昂贵。折衷方案是使用能够提供足够好结果的步长，然后将结果通过去噪滤波器，以尝试减少低频采样引入的伪影。

顾名思义，该技术在屏幕空间中工作，类似于屏幕空间环境光遮蔽 (SSAO) 等其他技术。对于给定的片段，我们首先确定它是否会产生反射。如果会产生反射，我们会根据表面法线和视线方向确定反射光线的方向。

接下来，我们沿着反射射线方向移动给定的迭代次数和步长。在每一步中，我们都会检查深度缓冲区以确定是否击中任何几何体。由于深度缓冲区的分辨率有限，通常我们会定义一个增量值来确定我们是否将给定的迭代视为命中。

如果光线深度与深度缓冲区中存储的值之间的差异低于此增量，我们可以退出循环；否则，我们必须继续。此增量的大小可能会有所不同，具体取决于场景的复杂性，并且通常需要手动调整。

如果ray marching 循环击中可见几何体，我们会查找该片段的颜色值并将其用作反射颜色。否则，我们要么返回黑色，要么使用环境贴图确定反射颜色。

### Implementing ray-traced reflections

算法的大概流程为：

1. 从 G-buffer 开始处理。我们检查给定片段的粗糙度是否低于某个阈值。如果是，我们进入下一步。否则，我们不会进一步处理此片段。
2. 为了使该技术实时可行，我们每个片段只投射一条反射射线。我们将演示两种选择反射射线方向的方法：一种是模拟镜面，另一种是针对给定片段采样 GGX 分布。
3. 如果反射光线照射到某个几何体上，我们需要计算其表面颜色。我们向通过重要性采样选定的光源发射另一条光线。如果选定的光源可见，我们使用标准照明模型计算表面颜色。
4. 由于我们每个片段仅使用一个样本，因此最终输出将带有噪声，尤其是因为我们在每一帧随机选择反射方向。因此，光线追踪步骤的输出将由降噪器处理。我们实施了一种称为**时空方差引导滤波 (spatiotemporal variance-guided filtering，SVGF)** 的技术，该技术是专门为这种用例开发的。该算法将利用空间和时间数据来产生仅包含少量噪声的结果。
5. 最后，我们在照明计算过程中使用去噪数据来检索镜面颜色。

如步骤 4 所述，步骤 3 得到的结果噪声很多，不能直接用于我们的照明计算。在下一节中，我们将实现一个降噪器，以帮助我们消除大部分噪音。

### Implementing a denoiser

为了使反射通道的输出可用于照明计算，我们需要将其通过降噪器。我们实施了一种名为 SVGF 的算法，该算法旨在重建路径追踪的颜色数据。

SVGF 包含三个主要过程：

1. 首先，我们计算亮度的积分颜色和矩。这是算法的时序部分。我们将前一帧的数据与当前帧的结果相结合。
2. 接下来，我们计算方差的估计值。这是使用我们在第一步中计算出的一阶矩和二阶矩值来完成的。
3. 最后，我们执行五次小波滤波器。这是算法的空间部分。在每次迭代中，我们应用 5x5 滤波器以尽可能减少剩余噪声。