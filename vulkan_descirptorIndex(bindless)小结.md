最近在项目中遇到了 descriptor index 这个扩展，可以用它来实现 bindless，它可以让我们很方便的管理 gpu 中的资源，现在记录下来。

## 什么是bindless

问GPT，它如此回答：在图形学中，**bindless**（无绑定）是一种优化技术，旨在减少图形处理单元（GPU）在渲染过程中所需的状态切换和绑定操作。传统的图形渲染管线中，GPU需要频繁地绑定和解绑各种资源（如纹理、缓冲区等），这会带来一定的性能开销。**Bindless**技术通过允许GPU直接访问资源的指针或句柄，减少了这些绑定操作，从而提高了渲染性能。

现代图形 API（如 Vulkan、DirectX 12 和 Metal）基本上都可以实现 bindless，其中 DirectX 12 是走的最远的。

## Vulkan 中的 bindless

使用 [descriptor indexing extension](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_descriptor_indexing.html) 可以在在 Vulkan 实现 bindless 设计，该扩展在 Vulkan 1.2 中提升为核心。可以通过 [VkPhysicalDeviceDescriptorIndexingFeatures(3) (khronos.org)](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPhysicalDeviceDescriptorIndexingFeatures.html) 来查询设备是否支持，其中有三个字段比较重要：

- `runtimeDescriptorArray`：启用在 SPIR-V 中使用动态数组，就是形如 `[]` 的声明形式（`layout(binding = 0, set = 0) uniform sampler2D textures[]`）。如果没启用的话必须在 shader 代码中指定数组的大小。
- `descriptorBindingVariableDescriptorCount`：启用在 DescriptorSet Layout 的 Binding 中使用可变大小的 array of descriptor。
- `shader*NonUniformIndexing `：启用在 SPIR-V 中通过 nonuniformEXT Decoration 用于非 Uniform 变量下标索引资源数组。（这里的 `*` 指的是 sampledImage、uniform buffer 等）

当查询成功后，在创建 Vulkan 逻辑设备时启用 Descriptor Indexing 特性。

```c++
VkPhysicalDeviceDescriptorIndexingFeatures indexingFeatures{};
indexingFeatures.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DESCRIPTOR_INDEXING_FEATURES;
indexingFeatures.pNext = nullptr;
indexingFeatures.runtimeDescriptorArray = VK_TRUE;
indexingFeatures.descriptorBindingVariableDescriptorCount = VK_TRUE;
indexingFeatures.shaderSampledImageArrayNonUniformIndexing = VK_TRUE;

VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
createInfo.pNext = &indexingFeatures;
....
```

接下来在需要指定 bindless 的 binding 中的 `descriptorCount` 字段，注意这时指定的是运行时 array of descriptor 最大可能的上限，而不代表必须这些数量的资源。代码如下所示：

```c++
VkDescriptorSetLayoutBinding setLayoutBinding {};
// 描述符类型，此处为 Sampled Image
setLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE;
// 指定绑定的管线阶段为 FS
setLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;
// binding 号
setLayoutBinding.binding = 0;
// 描述符数量，此处表示最大可能使用的数量上限
setLayoutBinding.descriptorCount = 10;
```

**注意**，与 D3D12 类似，这种可变数量类型的 Binding 必须放在整个 Descriptor Set Layout 的最后。

最后在创建 Descriptor Set Layout 时指定 `VkDescriptorSetLayoutBindingFlagsCreateInfo` 结构，并在这个结构中对应的 binding flag 指定为 `VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT` 标志位，这样就完成 Bindless 的启用工作，代码如下：

```c++
VkDescriptorSetLayoutBindingFlagsCreateInfo setLayoutBindingFlags{};
setLayoutBindingFlags.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_BINDING_FLAGS_CREATE_INFO;
setLayoutBindingFlags.bindingCount = 1;
// 启用可变大小描述符数量标志位
VkDescriptorBindingFlagsEXT descriptorBindingFlags = VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT;
setLayoutBindingFlags.pBindingFlags = &descriptorBindingFlags;
// 指定 Descriptor Set Layout CreateInfo 扩展
descriptorSetLayoutCI.pNext = &setLayoutBindingFlags;
```



在 glsl 中，如果 array of descriptor 的 index 是 Non-Uniform 的，则必须加上 `nonuniformEXT` 限定符，如下：

```glsl
textures[nonuniformEXT(index + 1)]
```

### 有关 bindless 的其他功能

`VkPhysicalDeviceDescriptorIndexingFeatures` 中还指明了一些其他的功能：

- `descriptorBindingPartiallyBound`：支持部分绑定
  - **部分绑定**（Partially Bound）是指在描述符集中，某些 binding 可以不绑定资源，而只在一部分 binding 中绑定了资源。只要 shader 不访问这些没有绑定资源的 binding 即可。
  - 另外，描述符数组允许你在一个 `binding` 中绑定多个资源（如多个纹理、缓冲区等）。如果希望在描述符数组中只绑定部分资源，而其他资源槽位保持未绑定状态，那么在这一个 binding 中也需要启用部分绑定。
- `descriptorBindingSampledImageUpdateAfterBind`：指示描述符绑定是否支持在绑定后更新采样图像
  - 在提交到渲染队列后，依然可以更新 descriptor，从而使资源保持最新。在以前的情况下，更新 descriptor 会造成 commandBuffer 失效。
  - 在需要频繁更新资源或者动态调整资源绑定的情况下，这个功能很有用。比如动态的粒子系统。



另外 VK_KHR_buffer_device_address 这个扩展，可以让我们获取 vulkan buffer 在 GPU 中的地址，可以进一步增加 bindless 的灵活性。

