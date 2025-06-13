最近在实现基于 rayquery 的路径追踪的过程中接触到了 buffer_reference，它是一个 glsl 扩展，需要 shader 端和 vulkan host 端协同使用。其官方描述为：[GLSL/extensions/ext/GLSL_EXT_buffer_reference.txt at main · KhronosGroup/GLSL (github.com)](https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GLSL_EXT_buffer_reference.txt)

> 开发过程中参照了 [SaschaWillems/Vulkan: Examples and demos for the new Vulkan API (github.com)](https://github.com/SaschaWillems/Vulkan) 中的 rayquery、raytracingtextures、raytracinggltf 等示例。

## 使用背景

首先，我们为什么需要使用 buffer_reference 呢？对于我来说，是为了在 GPU 端减少显存用量和提升数据读取速度。以我在项目中的实际使用来说，经历了以下心路历程。

首先我是使用 vulkan 的 rayquery 来实现路径追踪，使用 vulkan 的 rayquery 时需要创建 vulkan 的加速结构，为了创建加速结构，我们使用了三个扩展：`VK_KHR_buffer_device_address`、`VK_KHR_deferred_host_operations`、`VK_EXT_descriptor_indexing`。注意`VK_KHR_buffer_device_address`，有了这个扩展，我们就能获取 vulkan buffer 在 GPU 中的地址了。在创建底层加速结构的时候我们使用了模型的顶点、顶点索引等数据，并将这些顶点数据的 buffer 的地址传给了加速结构，所以记住这一点：此时**在 GPU 端我们已经有了一份顶点数据了**。

在使用 rayquery 查询到光线和加速结构的交点之后，我们可以得到光线从起点到交点的时间，利用它可以求出交点的位置，但是可能不太准确，最准确的方式是找到交点所在的三角形，利用重心坐标来求得交点的坐标。rayquery 提供给了我们交点所在的实例ID `InstanceId` 和图元ID `PrimitiveIndex`。`InstanceId` 对应着第几个底层加速结构，`PrimitiveIndex` 对应着底层加速结构中的第几个图元（一般情况下就是三角形）。

有了这些索引，我们就需要去取数据，但是去哪取呢？还记得刚刚提到的“在 GPU 端我们已经有了一份顶点数据了”吗？如果不把它用起来就太可惜了。我们之前获取到了它在 GPU 端的地址，现在可以用上了。

> 在项目开发过程中，一开始组内专家告诉我说使用 buffer_reference 来访问模型顶点数据会导致应用卡死，所以他把型顶点数据又拷贝了一份，使用 ssbo 的形式传入 shader 中，然后访问。这样有两点坏处：1，占用了更多显存；2，ssbo 速度比 ubo 慢。所以我还是想着将 buffer_reference 利用起来，过程中踩了一些坑，后来也终于用起来了，虽然看上去帧率没有提升，但是减小了显存的用量。

## 使用方式

如 vulkan example 中所示：

```glsl
// bufferReferences.glsl
layout(push_constant) uniform BufferReferences {
	uint64_t vertices;
	uint64_t indices;
} bufferReferences;

layout(buffer_reference, scalar) buffer Vertices {vec4 v[]; };
layout(buffer_reference, scalar) buffer Indices {uint i[]; };
    
// geometrytypes.glsl
struct Vertex
{
  vec3 pos;
  vec2 uv;
};

struct Triangle {
	Vertex vertices[3];
	vec2 uv;
};

// This function will unpack our vertex buffer data into a single triangle and calculates uv coordinates
Triangle unpackTriangle(uint index, int vertexSize) {
	Triangle tri;
	const uint triIndex = index * 3;

	Indices    indices     = Indices(bufferReferences.indices);
	Vertices   vertices    = Vertices(bufferReferences.vertices);

	// Unpack vertices
	// Data is packed as vec4 so we can map to the glTF vertex structure from the host side
	for (uint i = 0; i < 3; i++) {
		const uint offset = indices.i[triIndex + i] * (vertexSize / 16);
		vec4 d0 = vertices.v[offset + 0]; // pos.xyz, n.x
		vec4 d1 = vertices.v[offset + 1]; // n.yz, uv.xy
		tri.vertices[i].pos = d0.xyz;
		tri.vertices[i].uv = d1.zw;
	}
	// Calculate values at barycentric coordinates
	vec3 barycentricCoords = vec3(1.0f - attribs.x - attribs.y, attribs.x, attribs.y);
	tri.uv = tri.vertices[0].uv * barycentricCoords.x + tri.vertices[1].uv * barycentricCoords.y + tri.vertices[2].uv * barycentricCoords.z;
	return tri;
}
```

`bufferReferences.glsl` 最下面使用 `buffer_reference` 定义了两个指针类型 `Vertices` 和 `Indices`。如果用 C++ 表示的话，它们可以分别表示为：

```c++
typedef vec4* Vertices;
typedef uint* Indices;
```

然后定义了 `bufferReferences` 用来接收传入的 buffer 地址数据。最后在使用时，先调用

```glsl
Indices    indices     = Indices(bufferReferences.indices);
Vertices   vertices    = Vertices(bufferReferences.vertices);
```

使用传入的 `uint64_t` 类型的地址来初始化一个指针（或者可以使用另一个指针来初始化这个指针），然后就可以使用偏移或者 `[]` 操作符来按照指针的类型读取指针所指地址的值了。

> 更详细的解释可以参照 [游戏引擎随笔 0x24：再论现代图形 API 的 Bindless（下） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/421175854)

## 使用中碰到的坑

目前 buffer_reference 在 PC 端（主要是 nVidia 平台和 intel 平台）运行的比较良好，并且 PC 端有验证层可以比较明确的看到哪里有错误，实际开发过程中也没有碰到太多问题。

问题主要出现在安卓端。目前碰到的坑有：

##### 1. 指针不能使用 `uint64_t` 类型的地址初始化

在 vulkan sample 中，指针是通过 `uint64_t` 类型的地址来初始化的，这样做的话需要开启 glsl 的扩展 `GL_EXT_shader_explicit_arithmetic_types_int64`，此扩展能使 `uint64_t` 和 `int64_t` 类型在 glsl 中可用，如果不开启此扩展的话，shader 中就不能使用 `uint64_t`。同时设备还需要支持 `shaderInt64` 功能，此功能可以通过 `vkGetPhysicalDeviceFeatures` 来查询。

目前高通芯片的安卓机不支持 `shaderInt64`，所以无法在 shader 中 `uint64_t`，也就不能用  `uint64_t` 的地址初始化 buffer_reference 指针了。

##### 2. 指针不能通过另一个指针初始化

在 [GLSL/extensions/ext/GLSL_EXT_buffer_reference.txt at main · KhronosGroup/GLSL (github.com)](https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GLSL_EXT_buffer_reference.txt) 中提到，如果不使用 `uint64_t`类型的地址初始化指针，可以使用另一个指针来初始化它。形式如下：

```glsl
layout(buffer_reference, scalar) buffer Vertices {vec4 v[]; };
layout(buffer_reference, scalar) buffer Indices {uint i[]; };

layout(binding = 4, set = 0) buffer SceneModelInfo {
	Vertices vertices;
	Indices indices;
} sceneModelInfo[];

...

Indices  indices  = Indices(sceneModelInfo[0].indices);
Vertices vertices = Vertices(sceneModelInfo[0].vertices);
```

这样的写法在 PC 端运行良好，但是在高通芯片的安卓机上会在 `vkCreateGraphicsPipelines()` 时报错，报错类型为 `VK_ERROR_UNKNOWN`，目前还不知道原因。（后面通过 [Android 上的 Vulkan 验证层  | Android NDK  | Android Developers](https://developer.android.com/ndk/guides/graphics/validation-layer?hl=zh-cn) 开启了验证层，验证层不报告详细错误信息。）

所以目前来看，在安卓端所有的指针初始化方式都不能用，只能在 storage buffer 和 uniform buffer 中直接初始化。在安卓端的 shader 代码中也不能创建其他新的指针，只能直接访问 storage buffer 和 uniform buffer 中的指针所指的数据。如下：

```glsl
layout(buffer_reference, scalar) buffer Vertices {vec4 v[]; };
layout(buffer_reference, scalar) buffer Indices {uint i[]; };

layout(binding = 4, set = 0) buffer SceneModelInfo {
	Vertices vertices;
	Indices indices;
} sceneModelInfo[];

...

Vertices vertices = Vertices(sceneModelInfo[0].vertices); // ERROR: 安卓端无法创建pipeline
vec4 d0 = vertices.v[offset + 0]; // ERROR: 跟着上面来，无法创建pipeline也就无法访问数据了

vec4 d1 = sceneModelInfo[0].vertices.v[offset + 1]; // SUCCESS: 安卓端只能直接使用 storage buffer 和 uniform buffer 中的指针
```

PS: 后来发现安卓端可以直接给指针赋值，不会导致 pipeline 创建失败：

```glsl
Vertices vertices = sceneModelInfo[0].vertices; // SUCCESS: 可以直接给指针赋值
vec4 d2 = vertices.v[offset + 2];
```

