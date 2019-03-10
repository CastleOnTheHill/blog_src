---
title: cpp-template总结
date: 2019-03-06 21:29:35
tags:
- cpp
- template
---

## 类模板
```cpp
// 声明
template<typename T>
class A {
public:
	T value;
	...
};

//使用
A<int> object1;
A<double> object2;
```
在实现方面，编译器通过传进来的参数，生成两份class A的代码。

## 函数模板

```cpp
// 声明
template<typename T>
const T& min(T arg) {const T&a, const T& b} {
    return b < a ? b : a;
}

// 使用
cout << min(2, 3);
cout << min(2.4, 3.5);
```
使用函数模板的时候，不需要先声明传递进去的类型，编译器会进行实参推导。


## 成员模板
```cpp
template <class T1, class T2>
struct some_struct {
		
		T1 a;
		T2 b;
		some_struct(T1 arg1, T2 arg2)
		    :a(arg1), b(arg2) {}
		template <class U1, class U2>
		some_struct(const some_struct<U1, U2>& p)
			: a(p.a), b(p.b) {}
}
```
模板里面的成员本身又是一个模板，这里成员模板使得子类能够作为参数去构建父类，如：

```cpp
class A {
    ...
}

class B :public A {
    ...
}
B b1;
B b2;
some_struct<B, B> temp1(b1, b2);
some_struct<A, A> temp2(temp1); 
// 此时T1，T2是A;U1，U2是B 
//如果不写成员模板就无法实现这样的操作
```

## 模板特化

特化是泛化的反面，是对于模板某些**独特的类型**做特殊的设计。特化需要整个特化，不能只特化原来模板的一部分。

```cpp
template <class T>
struct hash {
    ...
}

template<>
struct hash<char> {
    ...
}
```

## 模板偏特化
分两种情况，一种是参数个数上的偏特化，指的是模板的参数只特化前面一部分（不能有间隔，只特化第1,3,5参数是错误的）。

例如在标准库中（旧），当vector的第一个模板参数是bool的时候，如果大量的bool值都用原本的一个byte来存储有些不经济，于是对于bool这种情况单独处理。
```cpp
template <typename T, typename Alloc=...>
class vector {
	...
}

template<truename Alloc=...>
class vector<bool, Alloc> {
	...
}

```

第二种是范围上的偏特化。

```cpp
template <typename T>
class C {
	...
}

// T的范围由原来的任意类型，特化到只能是指针，范围缩小
template <typename T>
class C<T*> {
	...
}
```


## 参考
> 侯捷 c++程序设计