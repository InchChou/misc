# 简介

Renderdoc的简介不再赘述。在我们开发过程中，使用renderdoc来调试的目标设备主要是安卓手机，其主要原理是在安卓手机上安装renderdoc的apk，通过apk与电脑上的renderdoc gui进行通信来实现各项操作。所以我们只需要编译renderdoc的apk与电脑上的renderdoc gui。

# Renderdoc GUI编译

本节只叙述在windows环境下编译Renderdoc GUI的过程。

直接使用visual studio 2015 及以上的版本打开`renderdoc.sln`，使用”重定目标解决方案“来将Renderdoc工程的平台工具集版本和Windows SDK版本重定向到本机所安装的版本。除了Windows SDK，Renderdoc没有其他任何外部依赖。

在Windows上，推荐使用`Development`设置来编译，它是可调试的，又不会特别慢。生成解决方案之后，`qrenderdoc.exe`就可以直接打开了。如果需要调试安卓手机上面的应用，则需要编译renderdoc的apk。

# Renderdoc apk的编译

为了编译APK，首先需要检查自己是否有如下依赖：

1. CMake, [Link](https://cmake.org/)
2. MinGW-W64
3. Git
4. JDK8, [Link](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
5. Android SDK, [Link](https://developer.android.com/studio)
6. Android NDK, [Link](https://developer.android.com/ndk/downloads)

Renderdoc作者使用的是：NDK 14b, SDK tools 3859397, SDK build-tools 26.0.1, SDK platform android-23, Java 8。

本人使用`NDK r21e`,` SDK build-tools 29.0.2`, `SDK platform android-29`, `java 8` 编译成功。

首先设置环境变量：

```bat
set ANDROID_SDK=<path_to_sdk_root>
set ANDROID_NDK=<path_to_ndk_root>
set JAVA_HOME=<path_to_jdk_root>
```

进入renderdoc目录后，执行以下命令，编译32位apk：

```bat
mkdir build-android
cd build-android
cmake -DBUILD_ANDROID=On -DANDROID_ABI=armeabi-v7a -G "MinGW Makefiles" ..
mingw32-make -j12
```

重复上次操作，编译64位apk

```bat
mkdir build-android64
cd build-android64
cmake -DBUILD_ANDROID=On -DANDROID_ABI=arm64-v8a -G "MinGW Makefiles" ..
mingw32-make -j12
```

编译出来的apk都在`bin/`目录中，将他们拷到renderdoc GUI生成目录的`renderdoc\x64\Development\plugins\android`中。此时再打开renderdo GUI即可连接安卓手机。



## 其他注意事项

1. Renderdoc GUI和Renderdoc apk会校验编译时期的git commit id，所以需要使用同一笔提交来编译这两个组件
2. 使用vs2022时，需要修改`util/WindowsSDKTarget.props`中的win10 sdk为10.0，如下：

```diff
--- a/util/WindowsSDKTarget.props
+++ b/util/WindowsSDKTarget.props
@@ -12,7 +12,7 @@

        <!-- if we found the SDK version, use it. Otherwise fall back to 8.1. Since we require VS2015 which installed the 8.1 SDK at minimum we don't need to fallback any earlier (though we don't use anything from the 8.1 SDK either) -->
        <WindowsTargetPlatformVersion Condition="'$(RenderDocLatestWin10SDKVersion)'!=''">$(RenderDocLatestWin10SDKVersion)</WindowsTargetPlatformVersion>
-       <WindowsTargetPlatformVersion Condition="'$(RenderDocLatestWin10SDKVersion)'==''">8.1</WindowsTargetPlatformVersion>
+       <WindowsTargetPlatformVersion Condition="'$(RenderDocLatestWin10SDKVersion)'==''">10.0</WindowsTargetPlatformVersion>
 </PropertyGroup>

 </Project>
```



# 修改安卓端库名

有些手机游戏会检测加载的库，判断是否处于抓帧状态，这个时候游戏会闪退。需要改变renderdoc的库名：

```diff
diff --git a/renderdoc/CMakeLists.txt b/renderdoc/CMakeLists.txt
index f838d014e..af6e76658 100644
--- a/renderdoc/CMakeLists.txt
+++ b/renderdoc/CMakeLists.txt
@@ -629,7 +629,7 @@ if(ANDROID)
     endif()
     set_target_properties(renderdoc PROPERTIES LINK_FLAGS "${RDOC_LINK_FLAGS}")
     # rename output library
-    set_target_properties(renderdoc PROPERTIES OUTPUT_NAME "VkLayer_GLES_RenderDoc")
+    set_target_properties(renderdoc PROPERTIES OUTPUT_NAME "EGL.18")
 
     if(STRIP_ANDROID_LIBRARY AND ANDROID_STRIP_TOOL AND RELEASE_MODE)
         add_custom_command(TARGET renderdoc POST_BUILD
diff --git a/renderdoc/android/android.cpp b/renderdoc/android/android.cpp
index a2d7290f3..e8ca47a47 100644
--- a/renderdoc/android/android.cpp
+++ b/renderdoc/android/android.cpp
@@ -310,7 +310,8 @@ bool CheckAndroidServerVersion(const rdcstr &deviceID, ABI abi)
   rdcstr hostVersionName = GitVersionHash;
 
   // False positives will hurt us, so check for explicit matches
-  if((hostVersionCode == versionCode) && (hostVersionName == versionName))
+  // if((hostVersionCode == versionCode) && (hostVersionName == versionName))
+  if(hostVersionCode == versionCode)
   {
     RDCLOG("Installed server version (%s:%s) is compatible", versionCode.c_str(),
            versionName.c_str());
diff --git a/renderdoc/common/globalconfig.h b/renderdoc/common/globalconfig.h
index 2fe62571e..42add1c59 100644
--- a/renderdoc/common/globalconfig.h
+++ b/renderdoc/common/globalconfig.h
@@ -167,7 +167,7 @@ enum
 #define RENDERDOC_VULKAN_LAYER_NAME "VK_LAYER_RENDERDOC_Capture"
 #define RENDERDOC_VULKAN_LAYER_VAR "ENABLE_VULKAN_RENDERDOC_CAPTURE"
 
-#define RENDERDOC_ANDROID_LIBRARY "libVkLayer_GLES_RenderDoc.so"
+#define RENDERDOC_ANDROID_LIBRARY "libEGL.18.so"
 
 // This MUST match the package name in the build process that generates per-architecture packages
 #define RENDERDOC_ANDROID_PACKAGE_BASE "org.renderdoc.renderdoccmd"
```

