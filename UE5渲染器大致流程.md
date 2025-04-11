[UE5【理论】2.延迟渲染管线DeferredShadingPipeline - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/574117143)



在 `UE5杂项笔记.md` 中提到，引擎初始化完毕后，会调用`EngineTick()` 来对整个引擎进行 tick 更新引擎的状态，包括绘制场景，更新物体等等，这里的 tick 是引擎的核心部分。我们先看它是如何调用渲染器渲染场景的。

> - `FEngineLoop::Tick() ->` `GEngine->Tick()`
>   - `UUnrealEdEngine/UGameEngine::Tick() ->`
>     - `UEditorEngine::UpdateSingleViewportClient()/UGameEngine::RedrawViewports() ->`
>       - `FViewport::Draw() ->`
>         - `UGameViewportClient/FEditorViewportClient::Draw() ->`
>           - `GetRendererModule().BeginRenderingViewFamily()->`  `GetRendererModule()` 返回一个 `FRendererModule` 的实例
>             - `FSceneRenderer::CreateSceneRenderers() ->`
>               - `new FDeferredShadingSceneRenderer/FMobileSceneRenderer`
>             - `ENQUEUE_RENDER_COMMAND(FDrawSceneCommand) ->` 发送 `FDrawSceneCommand` 命令到渲染线程
>               - 渲染线程中 `RenderViewFamilies_RenderThread() ->`
>                 - `FDeferredShadingSceneRenderer::RenderHitProxies()/Render()` 

`RenderViewFamilies_RenderThread()` 中调用了 `SceneRenderer->RenderHitProxies()/Render()`。在最后这里 `Render` 是渲染整个场景的，`RenderHitProxies` 是渲染鼠标点击物体场景的，算是Render的简单版本。

> 关于 HitProxy 知乎上有文：[Learning UE5: HitProxy 原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/670360410)

`SceneRenderer` 是一个 `FSceneRenderer` 的子类的实例。`SceneRendering.h` 与 `SceneRendering.cpp` 中定义了 `FSceneRenderer` 类和它的一个子类 `FMobileSceneRenderer`（用于移动端渲染），`DeferredShadingRenderer.h/cpp` 中定义了另一个子类 `FDeferredShadingSceneRenderer`，用于延迟渲染。后续主要基于延迟渲染分析。

这里分析 `FDeferredShadingSceneRenderer::Render()` 流程（[UE5中的Render函数到底做了什么？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/642093164)）。

###  `FDeferredShadingSceneRenderer::Render()`

> [Unreal 渲染管线原理机制源码剖析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/641367884)
>
> [UE5【理论】2.延迟渲染管线DeferredShadingPipeline - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/574117143)
>
> 在 UE5.3 中，重构了 `Render()` 函数，将其中的第一步流程 `Scene->UpdateAllPrimitiveSceneInfos()` 工作包装成了一个 task，随后又经过了多次修改。主要的有：
>
> 1. [Commit 7173e0f](https://github.com/EpicGames/UnrealEngine/commit/7173e0f8fda45d874f91871e7bf1c8cb78317e71) 将 Render 函数中的 `Scene->UpdateAllPrimitiveSceneInfos()` 工作放入了新建的 `FSceneRenderer::UpdateScene()` 中
> 2. [Commit b9494a7](https://github.com/EpicGames/UnrealEngine/commit/b9494a7040175a19a2464c1db9552848b33774d5) 在 `FSceneRenderer::UpdateScene()` 将图元剔除放入异步task中（`BeginInitVisibility()`）
> 3. [Commit 9cd7556](https://github.com/EpicGames/UnrealEngine/commit/9cd755694f97946ad0e84806250d9fdf428cefc7) 将 InitView 重构为 TaskGraph，表现为 `FSceneRenderer::UpdateScene()` 中的 `BeginInitVisibility()` 变为 `LaunchVisibilityTasks()`
> 4. [Commit 7c7cff1 ](https://github.com/EpicGames/UnrealEngine/commit/7c7cff174ab43224090c54b36c7562248bf3d8ac) 将 `FSceneRenderer::UpdateScene()` 又重构成了 `FSceneRenderer::OnRenderBegin()`，将 `Scene->UpdateAllPrimitiveSceneInfos()` 放在 Landscape 和其他扩展调用及  `LaunchVisibilityTasks()` 之前。PS：此时将 `RenderFinish` 也改成了 `OnRenderFinish`，此时 RenderBegin 和 RenderFinish 可以都看作是事件 Task。
> 5. [Commit 99b678a](https://github.com/EpicGames/UnrealEngine/commit/99b678aba2cb915c1c25e325c29ba234e8625fd0) 将 View-Independent GPU Scene 更新移到 initViews 之前，以减少关键路径。即将 `Scene->GPUScene.Update()` 从 `Render()` 中移动到了 `FDeferredShadingSceneRenderer::BeginInitViews()` 中
> 6. [Commit 04d2b65](https://github.com/EpicGames/UnrealEngine/commit/04d2b6543a9241546756a9da52f1df706248c46d) 致力于通过 GPU 场景更新解决可见性重叠问题。这里在 `FScene` 中添加了一个 `Update()` 函数，用于同步或异步更新场景，而 `FScene::UpdateAllPrimitiveSceneInfos()` 算是 `Update()` 的特化同步版本，直接使用默认的同步参数调用 `Update()` 函数，同时无回调函数。以前版本的 `UpdateAllPrimitiveSceneInfos()` 可以在 [RendererScene.cpp_commit_56d764b](https://github.com/EpicGames/UnrealEngine/blob/56d764b0e71cddb66fdbb5dfc47f81de61c19465/Engine/Source/Runtime/Renderer/Private/RendererScene.cpp#L5416C1-L6666C2) 中找到。`OnRenderBegin()` 中将原本 `UpdateAllPrimitiveSceneInfos()` 之后的一些处理，如  Landscape 和其他扩展调用及  `LaunchVisibilityTasks()` 包装成 SceneUpdateParameters 中的回调函数，将 SceneUpdateParameters 传递给 `FScene::Update()` 函数。在 `Update` 函数中处理完原本 `UpdateAllPrimitiveSceneInfos()` 的逻辑后，调用回调函数，同时将 VisibilityTaskData 以 Lambda 捕获的方式传递回 `OnRenderBegin()`。

1. `FSceneRenderer::OnRenderBegin()`：进行渲染前更新
   1. 准备 `EUpdateAllPrimitiveSceneInfosAsyncOps` flag
   2. 准备 `FScene::FUpdateParameters SceneUpdateParameters` 参数：设置回调函数给它的 `Callbacks.PostStaticMeshUpdate`，回调函数中调用如下：
      1. `RayTracing::OnRenderBegin()`：更新光追相关的资源和 view
      2. `ViewFamily.ViewExtensions[ViewExt]->PreRenderViewFamily_RenderThread()`：针对每个ViewExtension，更新其ViewFamily
      3. 准备viewRect和viewState
      4.  `LaunchVisibilityTasks()` 启动了任务进行可见性判断，将返回的 `FVisibilityTaskData VisibilityTaskData` 通过捕获的方式传出到 `OnRenderBegin()`，再返回给 `Render()`。执行视锥体剔除Frustum Cull、遮挡剔除Occlusion Cull、相关性计算Compute Relevance
         1. 收集动态网格元素
         2. 设置 Mesh Passes
   3. `FScene::Update()`：使用 `SceneUpdateParameters` 参数调用当前场景的 `Update()` 函数，更新场景。这里的 `FScene::Update()` 函数与文章中的 `FScene::UpdateAllPrimitiveSceneInfos()` 函数流程基本一样，不过在最后会运行 `PostStaticMeshUpdate` 回调函数。
      1. `GPUSkinCache->DoDispatch()`：如果有 GPU 蒙皮，调用 GPU 更新它
      2. `RDG_EVENT_SCOPE(GraphBuilder, "UpdateAllPrimitiveSceneInfos")`：标志着图元场景信息开始更新了
      3. `SceneUpdateChangeSetStorage.PrimitiveUpdates.ForEachCommand(Lambda)`：获取场景更新中常用的类别。
      4. `UpdateAllLightSceneInfos()` 更新光照信息
      5. `UpdateRayTracingGroupBounds_XXX()`：更新光追组绑定
      6. `GPUScene.OnPreSceneUpdate()`：根据GPUScene中的 `PrimitiveUpdates` 中的command，更新图元的Dirty状态
      7. `SceneCulling->BeginUpdate()`：进行 SceneCulling 更新，会向RDG中添加 ComputeExplicitCellBounds Pass，然后根据计算结果调用一个task用来标记需要剔除的instance
      8. `SceneExtensionsUpdaters.PreSceneUpdate()`：针对 Extensions 进行更新
      9. `SCOPED_NAMED_EVENT(FScene_RemovePrimitiveSceneInfos, FColor::Red)`：移除不需要的PrimitiveSceneInfo。将要删除对象移动到该类型的末端，直到到达末尾
      10. `SCOPED_NAMED_EVENT(FScene_UpdatePrimitiveInstances, FColor::Emerald)`：在将primitive添加之前，释放其instance
      11. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(AddPrimitiveSceneInfos);SCOPED_NAMED_EVENT(FScene_AddPrimitiveSceneInfos, FColor::Green)`：将primitive添加进场景中的结构体以及RDG中分配的内存结构体
      12. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(UpdatePrimitiveTransform);SCOPED_NAMED_EVENT(FScene_AddPrimitiveSceneInfos, FColor::Yellow)`：更新primitve对应的Transform信息
      13. `SCOPED_NAMED_EVENT(FScene_UpdatePrimitiveInstances, FColor::Emerald)`：更新primitive instance。
      14. `UpdateRayTracingGroupBounds_UpdatePrimitives(UpdatedInstances)`：根据要更新的transform信息更新光追Group。
      15. `FPrimitiveSceneInfo::AddToScene(this, SceneInfosWithAddToScene)`：将更新后的PrimitiveSceneInfos添加进场景中，其中进行了很多光照信息的更新、碰撞盒信息的添加
      16. `SceneCullingUpdater.OnPostSceneUpdate()`：根据场景更新后的信息，处理cull后的结果
      17. `GPUScene.OnPostSceneUpdate()`：根据添加的primitive，更新对应GPU状态