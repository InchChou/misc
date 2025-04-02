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
>                 - `FDeferredShadingSceneRenderer::RenderHitProxies()/Render()` 这里 `Render` 是渲染整个场景的，`RenderHitProxies` 是渲染鼠标点击物体场景的，算是Render的简单版本
>
> 



> 关于 HitProxy 知乎上有文：[Learning UE5: HitProxy 原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/670360410)

`SceneRendering.h` 与 `SceneRendering.cpp` 中定义了 `FSceneRenderer` 类和它的一个子类 `FMobileSceneRenderer`（用于移动端渲染），`DeferredShadingRenderer.h/cpp` 中定义了另一个子类 `FDeferredShadingSceneRenderer`，用于延迟渲染。后续主要基于延迟渲染分析。

`FRendererModule::BeginRenderingViewFamily()` 调用到 `FRendererModule::BeginRenderingViewFamilies()`，