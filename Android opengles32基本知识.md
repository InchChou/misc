这几天需要在手机上调用 Geometry Shader，然后用 RenderDoc 来抓取应用来调试 GS，所以开发一个 demo 来调用 OpenGL ES。同时踩了一些坑，记录在此。

主要借助了 Android Studio 文档、[JimSeker/opengl: android OpenGL examples (github.com)](https://github.com/JimSeker/opengl) 和 [Opengl ES系列学习--序_红-旺永福的博客-CSDN博客](https://blog.csdn.net/sinat_22657459/article/details/89395495)、[Opengl ES_红-旺永福的博客-CSDN博客](https://blog.csdn.net/sinat_22657459/category_8874366.html)

## 第一步：使用 Android Studio 文档来开发 OpenGL ES demo

官方文档如下：[使用 OpenGL ES 显示图形  | Android 开发者  | Android Developers](https://developer.android.com/training/graphics/opengl)

主要步骤为：

1. 为 OpenGL ES 图形创建 Activity
2. 构建 GLSurfaceView 对象
3. 构建渲染程序类（Renderer）
4. 定义形状并绘制

其中每一步都涉及到创建一个 java 类：

```java
// OpenGLES32Activity.java
// 1. 为 OpenGL ES 图形创建 Activity
public class OpenGLES32Activity extends Activity {
}

// MyGLSurfaceView.java
// 2. 构建 GLSurfaceView 对象
class MyGLSurfaceView extends GLSurfaceView {
    ...
    setEGLContextClientVersion(3);
    ...
}

// MyGLRenderer.java
// 3. 构建渲染程序类（Renderer）
public class MyGLRenderer implements GLSurfaceView.Renderer {
}

// Triangle.java
// 4. 定义形状并绘制
public class Triangle {
}
```

其中踩的坑有：

1. 在创建 Activity 时，需要按照 [创建第二个 activity  | Android 开发者  | Android Developers](https://developer.android.com/training/basics/firstapp/starting-activity#CreateActivity) 的步骤来创建，它会在 `AndroidManifest.xml` 中添加所需的 `<activity>` 元素。同时我们可能还需要修改 `AndroidManifest.xml` 中的对应 `<activity>` 元素来将它设置为启动 activity:

   > ```xml
   > <activity
   >     android:name=".MainActivity"
   >     android:exported="true">
   >     <intent-filter>
   >         <action android:name="android.intent.action.MAIN" />
   > 
   >         <category android:name="android.intent.category.LAUNCHER" />
   >     </intent-filter>
   > </activity>
   > ```

2. 我们这里需要使用 OpenGL ES 3.2 api，但是文档中所用示例是 OpenGL ES 2.0，所以在引入包时，需要引入 `import android.opengl.GLES32;`。

## 第二步：将文档中的 shader 代码改成 GLES 3.2 版本

在默认情况下，如果在 shader 代码中不指定 GLSL 版本，它会使用 GLES 2.0 版本。而我们需要使用到 Geometry Shader，它在 GLES 3.2 版本才支持，所以我们需要指定版本。在 shader 代码的开头添加：

```glsl
#version 320 es
```

即可指定版本为 GLES 3.2。

在指定版本后，Android Studio 文档中的 Shader 代码就不可用了，编译它们的话会出现编译错误。使用

`String GLES32.glGetShaderInfoLog(int var0)` 接口可以获得它们的错误信息，如果返回为空则表示无错误。常见的错误有：

```
// 1.
Keyword 'attribute' is reserved

// 2.
L0002: Undeclared variable ‘gl_FragColor’
```

1. 这是因为 OpenGL ES 3.0 中将 2.0 的 `attribute` 改成了 `in`，顶点着色器的 `varying` 改成 `out`，片段着色器的 `varying` 改成了 `in`，也就是说顶点着色器的输出就是片段着色器的输入，另外 `uniform` 跟2.0用法一样。
2. OpenGL ES 2.0 的 `gl_FragColor` 和 `gl_FragData` 在 `3.0` 中取消掉了，需要自己定义 `out` 变量作为片段着色器的输出颜色，如 `out vec4 fragColor;`。

整体上和我们在 learnopengl 中学习的 `330 core` 语法非常类似。其他的错误见 [OpenGL ES 2.0升级到3.0配置win32环境以及编译所遇bug_NULL____的博客-CSDN博客](https://blog.csdn.net/lb377463323/article/details/77047221)

在将着色器 attach 到 program 并链接后，可以使用 `String GLES32.glGetProgramInfoLog(int var0)` 来检查 program 的状态，如果返回空则无错误。

在迁移到 GLES 3.2 后，顶点着色器的顶点数据最好使用 vao 和 vbo 的形式来传入，见 [你好，三角形 - LearnOpenGL CN (learnopengl-cn.github.io)](https://learnopengl-cn.github.io/01 Getting started/04 Hello Triangle/)

### 检查 GLES 版本

[检查 OpenGL ES 版本](https://developer.android.com/guide/topics/graphics/opengl#version-check)

使用以下两种方法可以检查 GLES 版本：

```java
// 1.
String version = GLES32.glGetString(GLES32.GL_VERSION);
Log.w(TAG, "Version: " + version );
// 2.
int[] vers = new int[2];
GLES32.glGetIntegerv(GLES32.GL_MAJOR_VERSION, vers, 0);
GLES32.glGetIntegerv(GLES32.GL_MINOR_VERSION, vers, 1);
```

如果创建的是低版本的 GLES 上下文，则需要通过实现 `GLSurfaceView.EGLContextFactory` 来创建更高版本的上下文。（此条存疑）

我在在支持 GLES3.2 版本的手机中运行 demo，在 `GLSurfaceView` 中通过 `setEGLContextClientVersion(3)` 指定版本也能创建 GLES 3.2 版本的上下文，这个接口接收的参数为 `int`，所以只能填 `3` 进去。在 google 中搜索 “Android 指定 GLES 3.2” 也没有搜到，只有创建 3.0 的。



## Demo 地址

[InchChou/OpenGLES32Demo (github.com)](https://github.com/InchChou/OpenGLES32Demo)

其中 `Points.java` 使用了 GLES 3.2 版本的 glsl，其中几何着色器参照了 [几何着色器 - LearnOpenGL CN (learnopengl-cn.github.io)](https://learnopengl-cn.github.io/04 Advanced OpenGL/09 Geometry Shader/)