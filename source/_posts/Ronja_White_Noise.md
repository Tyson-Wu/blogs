---
title: White Noise
date: 2021-07-08 18:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[White Noise](https://www.ronja-tutorials.com/post/024-white-noise/)

## Summary

在很多效果中需要用到随机数来生成纹理图案、又或者其他东西。下面我们以白色噪声图为例，来展示随机数用途。后面我们还会介绍其他使用随机数生成的具有一定组织结构的图案，例如泊林噪声图、和`vornoi`噪声图。本文是采用表面着色器来实现的，所以建议你先阅读我关于[表面着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)的介绍。
![](https://www.ronja-tutorials.com/assets/images/posts/024/Result.png)

## Scalar noise from 3d Input

在着色器中，我们很难将上一帧画面的数据保存到下一帧，因此我们的随机数必须依赖于着色器中可访问的参数，这样无论什么时候我们都可以得到固定的随机值。这里我们使用世界坐标来生成随机值。当然如果你想让你的噪声图动起来，可以引入时间变量。

因此我们需要在表面着色器的输入结构中加入世界坐标。另外因为我们打算通过随机数生成纹理图案，所以我们不需要纹理变量，相应的UV坐标也可以删除了。
```c++
struct Input {
    float3 worldPos;
};
```

接下来我们将实现随机噪声值生成函数，这样我们可以通过该函数很方便的制造随机数。首先我们的函数接收三维坐标参数，然后返回一个0-1之间的小数。将向量转换为标量最简单的方法就是点乘，但是点乘的结果可能非常大，所以我们使用`frac`函数只截取其中的小数部分。
```c++
float rand(float3 vec){
    float random = dot(vec, float3(12.9898, 78.233, 37.719));
    random = frac(random);
    return random;
}
```

如果我们在表面着色器函数中使用我们的随机函数，并以世界坐标为参数，将结果写入`Albedo`参数中，那么我们可以立马看到我们的随机值遍布在模型表面。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    o.Albedo = rand(i.worldPos);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/024/DotFrac.png)
![](https://www.ronja-tutorials.com/assets/images/posts/024/DotFracClose.png)

上面生成的噪声图有一个问题，就是看起来并不是那么随机，我们可以看到有很多条纹图案。虽然这个”随机函数“是我随便写的，但是它执行很快，也能满足我们当前一些简单的随机需求。我们再将上面的伪随机值乘以一个非常大的值，然后截取结果的小数部分，这样可以产生非常细的条纹，几乎观察不到。
```c++
float rand(float3 vec){
    float random = dot(vec, float3(12.9898, 78.233, 37.719));
    random = frac(random * 143758.5453);
    return random;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/024/SimpleRandom.png)

但是又有一个问题，就是乘以一个非常大的数，其结果很容易超出浮点数表示范围，例如我们的模型离世界原点非常远。
![](https://www.ronja-tutorials.com/assets/images/posts/024/FloatRange.png)

为了修复这个问题，我们可以在乘积之前，将点乘结果限定在非常小的范围，这里我使用三角函数。因为三角函数是在特殊的计算单元中执行，所以其性能消耗只比加减乘除高点。

```c++
//基于向量计算随机值
float rand(float3 value){
    //限制向量大小
    float3 smallValue = sin(value);
    //计算随机值
    float random = dot(smallValue, float3(12.9898, 78.233, 37.719));
    //防止超出范围
    random = frac(sin(random) * 143758.5453);
    return random;
}
```

## Different Input and Output

为了产生多维随机向量，我们可以沿着不同方向生成随机数，然后将结果合并成向量。但是不同方向的随机参数必须不同，这样不同方向的随机值才能不同。最简单的方法是将上面的固定向量改成变量，然后不同方向的随机值需要传入不同的向量。我们可以将上面的固定向量作为我们这个向量变量的默认参数，这样还可以以上面的方式调用。因为现在有一维、和三维随机数生成函数，所以我们需要给他们分别命名。
```c++
//生成一维随机数
float rand3dTo1d(float3 value, float3 dotDir = float3(12.9898, 78.233, 37.719)){
    //限制向量大小
    float3 smallValue = sin(value);
    //计算随机值
    float random = dot(smallValue, dotDir);
    //防止超出范围
    random = frac(sin(random) * 143758.5453);
    return random;
}
```

要生成三维随机数，我们可以将上面的方法调用三次。每次得到向量的一个维度值，每个维度使用不同的方向向量。这样我们可以得到一个彩色的随机噪声图。之所以将上面的方法执行三遍，而不是另外写一个直接生成三维向量的随机方法，是因为我们想让生成的随机向量的三个维度的值相互独立。
```c++
//生成三维随机向量
float3 rand3dTo3d(float3 value){
    return float3(
        rand3dTo1d(value, float3(12.989, 78.233, 37.719)),
        rand3dTo1d(value, float3(39.346, 11.135, 83.155)),
        rand3dTo1d(value, float3(73.156, 52.235, 09.151))
    );
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    o.Albedo = rand3dTo3d(i.worldPos);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/024/ColorfulWhiteNoise.png)

上面是使用三维向量来生成随机数，我们也可以使用二维向量，只要把前面的三维向量改成二维向量就行。还可以使用标量，不过原先的点乘操作就用不上了。我们可以一次性将这些函数都实现，然后放在一个`include`文件中，这样以后都不用再重写这些方法了。
```c++
//一维随机数

//使用三维向量计算一维随机数
float rand3dTo1d(float3 value, float3 dotDir = float3(12.9898, 78.233, 37.719)){
    //限制向量大小
    float3 smallValue = sin(value);
    //计算随机值
    float random = dot(smallValue, dotDir);
    //防止超出范围
    random = frac(sin(random) * 143758.5453);
    return random;
}

float rand2dTo1d(float2 value, float2 dotDir = float2(12.9898, 78.233)){
    float2 smallValue = sin(value);
    float random = dot(smallValue, dotDir);
    random = frac(sin(random) * 143758.5453);
    return random;
}

float rand1dTo1d(float3 value, float mutator = 0.546){
	float random = frac(sin(value + mutator) * 143758.5453);
	return random;
}

//二维随机数

float2 rand3dTo2d(float3 value){
    return float2(
        rand3dTo1d(value, float3(12.989, 78.233, 37.719)),
        rand3dTo1d(value, float3(39.346, 11.135, 83.155))
    );
}

float2 rand2dTo2d(float2 value){
    return float2(
        rand2dTo1d(value, float2(12.989, 78.233)),
        rand2dTo1d(value, float2(39.346, 11.135))
    );
}

float2 rand1dTo2d(float value){
    return float2(
        rand2dTo1d(value, 3.9812),
        rand2dTo1d(value, 7.1536)
    );
}

//三维随机数

float3 rand3dTo3d(float3 value){
    return float3(
        rand3dTo1d(value, float3(12.989, 78.233, 37.719)),
        rand3dTo1d(value, float3(39.346, 11.135, 83.155)),
        rand3dTo1d(value, float3(73.156, 52.235, 09.151))
    );
}

float3 rand2dTo3d(float2 value){
    return float3(
        rand2dTo1d(value, float2(12.989, 78.233)),
        rand2dTo1d(value, float2(39.346, 11.135)),
        rand2dTo1d(value, float2(73.156, 52.235))
    );
}

float3 rand1dTo3d(float value){
    return float3(
        rand1dTo1d(value, 3.9812),
        rand1dTo1d(value, 7.1536),
        rand1dTo1d(value, 5.7241)
    );
}
```

然后我们将上面的随机生成的函数都放到一个叫`WhiteNoise.cginc`的文件中。并且在我们的着色器中引用它。
```c++
#include "WhiteNoise.cginc"
```

为了防止我们多次误引用同一个文件，我们可以使用宏命令来规避这个问题。
```c++
#ifndef WHITE_NOISE
#define WHITE_NOISE

//我们的包含库内容

#endif
```

## Cells

现在我们实现了通过世界坐标来生成随机颜色，这些颜色块非常小，当我们移动物体时，颜色也会快速变化。如果我们想让颜色块变大，我们可以将空间进行划分，所有处在同一块中的点使用同一个随机值。我们这里可以使用取整的方法，这样所有整数之间的小数对应的点都将使用同一随机值。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = floor(i.worldPos);
    o.Albedo = rand3dTo3d(value);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/024/1Cells.png)

现在我们得到泾渭分明的色块，然后我们可以修改色块的大小。
```c++
Properties {
    _CellSize ("Cell Size", Vector) = (1,1,1,0)
}
```
```c++
float3 _CellSize;
```

我们这将世界坐标除以色块尺寸。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = floor(i.worldPos / _CellSize);
    o.Albedo = rand3dTo3d(value);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/024/Cells.png)

## Source

```c++
Shader "Tutorial/024_white_noise/random" {
	Properties {
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "WhiteNoise.cginc"

		struct Input {
			float3 worldPos;
		};

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos;
			o.Albedo = rand3dTo3d(value);
		}
		ENDCG
	}
	FallBack "Standard"
}
```
```c++
Shader "Tutorial/024_white_noise/cells" {
	Properties {
		_CellSize ("Cell Size", Vector) = (1,1,1,0)
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "WhiteNoise.cginc"

		float3 _CellSize;

		struct Input {
			float3 worldPos;
		};

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = floor(i.worldPos / _CellSize);
			o.Albedo = rand3dTo3d(value);
		}
		ENDCG
	}
	FallBack "Standard"
}
```
```c++
#ifndef WHITE_NOISE
#define WHITE_NOISE

//to 1d functions

//get a scalar random value from a 3d value
float rand3dTo1d(float3 value, float3 dotDir = float3(12.9898, 78.233, 37.719)){
	//make value smaller to avoid artefacts
	float3 smallValue = sin(value);
	//get scalar value from 3d vector
	float random = dot(smallValue, dotDir);
	//make value more random by making it bigger and then taking the factional part
	random = frac(sin(random) * 143758.5453);
	return random;
}

float rand2dTo1d(float2 value, float2 dotDir = float2(12.9898, 78.233)){
	float2 smallValue = sin(value);
	float random = dot(smallValue, dotDir);
	random = frac(sin(random) * 143758.5453);
	return random;
}

float rand1dTo1d(float3 value, float mutator = 0.546){
	float random = frac(sin(value + mutator) * 143758.5453);
	return random;
}

//to 2d functions

float2 rand3dTo2d(float3 value){
	return float2(
		rand3dTo1d(value, float3(12.989, 78.233, 37.719)),
		rand3dTo1d(value, float3(39.346, 11.135, 83.155))
	);
}

float2 rand2dTo2d(float2 value){
	return float2(
		rand2dTo1d(value, float2(12.989, 78.233)),
		rand2dTo1d(value, float2(39.346, 11.135))
	);
}

float2 rand1dTo2d(float value){
	return float2(
		rand2dTo1d(value, 3.9812),
		rand2dTo1d(value, 7.1536)
	);
}

//to 3d functions

float3 rand3dTo3d(float3 value){
	return float3(
		rand3dTo1d(value, float3(12.989, 78.233, 37.719)),
		rand3dTo1d(value, float3(39.346, 11.135, 83.155)),
		rand3dTo1d(value, float3(73.156, 52.235, 09.151))
	);
}

float3 rand2dTo3d(float2 value){
	return float3(
		rand2dTo1d(value, float2(12.989, 78.233)),
		rand2dTo1d(value, float2(39.346, 11.135)),
		rand2dTo1d(value, float2(73.156, 52.235))
	);
}

float3 rand1dTo3d(float value){
	return float3(
		rand1dTo1d(value, 3.9812),
		rand1dTo1d(value, 7.1536),
		rand1dTo1d(value, 5.7241)
	);
}

#endif
```

希望我的教程能够对你有所帮助。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/WhiteNoise.cginc](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/WhiteNoise.cginc)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/white_noise_random.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/white_noise_random.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/white_noise_cells.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/024_White_Noise/white_noise_cells.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
