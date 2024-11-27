## UE5 根项目的程序入口

在 windows 下编译并运行 UE5，首先运行 `Setup.bat` 脚本，最好使用命令行运行，有可能会报错，然后再运行 `GenerateProjectFiles.bat` 脚本，生成 VS 项目。使用 VS 打开 `UE5.sln` 即可。

在编译完 UE5 后，在 VS 中运行 UE5 项目，会出现一个 **“虚幻编辑器”** 的进度条窗口，这个进度条窗口的程序入口在 `Engine/Source/Runtime/Launch/Private/Windows/LaunchWindows.cpp` 的 `WinMain()` 中，然后调用了 `LaunchWindowsStartup()`。在进行一系列判断和设置后，调用了 `Engine/Source/Runtime/Launch/Private/Launch.cpp` 中的 **`GuardedMain()`**，这个 `GuardedMain()` 函数在所有平台（windows，linux，macos 等）中都会调用到，是程序实际的入口。

在 `GuardedMain()` 中，首先使用 `FTaskTagScope Scope(ETaskTag::EGameThread)` 开启游戏线程，然后使用 `FScopedSlowTask SlowTask` 来加载引擎初始化所需资源。然后判断是否在编辑器中，如果在的话就调用 `EditorInit(GEngineLoop)` 来初始化编辑器，它位于 `Engine/Source/Editor/UnrealEd/Private/UnrealEdGlobals.cpp` 中。

在 `EditorInit()` 加载上次使用的目录以及进行一些必要操作后，就会使用 `FModuleManager::LoadModuleChecked<IMainFrameModule>(TEXT("MainFrame"))` 获取 MainFrame 模块，然后初始化它，调用 `CreateDefaultMainFrame()` （位于 `Engine/Source/Editor/MainFrame/Private/MainFrameModule.cpp`）。在 MainFrame 中，会调用 Providers 来创建 ContentWidget，用于显示一些内容（只有一个激活的 provider），这里主要是 `FProjectDialogProvider`。在创建完成后，会调用 `MainFrameHandler->ShowMainFrameWindow()` 来显示对话框窗口。`FProjectDialogProvider` 会调用 `FGameProjectGenerationModule` 创建一个 GameProjectDialog，这里的对话框是 `SProjectDialog`，创建时，会调用 `SProjectDialog::Construct()` 来构造项目对话框。

在 `SProjectDialog::Construct()` 中，创建了一个项目浏览器 `SProjectBrowser`，同样也调用 `Construct()` 函数来构造。

在 “虚幻编辑器” 的进度条走完之后，出来的窗口是 **“虚幻项目浏览器”**，即刚才创建的 `SProjectBrowser`，。这个窗口用于打开之前创建的项目，或者新建项目。

在 **`GuardedMain()`** 中，完成上述操作后，会调用一个循环来进行 tick 操作：`		while( !IsEngineExitRequested()) { EngineTick(); }`。这个后续再讨论。

## 生成的 UE5 项目的程序入口

在用 “虚幻项目浏览器” 生成 C++ 项目后，在目的目录会生成一个 `<项目名>.sln` 的工程文件（蓝图项目不会生成此文件）。在编译运行 `<项目名>` 的 VS 项目之后，会打开真正的 "虚幻编辑器" 窗口，用来编辑和运行虚幻游戏。

在这个 "虚幻编辑器" 窗口中，程序入口也在 `Engine/Source/Runtime/Launch/Private/Windows/LaunchWindows.cpp` 的 `WinMain()` 中，因为它本质上与这篇文章最开始的 **“虚幻编辑器”**  是同一个东西，只不过后续会走不同分支，在这个游戏项目中，不会通过 provider 来创建 MainFrameContent，而是通过 `FGlobalTabmanager` 来创建 content。

并且游戏会作为 module 注册到 module manager 中，由 `IMPLEMENT_PRIMARY_GAME_MODULE` 注册。可见 [UE 模块的加载与启动分析 | 虚幻社区知识库 (ue5wiki.com)](https://ue5wiki.com/wiki/24007/)。在 UE5 的编辑器模式下，默认情况 `LinkType` 是 `TargetLinkType.modular`，即模块式的，需要链接游戏工程的动态库。

`IMPLEMENT_PRIMARY_GAME_MODULE` 宏最后也是调用 `IMPLEMENT_MODULE` 宏，这个宏定义了一个静态变量，根据 C++ 知识，这个静态变量在动态库被加载时会自动创建，或者被静态链接后也会自动创建。对于 `MONOLITHIC` 模式来说，变量是 `static FStaticallyLinkedModuleRegistrant< ModuleImplClass > ModuleRegistrant##ModuleName( TEXT(#ModuleName) )`；对于 Modular 模式来说，变量是 `static FModuleInitializerEntry ModuleName##InitializerEntry(TEXT(#ModuleName), Initialize##ModuleName##Module, TEXT(UE_MODULE_NAME))`。比如我的项目叫做 `MyDemo`，得到的变量为 `static FModuleInitializerEntry MyDemoInitializerEntry(TEXT("MyDemo", InitializeMyDemoModule, TEXT(UE_MODULE_NAME)))`，这里的 `InitializeMyDemoModule` 是一个函数，它被包装到这个 Entry 结构体中了，它 new 一个 `ModuleImplClass` （这里是 `FDefaultGameModuleImpl`）然后返回指针。

后续模块通过 `FModuleManager` 中的接口来加载、初始化和运行。

在 `FModuleManager::LoadModuleWithFailureReason()` 中，如果 modular 模式，会寻找名字对应的动态库，找到好，就调用 `FInitializeModuleFunctionPtr InitializeModuleFunctionPtr = FModuleInitializerEntry::FindModule(*InModuleName.ToString());` 来获取 `Initialize<Name>Module` 指针（即 `InitializeMyDemoModule` 函数），然后调用这个函数指针实例化出来一个真正的模块。之后初始化、运行它。