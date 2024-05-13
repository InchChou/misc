### Disney BSDF 参数与 gltf 中的参数对应

如 [glTF™ 2.0 Specification (khronos.org)](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#materials) 所说，在 gltf 2.0 使用金属粗糙度材料模型（pbrMetallicRoughness）。在这个模型中，只包含了材质的基色 base color、金属性 metalness 和粗糙度 roughness，以及这三个属性对应的 texture。

除了金属粗糙度材料模型外，gltf 2.0 材质还包含一些其他业界通用的属性：法线贴图 normalTexture、遮挡贴图 occlusionTexture、发射贴图和系数 emissive[Texture/factor]、透明度属性 alphaMode alphaCutoff、双面属性 doubleSided。

上面这些属性已经可以应对绝大部分场景了。但是我们的目标是进一步提高渲染的真实程度，所以我们寻找更加贴近现实的算法，即 Disney BSDF，它使用了更加多的参数来控制光线在材质表面或者材质中的传递。而这些参数在 gltf 2.0 标准的材质系统中是没有的，而是以材质扩展的形式（`KHR_materials_xxxx`）提供出来的。为了使算法统一起来，我们需要将 Disney BSDF 参数与 gltf 中的参数对应起来。

| Disney BSDF 参数     | gltf 扩展参数                                          |
| ------------------ | -------------------------------------------------- |
| anisotropic        | KHR_materials_anisotropy : anisotropyStrength      |
| clearCoat          | KHR_materials_clearcoat : clearcoatFactor          |
| clearCoatRoughness | KHR_materials_clearcoat : clearcoatRoughnessFactor |
| ior                | KHR_materials_ior : ior                            |
| specular           | KHR_materials_specular : specularFactor            |
| specularTint       | unknown                                            |
| subsurface         | WIP: KHR_materials_subsurface :                    |
| scatterDistance    | WIP: KHR_materials_subsurface : scatterDistance    |
| sheen              | KHR_materials_sheen : sheenRoughnessFactor         |
| sheenTint          | unknown                                            |
| specTrans          | 存疑 KHR_materials_transmission : transmissionFactor |

在 blender 中，直接支持了 Disney BSDF，其材质被称为 Principled BSDF（原理化BSDF）。其中一些参数的中英文分别为：

| Principled BSDF | 原理化BSDF |
| --------------- | ------- |
| Subsurface      | 次表面     |
| Specular        | 高光      |
| Transmission    | 透射      |
| Coat            | 涂层      |
| Sheen           | 边缘光泽    |
| Emission        | 自发光     |


