这两天下载了 OGRE 引擎的代码，开始编译，其使用的构建系统为 CMake，这也是现在绝大多数 C++ 开源项目所使用的构建系统。

发现在用 CMake 构建时需要下载依赖，而在公司的网络条件下总是下载不下来，经过搜索后解决，现在将其记录下来。

## CMake下载依赖

一般的依赖都是压缩包的形式，方便传输。在 CMake 中下载文件时使用的时 `file` 命令，在 `file()` 命令的第一个参数中，使用 `DOWNLOAD` 子命令即可下载指定文件，详见 CMake 文档：[file — CMake 3.23.1 Documentation](https://cmake.org/cmake/help/latest/command/file.html#transfer)。

在 OGRE 中是如下形式：

```cmake
file(DOWNLOAD
    https://github.com/ocornut/imgui/archive/v1.85.tar.gz
    ${PROJECT_BINARY_DIR}/imgui.tar.gz)
```

可以将 `https://github.com/ocornut/imgui/archive/v1.85.tar.gz` 下载为 `imgui.tar.gz`。

但是我在实际构建过程中却发现始终下载不成功，压缩包始终是 0 KB，导致构建失败，因此猜想是代理问题。经过搜索后发现两个链接 [curl - File download ignores HTTP_PROXY set in CMakeLists - Stack Overflow](https://stackoverflow.com/questions/51883769/file-download-ignores-http-proxy-set-in-cmakelists) 和  [Fetching dependencies in CMake behind proxy - Stack Overflow](https://stackoverflow.com/questions/68724873/fetching-dependencies-in-cmake-behind-proxy)。

下面解释一下原理

## CMake代理

CMake 的代理依赖于环境变量 `HTTP_PROXY` 和 `HTTPS_PROXY`。在公司的电脑中，虽然设置了代理，但是并没有这两个环境变量，所以解决方式有两种：

1. 在系统环境中设置这两个变量，或者在命令行中设置；
2. 在 CMakeList 中设置这两个环境变量。

经过思考和，使用第一种方式设置的话，可能会影响其他软件的正常使用，并且在命令行中设置的环境变量也只在当前终端窗口中起作用，所以使用了第二种方式，这样可以避免对其他软件的影响。

主要方式为：修改主 `CMakeLists.txt`，将以下两行添加到文件的顶端：

```cmake
set(ENV{HTTP_PROXY}  "myproxy:8080")
set(ENV{HTTPS_PROXY}  "myproxy:8080")
```

再构建即可成功下载依赖。

## CMake 管理项目的release 和debug

一个`c/c++`库，在编译的时候，可以选择编译是否带调试信息，带调试信息的就是`Debug`版，不带调试信息的就是`Release`版。 在`CMakeLists.txt`里一般不会制定当前工程是否是`Debug`还是`Release`， 这个信息可以通过`CMake`的命令参数传输进去，使用方法如下：

```cmake
cmake .. -DCMAKE_BUILD_TYPE=Debug -G xxx -B xxx
cmake .. -DCMAKE_BUILD_TYPE=Release -G xxx -B xxx
```

对于 OGRE 来说，有四个选项：

- Debug
- MinSizeRel
- RelWithDebInfo
- Release

它默认的选项为 RelWithDebInfo。在 Configure 过程中会编译一些依赖，这些依赖就是使用 RelWithDebInfo 来编译的。所以在编译OGRE的时候就需要选择 `RelWithDebInfo`，否则的话就需要通过命令行来生成工程，同时带上诸如 `-DCMAKE_BUILD_TYPE=Debug` 的参数。