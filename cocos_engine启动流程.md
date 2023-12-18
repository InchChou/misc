cocos 3.6后的启动流程稍有改动，与网上说的不太一样。

> [Cocos Creator 源码解读：引擎启动与主循环 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/647616112)

### 启动流程

在 BaseGame.cpp 的 `BaseGame::init()` 中，经过一些初始化操作后，使用 `runScript("main.js")` 调用构建项目下生成的 `main.js` 脚本文件。

### main.js

首先读取了 `src/import-map.json` 中的需要导入的条目，这个 json 文件中最关键的就是 `"cc":"./cocos-js/cc.js"`  这个条目。

然后调用了 `System.import('./src/application.js')` 导入 application.js 脚本。每行注释如下：

```js
System.import('./src/application.js')
.then(({ Application }) => {
    return new Application(); // 使用application.js创建application
}).then((application) => {
    return System.import('cc').then((cc) => { // 导入cc条目，即上述的cc.js，创建了cc实例
        require('jsb-adapter/engine-adapter.js');
        return application.init(cc); // 使用cc实例初始化applicatin
    }).then(() => {
        return application.start(); // 调用 start 运行application
    });
}).catch((err) => {
    console.error(err.toString() + ', stack: ' + err.stack);
});
```

这里的 cc 实例就是 cocos engine 实例。

### application.start()

这里设置了一些启动参数，然后调用 `cc.game.run()` 来启动 cc.game。

### cc.game

`cc.game` 对象是 `cc.Game` 类的一个实例，`cc.game` 包含了游戏主体信息并负责驱动游戏。

说人话，`cc.game` 对象就是管理引擎生命周期的模块，启动、暂停和重启等操作都需要用到它。

其源码在 `cocos\game\game.ts`。

```js
public run (onStart?: Game.OnStart) {
    if (onStart) {
        this.onStart = onStart;
    }
    if (!this._inited || EDITOR_NOT_IN_PREVIEW) {
        return;
    }
    this.resume();
}

public resume () {
    if (!this._paused) { return; }
    input._clearEvents();
    this._paused = false;
    this._pacer?.start();
    this.emit(Game.EVENT_RESUME);
}
```

这里又调用了 `_pacer.start()` 。这里 Pacer 是一组接口，相当于 c++ 的纯虚类。

```typescript
declare module 'pal/pacer' {

    export class Pacer {
        get targetFrameRate (): number;
        set targetFrameRate (val: number);
        onTick: (() => void) | null;
        start (): void;
        stop (): void;
    }
}
```

在 `pal/pacer/` 目录下，有一些 Pacer 的实例。如 `Pacer-native.ts` 的 Pacer 实例。部分代码如下：

```typescript
export class Pacer {
    constructor () {
        this._updateCallback = () => {
            if (this._isPlaying) {
                this._rafHandle = requestAnimationFrame(this._updateCallback);
            }
            if (this._onTick) {
                this._onTick();
            }
        };
    }
    ...
    start (): void {
        if (this._isPlaying) return;
        this._rafHandle = requestAnimationFrame(this._updateCallback);
        this._isPlaying = true;
    }
}
```

`start()` 函数设置了 `requestAnimationFrame` 的回调函数为自己 `_updateCallback` 函数，而这个函数中最主要的是 `_onTick`。

> requestAnimationFrame 是 js 中的一个定时器函数，比较复杂，这里简单理解为定时执行设置的回调函数。

### onTick()

onTick 函数是由 cc.game 在 `initPacer()` 中设置的，设置的函数为 `game._updateCallback ()` 函数。这个函数做了实际的 tick 工作：

1. 如果 SplashScreen 没有更新完，则继续更新
2. 如果需要加载 scene，则调用 `director.loadScene()` 来加载场景
3. 都做完了，则调用 `director.tick()` 来进行 tick。

> director 中之前有 mainLoop，现在统一改为 tick 函数。



至此除资源加载外的启动流程已经差不多了，后面就是加载场景资源以及 tick 了，这些都是 director 的重点。