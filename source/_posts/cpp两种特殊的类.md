---
title: cpp两种特殊的类
date: 2019-03-06 21:26:00
tags:
- cpp
- function_like_class
- point_like_class
---
## C++ point-like class

point-like class就是行为像指针的类。point-like class的成员中一定有一个真正的指针，class通过重载 `*`和`->`等指针的操作来实现pointer-like。

```cpp
//定义
template<typename T>
class shared_ptr {
public:
	shared_ptr(T* p): ptr(p) {
		...
	}
	T& operator *() {
		return *ptr;
	}
	T* operator ->() {
		return ptr;
	}
private:
	T* ptr;
};
//使用
class object {
public:
	...
	void some_method(){...}
};
object temp;
shared_ptr<object> pointer(&temp);

pointer->some_method(); 
/*
->的重载在C++中比较特殊，pointer-> 在调用了->之后，返回来的ptr仍然会使用->作用到some_method;
(*pointer).some_method();
*/
```
pointer-like class的例子：STL的迭代器，C++11的智能指针。

## function-like class

function-like class 就是类能像函数一样通过括号传入参数执行，重点就在重载括号。也被称为**仿函数**。
```cpp
template <class T>
class Identify {
public:
	const T& operator() (const T& x ){return x;}
};
```

## 参考
> 侯捷 c++程序设计
