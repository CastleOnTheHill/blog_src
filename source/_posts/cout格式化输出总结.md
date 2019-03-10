---
title: cout格式化输出总结
date: 2019-03-10 20:20:44
tags:
- cpp
- cout
---
(带补充完整)
## 设置输出最小宽度
`setw(n)`对应的只是宽度的最小值，如果要输出的长度本身就超过n则不起作用。当宽度小于最小值的时候，`left`是添加填充字符到右，`right`是添加填充字符到左，而`internal`是添加填充字符到内部选定点。 `left` 与 `right` 应用到任何输出，而 `internal` 应用到整数、浮点和货币输出。

```
//来源：https://zh.cppreference.com/w/cpp/io/manip/left
//Left fill:
-1.23*******
0x2a********
USD *1.23***
 
//Internal fill:
-*******1.23
0x********2a
USD ****1.23
 
//Right fill:
*******-1.23
********0x2a
***USD *1.23
```
## 设置浮点数精度
`setprecision(n)`单独使用，精度n指的是**整个浮点数的位数**，而不是小数的位数，如果浮点数的整数部分位数大于n，那么会采用科学计数法输出，保证n位。

`setprecision(n)`与`fixed`(用定点记法生成浮点类型)同时使用，输出浮点数的小数位数就是n了（为什么会这样还不清楚）。[定点数的解释](https://baike.baidu.com/item/%E5%AE%9A%E7%82%B9%E6%95%B0/11030127?fr=aladdin)

```cpp
double x = 131235.1415, y = 3.14159;
cout << setprecision(4);
//整个浮点数保留4位
cout << x << endl; //1.312e+005
cout << y << endl; //3.142

cout << fixed;
// 小数部分保留4位
cout << x << endl;//131235.1415
cout << y << endl;//3.1416


```