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

# 用到的脚本和patch

`BuildAPK_Release.ps1` 如下所示：

```powershell
$env:JAVA_HOME="D:\PortablePrograms\jdk-17.0.2"
$env:Path="$env:JAVA_HOME\bin;$env:Path"

$env:ANDROID_SDK="D:\PortablePrograms\AndroidSDK"
$env:ANDROID_NDK="D:\PortablePrograms\AndroidSDK\ndk\27.2.12479018"

$RDCRootPath="D:\chou\proj\analyzeTools\graphics\renderdoc\renderdoc"

$x = Split-Path -Parent $MyInvocation.MyCommand.Definition

function GenerateMakefile{
    cd $RDCRootPath
    mkdir build-android
    cd build-android
    cmake -DBUILD_ANDROID=On -DANDROID_ABI=armeabi-v7a -DCMAKE_BUILD_TYPE=Release -DSTRIP_ANDROID_LIBRARY=On -G "MinGW Makefiles" ..

    cd $RDCRootPath
    mkdir build-android64
    cd build-android64
    cmake -DBUILD_ANDROID=On -DANDROID_ABI=arm64-v8a -DCMAKE_BUILD_TYPE=Release -DSTRIP_ANDROID_LIBRARY=On -G "MinGW Makefiles" ..

    cd $x
}

function CompileApk{
    cd $RDCRootPath
    cd "build-android"
    mingw32-make.exe -j12

    cd $RDCRootPath
    cd "build-android64"
    mingw32-make.exe -j12

    cd $x
}

function CopyApkToDir{
    $targetFolder="$RDCRootPath\x64\Release\plugins\android"

    $source1="$RDCRootPath\build-android\bin\org.renderdoc.renderdoccmd.arm32.apk"
    $source2="$RDCRootPath\build-android64\bin\org.renderdoc.renderdoccmd.arm64.apk"

    $exist=Test-Path -Path $targetFolder
    if (-not $exist) {
        New-Item $targetFolder -type Directory
        Write-Host("Make new directory $targetFolder")
    }
    Copy-Item $source1 -Destination $targetFolder -Force
    Copy-Item $source2 -Destination $targetFolder -Force
    Write-Host("Copy done.")

    cd $x
}

Write-Host("Functin choices:")
Write-Host("     1. GenerateMakefile")
Write-Host("     2. CompileApk")
Write-Host("     3. CopyApkToDir")
Write-Host("")

$a = Read-Host("     Your choice is")

# Write-Host($a)

if ($a -eq "1") {
    GenerateMakefile
} elseif ($a -eq "2") {
    CompileApk
} elseif ($a -eq "3") {
    CopyApkToDir
} else {
    Write-Host("Please input 1 or 2 or 3")
}

cmd /c "pause"
```



某些高通手机会在 destroy egl context 时 crash，此时需要修改：

```diff
diff --git a/renderdoc/driver/gl/egl_hooks.cpp b/renderdoc/driver/gl/egl_hooks.cpp
index aa288bdbe..b17310ad9 100644
--- a/renderdoc/driver/gl/egl_hooks.cpp
+++ b/renderdoc/driver/gl/egl_hooks.cpp
@@ -399,6 +399,7 @@ HOOK_EXPORT EGLBoolean EGLAPIENTRY eglDestroyContext_renderdoc_hooked(EGLDisplay
   eglhook.driver.SetDriverType(eglhook.activeAPI);
   {
     SCOPED_LOCK(glLock);
+    EGL.MakeCurrent(dpy, 0L, 0L, ctx); // must MakeCurrent before delete context, may crash on qcom devices
     eglhook.driver.DeleteContext(ctx);
     eglhook.contexts.erase(ctx);
   }
```



`ALooper_pollAll` 被弃用了，替换如下：

```diff
diff --git a/renderdoccmd/renderdoccmd_android.cpp b/renderdoccmd/renderdoccmd_android.cpp
index df90d14cc..e441bef34 100644
--- a/renderdoccmd/renderdoccmd_android.cpp
+++ b/renderdoccmd/renderdoccmd_android.cpp
@@ -523,7 +523,7 @@ void android_main(struct android_app *state)
       }
     }
 
-    if(ALooper_pollAll(1, nullptr, &events, (void **)&source) >= 0)
+    if(ALooper_pollOnce(1, nullptr, &events, (void **)&source) >= 0) //ALooper_pollAll is deprecated
     {
       if(source != NULL)
         source->process(android_state, source);
```



某些 astc 纹理不符合spec要求，导致应用crash，修改如下：

```diff
diff --git a/renderdoc/driver/gl/wrappers/gl_texture_funcs.cpp b/renderdoc/driver/gl/wrappers/gl_texture_funcs.cpp
index 7b72fd584..33999a5cd 100644
--- a/renderdoc/driver/gl/wrappers/gl_texture_funcs.cpp
+++ b/renderdoc/driver/gl/wrappers/gl_texture_funcs.cpp
@@ -3814,7 +3814,8 @@ void WrappedOpenGL::StoreCompressedTexData(ResourceId texId, GLenum target, GLin
           // image size is not an integer multiple of the block size, so we need to take into
           // account that in the loop
           size_t roundedUpHeight = AlignUp((uint32_t)height, blockSize[1]);
-          for(size_t y = 0; y < roundedUpHeight; y += blockSize[1])
+          // some astc texture don't meet the spec, height is smaller than roundedUpHeight, cause overflow
+          for(size_t y = 0; y < roundedUpHeight && y < (uint32_t)height; y += blockSize[1])
           {
             memcpy(cdData.data() + dstOffset, srcPixels + srcOffset, srcRowSize);
             srcOffset += srcRowSize;

```



某些应用启用了不支持的接口，会导致抓帧关闭，此时需要手动开启，修改如下：

```diff
diff --git a/renderdoc/driver/gl/gl_driver.cpp b/renderdoc/driver/gl/gl_driver.cpp
index c1e92526a..65a53ab8a 100644
--- a/renderdoc/driver/gl/gl_driver.cpp
+++ b/renderdoc/driver/gl/gl_driver.cpp
@@ -2087,6 +2087,19 @@ void WrappedOpenGL::SwapBuffers(WindowingSystem winSystem, void *windowHandle)
     if(overlay & eRENDERDOC_Overlay_Enabled)
     {
       int flags = 0;
+      // maually enable capture for some function
+      const char* enableCaptureunsupportedFunctions[] = {
+        "glEGLImageTargetTexture2DOES"
+      };
+      size_t enableCaptureunsupportedFunctionsLength = sizeof(enableCaptureunsupportedFunctions) / sizeof(const char*);
+      for (size_t index = 0; index < enableCaptureunsupportedFunctionsLength; ++index)
+      {
+        if (m_UnsupportedFunctions.count(enableCaptureunsupportedFunctions[index]) > 0)
+        {
+          RDCWARN("Unsupported %s is used, but we enabled capture.", enableCaptureunsupportedFunctions[index]);
+          m_UnsupportedFunctions.erase(enableCaptureunsupportedFunctions[index]);
+        }
+      }
       // capturing is disabled if unsupported functions have been used, or this context is legacy
       if(ctxdata.Legacy() || !m_UnsupportedFunctions.empty())
         flags |= RenderDoc::eOverlay_CaptureDisabled;

```

