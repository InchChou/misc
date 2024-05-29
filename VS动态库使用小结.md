## 前言

最近在使用 visual studio 动态库工程时碰到了一些小问题，现在记录下来。

起因是想要用 visual studio 的项目引用功能。参见“VisualStudio项目引用简介.md”。在做实验的时候发现，dll 项目没有生成 lib 库（一般随 dll 生成的会有一个 lib 库，里面是导出的符号表），导致依赖于 dll 项目的主项目无法[隐式链接]([https://learn.microsoft.com/zh-cn/cpp/build/linking-an-executable-to-a-dll?view=msvc-170#implicit-linking)成功。经过搜索后发现，是因为没有显式导出符号表。如果需要显式导出符号表，需要使用 `__declspec(dllexport)` 来声明需要导出的函数，这样在编译 dll 的时候才会导出符号表。

### 用法

主要参照微软文档：[演练：创建和使用自己的动态链接库 (C++) | Microsoft Learn](https://learn.microsoft.com/zh-cn/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp?view=msvc-170)

在 `MathLibrary.h` 中，定义了一段宏

```cpp
#ifdef MATHLIBRARY_EXPORTS
#define MATHLIBRARY_API __declspec(dllexport)
#else
#define MATHLIBRARY_API __declspec(dllimport)
#endif

MATHLIBRARY_API unsigned fibonacci_index();
```

在 MathLibrary 项目中，会在预处理器宏中添加 `MATHLIBRARY_EXPORTS`，此时MathLibrary 项目中的 `MathLibrary.cpp` include 了 `MathLibrary.h`，在预处理时，

`MATHLIBRARY_EXPORTS` 就生效了，此时 `MATHLIBRARY_API` 宏会对函数声明设置 `__declspec(dllexport)` 修饰符。 此修饰符指示编译器和链接器从 DLL 导出函数或变量，以便其他应用程序可以使用它。

在其他项目（如 A 项目）使用 MathLibrary 库时，也会include `MathLibrary.h` 头文件，此时 A 项目的预处理器宏未定义 `MATHLIBRARY_EXPORTS`，所以 `MATHLIBRARY_API` 宏会对函数声明设置 `__declspec(dllimport)` 修饰符。此修饰符指示编译器和链接器从 DLL 导入函数或变量，供此项目使用。

> 当库是静态库时，不需要使用这些东西，所有符号是默认导出的。

### 经典模板

在使用 khronos ktx 库时，发现它的头文件 `ktx.h` 写的不错，摘录如下：

```cpp
#if defined(KHRONOS_STATIC)
  #define KTX_API
#elif defined(_WIN32) || defined(__CYGWIN__)
  #if !defined(KTX_API)
    #if __GNUC__
      #define KTX_API __attribute__ ((dllimport))
    #elif _MSC_VER
      #define KTX_API __declspec(dllimport)
    #else
      #error "Your compiler's equivalent of dllimport is unknown"
    #endif
  #endif
#elif defined(__ANDROID__)
  #define KTX_API __attribute__((visibility("default")))
#else
  #define KTX_API
#endif

KTX_API KTX_error_code KTX_APIENTRY
ktxTexture_CreateFromMemory(const ktx_uint8_t* bytes, ktx_size_t size,
                            ktxTextureCreateFlags createFlags,
                            ktxTexture** newTex);
```

第一二行的 `#if defined(KHRONOS_STATIC)` 是指如果生成的是静态库，则定义 `KTX_API` 宏为空，即不使用修饰符。如果是 dll 动态库的话，则需要给预处理传递定义 `KTX_API` 为 `__declspec(dllexport)`。

后面的都是平台相关的，以后编写库，如果需要导出符号表的话可以参考这个写法。
