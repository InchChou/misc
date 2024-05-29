## 前言

在使用 VisualStudio 编译 renderdoc windows 客户端的时候，发现了一个现象：renderdoc 项目作为最核心的项目，它输出为动态库，并且导出了带符号表的 lib 库，qrenderdoc 作为客户端界面输出成 exe 文件，按照我一贯的思想， qrenderdoc 本来应该链接 renderdoc.lib 的，但是在项目的链接输入属性里并没有看到 renderdoc.lib，并且连解决方案里其他的生成库都没有，只有 qt 和系统相关的 lib 库。

这让我有点疑惑，它能正常运行，debug 的时候也能正常跳转到 renderdoc 项目里，说明确实链接到了 renderdoc.lib，那么它是怎么链接的呢？

这时候我观察 qrenderdoc 项目，逐个点开它的所有目录，发现了有一个特殊的目录，叫做 **“引用”**。点开它，发现里面有两个条目：renderdoc 和 version。观察 renderdoc 引用条目的所有属性，发现它就是指向的 renderdoc 项目。这里就说的通了，它通过引用来隐式地实现了对 renderdoc 输出库的链接和调用。

## VS 项目中的引用

微软关于 VS 项目中的引用的文档链接为：[管理项目中的引用 - Visual Studio (Windows) | Microsoft Learn](https://learn.microsoft.com/zh-cn/visualstudio/ide/managing-references-in-a-project?view=vs-2022)，[在 C++ 项目中使用库和组件 | Microsoft Learn](https://learn.microsoft.com/zh-cn/cpp/build/adding-references-in-visual-cpp-projects?view=msvc-170)

在添加引用后，VS 会自动创建项目的依赖项，即如果 A 引用了 B，那表示 A 依赖于 B，如果需要生成 A 项目，则必须先生成 B 项目。

### 关于引用的属性

如 visual studio 和文档中所述，引用的属性如下：

- 生成：以下属性可用于各种类型的引用。 它们使你可以指定如何用引用进行生成。
  - 复制本地（Copy Local）：指定是否在生成期间自动将所引用的程序集复制到目标位置。
  - 复制本地附属程序集 (Copy Local Satellite Assemblies)：指定是否在生成期间将所引用程序集的附属程序集复制到目标位置。 仅在“复制本地”为 **`true`** 时使用。
  - 引用程序集输出（Reference Assembly Output）：指定在生成过程中使用此程序集。 如果是 **`true`**，则在生成期间在编译器命令行上使用此程序集。
- 项目引用：以下属性定义从“引用”窗格中选中的项目到同一解决方案中另一项目的项目到项目引用。 有关详细信息，请参阅[管理项目中的引用](https://learn.microsoft.com/zh-cn/visualstudio/ide/managing-references-in-a-project)。
  - 链接库依赖项（Link Library Dependencies）：此属性为 True 时，项目系统会将不相关项目生成的 LIB 文件链接到相关项目。 通常，需要指定为 True。
  - 使用库依赖项输入（Use Library Dependency Inputs）：当此属性为 False 时，项目系统不会将库中由不相关项目生成的 OBJ 文件链接到相关项目。 因此，此值会禁用增量链接。 通常，需要指定为 False，因为如果存在多个不相关项目，则构建应用程序可能会花很长时间。
  - 项目标识符(Project Identifier)：唯一标识独立项目。 属性值是不可修改的内部系统 GUID。
- 引用：参见 [只读引用属性（COM 和 .NET）](https://learn.microsoft.com/zh-cn/cpp/build/adding-references-in-visual-cpp-projects?view=msvc-170#read-only-reference-properties-com--net)。

### 使用示例

这里我们假定 A 是 exe 项目，B 是 lib 库项目，A 使用了 B 中的函数或方法，则需要将 B 链接到 A。此时推荐使用 VS 的项目引用功能。在 A 项目的“引用”节点中右击，选择“添加引用”，勾选 B 项目即可，此时 A 项目就引用了 B 项目，“引用”节点中会出现 B 项目。但此时还未链接，需要将“引用”节点中 B 项目的“项目引用 -> 链接库依赖项” 设置为 `True`，此时即可链接。

> 将“链接库依赖项” 设置为 `True` 等同于在 A 项目的 “属性 -> 链接器 -> 输入 -> 附加依赖项” 中添加 B 项目的生成 lib库。

> 如果 B 项目是 dll 项目，会发现生成项目的时候只会生成 dll 文件，不会生成 lib 文件，导致链接失败。这是因为 lib 文件实际上是导出的符号表，dll 项目默认情况下不会导出符号表，如果要生成 lib 库，则在需要导出的 api 中添加 `__declspec(dllexport)` 或者 `__declspec(dllimport)` 才行。

默认情况下，创建的 vs 工程的输出目录为 `$(SolutionDir)$(Platform)\$(Configuration)\`，如果是x64 平台、 Debug 配置，则目录为 `<sln文件所在的目录>\x64\Debug\`。使用了引用，并且打开“链接库依赖项”后，无论生成的库在哪里，都能链接到。

> 但是需要注意的一点是，如果可执行程序项目所依赖的项目是动态库项目的话，在不做其他设置的情况下，运行可执行程序时，需要动态库与可执行程序在同一个目录下，所以最好将这些项目的输出目录设置为相同。

### 使用小结

在使用 Visual Studio 开发 C/C++ 项目时，可以使用项目引用功能来方便管理项目之间的引用，使用它可以省去在项目中指定 “属性 -> 链接器 -> 输入 -> 附加依赖项” 的工作。

使用引用时，需要添加引用，选定所需要引用的项目，然后将引用条目的链接库依赖项设置为 True 即可。

> 引用中 “生成” 属性在 C++ 项目中基本用不到，它是与程序集有关的，搜索了一下，程序集是 NET 应用程序的部署单元，等以后接触了 C# 和 NET 开发再探究里面的内容。

### PS：CMake 尝试

本来想用 CMake 看能否设置 VS 项目引用，结果发现只有 `add_dependencies` 和 `target_link_libraries` 能影响部分属性，以示例中的引用关系为例，使用这两个命令，会在 A 的引用中添加 B，但是不会修改 A 项目 “引用”节点中 B 项目的“项目引用 -> 链接库依赖项” 属性，意味着无法自动链接。

最后找到了一个很 trick 的方法，但是不推荐使用：

```cmake
add_subdirectory(PrintSome)

add_dependencies(ProjectDependenciesTest PrintSome)

set_target_properties(ProjectDependenciesTest 
    PROPERTIES VS_DOTNET_REFERENCEPROP_PrintSome_TAG_LinkLibraryDependencies TRUE
)
```

这里 `add_dependencies` 添加了对 PrintSome 的引用，`set_target_properties(.. VS_DOTNET_REFERENCEPROP_PrintSome_TAG_LinkLibraryDependencies TRUE)` 修改了这个引用的 `LinkLibraryDependencies` 属性为 True。

`VS_DOTNET_REFERENCEPROP_PrintSome_TAG_LinkLibraryDependencies ` 来源于[VS_DOTNET_REFERENCEPROP_\<refname\>_TAG_\<tagname\>](https://cmake.org/cmake/help/latest/prop_tgt/VS_DOTNET_REFERENCEPROP_refname_TAG_tagname.html)。这个属性实际上是给 .NET 项目使用的，这里我强行使用了，将 `<refname>` 替换成被应用的项目名，`<tagname>` 替换成 `LinkLibraryDependencies` 标签名即可。

或者通过 [How to set Visual-Studio LinkLibraryDependencies property to yes through CMAKE - Stack Overflow](https://stackoverflow.com/questions/42067674/how-to-set-visual-studio-linklibrarydependencies-property-to-yes-through-cmake) 这种方式。

最后在 cmake 的 gitlab 上找到了一些 issue 来讨论 LinkLibraryDependencies 的，但都没有太多意义。

> [VS: add_dependencies adds a link dependency for VisualStudio targets (#20888) · 议题 · CMake / CMake · GitLab (kitware.com)](https://gitlab.kitware.com/cmake/cmake/-/issues/20888)

在 [Source/cmGlobalVisualStudioGenerator.cxx#L417](https://gitlab.kitware.com/cmake/cmake/-/blob/master/Source/cmGlobalVisualStudioGenerator.cxx#L417) 的注释中，写到：“VS 8 及更高版本提供了项目文件“LinkLibraryDependencies”设置来选择是否激活被引用项目自动链接。除了链接外部项目文件时，我们会禁用它“。意思就是这个功能默认不打开，除非引用的是外部项目。

所以 CMake 暂时还是使用显式链接库的方式，应该是考量了兼容性。