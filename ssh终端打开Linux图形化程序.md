## Introduction

> 参照了链接：[SSH终端怎们正确打开Linux gui程序-Window System？ - 掘金 (juejin.cn)](https://juejin.cn/post/7060406899237191716)

在学习 TinyGL 的过程中，通过 ssh 连接到了远程的 Linux 机器，在 Linux 环境下编译好之后，想要运行它的示例程序，结果发现有一些问题，现在记录在下：

1. 使用 xshell 的时候运行程序会直接弹窗：

```
需要 Xmanager 软件来处理 X11 转发请求.
当你安装 Xmanager 时，你可以直接在 Windows 中使用从 Xshell 运行的 X11 程序，例如 xterm 和 gnome-terminal。
```

意思是让我们安装与 Xshell 配套的 Xmanager。点击 <kbd>取消</kbd> 后显示 `Could not open X display`。未能正常运行。

2. 使用 MobaXterm 的时候运行程序，能正常运行，弹出 GUI 弹窗。
3. 使用 Tabby 的时候运行程序，会报如下错误：

```bash
 SSH   X  Could not connect to the X server: Error: connect ECONNREFUSED 127.0.0.1:6000
 SSH      Tabby tried to connect to {"host":"localhost","port":6000} based on the DISPLAY environment var (localhost:0)
 SSH      To use X forwarding, you need a local X server, e.g.:
 SSH      * VcXsrv: https://sourceforge.net/projects/vcxsrv/
 SSH      * Xming: https://sourceforge.net/projects/xming/
Could not open X display
```

意思是需要一个本地的 X server，比如 VcXsrv 和 Xming

## 原因

X Window System 的介绍见 [X窗口系统 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/X視窗系統)

简单来说就是 Linux(或者 POSIX)系统要支持图形化界面程序，需要 `Window System`。现在大多数 Linux 发行版都是用的是 `X Window System`。苹果的`OSX`系统使用`Quartz Compositor` Window System。

`Window System`核心是`DISPLAY SERVER`（或者称为window server、compositor）。一个调用了`DISPLAY SERVER`来显示图形化的程序称之为该`DISPLAY SERVER`的客户端（**client**）。

既有 **client**（调用DISPLAY SERVER服务的gui程序），也有 **server**（DISPLAY SERVER），它们的交互就涉及到协议（protocol），这种协议就称为 **display server protocol**。目前X Window System的X DISPLAY SERVER使用的协议就是X 11（11表示的是版本）。

但是与我们的直觉相反，当我们在 Linux 上打开一个 GUI 程序的时候，该程序会调用 `DISPLAY SERVER` 的图形化显示服务。这个时候 `DISPLAY SERVER` 是服务端，而 GUI 程序是客户端。

> 这里是远端调用本地，也就是GUI程序调用`DISPLAY SERVER`。

那到底`DISPLAY SERVER`在本地哪里呢？

可能是你本地 Linux 操作系统中自带的，也可能是你在 Windows 系统中手动安装的（Windows 系统默认没有display server服务），还可能是 ssh 客户端工具（例如 MobaXterm）内置提供的等等。但是总之，你想在本地启动 Linux GUI 程序，本地一定是要有 `DISPLAY SERVER` 服务的。

Linux GUI 程序通过环境变量 `DISPLAY` 来查找设置的服务地址。

而 X11 Forwarding能够将本地配置的DISLAY SERVICE转发到Forwarding服务启动的X SERVER上。

> 使用`MobaXterm`连接远程服务器时，该 ssh 客户端会自动启动一个 `X DISPLAY SERVER` 服务并开启 `X11 Forwarding`，保证你始终能正确打开Linux GUI 程序。所以在开头的情况中只有 MobaXterm 能打开 GUI 程序。

## 解决方法

1. 在本机上安装常用的 `DISPLAY SERVER`。
2. 设置 X11 Frowarding 转发，设置环境变量 `DISPLAY`。

针对步骤 1，经过搜索后推荐使用 **VcXsrv**，其网址见 tabby 的报错。

针对步骤 2，MobaXterm 会自动设置转发并设置环境变量；Xshell 需要自己设置转发，设置后重新打开 ssh 会话窗口，会自动设置环境变量 `DISPLAY`；tabby 需要自己设置转发，设置后重新打开 ssh 会话窗口，会自动设置环境变量 `DISPLAY`。

由于 tabby 是好用的开源的终端软件，所以以它来做演示。

### VcXsrv 的使用

在下载好 VcXsrv 后，打开 `XLaunch`，`display settings` 有四个选项，按自己需求选择，`Display number` 使用默认的 `-1`，然后点击 <kbd>下一页</kbd>；选择 `Start no client`，点击 <kbd>下一页</kbd>；勾选 `Clipboard`、`Primary Selection` 和 `Native opengl`，点击 <kbd>下一页</kbd>；点击 <kbd>完成</kbd>。此时 `XLaunch` 会变成小图标收拢在任务栏中。

#### tabby 的设置

点击 tabby 的 ⚙ 图标，打开设置，点击  <kbd>Profiles&connections</kbd> ->  <kbd>ADVANCED</kbd> ->  <kbd>SSH</kbd> ->  <kbd>ADVANCED</kbd>，打开 `X11 forwarding`，然后保存重新打开 ssh 会话，执行 GUI 程序，即可正常运行程序。