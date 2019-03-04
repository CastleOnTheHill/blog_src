---
title: 转换函数与explicit
date: 2019-03-05 00:00:37
tags:
- cpp
---
#### 转换函数
C++中存在转换一种特殊的函数--**转换函数**,它没有返回值，作用是将我们的对象转换成其他任意的类型，这个函数常常是由编译器自动调用的。

```cpp
class Fraction {
public:
    Fraction(int num, int den = 1)
    :a(num), b(den) {}
    Fraction operator +(const Fraction& f) {
        return Fraction(...)
    }
    // 转换函数
    operator double() const {
        return (double)a / b;
    }
private:
    int a;
    int b;
    ...
}

...
    // main 中
    Fraction f(3, 5);
    double temp1 = 0.5 + f; // 在这里，编译器会自动调用转换函数将f转为double型
```

#### non-explicit构造函数

non-explicit构造函数是能被编译器隐式调用的构造函数，用于在需要的时候自动将某些数据转变成对象，当non-explicit构造函数只需要填一个参数的时候，就能够起到和转换函数类似的效果。

但是这样的隐式调用可能会带来意想不到的麻烦，看下面的例子：
```cpp
// 承接上面例子
    double temp2 = f + 0.5;
/*
在这里编译器会报歧义，因为此处有两种两种解释方法都行得通
1. 和上面的例子一样，直接调用转换函数将f转换为double，相加之后赋值给temp2。
2.  （1）先隐式调用构造函数Fraction(0.5)，将0.5转为Fraction对象
    （2）再调用Fraction重载的+号，计算两个分数之和
    （3）最后再调用转换函数将返回的Fraction对象转为double类型赋值给temp2。
*/
```
此时就需要`explicit`关键字了,在Fraction的构造函数上加上这个关键字`explicit Fraction(int num, int den = 1)`，编译器就不会隐式调用构造函数，就不会产生歧义。

#### 参考
> 侯捷 c++程序设计