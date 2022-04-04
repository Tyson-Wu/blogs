---
title: Tiling Noise
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
[Tiling Noise](https://www.ronja-tutorials.com/post/029-tiling-noise/)

前面我们实现的噪声图都是可以沿着某一方向无线延申，并且没有明显重复的迹象。但是有时候我们希望我们的噪声图在超过一定范围后重复显示，特别是当我们希望将噪声图烘焙下来的时候，用于存储的图片不可能无限大。本文我们将会介绍如何实现噪声周期性重复出现，以及如何使用uv坐标来制造噪声。

本文将使用柏林噪声和维诺噪声实现的叠加类噪声，来阐述这种周期性噪声的原理。当然这种周期性噪声可以基于任何噪声，以及适用于任何着色器。














希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
