> [剖析虚幻渲染体系（04）- 延迟渲染管线 - 0向往0 - 博客园 (cnblogs.com)](https://www.cnblogs.com/timlly/p/14732412.html)

其部分声明和定义在：`Engine\Source\Runtime\Renderer\Private\DeferredShadingRenderer.h`

查看源码，可以看到 `FDeferredShadingSceneRenderer `主要包含了Mesh Pass、Lumen、大气、光源、阴影、半透明、光线追踪、反射、可见性等几大类接口。

> “继续分析`FScene::UpdateAllPrimitiveSceneInfos`的过程” 这一部分，由之前的《UE5大致渲染流程》提到在 `FScene` 中添加了一个 `Update()` 函数，用于同步或异步更新场景，其逻辑与 `FScene::UpdateAllPrimitiveSceneInfos()` 基本一致。而 `FScene::UpdateAllPrimitiveSceneInfos()` 算是 `Update()` 的特化同步版本。我们就分析 `FScene::Update` 函数

### `FScene::Update`

`FScene::Update` 过程：

1. 首先对 `RemovedLocalPrimitiveSceneInfos` 和 `AddedLocalPrimitiveSceneInfos` 进行排序，分别对应着之前存在于场景中并将被移除的图元，和不存在场景中并将被添加的图元。
2. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RemovePrimitiveSceneInfos)`：在删除操作中，将要删除的 proxy 移到 `PrimitiveSceneProxies` 数组（在数组中，proxy 按 type 连续存储）最末端，配合 `PrimitiveSceneProxies` 和 `TypeOffsetTable`, 可以减少删除元素的移动或交换次数。然后移除 `DistanceFieldSceneData` 和 Lumen 数据中的 Primitive 及相关信息。
3. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(AddPrimitiveSceneInfos)`：处理图元增加时，将 Primitive 相关信息的那些数组增加空间。然后增加新的 Primitive 信息，将这些信息赋值为 `AddedLocalPrimitiveSceneInfos` 中的信息。更新 `TypeOffsetTable`。同样的，根据 TypeOffsetTable 通过交换操作将相同 Type 的 Primitive 放在一起。然后添加 `DistanceFieldSceneData` 和 Lumen 数据中的 Primitive 及相关信息。
4. `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(UpdatePrimitiveTransform)`：更新图元变换矩阵

从代码可以看出来，一直到第二个 `CSV_SCOPED_TIMING_STAT_EXCLUSIVE(UpdatePrimitiveInstances)` 后，`FScene::Update` 的第一部分的作用是删除、增加图元，以及更新图元的所有数据，包含变换矩阵、自定义数据、距离场数据等。

在UE5.5 `FScene::Update` 剩下的逻辑与博客中的不同。

1. `GPUScene.OnPostSceneUpdate()`：在 `FScene` 更新后调用
   1. `AddPrimitiveToUpdate()`：对应着博客里面的 `AddPrimitiveToUpdateGPU()`，将 primitive 加入到待 GPUScene 的更新列表。



与博客中不一样，直接在 `FScene::Update()` 中就调用了 `GPUScene.Update()`，其调用时间点在 `AddStaticMeshesTask`、`UpdateReflectionSceneData`、`SCOPED_NAMED_EVENT(UpdateUniformBuffers, FColor::Emerald)` 之后。

1. `UpdateInternal()`：直接调用封装的 `UpdateInternal` 函数。
   1. 首先检查是否设置了每帧上传全部图元，上传的话就将所有图元再加入 `PrimitivesToUpdate` 数组中，并且更改对应的标志位。
   2. 使用 RDG 创建一些资源，如 `FUploadDataSourceAdapterScenePrimitives`、`FRegisteredBuffers`。
   3. `UploadGeneral<FUploadDataSourceAdapterScenePrimitives>()`：调用模板函数来进行统一上传，这个函数使用 Adapter 抽象数据源。
      1. 将上传过程封装成 Task
      2. 对于所有需要上传的 PrimitiveData，使用 `FInstanceBatcher` 及其 `QueueInstances` 方法，将这些数据组织成批次。
      3. 在 Task 中并行执行 `Upload Primitives`、`Upload Instances`，在最后顺序执行 `Upload Lightmap` ，。
      4. 在 `Upload XXX` 时，将步骤 1.3.2 得到的批次中的 primitive 依次调用`FInstanceSceneShaderData` 的 `BuildXXX()` 系列函数构建上传数据，然后添加到 `TaskContext` 中的对应 Uploader 中。
      5. 在并行执行完 `Upload XXX` 后，调用 `XXXUploader.End()` 函数（即 `FRDGAsyncScatterUploadBuffer::End()`） 来构建 `ScatterUpload` Pass，进行实际的数据上传工作。
   4. 如果传入的 `FUpdateInstanceFromComputeCommand` 不为空，则使用 GPU 的 Compute 来更新 instance data。（此处不理解）

在 `GPUScene.Update()` 之后，添加一个 Task 用于清理要删除的 PrimitiveSceneInfos。

### `BeginInitViews()` 与 `EndInitViews()`

UE5 与 UE4 不同，现在没有了 `InitViews()`，取而代之的是 `BeginInitViews()` 与 `EndInitViews()`，并且在 `BeginInitViews()` 之前，渲染器会更新 virtual texture、启动更新光照 Function Atlas 贴图任务、更新天空大气、更新 lumen 场景、启动 Nanite 可见性任务、准备距离场 Scene、设置 ShadowSceneRenderer 的更新任务、更新各个模块的 Atlas 贴图。

#### `BeginInitViews()` 

原po对于 `InitViews` 写道：

> 它的处理的渲染逻辑很多且重要：可见性判定，收集场景图元数据和标记，创建可见网格命令，初始化Pass渲染所需的数据等等。

由于将它拆分成了两个步骤，`BeginInitViews()` 简单一些：可见性判定，收集场景图元数据和标记，创建可见网格命令。

1. `PreVisibilityFrameSetup()`：创建可见性帧设置预备阶段。
2. `TaskDatas.VisibilityTaskData->StartGatherDynamicMeshElements()`：尽早激活 `DynamicMeshElementsPrerequisites` 任务，开始处理动态网格数据，并在处理完后收集网格数据。
3. `BeginInitDynamicShadows()`：在最终确定可见性之前尝试尽早启动动态阴影任务。
4. `InstanceCullingManager.RegisterView()`：为 instance culling 创建 view 的 GPU 端表示
5. 初始化特效系统 `FXSystem`
6. `LumenScenePDIVisualization()`：创建 Lumen 场景的 PDI 可视化表示
7. `InitSkyAtmosphereForViews()`：为每个 View 初始化天空大气效果
8. `View.InitRHIResources()`：初始化每个 View 的 uniform buffer。
9. `TaskDatas.VisibilityTaskData->ProcessRenderThreadTasks()`：统一处理可见性任务
   1. `StartGatherDynamicMeshElements()`：即步骤 2。
   2. `ViewPacket.BeginInitVisibility()`：