从 vs2017 开始，vs 正式支持 CMake 项目。

也就是说我们可以直接在 vs 中管理 CMake 工程。下面以 vs2022 为例，说明如何调试 CMake 项目

## 打开 CMake 工程

1. 打开 vs2022，选择 <kbd>打开本地文件夹(F)</kbd>，选择 CMake 项目的文件夹（即根 CMakeLists.txt 所在的目录）。
2. vs 会自动展开 CMake 项目，初次打开时右侧的 `解决方案资源管理器` 显示为  `解决方案资源管理器 - 文件夹视图` 。
3.  点击 <kbd>项目(P)</kbd> -> <kbd>配置缓存(C)</kbd> 生成 CMake 缓存。
4. 选择 `CMake 概述页` 中的 <kbd>打开CMake设置编辑器</kbd>，就可以打开类似于 CMake-GUI 的配置界面；这个界面也可以通过  <kbd>项目(P)</kbd> -> <kbd>xxx 的CMake设置</kbd> 打开；这个设置对应着由 vs 刚刚生成的 `CMakeSettings.json` 文件。
5. 根据需要修改 CMake 配置，修改完成后保存，vs会自动配置这些设置。
6. 点击  <kbd>生成(B)</kbd> -> <kbd>全部生成</kbd> 或者其他的生成选项即可编译项目。

## 切换`解决方案资源管理器`视图

点击  `解决方案资源管理器 - 文件夹视图`  标题下面的 <kbd>在解决方案和可用视图之间切换</kbd> 按钮，可以切换解决方案视图，默认情况下 CMake 项目有两种视图：`文件夹视图` 和 `CMake 目标视图`。当前激活的视图会以粗体字显示。

双击 <kbd>文件夹视图</kbd> 或者 <kbd>CMake 目标视图</kbd> 即可切换视图。可以按需切换，但是推荐在 <kbd>CMake 目标视图</kbd> 下工作。

## 调试 CMake 项目

直接点击调试按钮或者 <kbd>F5</kbd> 即可进行调试。

### 改变程序的执行目录

在程序中涉及到相对目录访问时，可能会需要修改执行目录，修改方法如下

1. 修改 `解决方案资源管理器` 视图为 `CMake 目标视图`；
2. 在`xxx 项目` > `xxx （可执行文件）` 中右击，选择 `🔧 添加调试配置`；
3. 在打开的 `launch.vs.json` 文件中的 `configurations` 数组中添加条目 `"currentDir"`，值为自己需要的值，如果想要执行路径是当前目录，输入 `"${workspaceRoot}"` 即可，这个变量是 vs 内置变量。

> [How to set working directory for Visual Studio 2017 RC CMake Project - Stack Overflow](https://stackoverflow.com/questions/41864259/how-to-set-working-directory-for-visual-studio-2017-rc-cmake-project)



