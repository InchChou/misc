在 API version 30以后 `systemUiVisibility` 属性被废弃了，在网上搜到的安卓全屏显示的文章基本上都是使用的这个属性，所以最好不用，使用安卓文档里推荐的  `WindowCompat` 和 `WindowInsetsControllerCompat`。

### 遇到的问题

1. 标题栏会自己显示
2. 状态栏会自己显示
3. 隐藏状态栏后状态栏那一块还是不显示app中的内容

现在对碰到的三个问题一一解决。

##### 1. 标题栏会自己显示

这个标题栏其实是 `ActionBar`，见 [设置应用栏  | Android 开发者  | Android Developers (google.cn)](https://developer.android.google.cn/develop/ui/views/components/appbar/setting-up?hl=zh-cn)

如果 activity 是继承自 `AppCompatActivity`，则使用默认主题时会自动带一个 `ActionBar`，使用  AppCompat 的其中一个 `NoActionBar` 主题可以将其关掉，如下：

```xml
<application
    android:theme="@style/Theme.AppCompat.Light.NoActionBar"
    />
```

##### 2. 状态栏会自己显示

在使用 `NoActionBar` 主题后，没有`ActionBar`了，但是还是有状态栏，在 Android 中，状态栏`statusBar` 和导航栏 `navigationBar` 统称为系统栏 `systemBar`，这时可以用两种方式关掉系统栏：

1. 使用 `WindowInsetsControllerCompat.hide()` 方法，见[在沉浸模式下隐藏系统栏  | Android 开发者  | Android Developers](https://developer.android.com/develop/ui/views/layout/immersive?hl=zh-cn#java)

首先在 Activity 的 `onCreate()` 方法中获取 `WindowInsetsControllerCompat`，然后用它来隐藏系统栏

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...

    WindowInsetsControllerCompat windowInsetsController =
            WindowCompat.getInsetsController(getWindow(), getWindow().getDecorView());
    indowInsetsController.hide(WindowInsetsCompat.Type.systemBars());
```

此时打开 app 就会发现，系统栏隐藏了，用手在屏幕边缘划一下，系统栏才会出现。但是此时有一个问题，系统栏重新出现后不会再隐藏了，此时需要设置系统栏的行为：

```
windowInsetsController.setSystemBarsBehavior(
            WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
    );
```

`BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE` 表示通过系统手势暂时显示隐藏的系统栏。

2. 设置 `theme` 中的 `android:windowFullscreen` 属性为 `true`：

```xml
<item name="android:windowFullscreen">true</item>
```

这种方法可以达到与 1 同样的效果，即打开 app 时状态栏隐藏，用手在屏幕边缘划一下，状态栏出现一小会，然后隐藏。但是不能隐藏导航栏。

> 这个主题参数与 `android.view.WindowManager.LayoutParams#FLAG_FULLSCREEN` 有关，在 **API level 30** 时废弃。文档中推荐使用 `WindowInsetsController#hide(int)` with `Type#statusBars()` 代替。但是貌似 `themes.xml` 中还可以继续用

##### 3. 隐藏状态栏后状态栏那一块还是不显示app中的内容

这是因为使用的手机是刘海屏，根据安卓文档中的说法，[默认行为](https://developer.android.com/develop/ui/views/layout/display-cutout?hl=zh-cn#default-behaviour) 是“在未设置特殊标志的竖屏模式下，在有刘海屏的设备上，状态栏的高度会调整为至少与刘海屏一样高，并且您的内容会显示在下方区域“。

此时可以参照 [支持刘海屏  | Android 开发者  | Android Developers](https://developer.android.com/develop/ui/views/layout/display-cutout?hl=zh-cn) 中的处理刘海区域章节来设置刘海模式：

1. 设置样式：

```xml
<style name="ActivityTheme">
  <item name="android:windowLayoutInDisplayCutoutMode">
    shortEdges <!-- default, shortEdges, or never -->
  </item>
</style>
```

2. 编程

```java
WindowManager.LayoutParams lp = getWindow().getAttributes();
        lp.layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
```

使用以上任何一种方式即可。

根据文档所说：”在尝试将应用呈现到刘海区域之前，请确保您的应用已配置为[无边框显示内容](https://developer.android.com/develop/ui/views/layout/edge-to-edge?hl=zh-cn)”。所以搜先需要将引用配置为**无边框显示内容**。这可以通过在 `Activity` 的 `onCreate` 中调用 `enableEdgeToEdge` 来在应用中启用全屏显示模式（依赖于`androidx.activity:activity`）。（但是试了一下，使用这个方式在 AppCompatActivity 中会导致依赖性问题，还未有解决方案，所以使用下面的方式。后面搜了一下，貌似这个只能在kotlin中使用）。

或者[手动设置无边框显示](https://developer.android.com/develop/ui/views/layout/edge-to-edge-manually?hl=zh-cn)，使用 [`WindowCompat.setDecorFitsSystemWindows(window, false)`](https://developer.android.com/reference/androidx/core/view/WindowCompat?hl=zh-cn#setDecorFitsSystemWindows(android.view.Window, boolean)) 在系统栏后面布置应用。

### 总结

在新版本安卓中（API version >= 30），将应用设置成全屏的步骤如下所示：

1. 使用 `NoActionBar` 主题；
2. 使用 `WindowInsetsControllerCompat` 关闭系统栏；
3. 使用 `WindowCompat.setDecorFitsSystemWindows(window, false)` 将应用设置为可以在系统栏后布置应用；
4. 设置刘海的 cutMode 属性，使用 theme 的 `<item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>` 或者编程式的 `getWindow().getAttributes().layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;` 均可。



