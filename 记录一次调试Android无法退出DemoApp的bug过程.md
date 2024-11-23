## 背景

为了调试 vulkan 渲染器在 Android 上的表现，我编写了一个 demo app，没有 java 代码，全是 native 代码。在使用过程中发现了一个问题：点击返回按钮或者使用返回手势，app 不关闭。只能先最小化然后关闭，为了实现能正常关闭它，调试了两天，现记录在案

## Demo 写法

因为使用 vulkan 来做渲染器，所以注定了要编写 native 代码，又因为没有编写 jni 接口，所以编写了一个纯 native 的应用。应用的主入口是 `void android_main(struct android_app* app)`：

```c++
void HandleAppCommand(android_app * app, int32_t cmd)
{
    // implementation...
}

int32_t HandleAppInput(struct android_app* app, AInputEvent* event)
{
    // implementation...
    return 1;
}

android_app* androidApp;
void android_main(struct android_app* app) {
    PTEngine* engine = new PTEngine();

    app->userData = engine;
    app->onAppCmd = HandleAppCommand;
    app->onInputEvent = HandleAppInput;

    androidApp = app;

    engine->Run();

    delete engine;
}
```

`app->onAppCmd` 回调函数：是需要填充它的，用于处理主要的 app 相关的命令（`APP_CMD_*`），如 `APP_CMD_INIT_WINDOW`、`APP_CMD_TERM_WINDOW` 等。

`app->onInputEvent` 回调函数：用于处理输入事件（`AINPUT_EVENT_TYPE_*`），如触控事件、点击事件等。

## 调试过程

首先在 `APP_CMD_TERM_WINDOW` 分支中打断点 1，点击返回按钮后，断点不进入。然后在程序运行后，将断点 2 打在 `HandleAppCommand()` 函数的入口处，点击返回按钮后，断点 2 还是不进入，所以怀疑 `APP_CMD_TERM_WINDOW` 命令被拦截了或者是没有生成。但是看了所有的代码，没有其他地方有处理此命令的地方。

但是 vulkan 官方示例的 Android demo 能正常退出，因此跟官方示例进行对比，发现大体流程一样。但是官方示例中处理了输入的 key 的事件。然后开始对比两个 demo 的 log 输出。

发现官方示例中，点击返回按钮后，有这么一行 log：

```log
Activity                de....illems.vulkanTrianglevulkan13  I  dispatchKeyEvent KEYCODE_BACK, action:  0
```

它由 Activity 发出，表明了分发 `KEYCODE_BACK` 事件，但是官方示例中并没有对 `AKEYCODE_BACK` 处理。

检查我的 demo 的 log，发现并没有 `dispatchKeyEvent KEYCODE_BACK` 相关的信息，所以怀疑是 `AKEYCODE_BACK` 被拦截了。

检查代码，发现所有的输入都由 `HandleAppInput()` 来处理，其中并没有处理 `AKEYCODE_BACK` 事件，再仔细对比代码，发现官方示例中，`app->onInputEvent` 回调函数默认情况下是返回 0，只有在处理了它感兴趣的事件，如 `AMOTION_EVENT_ACTION_UP` 时，才返回 0。而我的 demo 代码中，默认情况下是返回 1，所以尝试着将默认情况的返回改为 0，再进行调试。

此时发现返回按钮点击之后能将应用退出了，在 log 中也出现了 `dispatchKeyEvent KEYCODE_BACK`。所以罪魁祸首就是 `app->onInputEvent` 回调函数的返回值，在没有处理感兴趣的事件时，应该返回 0，不能无脑返回 1。

## 解释

再次阅读 的函数声明：

```c++
    // Fill this in with the function to process input events.  At this point
    // the event has already been pre-dispatched, and it will be finished upon
    // return.  Return 1 if you have handled the event, 0 for any default
    // dispatching.
    int32_t (*onInputEvent)(struct android_app* app, AInputEvent* event);
```

注释翻译：用于处理输入事件的函数。在这个时候，事件已经被预先分发，并且在返回时将完成处理。如果你已经处理了该事件，返回 1；否则返回 0 以进行默认的事件分发。

所以我们必须只在处理自己需要处理的事件后，将返回值设为 1，设置为 1 后，该事件不会进行后续的分发。对于没有处理的事件，我们必须返回为 0，让系统继续处理此类事件，由于此类事件可能会被其他的组件或者流程使用，如果不返回 0，其他的组件不能处理此事件，会导致一些默认行为失效，比如我碰到的不能返回的情况。

总的来说，回调函数的返回值不能乱填，需要仔细阅读文档和注释。