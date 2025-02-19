首先最好参照

[为虚幻引擎C++项目设置Visual Studio开发环境 | 虚幻引擎 5.5 文档 | Epic Developer Community (epicgames.com)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/setting-up-visual-studio-development-environment-for-cplusplus-projects-in-unreal-engine) 和 [编译虚幻引擎C++游戏项目 | 虚幻引擎 5.5 文档 | Epic Developer Community (epicgames.com)](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/compiling-game-projects-in-unreal-engine-using-cplusplus) 配置 visual studio，并且参照 UE5 的 release note（如 [Unreal Engine 5.5 Release Notes | Unreal Engine 5.5 Documentation | Epic Developer Community (epicgames.com)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.5-release-notes) ）来配置 MSVC 版本和 windows sdk 版本。

## 1. 在 visual studio 中编译报 error MSB3073

这个错误在我打开UE源码目录下的 `UE5.sln` （虚幻项目浏览器）和游戏项目目录下的 `<游戏项目名>.sln` 之后进行编译时都遇到过，并且主要在游戏项目中碰到。报错如下：

```log
Microsoft Visual Studio\2022\Community\MSBuild\Microsoft\VC\v170\Microsoft.MakeFile.Targets(44,5): error MSB3073: 命令“UnrealEngine\Engine\Build\BatchFiles\Build.bat -Target="<游戏项目名>Editor Win64 Debug -Project=\"<游戏项目名>\<游戏项目名>.uproject\"" -Target="ShaderCompileWorker Win64 Development -Project=\"<游戏项目名>\<游戏项目名>.uproject\" -Quiet" -WaitMutex -FromMsBuild -architecture=x64”已退出，代码为 6。
```

这个问题搜索了半天，说的是 Live Coding 的问题：[UE5.1 VS2022 C++ Build Error With MSB3073 - Development / Platform & Builds - Epic Developer Community Forums (unrealengine.com)](https://forums.unrealengine.com/t/ue5-1-vs2022-c-build-error-with-msb3073/694392/9)。

论坛中的解决方法是关闭 Live Coding。因为 VS 的编译与 Live Coding 冲突了。

但是我在游戏项目的 sln 中，根本无法打开 UE Editor，所以没办法关闭 Live Coding。所以得新键一个蓝图工程，打开后在 UE Editor 中关闭 Live Coding。

思考后打开任务管理器，发现有个 UE 图标的进程叫 `LiveCodingConsole`，将其结束任务，再次编译，发现不报错、能编译通过了。

所以推测是在”虚幻项目浏览器“中创建项目之后，它启动了 `LiveCodingConsole`，导致我们在 VS 中编译发生冲突。

类似的实验发生在 [UE4&5 C++项目报错“C1083”和“MSB3073代码6”原因解析与解决方法 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/opus/649752398065041431) 中，有结论如下：

> 在引擎关闭状态下，VS可以成功生成，因为不需要实时显示在引擎中，代码处于非活动（unactive）状态。
>
> 但在引擎开启状态下，引擎为了保护代码的实时正确显示，所以不允许直接通过VS编译，必须使用‍**ctrl+alt+f11**调用实时代码编写然后编译通过才可以在引擎中显示。

UE 文档 [使用Live Coding在运行时实时重新编译虚幻引擎应用](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/using-live-coding-to-recompile-unreal-engine-applications-at-runtime) 中也提到：

> 所有的新版本虚幻引擎安装都默认启用Live Coding。当你打开IDE时Live Coding控制台会自动启动，但是处于隐藏状态。如果控制台隐藏了，它会在你启动Live Coding构建的时候打开。

所以在 UE Editor 开启时，不使用 VS 编译；在关闭 UE Editor 后，使用 VS 编译时，检查 `LiveCodingConsole` 是否关闭，如果没有关闭，则需要手动退出。