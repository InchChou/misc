最近在使用 Android NDK 编译 C++ 项目的时候发现了关于左值右值的问题，报错如下：

```bash
note: candidate function not viable: expects an l-value for 4th argument
```

这里报错的意思是函数期望第四个参数是右值。报错涉及到了 C++ 的参数引用与左值右值。

需要调用的函数有如下形式：

```c++
void function(..., ..., ..., Object& obj);
```

其中第四个参数是引用，而我在调用这个函数的时候是如下形式：

```c++
Object otherFunc()
{
    return Object(...);
}

int main()
{
    function(..., ..., ..., otherFunc());
}
```

这里 `ohterFunc()` 返回了一个局部变量，离开作用域会消亡，可以看作是一个右值：不可以成为其他引用的初始值，不能够作为左值使用。

即

```c++
Object& a = otherFunc(); // 错误，非常量引用的初始值必须为左值
```

所以在我这个 case 中会报编译错误。



在 visual studio 2019 和 2022 中不报这个错误，应该是 msvc 编译器的问题。