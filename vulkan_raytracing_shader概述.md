## 概述

在 `vulkan_sample之raytracing_basic.md` 中提到，raytracing 中有多种阶段的 shader：

- Ray Generation
- Miss
- Closest Hit
- Intersection
- Any Hit
- Callable

有了 Shader 我们可以创建着色器绑定表 SBT 来让光追管线运行时所有着色器可以随时被选择使用。

### Ray Gen

调用 `vkCmdTraceRaysKHR` 后执行光线追踪调度，首先启动的是 Ray Gen Shader，一个典型的 Ray Gen Shader 如下：

```glsl
#version 460
#extension GL_EXT_ray_tracing : require

layout(binding = 0, set = 0) uniform accelerationStructureEXT topLevelAS; // 表示场景的TLAS
layout(binding = 1, set = 0, rgba8) uniform image2D image; // 存储最终结果的图像
layout(binding = 2, set = 0) uniform ... // 一些uniform变量

layout(location = 0) rayPayloadEXT vec3 hitValue; // 光线载荷，在光追管线中进行修改。

void main()
{
    const vec2 pixelCenter = vec2(gl_LaunchIDEXT.xy) + vec2(0.5); // 计算当前发射光线对应的像素中心
    ... // 计算光线原点和方向
    traceRayEXT(topLevelAS, rayFlags, cullMask, 0, 0, 0, origin.xyz, tmin, direction.xyz, tmax, 0); // 启动光线遍历
    imageStore(image, ivec2(gl_LaunchIDEXT.xy), vec4(hitValue, 0.0)); // 将每条光线的载荷写入图像
}
```

正如注释所说，Ray Gen Shader 的大致流程为：

1. 计算当前发射光线对应的像素中心。因为如同 [Ray Tracing in One Weekend Series](https://raytracing.github.io/) 中 book 1 的 figure 3 所说，为了得到一张图像，我们将图像作为相机的 viewport，然后从相机的中心往 vieport 中的各个像素中心点发射光线，穿过每个像素的光线经过遍历后得到结果，这个结果就是各个像素的值。

![figure 3](https://raytracing.github.io/images/fig-1.03-cam-geom.jpg)

2. 计算光线原点和方向。知道了相机原点和每条相机需要穿过的像素中心点的位置，就可以算出光线方向。
3. 调用 `traceRayEXT` 启动光线遍历。这个函数有许多参数，各参数如下：

```glsl
traceRayEXT(topLevelAS,         // acceleration structure
            gl_RayFlagsNoneEXT, // rayFlags
            0xFF,               // cullMask
            0,                  // sbtRecordOffset
            0,                  // sbtRecordStride
            0,                  // missIndex
            origin,             // ray origin
            0.1,                // ray min range
            rayDir,             // ray direction
            100000.0,           // ray max range
            0                   // payload (location = 0)
           );
```

这里的 ray min/max range 对应着 `Ray Tracing in One Weekend Series` 光线遍历中经过的时间 t。

4. 将每条光线的载荷写入图像。这里涉及到两个内置变量：
   1. `gl_LaunchSizeEXT`：对应着调用 `vkCmdTraceRaysKHR` 时指明的图像尺寸；
   2. `gl_LaunchIDEXT`：对应的是正在渲染的像素的整数坐标。

> 在 glsl 中与 Ray Tracing 有关的内置变量和函数可以在 glsl 的扩展文档中 [GLSL/extensions/ext/GLSL_EXT_ray_tracing.txt at main · KhronosGroup/GLSL (github.com)](https://github.com/KhronosGroup/GLSL/blob/main/extensions/ext/GLSL_EXT_ray_tracing.txt) 中找到。glsl 本身的内置变量和函数可以在它的 spec 文档中找到，如[The OpenGL® Shading Language, Version 4.60.8 (khronos.org)](https://registry.khronos.org/OpenGL/specs/gl/GLSLangSpec.4.60.pdf)

如果光线遍历过程中无交点，则调用 Miss Shader，一般情况下 Miss Shader 会采样环境贴图。

### Intersection

Intersection Shader 允许计算光线与用户定义的几何体的相交情况。每当光线集中图元的 AABB 时，就会执行相交着色器。如果计算出相交的话，着色器需要调用 `reportIntersectionEXT` 来报告相交，然后调用当前的命中着色器，如果相交发生在当前光线间隔内，则调用与当前 intersection shader 对应的 any-hit shader 与 closest-hit shader。发生在当前光线间隔外，则拒绝命中并返回 false。

报告交点函数为：`bool reportIntersectionEXT(float hitT, uint hitKind)`，`hitT` 提交为当前光线新的 `gl_RayTmaxEXT`，即光线命中时经过的时间，`hitKind` 提交为当前光线新的 `gl_HitKindEXT`，并且返回 true。

> PS：在 any-hit 和 closest-hit shader 中可以访问 `gl_HitTEXT`，它是 `gl_RayTmaxEXT` 的别名。

在目前的情况中，只有当 shaderGroup 的类型为 `VK_RAY_TRACING_SHADER_GROUP_TYPE_PROCEDURAL_HIT_GROUP_KHR` 时，才会指定 Intersection Shader。此时可能会同时指定 any-hit shader 与 closest-hit shader。

它无法读取或者修改光线的 payload。

### Any-Hit

Intersection Shader 报告位于光线当前 `[tmin,tmax]` 内的相交后，执行任意命中着色器。 任意命中着色器的主要用途是通过编程方式决定是否接受相交。 除非着色器调用 `IgnoreIntersectionKHR` 指令，否则将接受相交。 Any-Hit Shader 对相应 Intersection Shader 生成的属性具有只读访问权限，并且可以读取或修改光线有效负载。

### Closest-Hit

如果光线遍历过程中有交点，则对最近的交点调用 Closest Hit Shader。一个最简单的 Closest Hit Shader 如下：

```glsl
#version 460
#extension GL_EXT_ray_tracing : enable
#extension GL_EXT_nonuniform_qualifier : enable

layout(location = 0) rayPayloadInEXT vec3 hitValue; // 光线载荷，在此着色器中修改
hitAttributeEXT vec2 attribs; // 光线与三角形交点的重心坐标的权重

void main()
{
  const vec3 barycentricCoords = vec3(1.0f - attribs.x - attribs.y, attribs.x, attribs.y);
  hitValue = barycentricCoords;
}
```

这里有两个要注意的地方：

1. `rayPayloadInEXT`：与 Ray Gen Shader 中的 `rayPayloadEXT vec3 hitValue` 对应，`rayPayloadEXT ` 可以在 Ray Gen、closest-hit、miss shader 中访问，当它被传给 `traceRayEXT()` 时，会变成与之相关的输入传递。即变成 `rayPayloadInEXT vec3 hitvalue`，在 closest-hit、any-hit、miss shader 中访问。
2. `hitAttributeEXT`：表示光线与三角形交点的重心权重。所谓重心权重，可以如 [图7](https://raytracing.github.io/images/fig-2.07-quad-coords.jpg) 这样理解，三角形ABC有三个顶点A、B、C，如果以A为原点(0, 0)，则B点为(0,1)，C点为(1,0)，则位于三角形ABC中的点坐标为(u,v)，其中(u>=0, v>=0, u+v <=1)。(u,v)为重心权重。

有了 `hitAttributeEXT` 怎么计算交点位置呢？其实有两种方法：

1. 直接使用光线的原始点和光线防线，还有命中时经过的时间来计算。计算公式为：`point = gl_WorldRayOriginEXT + gl_WorldRayDirectionEXT * gl_HitTEXT`

2. 可以通过访问传入的顶点数据vertex data和顶点索引数据indices data，着色器中有个内置变量 `gl_PrimitiveID` 指明的是命中的图元的 ID，这里如果传入的 BLAS 是三角形的话，则表示的是第几个三角形，如果 BLAS 是 AABB，则表示第几个 AABB。大多数情况下是三角形，则可以通过 `gl_PrimitiveID` 在顶点数据中将对应三角形的顶点数据取出来，然后通过 `hitAttributeEXT` 对应的重心坐标，对顶点数据进行重心插值即可得到焦点位置。这里有个好处是知道了顶点数据，后续可以用来采样等。

最后，在 Closest-Hit Shader 中，还可以继续发射光线，这根光线一般用来判断是否在阴影中。

### Miss

未命中着色器**可以**访问光线有效负载，并**可以**通过管道跟踪光线指令跟踪新光线，但**无法**访问属性，因为它们不与焦点有关联。

### Callable

可调用着色器可以访问 callable 负载，其工作方式与光线有效负载类似，以执行子例程工作。

Callable Shader 可以在其他 Shader 中通过执行 `ExecuteCallableKHR` 来调用。