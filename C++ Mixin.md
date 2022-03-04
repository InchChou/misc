[C++ Mixin初探_jiang4357291的博客-CSDN博客](https://blog.csdn.net/jiang4357291/article/details/103325488)

[C++继承和组合——带你读懂接口和mixin，实现多功能自由组合_阿里云云栖号-CSDN博客](https://blog.csdn.net/yunqiinsight/article/details/80017968)

[c++ - What are Mixins (as a concept) - Stack Overflow](https://stackoverflow.com/questions/18773367/what-are-mixins-as-a-concept)

[Mixin-Based Programming in C++ | Dr Dobb's (drdobbs.com)](https://www.drdobbs.com/cpp/mixin-based-programming-in-c/184404445)

## 背景

在阅读源码时发现了下面的一种写法

```c++
template<class T>
class Base : public T
{
    ......
}
```

最终确定这是c++实现mixin的方式，也就是Template Parameters as Base Classes，那么为什么会诞生这种较复杂的语法设计呢？其存在的意义是什么？

在C++中多重继承存在引发钻石继承（菱形继承）的风险。于是C++ 引入了虚继承，但虚继承在某些场景下也是有一定弊端的，为此C++ 作者发明了[Template Parameters as Base Classes](https://books.google.com.hk/books?id=PSUNAAAAQBAJ&lpg=PA767&ots=DrtrIigY4J&dq=27.4. Template Parameters as Base Classes&hl=zh-CN&pg=PA767#v=onepage&q&f=false)，这便是C++ 中实现mixin的方式。

## 结论

mixin是一种设计思想，用于将相互独立（正交）的类的功能组合在一起，以一种安全的方式来模拟多重继承。而c++中mixin的实现通常采用Template Parameters as Base Classes的方法，既实现了多个不交叉的类的功能组合，又避免了多重继承的问题。