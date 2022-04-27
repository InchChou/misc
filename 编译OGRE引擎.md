OGRE 的文档中介绍了如何编译，它使用的构建工具是 CMake，通过 CMake 生成工程。在 Windows 中推荐使用 MSVC 来编译工程。

## CMake 生成工程与 OGRE 组件

使用 CMake 生成工程的方法就不多赘述，现在需要介绍一下 OGRE 的组件。

OGRE 组件在 CMake 中是 `OGRE_BUILD_COMPONENT_XXX` 的形式，我们需要选中或取消它们来生成对应的 project。其中有几个组件有依赖项：

- `OGRE_BUILD_COMPONENT_CSHARP`：依赖 MSVC 的 C# 编译器，如果 MSVC 中没有下载 C# 编译器，则需要取消勾选。
- `OGRE_BUILD_COMPONENT_JAVA`：依赖 JDK，如果没有配置 JDK 环境，则需要取消勾选。
- `OGRE_BUILD_COMPONENT_PYTHON`：依赖 Python，如果没有配置 Python 环境，则需要取消勾选。

CMake 中其他的选项按自己需要开启。

## CMake 中的依赖项下载

OGRE 依赖一些第三方库，如 freetype、imgui、pugixml、SDL2、assimp、zlib等，对于这些依赖，CMake 在 Configure 过程中会去下载并解压，但是如果网络环境不支持的话，则需要自己手动下载并拷贝到编译目录中，然后修改 Components/Overlay/CMakeLists.txt 及 Dependencies.cmake 文件，将下载文件的部分注释掉，如下所示：

```diff
diff --git a/CMake/Dependencies.cmake b/CMake/Dependencies.cmake
index b80742a13..6397cdb1b 100644
--- a/CMake/Dependencies.cmake
+++ b/CMake/Dependencies.cmake
@@ -70,9 +70,9 @@ set(CMAKE_FRAMEWORK_PATH ${CMAKE_FRAMEWORK_PATH} ${OGRE_DEP_SEARCH_PATH})

 if(OGRE_BUILD_DEPENDENCIES AND NOT EXISTS ${OGREDEPS_PATH})
     message(STATUS "Building pugixml")
-    file(DOWNLOAD
-        https://github.com/zeux/pugixml/releases/download/v1.10/pugixml-1.10.tar.gz
-        ${PROJECT_BINARY_DIR}/pugixml-1.10.tar.gz)
+    # file(DOWNLOAD
+    #     https://github.com/zeux/pugixml/releases/download/v1.10/pugixml-1.10.tar.gz
+    #     ${PROJECT_BINARY_DIR}/pugixml-1.10.tar.gz)
     execute_process(COMMAND ${CMAKE_COMMAND}
         -E tar xf pugixml-1.10.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
     execute_process(COMMAND ${BUILD_COMMAND_COMMON}
@@ -85,9 +85,9 @@ if(OGRE_BUILD_DEPENDENCIES AND NOT EXISTS ${OGREDEPS_PATH})
     #find_package(Freetype)
     if (NOT FREETYPE_FOUND)
         message(STATUS "Building freetype")
-        file(DOWNLOAD
-            https://download.savannah.gnu.org/releases/freetype/freetype-2.10.1.tar.gz
-            ${PROJECT_BINARY_DIR}/freetype-2.10.1.tar.gz)
+        # file(DOWNLOAD
+        #     https://download.savannah.gnu.org/releases/freetype/freetype-2.10.1.tar.gz
+        #     ${PROJECT_BINARY_DIR}/freetype-2.10.1.tar.gz)
         execute_process(COMMAND ${CMAKE_COMMAND}
             -E tar xf freetype-2.10.1.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
         # patch toolchain for iOS
@@ -110,9 +110,9 @@ if(OGRE_BUILD_DEPENDENCIES AND NOT EXISTS ${OGREDEPS_PATH})

     if(MSVC OR MINGW OR SKBUILD) # other platforms dont need this
         message(STATUS "Building SDL2")
-        file(DOWNLOAD
-            https://libsdl.org/release/SDL2-2.0.20.tar.gz
-            ${PROJECT_BINARY_DIR}/SDL2-2.0.20.tar.gz)
+        # file(DOWNLOAD
+        #     https://libsdl.org/release/SDL2-2.0.20.tar.gz
+        #     ${PROJECT_BINARY_DIR}/SDL2-2.0.20.tar.gz)
         execute_process(COMMAND ${CMAKE_COMMAND}
             -E tar xf SDL2-2.0.20.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
         execute_process(COMMAND ${CMAKE_COMMAND}
@@ -127,10 +127,10 @@ if(OGRE_BUILD_DEPENDENCIES AND NOT EXISTS ${OGREDEPS_PATH})

     if(MSVC OR MINGW OR SKBUILD) # other platforms dont need this
       message(STATUS "Building zlib") # only needed for Assimp
-      file(DOWNLOAD
-          http://zlib.net/zlib-1.2.12.tar.gz
-          ${PROJECT_BINARY_DIR}/zlib-1.2.12.tar.gz
-          EXPECTED_HASH SHA256=91844808532e5ce316b3c010929493c0244f3d37593afd6de04f71821d5136d9)
+      # file(DOWNLOAD
+      #     http://zlib.net/zlib-1.2.12.tar.gz
+      #     ${PROJECT_BINARY_DIR}/zlib-1.2.12.tar.gz
+      #     EXPECTED_HASH SHA256=91844808532e5ce316b3c010929493c0244f3d37593afd6de04f71821d5136d9)
       execute_process(COMMAND ${CMAKE_COMMAND}
           -E tar xf zlib-1.2.12.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
       execute_process(COMMAND ${BUILD_COMMAND_COMMON}
@@ -141,9 +141,9 @@ if(OGRE_BUILD_DEPENDENCIES AND NOT EXISTS ${OGREDEPS_PATH})
           --build ${PROJECT_BINARY_DIR}/zlib-1.2.12 ${BUILD_COMMAND_OPTS})

       message(STATUS "Building Assimp")
-      file(DOWNLOAD
-          https://github.com/assimp/assimp/archive/v5.1.6.tar.gz
-          ${PROJECT_BINARY_DIR}/v5.1.6.tar.gz)
+      # file(DOWNLOAD
+      #     https://github.com/assimp/assimp/archive/v5.1.6.tar.gz
+      #     ${PROJECT_BINARY_DIR}/v5.1.6.tar.gz)
       execute_process(COMMAND ${CMAKE_COMMAND}
           -E tar xf v5.1.6.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
       execute_process(COMMAND ${BUILD_COMMAND_COMMON}
diff --git a/Components/Overlay/CMakeLists.txt b/Components/Overlay/CMakeLists.txt
index ec67ef51a..ee1d22c4a 100644
--- a/Components/Overlay/CMakeLists.txt
+++ b/Components/Overlay/CMakeLists.txt
@@ -22,9 +22,9 @@ if(OGRE_BUILD_COMPONENT_OVERLAY_IMGUI)
   set(IMGUI_DIR "${PROJECT_BINARY_DIR}/imgui-1.85" CACHE PATH "")
   if(NOT EXISTS ${IMGUI_DIR})
     message(STATUS "Downloading imgui")
-    file(DOWNLOAD
-        https://github.com/ocornut/imgui/archive/v1.85.tar.gz
-        ${PROJECT_BINARY_DIR}/imgui.tar.gz)
+    # file(DOWNLOAD
+    #     https://github.com/ocornut/imgui/archive/v1.85.tar.gz
+    #     ${PROJECT_BINARY_DIR}/imgui.tar.gz)
     execute_process(COMMAND ${CMAKE_COMMAND}
         -E tar xf imgui.tar.gz WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
   endif()
```

这样在Configure 过程中就不会下载，而是用自己预先下载好的文件了。

> 这里的 CMake 命令 `file(DOWNLOAD ...)` 是文件的下载命令

## `_Py_object_GC_UnTrack` 错误

在编译过程中如果发现找不到 `_Py_object_GC_UnTrack` 的错误，是因为我们使用的 Python 版本为3.8以上。OGRE 使用了 SWIG 来生成 Python Binding，而在 Python 3.8 中将`_Py_object_GC_UnTrack`更名为 `Py_object_GC_UnTrack`，所以找不到。

解决方式为：将 swig 更新到 4.0.2.

## 编译操作

由于 OGRE 的 CMakeLists.txt 默认配置是生成 `RelWithDebInfo` 类型的 Build，所以使用 vs 打开 sln 解决方案文件后，需要将编译类型配置选择为  `RelWithDebInfo` 而不是默认的 `Debug`，这样才能编译。

修改好后，首先右击 `ALL_BUILD`项目选择“生成”，然后同样生成 `INSALL` 项目。

最后 `SampleBrowser` 设置为启动项目，即可运行 OGRE 的 Demo。