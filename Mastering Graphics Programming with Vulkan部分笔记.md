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

## Chapter 6 GPU-Driven Rendering

此章中，会有：

- 将大的 mesh 分解为小的 meshlet
- 使用 task 和 mesh shader 处理 meshlet，实现背面和视锥体剔除
- 使用 compute shader 实现有效的遮挡剔除
- 使用 indirect drawing functions 在 GPU 上生成 draw commands