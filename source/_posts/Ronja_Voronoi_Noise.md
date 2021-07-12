---
title: Voronoi Noise
date: 2021-07-12 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Voronoi Noise](https://www.ronja-tutorials.com/post/028-voronoi-noise/)

## Summary

还有一种噪声叫做维诺噪声。维诺噪声需要一堆随机点，然后基于这些相互最近的点之间构成的图案来生成噪声。维诺噪声和之前的噪声类似，也是基于噪声色块，只不过值类噪声对应色块的随机值表示的是灰度，而泊林噪声对应色块的随机值表示的是灰度变化趋势，维诺噪声对应色块的随机值表示的是色块中的某个随机位置。基于色块的计算相对简单，同时可重复，适用于并行计算。在阅读本文之前，建议你先对[着色器基础](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)有所了解，并且知道如何[在着色器中生成随机值](https://tyson-wu.github.io/blogs/2021/07/08/Ronja_White_Noise/)。
![](https://www.ronja-tutorials.com/assets/images/posts/028/Result.gif)

## Get Cell Values

