在三维图形学中，绝大部分旋转都使用四元数来表示，四元数的好处不多赘述，现在来研究一下使用四元数如何创建一个FPS相机。

## `Camera`结构体

基本上一个 `Camera` 都可以用下面的结构体表示：

```c++
struct Camera { 
  vec3 position; // Position in world space.
  quat orientation; // Orientation in world space.
}
```

- `position`：使用一个 `vec3` 表示了相机在世界坐标中的位置坐标；
- `orientation`：使用一个四元数表示了相机在世界坐标中的朝向。

在这里假设世界坐标系为 z 轴从屏幕向外指出，x 轴朝右，y 轴朝上。

假设相机初始方向为看向 z 轴负方向，右方向是 x 轴正方向，则 `front = (0.0, 0.0, -1.0)`，`up = (0.0, 1.0, 0.0)`。

### 移动相机

移动相机相对来说比较简单，首先算出当前相机的朝向和右方向，更新 `position` 即可：

```c++
void MoveCamera()
{
    ...
    float cameraSpeed = 0.05f; // adjust accordingly
    Vec3 cameraFront(0.0, 0.0, -1.0f);
    Vec3 cameraRight(1.0, 0.0, 0.0f);
    Vec3 curCameraFront = orientation * cameraFront;
    Vec3 curCameraRight = orientation * cameraRight;

    if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
        position += curCameraFront * cameraMoveSpeed;
    if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
        position -= curCameraFront * cameraMoveSpeed;
    if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
        position -= curCameraRight * cameraMoveSpeed;
    if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
        position += curCameraRight * cameraMoveSpeed;
}
```

### 旋转视角

为了旋转视角，需要更新 `orientation` 四元数，这里需要用到四元数乘法，四元数乘法的意义即将多个旋转组合起来。

#### yaw-pitch-roll

由于创建的是 fps 相机，所以只有两个维度：向上下看，向左右看。视角不会翻滚。如果使用欧拉角来类比，则只有 yaw、pitch，没有 roll。

> yaw-pitch-roll 角是泰特布莱恩角的一种顺序（确切的说是它的 $z-y^{\prime}-x^{\prime\prime}$(intrinsic rotations)次序，见 [Euler angles § Tait–Bryan angles](https://en.wikipedia.org/wiki/Euler_angles#Tait–Bryan_angles)）。关于它的介绍见 [Yaw, pitch, and roll](https://en.wikipedia.org/wiki/Yaw,_pitch,_and_roll)。
>
> 想象着正在驾驶飞机平行于地面向北飞，则各轴如下：
>
> yaw 轴位于机身重心，指向飞机底部，垂直于机翼和机身参考线，positive yawing motion（正偏航）表示飞机的机头向右运动；
>
> pitch 轴（也称 transverse or lateral axis，横轴）位于重心指向右边，平行于机翼。
>
> roll 轴（也称 longitudinal axis，纵轴）位于重心指向前方，平行于机身参考线。围绕这个轴的运动称为 **roll**（翻滚）。正向滚动运动升高左翼并降低右翼。
>
> 将 yaw-pitch-roll 角按照泰特布莱恩角的顺序对应起来，则应该如下：
>
> 假设飞机平行于地面向北飞，世界坐标轴为：x 轴指向北方，y 轴对应东方，z 轴垂直于地面向下。飞机坐标轴为：X 轴对应 roll 轴，Y 轴对应 pitch 轴，Z 轴对应着 yaw 轴。刚开始的时候飞机坐标轴与世界坐标轴重合。$z-y^{\prime}-x^{\prime\prime}$(intrinsic rotations)次序应该是先绕 Z （与 z 轴重合）轴旋转，此时为 yaw 角；然后绕着旋转后的 Y 轴旋转，此时为 pitch 角；最后绕着旋转两次后的 X 轴旋转，此时为 roll 角。

注意上面的定义，旋转的次序与旋转轴都很重要，并且得到的 yaw-pitch-roll 角都是相对于初始状态进行一整轮旋转后的角，即如果进行第二轮旋转的话，得到的数据也应该是相对于初始状态的。

#### 使用鼠标控制方向

如 yaw-pitch-roll 一节中提到，只有两个维度：向上下看，向左右看。则需要用鼠标来控制 yaw 和 pitch。如 [摄像机 - 鼠标输入](https://learnopengl-cn.github.io/01 Getting started/09 Camera/#_7) 中所说我们需要计算鼠标距离上一帧的偏移量，然后把偏移量添加进摄像机的 yaw 和 pitch 中。

```c++
float xoffset = xpos - lastX;
float yoffset = lastY - ypos; // 注意这里是相反的，因为y坐标是从底部往顶部依次增大的
lastX = xpos;
lastY = ypos;
```

添加到全局变量中：

```c++
yaw   += xoffset;
pitch += yoffset;
```

有了这两个变量就可以计算旋转四元数了，在这里坐标系如下：

- 世界坐标系 xyz 为 x 轴为屏幕向右，y 轴为屏幕向上，z 轴为从屏幕向外指；
- 相机的坐标系 XYZ 在初始情况下与世界坐标系 xyz 重合，假设相机的初始状态是看向 Z 轴负方向，右方向是沿 X 轴方向，上方向是沿 Y 轴正方向，。

```
// Rotation around Y axis, in other words, look left and right
Quat rotationX = AngleAxis(yaw  , Vec3(0.0f, 1.0f, 0.0f));
// Rotation around X axis, in other words, look up and down
Quat rotationY = AngleAxis(pitch, Vec3(1.0f, 0.0f, 0.0f));
```

这里的 `AngleAxis()` 为使用一个旋转轴与旋转角度来计算四元数的常用函数，得到的 `rotationX` 为偏航角四元数，`rotationY` 为俯仰角四元数。

按照 yaw-pitch-roll 角的定义来说，首先应绕 Y（与 y 轴重合）轴旋转 yaw 角，然后绕着旋转后的 x 轴旋转 pitch 角：

#### 计算

##### **方案一**

在一开始的时候想的很简单，没有使用全局的 `yaw` 和 `pitch` 变量，只使用了鼠标偏移量 `xoffset` 和 `yoffset`，相对于上一帧的相机旋转四元数来改变当前帧的相机旋转四元数（后面都将相机旋转四元数称为 `cameraRotation`）。

在更新 `cameraRotation`时也只是把两个方向上的旋转四元数和相机旋转四元数乘起来：`cameraRotation = rotationX * rotationY * cameraRotation`。如下所示：

```c++
// Rotation around Y axis, in other words, look left and right
Quat rotationX = AngleAxis(xoffset, Vec3(0.0f, 1.0f, 0.0f));
// Rotation around X axis, in other words, look up and down
Quat rotationY = AngleAxis(yoffset, Vec3(1.0f, 0.0f, 0.0f));
// update cameraRotation
cameraRotation = rotationX * rotationY * cameraRotation;
```

但是很快就发现问题了，**随便转两下就发现相机翻滚（roll）了**，这种情况在 fps 相机中不应该出现，在经过长时间的搜索后，在 [rotation - How to keep my Quaternion-using FPS camera from tilting and messing up? - Game Development Stack Exchange](https://gamedev.stackexchange.com/questions/30644/how-to-keep-my-quaternion-using-fps-camera-from-tilting-and-messing-up/30669#30669) 中发现了解决方案，原来是四元数乘法顺序的问题，需要写成 `cameraRotation = rotationX * cameraRotation * rotationY`，首先乘以偏航角，然后是 `cameraRotation`，最后是俯仰角，问题解决，但是回答中并没有解释为什么要这么做。

> 更新：在 [rotation - I'm rotating an object on two axes, so why does it keep twisting around the third axis? - Game Development Stack Exchange](https://gamedev.stackexchange.com/questions/136174/im-rotating-an-object-on-two-axes-so-why-does-it-keep-twisting-around-the-thir) 中讨论了这么做的原因。

我尝试过将偏航角和俯仰角的位置换一下，变成 `cameraRotation = rotationY * cameraRotation * rotationX`，依然会出现翻滚，所以上面这个顺序是唯一能成功的。

##### 方案二

使用全局的 `yaw` 和 `pitch` 变量，然后使用这两个变量和相机的初始旋转矩阵去更新 `cameraRotation`。如下：

```c++
Quat initCameraRotation = ... //
...
// Rotation around Y axis, in other words, look left and right
Quat rotationX = AngleAxis(yaw  , Vec3(0.0f, 1.0f, 0.0f));
// Rotation around X axis, in other words, look up and down
Quat rotationY = AngleAxis(pitch, Vec3(1.0f, 0.0f, 0.0f));
// update cameraRotation
cameraRotation = rotationX * rotationY * initCameraRotation;
```

使用这种方式可以较好的解决翻滚问题，同时还能限制俯仰角和偏航角。

但是，由于我相机的初始旋转四元数是不进行任何旋转的四元数（实部为 1），所以更新 `cameraRotation` 实际上相当于 `cameraRotation = rotationX * rotationY`，这里 `initCameraRotation` 放在任何位置都没关系，有关系的是 `rotationX` 和 `rotationY` 的顺序，我试过其他顺序，只有 `rotationX` 在 `rotationY` 前面才行，不然会出现翻滚。

##### 原理与坑

这里有许多坑，因为四元数乘法不满足交换律（noncommutative），所以当顺序不一样时，四元数乘法的意义也不一样。稍微详细一点的说明可以参见 [Quaternions (paroj.github.io)](https://paroj.github.io/gltut/Positioning/Tut08 Quaternions.html) 、 [Camera-Relative Orientation (paroj.github.io)](https://paroj.github.io/gltut/Positioning/Tut08 Camera Relative Orientation.html) 、[rotation - I'm rotating an object on two axes, so why does it keep twisting around the third axis? - Game Development Stack Exchange](https://gamedev.stackexchange.com/questions/136174/im-rotating-an-object-on-two-axes-so-why-does-it-keep-twisting-around-the-thir)以及 [c++ - Quaternion-based First Person View Camera - Stack Overflow](https://stackoverflow.com/questions/49609654/quaternion-based-first-person-view-camera) 中的讨论。所以在将这些四元数乘起来的时候需要格外小心。

在我的这两个方案中，都需要先乘偏航角四元数，再成俯仰角四元数，后续分别用 `Yaw` 和 `Pitch` 代替。要注意的是，这里的偏航角和俯仰角在计算过程中都是围绕世界坐标系的 y 轴和 x 轴旋转的。为了探究它，设有两个点 P1、P2，使用 P 表示两个点中的任意一个，R 为旋转矩阵`R3 * R2 * R1 * P` 表示先将 P 进行 R1 旋转，再进行 R2 旋转，最后进行 R3 旋转，即最靠近 P 的旋转矩阵最先起作用，使用四元数乘以点同样表示将点进行旋转，且作用次序旋转矩阵类似。

`Yaw * Pitch * P ` 即相当于先将 P 绕世界坐标的 x 轴旋转，然后再绕世界坐标的 y 轴旋转；`Pitch * Yaw * P` 即相当于先将 P 绕世界坐标的 y 轴旋转，然后再绕世界坐标的 x 轴旋转。进行这两次旋转之后，P 的位置会不一样，并且 P1 到 P2 的方向 $\vec{P_{1}P{2}}$也不同。正是因为顺序的原因，所以造成了翻滚。可以使用 [Quaternions - Visualisation](https://quaternions.online/) 来可视化这些旋转（这个网站中的 P1 为原点，P2 为 (1, 0, 0)），如果标志轴依然在 xz 平面上，则无翻滚。

**先来看看方案一**

使用 `cameraRotation = Yaw * cameraRotation * Pitch` 能解决翻滚的原因。首先四元数乘法相当于组合（compose）旋转，假设相机一开始是无旋转的，即 `cameraRotation` 实部为 1，进行一次旋转、第二次旋转、第三次旋转后分别为
$$
\begin{align}
\text{cameraRotation}_{1} &= \text{Yaw}_{1} * \text{cameraRotation} * \text{Pitch}_{1} = \text{Yaw}_{1} * \text{Pitch}_{1} \\
\text{cameraRotation}_{2} &= \text{Yaw}_{2} * \text{cameraRotation}_{1} * \text{Pitch}_{2} = \text{Yaw}_{2} * \text{Yaw}_{1} * \text{Pitch}_{1} * \text{Pitch}_{2} \\
\text{cameraRotation}_{3} &= \text{Yaw}_{3} * \text{cameraRotation}_{2} * \text{Pitch}_{3} = \text{Yaw}_{3} * \text{Yaw}_{2} * \text{Yaw}_{1} * \text{Pitch}_{1}* \text{Pitch}_{2} * \text{Pitch}_{3}
\end{align}
$$
则在第三次旋转后 P 点所在位置应该为 $\text{Yaw}_{3} * \text{Yaw}_{2} * \text{Yaw}_{1} * \text{Pitch}_{1} * \text{Pitch}_{2} * \text{Pitch}_{3} * \text{P}$，即

- 第一批旋转：首先绕世界坐标系的 x 轴旋转 Pitch3，再绕世界坐标系的 x 轴旋转 Pitch2，再绕世界坐标系的 x 轴旋转 Pitch1；
- 第二批旋转：首先绕世界坐标系的 y 轴旋转 Yaw1，再绕世界坐标系的 y 轴旋转 Yaw2，再绕世界坐标系的 y 轴旋转 Yaw3。

意思是将 Pitch 旋转和 Yaw 旋转都积累在一起，效果与 $\text{Yaw} * \text{Pitch}* \text{P}$ 形式一样，其中$$\text{Yaw} = \text{Yaw}_{3} + \text{Yaw}_{2} + \text{Yaw}_{1}, \text{Pitch} = \text{Pitch}_{1} + \text{Pitch}_{2} + \text{Pitch}_{3}$$。

如果不将 Pitch 和 Yaw 放在两边，则第三次后是形如  $\text{Yaw}_{3} * \text{Pitch}_{3} * \text{Yaw}_{2} * \text{Pitch}_{2} * \text{Yaw}_{1} * \text{Pitch}_{1} * \text{P}$ 的形式，首先绕世界坐标系的 x 轴旋转，再绕世界坐标系的 y 轴旋转，然后再绕世界坐标系的 x 轴旋转，再绕世界坐标系的 y 轴旋转，此时就会发生翻滚。

并且将 Pitch 和 Yaw 放在两边时一定是先 Yaw 再 Pitch 的的形式，不然的话也会发生翻滚，这个原因与方案二中的使用固定乘法顺序的原因一样，在下面讨论。

**再来看看方案二**

方案二中每帧更新的时候，四元数乘法中使用的相机四元数始终是相机的初始四元数，即只绕世界坐标系的 x 轴旋转一次，再绕世界坐标系的 y 轴旋转一次。

至于为什么乘法顺序是  `Yaw * Pitch`，还需要理解四元数乘法的意义。四元数乘法本质上是矩阵乘法，对于矩阵乘法我们见的最多的就是 MVP 矩阵，对应着世界空间、观察空间和裁剪空间。

在四元数乘法中涉及到局部坐标(local axis)与世界坐标(world axis)，讨论可以参见 [简单实例理解Unity世界坐标和局部坐标下四元数旋转（四元数乘法）_Veropatrinica的博客-CSDN博客_unity 四元数旋转](https://blog.csdn.net/shanwenkang/article/details/103956503)。

> 在 [rotation - I'm rotating an object on two axes, so why does it keep twisting around the third axis? - Game Development Stack Exchange](https://gamedev.stackexchange.com/questions/136174/im-rotating-an-object-on-two-axes-so-why-does-it-keep-twisting-around-the-thir) 中提到：“So we can fix the first-person camera with the mantra "Pitch Locally, Yaw Globally"” 以及 “but left = more global / right = more local is a common choice)”

总的来说可以理解为这样，如果有两个四元数  $p$ 和 $q$，分别代表了旋转 $P$ 和 $Q$，则 $pq$ 表示两个旋转的合成，以应用顺序来看即先旋转 $Q$ 再旋转 $P$，但其实还可以这样理解：

**假设 $P$ 是世界坐标中的旋转，$Q$ 是局部坐标中的旋转，物体处于局部坐标中，求应用 $pq$ 旋转后物体在世界坐标的位置和姿态——$pq$ 从左往右看表示先在世界坐标中将局部坐标旋转 $P$ 得到旋转后的局部坐标系，然后再在旋转后的局部坐标对物体应用旋转 $Q$。**

依然以相机中的 yaw-pitch-roll 角举例，其实它的顺序是先绕世界坐标中的 Z 轴将整个局部坐标系进行旋转（Yaw），再绕局部坐标系中的 x 轴进行旋转（Pitch），再绕局部坐标的 z 轴进行旋转（Roll）。所以不算上 Roll 旋转的话，顺序就是 `Yaw * Pitch`。

> 如果算上 Roll 旋转的话，顺序就是 `Yaw * Pitch * Roll`。

同理，如果四元数用在骨骼动画中的话， $pq$ 中的 $q$ 表示在子节点局部坐标系中的旋转，$p$ 表示子节点局部坐标系在父节点局部坐标系中的旋转。



在一些博客中，推荐 FPS 相机使用方案二。