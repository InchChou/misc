> [剖析虚幻渲染体系（04）- 延迟渲染管线 - 0向往0 - 博客园 (cnblogs.com)](https://www.cnblogs.com/timlly/p/14732412.html)

其部分声明和定义在：`Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.h`

查看源码，可以看到 `FDeferredShadingSceneRenderer `主要包含了Mesh Pass、Lumen、大气、光源、阴影、半透明、光线追踪、反射、可见性等几大类接口。

> “继续分析`FScene::UpdateAllPrimitiveSceneInfos`的过程” 这一部分，由之前的《UE5大致渲染流程》提到在 `FScene` 中添加了一个 `Update()` 函数，用于同步或异步更新场景，其逻辑与 `FScene::UpdateAllPrimitiveSceneInfos()` 基本一致。而 `FScene::UpdateAllPrimitiveSceneInfos()` 算是 `Update()` 的特化同步版本。我们就分析 `FScene::Update` 函数

`FScene::Update` 过程：

1. 首先对 `RemovedLocalPrimitiveSceneInfos` 和 `AddedLocalPrimitiveSceneInfos` 进行排序，分别对应着之前存在于场景中并将被移除的图元，和不存在场景中并将被添加的图元。