---
title: new和delete的重载
date: 2019-03-25 16:28:51
tags:
- cpp
- new
- delete
- 内存池
---
new和delete是一个表达式，执行的过程中会被分解。
## new和delete的分解
### 单个对象的new和delete

表达式`String* ps = new String("hello");`会被分解为
```cpp
void* mem = operator new(sizeof(String)) //在这个内部调用了malloc
ps = static_cast<String*>(mem); //类型的转换
ps->String::String("hello");  //显示地调用构造函数
```

表达式`delete ps;`会被分解为
```cpp
ps->String::~String(); //显示调用析构函数
operator delete(ps); // 释放内存，内部调用free
```

### 对象数组的new和delete

表达式`String* ps = new String[5];` 会被分解为
```
void* mem = operator new(sizeof(String) * 5 + 4); 
//new一个数组的时候会有一个int来存储数组的大小
ps = static_cast<String*>(mem);
ps->String();
(ps + 1)->String();
....//共调用五次
```

表达式`delete[] ps;` 会被分解为
```cpp
ps->String::~String(); //显示调用析构函数
(ps + 1)->String::~String();
...//共调用五次
operator delete(ps); // 释放内存，内部调用free
```

**由此，如果申请了一个对象数组(`new[]`)，却使用 delete删除，那么只会调用第一个对象的析构函数，未调用析构函数的对象之前如果申请了堆空间，就会发生内存泄露。**

## new和delete的重载

在对象的public成员函数中重载(可以是static)，可以自定义对象的new的过程，用于内存池：大致就是重载的`operator new`中不使用`malloc`而是返回一个已分配好了的内存的指针，`operator delete`中不使用`free`，而是将内存返回给内存池，整个过程不向系统索要内存返还内存，没有用户态到系统态的切换，更加快速。

```cpp
class Foo {
    public:
    void* operator new(size_t);
    void operator delete(void*, size_t); //size_t可有可无
    
    void* operator new[](size_t);
    void operator delete[](void*, size_t); //size_t可有可无    
}
```

此后`new Foo`, `delete Foo`, `new Foo[n]`, `delete[] p`就会使用自定义的操作了。

如果重载了new和delete却不想使用，可以`::new`和`::delete`调用默认的`operator new`和`operator delete`。

> 参考 C++程序设计