Swift 是苹果开发的一个语言，现在已经开源了。其官方网站为：https://www.swift.org。学习 Swift 主要看它的文档，它的文档很齐全。

首先看文档的 Getting Started 部分，如果想用 Swift，得先安装它，在 windows 下安装过程参考[Swift.org - Windows Installation Options](https://www.swift.org/install/windows/#installation-via-windows-package-manager)，可以通过包管理器或者安装程序[安装程序](https://www.swift.org/download/)来安装。安装完后使用 `swift --version` 来测试是否安装好。这是一系列开发工具，被称为 [Swift Package Manager (SwiftPM)](https://www.swift.org/package-manager/)，主要通过命令行来使用。

命令行的文档如下：[Swift.org - Build a Command-line Tool](https://www.swift.org/getting-started/cli-swiftpm/)。

为了写一个小程序，首先需要Bootstrapping，这里表现为通过命令行来初始化一个包：

```
$ mkdir MyCLI
$ cd MyCLI
$ swift package init --name MyCLI --type executable
```

这会成成一个新的名为 MYCLI 的文件夹，其中包含以下文件：

```
.
├── Package.swift
└── Sources
    └── main.swift
```

`Package.swift` 是 Swift 的清单文件。 您可以在其中保存项目的元数据以及依赖项。 

`Sources/main.swift` 是应用程序入口点，我们将在其中编写应用程序代码。

事实上，SwiftPM 生成了一个“Hello, world!” 为我们项目！ 我们可以通过在终端中运行 `swift run` 来运行该程序。

```
$ swift run MyCLI
Building for debugging...
[3/3] Linking MyCLI
Build complete! (0.68s)
Hello, world!
```

