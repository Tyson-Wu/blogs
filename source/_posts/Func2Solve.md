---
title: 二次函数解析
date: 2021-06-19 15:01:00
categories:
- [Math, Function]
tags:
- Math
- Function
---

## 问题
如下如所示，已知A、B两点坐标，以及二次函数顶点的y坐标，求解二次函数的方程式：
![](/blogs/images/src/func.jpg)

## 思路
首先，假设二次函数方程为：
```
a * x * x + b * x + c = y   ------------- 1 
```
同时假设，A、B、P三个点的坐标分别为`(x1, y1)`、`(x2, y2)`、`(x3, y3)`，其中`x3`未知。
因为A、B、P三个点都在函数曲线上，所以三个点都满足方程：
```
a * x1 * x1 + b * x1 + c = y1   ------------- 2
a * x2 * x2 + b * x2 + c = y2   ------------- 3
a * x3 * x3 + b * x3 + c = y3   ------------- 4
```
将上面的式子2减去式3得到`式子5`
```
a * (x1 * x1 - x2 * x2)+ b * (x1 – x2) = y1 – y2   ------------- 5
```
`式子5`中消去参数c，只有a、b两个参数。将b用a表示，得：
```
b = ((y1 – y2) – a * (x1 * x1 – x2 * x2))/(x1 – x2) 
 = (y1 – y2)/(x1 – x2) – a * (x1 * x1 – x2 * x2)/(x1 – x2) ------------- 6
```
又因为:
```
x1 * x1 – x2 * x2 = (x1 + x2) * (x1 – x2) ------------- 7
```
所以：
```
b = (y1 – y2)/(x1 – x2) – a * (x1 + x2) ------------- 8
```
令：
```
m = (y1 – y2) / (x1 – x2)
n = x1 + x2
```
所以：
```
b = m – a * n------------- 9
```
因为二次函数的顶点坐标为(-b/(2a), (4ac-b*b)/(4a))，这一点可以查百度。因此：
```
x3 = -b/(2a) ------------- 10
y3 = (4ac – b*b)/(4a)
```
所以：
```
c = y3 + b * b /(4 a)------------- 11
```

=================================分段线=======================
将`式9、10`代入`式2`：
```
y1 – y3 = a * x1 * x1 + (m – a * n) + (m –a * n) *(m –a * n)/(4a)
```
化简得：
```
(4 * x1 * x1 – 4 * n * x1 + n * n) * a * a 
   + (4 * m * x1 – 2 * m * n – 4 * (y1 – y3)) * a
     + m * m = 0--------------12
```
`式子12`是关于a的二次方程
假设：
```
A1 = 4 * x1 * x1 – 4 * n * x1 + n * n
B1 = 4 * m * x1 – 2 * m * n – 4 * (y1 – y3)
C1 = m * m
```
那么`式12`可以写成：
```
 A1 * a * a + B1 * b + C1 = 0--------------13
 ```
二次方程的有两个根：
```
a1 = -(B1 + sqrt(4A1C1 – B1 * B1))/(2* A1)
a2 = -(B1 - sqrt(4A1C1 – B1 * B1))/(2* A1)
```
这里可以取a1、a2中的任意一个作为a的值
根据式8可以计算b
根据式11可以计算c
根据式10可以计算x3
从上面分段线开始，是基于`式2`计算所有的参数，也就是基于点`(x1, y1)`，我们也可以基于`式3`进行计算，也就是基于点`(x2,y2)`，但是计算出的结果都是一致的

## 实现源码
```
void Start()
    {
        float x1 = 1.3245f;
        float y1 = 0.342f;
        float x2 = 11.24f;
        float y2 = 6.324f;
        float y3 = -8.213f;
        if(Solve(x1, y1, x2, y2, y3, out var x3, out var a, out var b, out var c))
        {
            Debug.LogError(string.Format("{0} == {1}", y1, func(a, b, c, x1)));
            Debug.LogError(string.Format("{0} == {1}", y2, func(a, b, c, x2)));
            Debug.LogError(string.Format("{0} == {1}", y3, func(a, b, c, x3)));
        }
        else
        {
            Debug.LogError("error");
        }
    }
    float func(float a, float b, float c, float x)
    {
        return a * x * x + b * x + c;
    }

    // a * x * x + b * x + c = y
    bool Solve(float x1, float y1, float x2, float y2, float y3, out float x3, out float a, out float b, out float c)
    {
        float eps = 0.00001f;
        bool rst = true;
        float m, n;
        m = (y1 - y2) / (x1 - x2);
        n = x1 + x2;

        float A1, B1, C1;
        A1 = 4 * x1 * x1 - 4 * n * x1 + n * n;
        B1 = 4 * m * x1 - 2 * m * n - 4 * (y1 - y3);
        C1 = m * m;

        float A2, B2, C2;
        A2 = 4 * x2 * x2 - 4 * n * x2 + n * n;
        B2 = 4 * m * x2 - 2 * m * n - 4 * (y2 - y3);
        C2 = m * m;

        float a1_1, a1_2;
        a1_1 = -(B1 + Mathf.Sqrt(B1 * B1 - 4 * A1 * C1)) / (2 * A1);
        a1_2 = -(B1 - Mathf.Sqrt(B1 * B1 - 4 * A1 * C1)) / (2 * A1);
        float a2_1, a2_2;
        a2_1 = -(B2 + Mathf.Sqrt(B2 * B2 - 4 * A2 * C2)) / (2 * A2);
        a2_2 = -(B2 - Mathf.Sqrt(B2 * B2 - 4 * A2 * C2)) / (2 * A2);

        if (Mathf.Abs(a1_1 - a2_1)< eps)
        {
            a = a1_1;
        }
        else if (Mathf.Abs(a1_2 - a2_2) < eps)
        {
            a = a1_2;
        }
        else
        {
            a = 1;
            rst = false;
        }
        b = m - a * n;
        c = y3 + b * b / (4 * a);
        x3 = - b / (2 * a);
        return rst;
    }
```