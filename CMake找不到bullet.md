这个问题是在编译 OGRE v13.4.0 版本的时候发现的，在 OGRE 的这个版本引入了 bullet 物理库，同时提供了 bullet 的 demo，所以尝试将 bullet 相关的东西一起编译进来。

## 直接使用 CMake-GUI 生成并编译

在直接使用 CMake-GUI 生成并编译时并没有发现有太多报错，直到使用 vs 2022 打开解决方案并生成 `ALL_BUILD` 之后才报错 `无法打开 OgreBullet.lib`，查看了编译目录后发现并没有生成 `OgreBullet.lib`。

虽然此时 `SampleBrowser` 能编译成功并打开，但里面并没有 bullet demo，所以还是要分析为什么 OgreBullet 编译不成功。

### 分析

在 vs 工程中寻找 OgreBullet 项目，并没有发现有它，同时在编译目录的 `Dependencies\lib\` 中发现生成了 bullet 的相关库，为 `BulletCollision_RelWithDebInfo.lib` 的形式，所以很奇怪为什么 OgreBullet 未被生成。

由于 OgreBullet 是子项目，而现在在 vs 工程中并没有出现，所以肯定是某种开关导致它未被添加进去，所以要找未被添加进去的原因。

在 OGRE 代码目录中发现有 `OgreBullet.cpp` 文件，并且其处于 `Components\Bullet\src` 目录下，所以推测这个目录就是 `OgreBullet`  项目的目录，在 `Components\Bullet` 目录下有 `CMakeLists.txt` 文件，打开它，发现其中有 `add_library(OgreBullet xxxx)` 语句，证明了我们的猜测。

继续在 OGRE 的 CMakeLists.txt 中寻找线索，在 `Components\CMakeLists.txt` 中有这么一个条件判断：

```cmake
if (OGRE_BUILD_COMPONENT_BULLET)
    add_subdirectory(Bullet)
endif()
```

于是找到了 OgreBullet 的开关，继续顺藤摸瓜，找 `OGRE_BUILD_COMPONENT_BULLET` 相关的配置，发现了这么一句：

```cmake
cmake_dependent_option(OGRE_BUILD_COMPONENT_BULLET "Build Bullet physics component" TRUE "BULLET_FOUND" FALSE)
```

这个选项的意思是 `OGRE_BUILD_COMPONENT_BULLET` 依赖于 `BULLET_FOUND` 选项，如果 `BULLET_FOUND` 是 `TRUE`，则 `OGRE_BUILD_COMPONENT_BULLET` 为 `TRUE`，否则为 `FALSE`。此外 `OGRE_BUILD_COMPONENT_BULLET` 与 `OGRE_BUILD_COMPONENT_BITES` 等选项的形式一样，后者能在 CMake-GUI中能自行选择开关，但前者没有出现，应该是被直接置为 `FALSE` 了。

继续寻找 `BULLET_FOUND` 选项，发现了两句：

```cmake
# Bullet
find_package(Bullet QUIET)
macro_log_feature(BULLET_FOUND "Bullet" "Bullet physics" "https://pybullet.org")
```

第一句是静默查找 Bullet，第二局是一个 log 宏，用于打印 `BULLET_FOUND` 的状态，推测是能找到 Bullet 库的话，就将 `BULLET_FOUND` 置为 `TRUE`，否则为 `FALSE`。为了调试，将 `QUIET` 去除，并加上 `message(STATUS "zyc Bullet found status ${BULLET_FOUND}")` 打印 `BULLET_FOUND` 的值，log 如下：

```
Could NOT find Bullet (missing: BULLET_DYNAMICS_LIBRARY BULLET_COLLISION_LIBRARY BULLET_MATH_LIBRARY BULLET_SOFTBODY_LIBRARY) 
zyc Bullet found status FALSE
```

证明了我们的猜想，但是 `BulletDynamics`、`BulletCollision` 等都已经编译出来了，**为什么会找不到这些库**。

同时用 everything 搜一下 `FindBullet`，发现在 `<CMake安装路径>\share\cmake-3.23\Modules` 目录中存在 `FindBullet.cmake`，打开观察一下：

```cmake
  BULLET_FOUND - Was bullet found
...
# Find the libraries

_FIND_BULLET_LIBRARY(BULLET_DYNAMICS_LIBRARY        BulletDynamics)
_FIND_BULLET_LIBRARY(BULLET_DYNAMICS_LIBRARY_DEBUG  BulletDynamics_Debug BulletDynamics_d)
_FIND_BULLET_LIBRARY(BULLET_COLLISION_LIBRARY       BulletCollision)
_FIND_BULLET_LIBRARY(BULLET_COLLISION_LIBRARY_DEBUG BulletCollision_Debug BulletCollision_d)
...
```

发现确实是用 `BULLET_FOUND` 来指示 bullet 是否已被找到，而寻找的语句为 CMake 内置的 `_FIND_BULLET_LIBRARY`，这里变量的名字与报错中未找到 bullet 的log 中的名字一样，同时注意到**这些库的名字**，发现只有无后缀和 `_d` 两种，分别对应着无后缀的变量 `XXX_LIBRARY` 和有后缀的变量 `XXX_LIBRARY_DEBUG` 两种，推测是否是这些名字导致的，所以在 `Dependencies\lib\` 中将 bullet 库相关的 `lib` 文件的 `_RelWithDebInfo` 删掉再点击 CMake-GUI 的 <kbd>Configure</kbd> 按钮，查看 log，发现能找到 bullet 库了：

```
Found Bullet: D:/chou/proj/HRTEngine/ogre_build/Dependencies/lib/BulletDynamics.lib  
zyc Bullet found status TRUE
```

再点击第二次 <kbd>Configure</kbd> 按钮，发现无报错，再点击 <kbd>Generate</kbd> 按钮生成工程文件。

打开工程文件，发现在 `Components` 文件夹中已经有了 `OgreBullet` 工程，再生成 `ALL_BUILD` 工程，运行 `SampleBrowser`，出现了 bullet 的 sample。至此就 OK 了。

## 原因

通过上面的分析，可以得出找不到 bullet 库的原因为以下几点：

1. 通过 CMake-GUI 打开 OGRE，默认情况下是 `CMAKE_BUILD_TYPE` 是  `RelWithDebInfo`，所以子项目的 `CMAKE_BUILD_TYPE` 也集成了父项目的 type，有时候就会生成带 `_RelWithDebInfo` 后缀的 lib 文件，bullet 库就是如此
2. 而在调用 `find_package(Bullet)` 时调用了 CMake 的 `FindBullet.cmake`，寻找的时候只寻找了不带后缀或只带 `_d` 后缀的 bullet 库，所以会找不到 bullet 库
3. 找不到 bullet 库就会导致 `BULLET_FOUND` 为 `FALSE`，然后导致 `OGRE_BUILD_COMPONENT_BULLET` 为 `FALSE`，最后无法执行 `add_subdirectory(Bullet)`，所以不会生成 OgreBullet.lib。

## 解决办法

在 CMake-GUI 生成 OGRE 的依赖之后，将 `Dependencies\lib\` 中 bullet 库的后缀删掉，然后删掉 CMake Cache 后再运行一遍 Configure即可

## 解决办法（推荐）

修改 `CMake/Dependencies.cmake` ，注释掉 `file(DOWNLOAD XXX/bullet/xxx)`语句和下面的解压语句，手动下载 bullet 库，并解压到编译目录的根目录。

然后修改 bullet 的 `CMakeLists.txt` 中的某些语句：

```cmake
SET(CMAKE_DEBUG_POSTFIX "_Debug" CACHE STRING "Adds a postfix for debug-built libraries.")
SET(CMAKE_MINSIZEREL_POSTFIX "_MinsizeRel" CACHE STRING "Adds a postfix for MinsizeRelease-built libraries.")
SET(CMAKE_RELWITHDEBINFO_POSTFIX "_RelWithDebugInfo" CACHE STRING "Adds a postfix for ReleaseWithDebug-built libraries.")
```

将这些 `POSTFIX` 的值都设为空即可。