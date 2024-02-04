在调试 Vulkan 应用的过程中，发现使用 Android Studio 的模拟器无法打开 Vulkan 应用，debug 之后发现是连 PhysicalDevice 都无法获取，获取到的 count 一直是 0，所以无法进行下去，意思就是模拟器无法支持 Vulkan。

在一番搜索后，发现了模拟器的版本说明，里面提到了在 29.0.6 版本开始对 Vulkan 进行了支持：

[模拟器版本说明  | Android Studio  | Android Developers](https://developer.android.com/studio/releases/emulator?hl=zh-cn#29.0.6-vulkan)

但是在打开开关后，依然无法运行 Vulkan 引用，这回是在 `vkQueuePresentKHR` 崩溃，连带模拟器一起闪退。并且在启动模拟器时会一直崩溃，崩溃几次才能启动。

感觉是模拟器的 Vulkan 支持还没做好，可能还需要等。