---
title: Perlin Noise
date: 2021-07-09 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Perlin Noise](https://www.ronja-tutorials.com/post/026-perlin-noise/)

## Perlin Noise

泊林噪声也是非常常用的一种噪声。和上一篇值类噪声非常相似，泊林噪声也是基于噪声色块，是“梯度噪声”实现方式的一种，所以容易产生重复、光滑的效果。它们之间的区别在于，值类噪声是基于噪声色块的插值，而泊林噪声是基于噪声变化的趋势。因为噪声部分内容复杂，建议你先从噪点、和值类噪声开始学习。
![](https://www.ronja-tutorials.com/assets/images/posts/026/Result.gif)

## Gradient Noise in one Dimension

泊林噪声是多维梯度噪声中的一种。因为一维梯度噪声比较简单，所以我们先从它入手。

首先我们实现一维噪声生成着色器，并且把所有相关代码封装到`noise`函数中，方便阅读。
```c++
float gradientNoise(float value){
    float previousCellNoise = rand1dTo1d(floor(value));
    float nextCellNoise = rand1dTo1d(ceil(value));
    float interpolator = frac(value);
    interpolator = easeInOut(interpolator);
    return lerp(previousCellNoise, nextCellNoise, interpolator);
}

void surf (Input i, inout SurfaceOutputStandard o) {
    float value = i.worldPos.x / _CellSize;
    float noise = perlinNoise(value);
    
    float dist = abs(noise - i.worldPos.y);
    float pixelHeight = fwidth(i.worldPos.y);
    float lineIntensity = smoothstep(2*pixelHeight, pixelHeight, dist);
    o.Albedo = lerp(1, 0, lineIntensity);
}
```

正如开篇提到的，泊林噪声并不是基于值插值，而是基于方向插值。因此我们首先要计算梯度。梯度方向可上可下，因此我们需要将噪声值从[0,1]区间变换到[-1,1]区间。

在得到梯度值后，我们就知道了当前点所在等高线的切线方向。因为线的方程是可以写成`base + inclination * variable`，这里`inclination`就是我们的梯度，而`base`是线的偏移，假设为0，输入值的余数部分`fraction`作为线的变量。那么线的方程可以简单的表达为`inclination * fractional`。
```c++
float gradientNoise(float value){
    float fraction = frac(value);

    float previousCellInclination = rand1dTo1d(floor(value)) * 2 - 1;
    float previousCellLinePoint = previousCellInclination * fraction;
    
    return previousCellLinePoint;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/Inclinations.png)

