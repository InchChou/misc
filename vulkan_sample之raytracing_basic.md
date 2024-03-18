这篇文章简单记录一下在学习 [SaschaWillems/Vulkan: Examples and demos for the new Vulkan API (github.com)](https://github.com/SaschaWillems/Vulkan) 的 raytracing basic 工程时，自己感觉需要注意的一些点。

学习时，有搜索网上的一些文章以帮助理解。

[Vulkan 光追之 Acceleration Structure - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/674891012)

[Vulkan Samples 阅读 -- Hardware Accelerated Ray Tracing（二）Reflections & Callable & Query_vkgetaccelerationstructurebuildsizeskhr-CSDN博客](https://blog.csdn.net/jiamada/article/details/119021907)

[从零开始的 Vulkan（七）：光线追踪 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/618620466)

[Vulkan ray tracing 与 rasterization 管线对比 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/632655624)

## 概述

在 vulkan 光追中，与光栅化区别最主要的两点就是场景的创建和光追管线。

光追中，创建场景时需要创建加速结构 Acceleration Structure，而不是像光栅化一样使用 vertex buffer 和 index buffer，

在光追中，渲染使用的管线是光追管线，在绑定光追管线后调用 `vkCmdTraceRaysKHR` 来启动光追管线。而在光栅化中，使用的是光栅化管线，绑定光栅化管线后调用 `vkCmdDraw` 来启动光栅化管线

> 由于 AccelerationStructure 过于长，后续在非代码块中使用 AS 代替它。

### AS

Vulkan_Sample 中，AS 的定义如下：

```cpp
struct AccelerationStructure {
	VkAccelerationStructureKHR handle;
	uint64_t deviceAddress = 0;
	VkDeviceMemory memory;
	VkBuffer buffer;
};
```

需要一个 `VkASKHR` 句柄`handle`，显存地址 `deviceAddress`，分配的 `VkDeviceMemory` 还有与之绑定成对出现的 `Vkbuffer`。

第一步需要创建 Bottom Level Acceleration Structure (BLAS)，它是最底层的描述我们物体具体的几何形状，如三角形网格、球体、盒子等。它包含了几何形状的顶点和索引数据，可以用来进行光线与物体的相交测试。所以第一步我们要输入我们的几何信息

#### accelerationStructureGeometry

```cpp
VkDeviceOrHostAddressConstKHR vertexBufferDeviceAddress{};
VkDeviceOrHostAddressConstKHR indexBufferDeviceAddress{};

vertexBufferDeviceAddress.deviceAddress = getBufferDeviceAddress(vertices.buffer);
indexBufferDeviceAddress.deviceAddress = getBufferDeviceAddress(indices.buffer);

uint32_t numTriangles = static_cast<uint32_t>(indices.count) / 3;
uint32_t maxVertex = vertices.count;

// Build
VkAccelerationStructureGeometryKHR accelerationStructureGeometry = vks::initializers::accelerationStructureGeometryKHR();
accelerationStructureGeometry.flags = VK_GEOMETRY_OPAQUE_BIT_KHR;
accelerationStructureGeometry.geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR;
accelerationStructureGeometry.geometry.triangles.sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_GEOMETRY_TRIANGLES_DATA_KHR;
accelerationStructureGeometry.geometry.triangles.vertexFormat = VK_FORMAT_R32G32B32_SFLOAT;
accelerationStructureGeometry.geometry.triangles.vertexData = vertexBufferDeviceAddress;
accelerationStructureGeometry.geometry.triangles.maxVertex = maxVertex;
accelerationStructureGeometry.geometry.triangles.vertexStride = sizeof(Vertex);
accelerationStructureGeometry.geometry.triangles.indexType = VK_INDEX_TYPE_UINT32;
accelerationStructureGeometry.geometry.triangles.indexData = indexBufferDeviceAddress;
accelerationStructureGeometry.geometry.triangles.transformData.deviceAddress = 0;
accelerationStructureGeometry.geometry.triangles.transformData.hostAddress = nullptr;
```

这里 `vertices.buffer` 与 `indices.buffer` 是之前创建出的 `VkBuffer`。然后在 `getBufferDeviceAddress` 中使用 `vkGetBufferDeviceAddressKHR()` 获取这些 buffer 在GPU中存放的位置。接下来指定的几何数据类型，这里我们使用的是三角形网络，也有其他类型比如instance实例对象，然后指定我们的三角形顶点数据，里面可能包含纹理，位置，颜色，法向量等。

#### VkAccelerationStructureBuildGeometryInfoKHR

接下来就需要把我们之前的 `accelerationStructureGeometry` 信息提供给 `VkASBuildGeometryInfoKHR`，由它来提供给 `vkGetAccelerationStructureBuildSizesKHR()`  获取 BLAS 所在gpu显存上需要分配空间的大小，将空间大小信息存入 `VkASBuildSizesInfoKHR` ，然后使用size信息调用 `createAccelerationStructureBuffer()` 来创建用于加速结构的 buffer

然后使用刚才分配的 buffer，调用 `vkCreateAccelerationStructureKHR()`，在这里我们在GPU上创建了整个BLAS的骨架，接下来就是真正build处理BLAS时候了。（调用这个函数后生成的加速结构的句柄放入了 `AS::handle`）

#### Build The BLAS

第一步我们需要选取一个真正进行build的地方，也就是"草稿"，肯定不能在骨架上，那里是保存的最终结果，因此需要一个中间区域**ScratchBuffer**

```c++
RayTracingScratchBuffer scratchBuffer = createScratchBuffer(accelerationStructureBuildSizesInfo.buildScratchSize);
```

然后再一次调用 `VkASBuildGeometryInfoKHR`

```c++
VkAccelerationStructureBuildGeometryInfoKHR accelerationBuildGeometryInfo{};
accelerationBuildGeometryInfo.sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_BUILD_GEOMETRY_INFO_KHR;
accelerationBuildGeometryInfo.type = VK_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL_KHR;
accelerationBuildGeometryInfo.flags = VK_BUILD_ACCELERATION_STRUCTURE_PREFER_FAST_TRACE_BIT_KHR;
accelerationBuildGeometryInfo.mode = VK_BUILD_ACCELERATION_STRUCTURE_MODE_BUILD_KHR;
//spec the dstAccelerationStructure
accelerationBuildGeometryInfo.dstAccelerationStructure = bottomLevelAS.handle;
accelerationBuildGeometryInfo.geometryCount = 1;
accelerationBuildGeometryInfo.pGeometries = &accelerationStructureGeometry;
// 中间区域临时缓存进行build的地方
accelerationBuildGeometryInfo.scratchData.deviceAddress = scratchBuffer.deviceAddress;

// 指定构建的范围区域，只对范围内处理提高效率
VkAccelerationStructureBuildRangeInfoKHR accelerationStructureBuildRangeInfo{};
accelerationStructureBuildRangeInfo.primitiveCount = numTriangles;
accelerationStructureBuildRangeInfo.primitiveOffset = 0;
accelerationStructureBuildRangeInfo.firstVertex = 0;
accelerationStructureBuildRangeInfo.transformOffset = 0;
std::vector<VkAccelerationStructureBuildRangeInfoKHR*> accelerationBuildStructureRangeInfos = { &accelerationStructureBuildRangeInfo };
```

注意这次和上次的区别，上次只是为了获取 size 信息，这次是有了目标 AS，来进行真正的 build，然后创建command buffer提交我们的命令，完成一次提交后删去之前的临时buffer

```c++
// Build the acceleration structure on the device via a one-time command buffer submission
// Some implementations may support acceleration structure building on the host (VkPhysicalDeviceAccelerationStructureFeaturesKHR->accelerationStructureHostCommands), but we prefer device builds
VkCommandBuffer commandBuffer = vulkanDevice->createCommandBuffer(VK_COMMAND_BUFFER_LEVEL_PRIMARY, true);
vkCmdBuildAccelerationStructuresKHR(
	commandBuffer,
	1,
	&accelerationBuildGeometryInfo,
	accelerationBuildStructureRangeInfos.data());
vulkanDevice->flushCommandBuffer(commandBuffer, queue);

deleteScratchBuffer(scratchBuffer);
```

最后再次获取 AS 在 GPU 中的地址：

```c++
VkAccelerationStructureDeviceAddressInfoKHR accelerationDeviceAddressInfo{};
accelerationDeviceAddressInfo.sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_DEVICE_ADDRESS_INFO_KHR;
accelerationDeviceAddressInfo.accelerationStructure = bottomLevelAS.handle;
bottomLevelAS.deviceAddress = vkGetAccelerationStructureDeviceAddressKHR(device, &accelerationDeviceAddressInfo);
```

> 在学习过程中，我对 Scratch Buffer 这个名字产生了疑问，在翻译软件上搜索 Scratch 是抓挠的意思，总感觉用在这里有点奇怪。后来在知乎上搜到一篇文章：[如何理解Vulkan中的Scratch Buffer这一概念 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/655998855)，Vulkan在建立BLAS的过程中会产生一些临时数据，这些临时数据需要一个Buffer来存放。而在调用Vulkan创建BLAS的接口时，Vulkan需要让我们来提供这个Buffer，即 Scratch Buffer。在这篇文章的评论区里面提到，scratch paper 有草稿纸的意思，所以 scratch paper 和 Scratch Buffer 的 Scratch 应该是“随便乱写”的意思，可以把 Scratch Buffer 理解成用于随便乱蹧的缓冲区。

#### BLAS流程总结

1. 读取模型数据index, vertex buffer，放入 VkASGeometryKHR
2. 由 VkASGeometryKHR -> VkASBuildGeoInfoKHR -> VkASBuildSizesInfoKHR，获取执行构建的内存 size 大小
3. 有 VkASBuildSizesInfoKHR -> createBLAS，分配足够大小的缓冲区来容纳加速结构（获取骨架）
4. 使用 buildScrathSize 创建临时 buffer，作为计算区域，再一次创建 VkASBuildGeoInfoKHR，这次指明目标 AS 以及临时 buffer 地址
5. 调用 vkCmdBuildASKHR 提交给 command buffer 完成构建，删去临时 buffer

创建 TLAS (Top Level Acceleration Structure) 的的流程与 BLAS 基本一致，不过是先用 `VkAccelerationStructureInstanceKHR` 引用创建好的 BLAS 的 deviceAddress，然后将 `VkAccelerationStructureGeometryKHR` 中的 `triangle` 部分改成 `instance` 引用即可。

### pipeline

Pipeline部分是rasterization pipeline 和 ray tracing pipeline之间的核心区别所在。
对于光线追踪，有两种管线可供选择：Ray Query 和Ray Tracing。
Ray Query更接近soft render的工程结构并且理解起来更容易，通过使用Compute shader进行处理，更适用于通用计算任务，如视频编解码和图像处理。
然而，Ray Tracing管线在实际工程中使用更广泛，更适合成像任务.

Ray tracing的一个挑战在于，与光栅化不同，无法根据不同的材质划分不同的渲染顺序来选择着色器。但是对于ray tracing 是不可能根据 intersect 的情况选择的，在 tracing 的过程中，**所有的shader都必须随时可用**，为解决这个问题，引入了**Shader Binding Table (SBT)**的概念.

#### pipeline 创建

光线追踪管线主要由三个部分组成：shader stage、fix-function、layout；与rasterization有明显区别的部分在shader部分。Ray tracing shader stage 中涉及到多个shader：

- Ray Generation Shader

在图像中的每一个像素都需要通过 tracing 来获取，在 soft render 中，光线的生成涉及到相机位置和 FOV（Field of View）配置。同样在光线追踪管线中，需要使用着色器内置函数 `traceRayEXT` 进行处理。

- Miss Shader

光线生成后，需要在场景中进行相交测试，判断光线与哪些物体相交。这里的光线不受材质限制，会检测所有可能的交点。当加速结构（AS）判断没有交点时，Miss 着色器开始工作。通常需要多个 Miss 着色器，因为 shadow ray 和 primary ray 需要分开处理。

- Closest Hit Shader

根据上一个着色器的相交测试结果，生成距离光线起点最近的交点，并计算光线的 radiance；类似rasterization pipeline 中 fragment shader 的作用，控制最终的图像结果。

- Intersection Shader

对于特定的图元进行 intersect test，因为有些特殊的图元可能不在 BLAS 中，所以需要自定义判断相交的条件。

- Any Hit Shader

通常用于透明场景，用于判断多个相交点中哪个点走Closest Hit shader，从而构建一些风格化效果。

- Callable Shader

可调用着色器是一个通用的着色器阶段，可以在任意光线追踪着色器阶段（包括可调用着色器自身）通过内置函数`CallShader`进行调用。任意命中着色器的入口点函数必须用[shader("callable")]标识，拥有唯一一个`inout`用户自定义参数。

而在 descriptor set 方面，ray tracing 还需要一些额外的信息，例如 TLAS 就需要通过 descriptor set 以告知 shader，大部分 descriptor set 的配置与 rasterization pipeline 类似。一般情况下我们需要有两个描述符：一个描述符指向顶层加速结构，另一个指向可以用于**写入像素数据的存储图像**，将其作为光线追踪的渲染目标。

在 RayTracing pipeline 中，每个 shader 都对应着一个 `VkPipelineShaderStageCreateInfo`，在创建好这些 Shader Module 后，可以将它们放入 shaderStages 数组中，这个数组中的条目可以被 shaderGroup 引用。一种或若干种 Shader 对应着一种 shader Group，其中有三种 group：

- `generalShader`：对应着 Ray Generation Shader、Miss Shader 或者 Callable Shader，group 的 `type` 为 `VK_RAY_TRACING_SHADER_GROUP_TYPE_GENERAL_KHR` 或 `VK_SHADER_UNUSED_KHR`。
- `closestHitShader`：对应着 Closet Hit Shader，group 的 `type` 为 `VK_RAY_TRACING_SHADER_GROUP_TYPE_TRIANGLES_HIT_GROUP_KHR`、`VK_RAY_TRACING_SHADER_GROUP_TYPE_PROCEDURAL_HIT_GROUP_KHR,` 或者 `VK_SHADER_UNUSED_KHR`。
- `anyHitShader`：对应着 Any Hit Shader，group 的 `type` 为 `VK_RAY_TRACING_SHADER_GROUP_TYPE_TRIANGLES_HIT_GROUP_KHR`、`VK_RAY_TRACING_SHADER_GROUP_TYPE_PROCEDURAL_HIT_GROUP_KHR` 或 `VK_SHADER_UNUSED_KHR`。
- `intersectionShader`：对应着 Intersection Shader，group 的 `type` 为 `VK_RAY_TRACING_SHADER_GROUP_TYPE_PROCEDURAL_HIT_GROUP_KHR`。

>  每个 shaderStage 条目对应一个 shaderGroup 条目。（暂时的理解，不知道是否为真）

所以手动构建 Pipeline 的步骤为：

1. 和常规着色器创建方式相同，将光追着色器编译并加载至 `VkShaderModule` 中；
2. 将这些 `VkShaderModule` 赋值进 `VkPipelineShaderStageCreateInfo` 中，构建关于后者的数组；
3. 创建一个 `VkRayTracingShaderGroupCreateInfoKHR` 的数组；每个创建的示例最终都会成为 SBT 中的绑定实体。此时由于 `VkPipelineShaderStageCreateInfo` 尚未分配设备地址(Device Memory)，着色器组通过它们在上述数组中的索引来链接各个着色器；
4. 上述的两个数组(通常加上一个管线布局 pipelineLayout)放入 `VkRayTracingPipelineCreateInfoKHR` 中，再使用 `vkCreateRayTracingPipelinesKHR` 创建光追管线；

#### SBT

**着色器绑定表（SBT）的作用是让光追管线运行时所有着色器可以随时被选择使用。**其本质上是一个包含着色器句柄（可能是设备地址）的表，类似于 C++ 的 vtable。一般情况下我们需要自己构建这个着色器绑定表。

在刚才我们我们创建了光追管线。接下来创建 SBT：

1. 刚才创建的光追管线将着色器组一起创建出来了，我们接下来要做的是获取着色器组句柄，并将其组装成着色器绑定表。可以使用 `vkGetRayTracingShaderGroupHandlesKHR` 来获取；
2. Vulkan 通过 `VkBuffer` 对象来分配和管理着色器绑定表，我们可以根据已有的信息，使用 `VK_BUFFER_USAGE_SHADER_BINDING_TABLE_BIT_KHR` 标志缓冲区，将着色器组句柄数据复制进去。但需要注意的是，Vulkan 无法帮我们实现着色器组句柄数据的内存对齐，需要我们自己手动完成，并满足 Vulkan 的要求，要求可以通过 `VkPhysicalDeviceRayTracingPipelinePropertiesKHR` 来获取。

```c++
uint32_t handleSize = properties.shaderGroupHandleSize;
uint32_t handleSizeAligned = alignedSize(properties.shaderGroupHandleSize, properties.shaderGroupHandleAlignment)
uint32_t groupCount = static_cast<uint32_t>(shaderGroups.size());
uint32_t sbtSize = groupCount * handleSizeAligned;
vkGetRayTracingShaderGroupHandlesKHR(device, pipeline, 0, groupCount, sbtSize, shaderHandleStorage.data())

createBuffer(..., xxxBindingTable, ...);

raygenShaderBindingTable.map();
missShaderBindingTable.map();
hitShaderBindingTable.map();
memcpy(raygenShaderBindingTable.mapped, shaderHandleStorage.data(), handleSize);
memcpy(missShaderBindingTable.mapped, shaderHandleStorage.data() + handleSizeAligned, handleSize);
memcpy(hitShaderBindingTable.mapped, shaderHandleStorage.data() + handleSizeAligned * 2, handleSize);
```

最后通过填写`VkStridedDeviceAddressRegionKHR`结构体来引用 SBT 缓冲区中存储着着色器组句柄数据的内存：

```c++
VkStridedDeviceAddressRegionKHR raygenShaderSbtEntry{};
raygenShaderSbtEntry.deviceAddress = getBufferDeviceAddress(raygenShaderBindingTable.buffer);
raygenShaderSbtEntry.stride = handleSizeAligned;
raygenShaderSbtEntry.size = handleSizeAligned;

VkStridedDeviceAddressRegionKHR missShaderSbtEntry{};
missShaderSbtEntry.deviceAddress = getBufferDeviceAddress(missShaderBindingTable.buffer);
missShaderSbtEntry.stride = handleSizeAligned;
missShaderSbtEntry.size = handleSizeAligned;

VkStridedDeviceAddressRegionKHR hitShaderSbtEntry{};
hitShaderSbtEntry.deviceAddress = getBufferDeviceAddress(hitShaderBindingTable.buffer);
hitShaderSbtEntry.stride = handleSizeAligned;
hitShaderSbtEntry.size = handleSizeAligned;
```

### 执行光线追踪

在了解完以上繁复的概念后，我们终于可以正式开始执行光线追踪了。需要注意的是，光线追踪管线的行为更接近计算管线而非光栅化管线，并且执行在渲染通道之外。

和光栅化一样，在调用 DrawCall 前需要完成一些准备工作，比如绑定管线和描述符集：

```c++
vkCmdBindPipeline(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_RAY_TRACING_KHR, pipeline);
vkCmdBindDescriptorSets(drawCmdBuffers[i], VK_PIPELINE_BIND_POINT_RAY_TRACING_KHR, pipelineLayout, 0, 1, &descriptorSet, 0, 0);
```

然后通过调用`vkCmdTraceRaysKHR`函数来执行光线追踪调度：

```c++
vkCmdTraceRaysKHR(
	drawCmdBuffers[i],
	&raygenShaderSbtEntry,
	&missShaderSbtEntry,
	&hitShaderSbtEntry,
	&callableShaderSbtEntry,
	width,
	height,
	1);
```

这个函数要求对光线生成、未命中、命中、可调用着色器绑定表分别填入对应的`VkStridedDeviceAddressRegionKHR`结构体来引用其 SBT 缓冲区；由于我们想对每个像素都发射出一条光线，所以将`width`和`height`指定为渲染区域的大小，并将`depth`设为1。

最后将存储图像的布局改变为 `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`，swapChain图像改变为 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`，使用 `vkCmdCopyImage` 将存储图像拷贝到 swapChain 图像中。再将swapChain图像改为 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` 用于显示。存储图像的布局改回 `VK_IMAGE_LAYOUT_GENERAL`，用于存储光追结果。

### 其他概念

其他概念都是 vulkan 光追引入的，但是在 raytracingbasic 中未使用到，所以只是简单描述一下。

VK_KHR_ray_query 扩展，它可以支持跟踪来自所有着色器类型的广西按，包括图形、计算和光线追踪管线。它不会引入额外的 API 入口点。只是未相关的 SPIR-V 和 GLSL 扩展（SPV_KHR_ray_query 和 GLSL_EXT_ray_query）提供 API 支持。

VK_KHR_pipeline_library 扩展，是管线库。它是一种特殊的管线，不能直接绑定和使用。但是可以被链接到其他管线。它没有直接引入任何 API 函数，也没有定义如何创建管线库。这些步骤留给其他相关扩展，目前是 `VK_KHR_ray_tracing_pipeline`。可以创建一个光追管线库，然后在一个光追管线中链接此库。

VK_KHR_deferred_host_operations 扩展，引入了一种跨多个线程分配 CPU 任务的机制。主要是允许应用程序创建和管理线程。只有特别注明延迟的操作才可以延迟。