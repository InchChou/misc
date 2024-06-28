## 背景

我有一个 Android Native 程序，想要使用触屏功能：

1. 使用单指拖动时，转动相机
2. 使用双指拖动时，更改其他参数。

## 过程

### GPT 生成的第一版

首先询问 GPT，得到了如下输入处理函数：

```c++
static float initial_x1 = 0.0f, initial_y1 = 0.0f;
static float initial_x2 = 0.0f, initial_y2 = 0.0f;
static int is_two_finger_drag = 0;
static int is_single_finger_drag = 0;

void single_finger_drag(float dx, float dy) {
    LOGI("Single-finger drag: dx = %f, dy = %f", dx, dy);
    // 在这里添加单指拖动时要执行的代码
    camera.rotate(Math::Vec3(-dy * camera.rotationSpeed,
                             dx * camera.rotationSpeed,
                             0.0f));
}

void two_finger_drag(float dx, float dy) {
    LOGI("Two-finger drag: dx = %f, dy = %f", dx, dy);
    // 在这里添加双指拖动时要执行的代码
}

void handle_input(struct android_app* app, AInputEvent* event) {
    if (AInputEvent_getType(event) == AINPUT_EVENT_TYPE_MOTION) {
        int32_t action = AMotionEvent_getAction(event);
        int32_t actionMasked = action & AMOTION_EVENT_ACTION_MASK;
        int32_t pointer_count = AMotionEvent_getPointerCount(event);

        if (pointer_count == 1) {
            float x1 = AMotionEvent_getX(event, 0);
            float y1 = AMotionEvent_getY(event, 0);

            if (actionMasked == AMOTION_EVENT_ACTION_DOWN) {
                initial_x1 = x1;
                initial_y1 = y1;
                is_single_finger_drag = 1;
            } else if (actionMasked == AMOTION_EVENT_ACTION_MOVE && is_single_finger_drag) {
                float dx = x1 - initial_x1;
                float dy = y1 - initial_y1;

                single_finger_drag(dx, dy);

                // Update initial positions for the next move event
                initial_x1 = x1;
                initial_y1 = y1;
            } else if (actionMasked == AMOTION_EVENT_ACTION_UP) {
                is_single_finger_drag = 0;
            }
        } else if (pointer_count == 2) {
            float x1 = AMotionEvent_getX(event, 0);
            float y1 = AMotionEvent_getY(event, 0);
            float x2 = AMotionEvent_getX(event, 1);
            float y2 = AMotionEvent_getY(event, 1);

            if (actionMasked == AMOTION_EVENT_ACTION_POINTER_DOWN) {
                initial_x1 = x1;
                initial_y1 = y1;
                initial_x2 = x2;
                initial_y2 = y2;
                is_two_finger_drag = 1;
            } else if (actionMasked == AMOTION_EVENT_ACTION_MOVE && is_two_finger_drag) {
                float dx1 = x1 - initial_x1;
                float dy1 = y1 - initial_y1;
                float dx2 = x2 - initial_x2;
                float dy2 = y2 - initial_y2;

                float avg_dx = (dx1 + dx2) / 2.0f;
                float avg_dy = (dy1 + dy2) / 2.0f;

                two_finger_drag(avg_dx, avg_dy);

                // Update initial positions for the next move event
                initial_x1 = x1;
                initial_y1 = y1;
                initial_x2 = x2;
                initial_y2 = y2;
            } else if (actionMasked == AMOTION_EVENT_ACTION_POINTER_UP) {
                int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK) >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
                if (pointerIndex == 0 || pointerIndex == 1) {
                    is_two_finger_drag = 0;
                }
            }
        }
    }
}
```

其中 `initial_x1/y1` 是用来记录第一个手指的位置，`initial_x2/y2` 是用来记录第二个手指的位置。

看上去这个处理函数是可以用的。但是有个大问题：**在双指拖动后，抬起一根手指，相机瞬间进行了巨大的旋转**。

打LOG查看 `dx` 和 `dy` 的值，发现在抬起一根手指时，`dx` 和 `dy` 有巨大的值，例如：`dx = -219.125000, dy = -401.625000`。

### GPT 生成的第二版

再次询问 GPT：“在双指拖动过程中，如果其中一个手指抬起，不仅需要正确重置 is_two_finger_drag 标志，还需要做其他的判断。请你改进，并且将改进后的代码与改进前的代码高亮出来。”得到如下代码：

```c++
...
void handle_input(struct android_app* app, AInputEvent* event) {
    if (AInputEvent_getType(event) == AINPUT_EVENT_TYPE_MOTION) {
        int32_t action = AMotionEvent_getAction(event);
        int32_t actionMasked = action & AMOTION_EVENT_ACTION_MASK;
        int32_t pointer_count = AMotionEvent_getPointerCount(event);

        if (pointer_count == 1) {
            float x1 = AMotionEvent_getX(event, 0);
            float y1 = AMotionEvent_getY(event, 0);

            if (actionMasked == AMOTION_EVENT_ACTION_DOWN) {
                initial_x1 = x1;
                initial_y1 = y1;
                is_single_finger_drag = 1;
            } else if (actionMasked == AMOTION_EVENT_ACTION_MOVE && is_single_finger_drag) {
                float dx = x1 - initial_x1;
                float dy = y1 - initial_y1;

                single_finger_drag(dx, dy);

                // Update initial positions for the next move event
                initial_x1 = x1;
                initial_y1 = y1;
            } else if (actionMasked == AMOTION_EVENT_ACTION_UP) {
                is_single_finger_drag = 0;
            }
        } else if (pointer_count == 2) {
            float x1 = AMotionEvent_getX(event, 0);
            float y1 = AMotionEvent_getY(event, 0);
            float x2 = AMotionEvent_getX(event, 1);
            float y2 = AMotionEvent_getY(event, 1);

            if (actionMasked == AMOTION_EVENT_ACTION_POINTER_DOWN) {
                initial_x1 = x1;
                initial_y1 = y1;
                initial_x2 = x2;
                initial_y2 = y2;
                is_two_finger_drag = 1;
                **is_single_finger_drag = 0; // 确保不再处理单指拖动**
            } else if (actionMasked == AMOTION_EVENT_ACTION_MOVE && is_two_finger_drag) {
                float dx1 = x1 - initial_x1;
                float dy1 = y1 - initial_y1;
                float dx2 = x2 - initial_x2;
                float dy2 = y2 - initial_y2;

                float avg_dx = (dx1 + dx2) / 2.0f;
                float avg_dy = (dy1 + dy2) / 2.0f;

                two_finger_drag(avg_dx, avg_dy);

                // Update initial positions for the next move event
                initial_x1 = x1;
                initial_y1 = y1;
                initial_x2 = x2;
                initial_y2 = y2;
            } else if (actionMasked == AMOTION_EVENT_ACTION_POINTER_UP) {
                int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK) >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
                if (pointerIndex == 0 || pointerIndex == 1) {
                    is_two_finger_drag = 0;

                    // 将剩下的一个手指转换为单指拖动
                    if (pointer_count == 2) {
                        int remainingPointerIndex = (pointerIndex == 0) ? 1 : 0;
                        initial_x1 = AMotionEvent_getX(event, remainingPointerIndex);
                        initial_y1 = AMotionEvent_getY(event, remainingPointerIndex);
                        **is_single_finger_drag = 1;**
                    }
                }
            }
        }
    }
}
...
```

其中 `**` 围起来的部分就是差别，GPT 解释如下：

**高亮的改进点**

1. 在 `AMOTION_EVENT_ACTION_POINTER_DOWN` 事件中重置 `is_single_finger_drag` 标志：

   ```c++
   is_single_finger_drag = 0; // 确保不再处理单指拖动
   ```

2. 在 `AMOTION_EVENT_ACTION_POINTER_UP` 事件中将剩下的一个手指转换为单指拖动：

   ```c++
   if (pointer_count == 2) {
       int remainingPointerIndex = (pointerIndex == 0) ? 1 : 0;
       initial_x1 = AMotionEvent_getX(event, remainingPointerIndex);
       initial_y1 = AMotionEvent_getY(event, remainingPointerIndex);
       is_single_finger_drag = 1;
   }
   ```

在试过之后，发现真的bug消失了，所以就是这两处修改奏效了。

为什么它会奏效呢？询问GPT，它没有给出太有效的信息，所以决定自己探索。

### 探索过程

注意到在 `AMOTION_EVENT_ACTION_POINTER_UP` 事件中将剩下的一个手指转换为单指拖动部分，有个 `remainingPointerIndex`，后续使用它来获取对应手指的坐标，并且赋给了第一个手指的初始值。

首先我们来看看GPT给我们的代码中的每个事件和事件动作（查看谷歌文档）：

- event（`AINPUT_EVENT_XXX`），使用 `AInputEvent_getType` 获取 event 的类型：
  - `AINPUT_EVENT_TYPE_MOTION`：表示事件是触摸事件；获取到 event 后，可以进行判断，我们这里只处理触摸事件。

- Motion event actions（`AMOTION_EVENT_ACTION_XXX`），使用 `AMotionEvent_getAction` 从 event 中获取 event actions：
  1. `AMOTION_EVENT_ACTION_DOWN`：按下的手势已开始，动作包含初始的起始位置；
  2. `AMOTION_EVENT_ACTION_MOVE`：按压手势期间发生了变化（在 `AMOTION_EVENT_ACTION_DOWN` 和 `AMOTION_EVENT_ACTION_UP` 之间）。动作包含最近的点，以及自上次按下或移动事件以来的任何中间点。
  3. `AMOTION_EVENT_ACTION_UP`：按下手势已完成，动作包含最终释放位置以及自上次按下或移动事件以来的任何中间点。
  4. `AMOTION_EVENT_ACTION_POINTER_DOWN`：非主要指针（手指）已按下。`AMOTION_EVENT_ACTION_POINTER_INDEX_MASK` 中的位指示哪个指针发生了变化。
  5. `AMOTION_EVENT_ACTION_POINTER_UP`：非主要指针已抬起。

我们还使用了 `pointer_count = AMotionEvent_getPointerCount(event)` 获取指针数目，文档中解释为：获取此事件所包含的数据指针的数量，永远大于等于1。这个比较好理解，当我们只用一根手指触摸时，它返回 1；用两个手指时，返回 2。

当指针数目为 1 时，我们使用了 event action 1、2、3 。数目为 2 时，我们使用了 event action 4、2、5。即当两个手指触控时，监测手指按下和抬起的 action 是 pointer 版本。

在各事件和事件动作对应的逻辑中打印log，来看各个事件和事件动作的顺序：

```c++
int32_t pointer_count = AMotionEvent_getPointerCount(event);
int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK)
        >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;

std::string actions[5] = {
        "ACTION_DOWN",
        "ACTION_MOVE",
        "ACTION_UP",
        "POINTER_DOWN",
        "POINTER_UP"
};
std::string curAction;
if (actionMasked == AMOTION_EVENT_ACTION_DOWN) {
    curAction = actions[0];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[1];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[2];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[3];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[4];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[5];
}
LOGI("pointer_count = %d, %s, pointerIndex = %d", pointer_count, curAction.c_str(), pointerIndex);
```

按照如下顺序进行第一种测试：

1. 按下一根手指
2. 按下第二根手指
3. 抬起第一根手指
4. 抬起第二根手指

可以看到log如下

```
pointer_count = 1, ACTION_DOWN, pointerIndex = 0  // 第一根手指按下
pointer_count = 1, ACTION_MOVE, pointerIndex = 0  // 按下后拖动或者未拖动都会显示MOVE
...xN
pointer_count = 1, ACTION_MOVE, pointerIndex = 0
pointer_count = 2, POINTER_DOWN, pointerIndex = 1 // 第二根手指按下
pointer_count = 2, ACTION_MOVE, pointerIndex = 0  // 按下后拖动或者未拖动都会显示MOVE
...xN
pointer_count = 2, ACTION_MOVE, pointerIndex = 0
pointer_count = 2, POINTER_UP, pointerIndex = 0   // 第一根手指抬起
pointer_count = 1, ACTION_MOVE, pointerIndex = 0  // 第二根手指继续按着
...xN
pointer_count = 1, ACTION_MOVE, pointerIndex = 0
pointer_count = 1, ACTION_UP, pointerIndex = 0    // 第二根手指抬起
```

其中 `xN` 表示前面一条log重复了 N 次。可以看到：

- 第一根手指按下时，事件 action 为 `ACTION_DOWN`，`pointer_count` 为 1，`pointerIndex` 为 0。
- 第二根手指按下时，事件 action 为 `POINTER_DOWN`，`pointer_count` 变为 2，`pointerIndex` 为 1。
- 第一根手指抬起时，事件 action 为`POINTER_UP`，`pointer_count` 仍为 2，`pointerIndex` 为 0。
- 第二根手指抬起时，事件 action 为`ACTION_UP`，`pointer_count` 为 1，`pointerIndex` 为 0。
- 不管有几根手指按下，不管有没有拖动，事件 action 都为 `ACTION_MOVE`，`pointerIndex` 都为 0。

按照如下顺序进行第二种测试：

1. 按下一根手指
2. 按下第二根手指
3. 抬起第二根手指
4. 抬起第一根手指

```
pointer_count = 1, ACTION_DOWN, pointerIndex = 0  // 第一根手指按下
pointer_count = 1, ACTION_MOVE, pointerIndex = 0  // 按下后拖动或者未拖动都会显示MOVE
...x N
pointer_count = 1, ACTION_MOVE, pointerIndex = 0
pointer_count = 2, POINTER_DOWN, pointerIndex = 1 // 第二根手指按下
pointer_count = 2, ACTION_MOVE, pointerIndex = 0  // 按下后拖动或者未拖动都会显示MOVE
...x N
pointer_count = 2, ACTION_MOVE, pointerIndex = 0
pointer_count = 2, POINTER_UP, pointerIndex = 1   // 第二根手指抬起
pointer_count = 1, ACTION_MOVE, pointerIndex = 0  // 第一根手指继续按着
...x N
pointer_count = 1, ACTION_MOVE, pointerIndex = 0
pointer_count = 1, ACTION_UP, pointerIndex = 0    // 第一根手指抬起
```

可以看到：

- 第一根手指按下时，事件 action 为 `ACTION_DOWN`，`pointer_count` 为 1，`pointerIndex` 为 0。
- 第二根手指按下时，事件 action 为 `POINTER_DOWN`，`pointer_count` 变为 2，`pointerIndex` 为 1。
- 第二根手指抬起时，事件 action 为`POINTER_UP`，`pointer_count` 仍为 2，`pointerIndex` 为 1。
- 第一根手指抬起时，事件 action 为`ACTION_UP`，`pointer_count` 为 1，`pointerIndex` 为 0。
- 不管有几根手指按下，不管有没有拖动，事件 action 都为 `ACTION_MOVE`，`pointerIndex` 都为 0。

两次实验不同点在第3步和第4步，抬起手指的顺序不同。在从两指切换到一指的过程中，事件 action 为`POINTER_UP`，如果抬起第一根手指，`pointerIndex` 为 0；如果抬起第二根手指，`pointerIndex` 为 1。

#### 推测

安卓支持了多点触控，所以可以推测：

- 当我们按下第一根手指（index = 0）时，开始有触摸事件，由 `ACTION_DOWN` action 指明第一根手指按下，此 action 在整个触摸过程中只出现一次。
- 当我们依次按下手指 0、1、2、3...（其index分别为 0、1、2、3...），屏幕上多于一根手指时，手指 i 的抬起和按下会由  `POINTER_XX` 系列事件 action 来指明，`pointerIndex` 指明的是第几个按下的手指。特别的，手指 0 的按下不会由 `POINTER_DOWN` 来指明。
- 不管有几根手指，只要有手指在屏幕上，就会有 `ACTION_MOVE`。

此推测需要进一步验证，但在我这个 case 中已经够用。

使用 GPT 给的第一版 code，再重复上面的两种测试，发现只有第二种测试会导致相机突然旋转过多：如果第一根手指一直保持在屏幕上，不会出现bug；如果从二指切换成单指时，抬起的是第一根手指，会出现bug。

#### Bug 分析

观察 GPT 给的 code，可以看到它使用的是 `AMotionEvent_getX/Y()` 来获取手指位置，查阅文档，此函数的两个参数分别为 `event` 和 `pointer_index`，获取此事件的**给定指针索引**的当前 X/Y 坐标。code 中写的逻辑是：当只有一根手指时，获取 index = 0 的指针坐标；当有两根手指时，获取 index = 0 和 index = 1 的指针坐标。

这个看上去很合理，但是如**推测**中所说，按在屏幕上的第一根手指 index 为 0，第二根手指 index 为 1，如果切换为单指时抬起第一根手指并且继续按着第二根手指，那么此时第二根手指的 index 究竟是多少呢？

这个问题可能就是引发bug的问题。

经过实验，可以知道此时第二根手指的 index 其实变成 0 了，此时在 `if (pointer_count == 1)` 分支中通过  `AMotionEvent_getX/Y(event, 0)` 获取到的手指位置其实是第二根手指的位置。而切换为单指之前，`AMotionEvent_getX/Y(event, 0)` 获取到的手指位置一直是第一根手指的位置。所以总的bug流程可以看作如下：

1. 在抬起第一根手指前，第一根手指和第二根手指的位置不同，`initial_x0/y0` 记录的是第一根手指的位置，`initial_x1/y1` 记录的是第二根手指的位置。
2. 在抬起第一根手指时（即 `if (actionMasked == AMOTION_EVENT_ACTION_POINTER_UP)` 分支），使用标志位切换到了单指模式。
3. 在第一次进入单指模式中的 move action 时，即 `if (actionMasked == AMOTION_EVENT_ACTION_MOVE && is_single_finger_drag)` 分支，此时通过  `AMotionEvent_getX/Y(event, 0)` 获取到的手指位置其实是第二根手指的位置，此位置与 `initial_x0/y0`（记录的是刚才的第一根手指的位置） 相减得到 `dx/dy`，由于两根手指的位置不同，所以 `dx/dy` 就很大，造成了 bug 的发生。

所以需要在抬起手指时，进行判断，并将 `initial_x0/y0` 的值正确更新。这种做法就是 GPT 给的修改后的做法。

#### 进一步探究

在进行多次实验后，无法找出  `pointer_index` 的确切规律，经过搜索后，发现还是得从谷歌的文档中来寻找整个机制和规律：[处理多点触控手势  | Views  | Android Developers](https://developer.android.com/develop/ui/views/touch-and-input/gestures/multi?hl=zh-cn)。文档中写的比较详细。

为了实现多点触控，我们需要跟踪指针，文档中说明如下：

您可以使用指针的索引和 ID 来跟踪 `MotionEvent` 中各个指针的位置。

- **索引**：`MotionEvent` 将指针信息存储在数组中。指针的索引就是指针在此数组中的位置。大多数 `MotionEvent` 方法都将触控点索引（而非指针 ID）作为参数。
- **ID**：每个指针还具有一个 ID 映射，该映射在轻触事件之间保持不变，让您能够在整个手势中跟踪单个指针。

各个指针在动作事件中以不确定的顺序出现。因此，一个事件中的指针索引可能会发生变化，但只要指针保持活跃状态，其指针 ID 就一定保持不变。您可以使用 `getPointerId()` 方法获取指针的 ID，以便在手势中的所有后续动作事件中跟踪指针。然后，对于连续动作事件，请使用 `findPointerIndex()` 方法获取给定触控点 ID 在相应动作事件中的触控点索引。

如需支持多个触摸指针，您可以在各个 `ACTION_POINTER_DOWN` 和 `ACTION_DOWN` 事件时间利用其 ID 缓存所有活动指针。在 `ACTION_POINTER_UP` 和 `ACTION_UP` 事件处从缓存中移除指针。您可能会发现这些缓存的 ID 有助于正确处理其他操作事件。例如，在处理 `ACTION_MOVE` 事件时，请查找每个缓存的活动指针 ID 的索引，使用 `getX()` 和 `getY()` 函数检索指针的坐标，然后将这些坐标与缓存的坐标进行比较，以了解哪些指针移动了。

只能将 `getActionIndex()` 函数用于 `ACTION_POINTER_UP` 和 `ACTION_POINTER_DOWN` 事件。请勿将此函数与 `ACTION_MOVE` 事件一起使用，因为此函数始终返回 `0`。

> 在 native 中只能通过 `AMotionEvent_getPointerId(event, index)` 来使用 index 获取 id。index 是通过 `(action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK) >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT` 的 方式来获取，所以我们得自己建立映射关系。



之前我都是使用的 index 来查看指针信息，接下来我们使用 id 查看。

```c++
int32_t pointer_count = AMotionEvent_getPointerCount(event);
int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK)
        >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
int32_t pointerId = AMotionEvent_getPointerId(event, pointerIndex);
std::string actions[5] = {
        "ACTION_DOWN",
        "ACTION_MOVE",
        "ACTION_UP",
        "POINTER_DOWN",
        "POINTER_UP"
};
std::string curAction;
if (actionMasked == AMOTION_EVENT_ACTION_DOWN) {
    curAction = actions[0];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[1];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[2];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[3];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[4];
} else if (actionMasked == AMOTION_EVENT_ACTION_MOVE) {
    curAction = actions[5];
}
if (!(actionMasked == AMOTION_EVENT_ACTION_MOVE)) {
    LOGI("pointer_count = %d, %s, pointerIndex = %d, pointerId = %d", pointer_count, curAction.c_str(), pointerIndex, pointerId);
}
```

经过测试，发现对于一直在屏幕上的指针，其 pointerID 不会变，但是其 index 会变。所有指针的 index 都不一样，且始终在 0 ~ pointer_count-1 之间。所以可以用如下循环来获取单词 event action 中所有的 index 和对应的 id。

```c++
for (int32_t i = 0; i < pointer_count; ++i) {
    LOGI("index %d -> PointerID %d", i, AMotionEvent_getPointerId(event, i));
}
```

所以可以使用缓存 pointerID 的方式来重构 GPT 给的函数，使其表达更精炼，鲁棒性更强。

经过与 GPT 的一番激烈讨论后，得到了以下的鲁棒性比较好的函数：

```c++
#include <android_native_app_glue.h>
#include <android/log.h>
#include <map>

#define LOGI(...) ((void)__android_log_print(ANDROID_LOG_INFO, "native-activity",__VA_ARGS__))
#define LOGW(...) ((void)__android_log_print(ANDROID_LOG_WARN, "native-activity",__VA_ARGS__))

struct PointerData {
    int32_t index;
    float current_x;
    float current_y;
};

std::map<int32_t, PointerData> pointer_map;
bool needUpdatePointerMap = false;

void single_finger_drag(float dx, float dy) {
    LOGI("Single-finger drag: dx = %f, dy = %f", dx, dy);
    // 在这里添加单指拖动时要执行的代码
}

void two_finger_drag(float dx, float dy) {
    LOGI("Two-finger drag: dx = %f, dy = %f", dx, dy);
    // 在这里添加双指拖动时要执行的代码
}

void handle_drag(AInputEvent* event) {
    if (pointer_map.size() < 1) {
        LOGW("Pointer map size is less than 1");
        return;
    }

    auto it1 = pointer_map.begin();
    float x1 = AMotionEvent_getX(event, it1->second.index);
    float y1 = AMotionEvent_getY(event, it1->second.index);
    float dx1 = x1 - it1->second.current_x;
    float dy1 = y1 - it1->second.current_y;

    if (pointer_map.size() == 1) {
        single_finger_drag(dx1, dy1);
        it1->second.current_x = x1;
        it1->second.current_y = y1;
    } else if (pointer_map.size() == 2) {
        auto it2 = std::next(it1);
        if (it2 == pointer_map.end()) {
            LOGW("Second pointer not found in pointer map");
            return;
        }
        float x2 = AMotionEvent_getX(event, it2->second.index);
        float y2 = AMotionEvent_getY(event, it2->second.index);
        float dx2 = x2 - it2->second.current_x;
        float dy2 = y2 - it2->second.current_y;
        float avg_dx = (dx1 + dx2) / 2.0f;
        float avg_dy = (dy1 + dy2) / 2.0f;
        two_finger_drag(avg_dx, avg_dy);
        it1->second.current_x = x1;
        it1->second.current_y = y1;
        it2->second.current_x = x2;
        it2->second.current_y = y2;
    }
}

void update_pointer_map(AInputEvent* event) {
    pointer_map.clear();
    int32_t pointer_count = AMotionEvent_getPointerCount(event);
    for (int32_t i = 0; i < pointer_count; ++i) {
        int32_t pointerId = AMotionEvent_getPointerId(event, i);
        PointerData data = {i, AMotionEvent_getX(event, i), AMotionEvent_getY(event, i)};
        pointer_map[pointerId] = data;
    }
}

void handle_input(struct android_app* app, AInputEvent* event) {
    if (AInputEvent_getType(event) == AINPUT_EVENT_TYPE_MOTION) {
        int32_t action = AMotionEvent_getAction(event);
        int32_t actionMasked = action & AMOTION_EVENT_ACTION_MASK;
        int32_t pointer_count = AMotionEvent_getPointerCount(event);

        switch (actionMasked) {
            case AMOTION_EVENT_ACTION_DOWN: {
                needUpdatePointerMap = true;
                update_pointer_map(event);
                break;
            }
            case AMOTION_EVENT_ACTION_POINTER_DOWN: {
                needUpdatePointerMap = true;
                int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK) >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
                int32_t pointerId = AMotionEvent_getPointerId(event, pointerIndex);
                PointerData data = {pointerIndex, AMotionEvent_getX(event, pointerIndex), AMotionEvent_getY(event, pointerIndex)};
                pointer_map[pointerId] = data;
                break;
            }
            case AMOTION_EVENT_ACTION_MOVE: {
                if (needUpdatePointerMap) {
                    update_pointer_map(event);
                    needUpdatePointerMap = false;
                }
                handle_drag(event);
                break;
            }
            case AMOTION_EVENT_ACTION_POINTER_UP: {
                needUpdatePointerMap = true;
                int32_t pointerIndex = (action & AMOTION_EVENT_ACTION_POINTER_INDEX_MASK) >> AMOTION_EVENT_ACTION_POINTER_INDEX_SHIFT;
                int32_t pointerId = AMotionEvent_getPointerId(event, pointerIndex);
                pointer_map.erase(pointerId);
                break;
            }
            case AMOTION_EVENT_ACTION_UP: {
                needUpdatePointerMap = true;
                pointer_map.clear();
                break;
            }
        }
    }
}

void android_main(struct android_app* state) {
    app_dummy();

    state->onInputEvent = handle_input;

    while (1) {
        int events;
        struct android_poll_source* source;

        while (ALooper_pollAll(0, NULL, &events, (void**)&source) >= 0) {
            if (source != NULL) {
                source->process(state, source);
            }

            if (state->destroyRequested != 0) {
                return;
            }
        }
    }
}
```

