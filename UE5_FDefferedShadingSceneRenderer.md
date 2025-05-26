> [剖析虚幻渲染体系（04）- 延迟渲染管线 - 0向往0 - 博客园 (cnblogs.com)](https://www.cnblogs.com/timlly/p/14732412.html)

其部分声明和定义在：`Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.h`

查看源码，可以看到 `FDeferredShadingSceneRenderer `主要包含了Mesh Pass、Lumen、大气、光源、阴影、半透明、光线追踪、反射、可见性等几大类接口。

> “继续分析`FScene::UpdateAllPrimitiveSceneInfos`的过程” 这一部分，由之前的《UE5大致渲染流程》提到在 `FScene` 中添加了一个 `Update()` 函数，用于同步或异步更新场景，其逻辑与 `FScene::UpdateAllPrimitiveSceneInfos()` 基本一致。而 `FScene::UpdateAllPrimitiveSceneInfos()` 算是 `Update()` 的特化同步版本。我们就分析 `FScene::Update` 函数

`FScene::Update` 过程：

1. 首先对 `RemovedLocalPrimitiveSceneInfos` 和 `AddedLocalPrimitiveSceneInfos` 进行排序，分别对应着之前存在于场景中并将被移除的图元，和不存在场景中并将被添加的图元。
2. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RemovePrimitiveSceneInfos)`：在删除操作中，将要删除的 proxy 移到 `PrimitiveSceneProxies` 数组（在数组中，proxy 按 type 连续存储）最末端，配合 `PrimitiveSceneProxies` 和 `TypeOffsetTable`, 可以减少删除元素的移动或交换次数。然后移除 `DistanceFieldSceneData` 和 Lumen 数据中的 Primitive 及相关信息。
3. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(AddPrimitiveSceneInfos)`：处理图元增加时，将 Primitive 相关信息的那些数组增加空间。然后增加新的 Primitive 信息，将这些信息赋值为 `AddedLocalPrimitiveSceneInfos` 中的信息。更新 `TypeOffsetTable`。同样的，根据 TypeOffsetTable 通过交换操作将相同 Type 的 Primitive 放在一起。然后添加 `DistanceFieldSceneData` 和 Lumen 数据中的 Primitive 及相关信息。
4. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(UpdatePrimitiveTransform)`：更新图元变换矩阵