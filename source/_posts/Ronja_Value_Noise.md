---
title: Value Noise
date: 2021-07-08 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Value Noise](https://www.ronja-tutorials.com/post/025-value-noise/)

[上一篇](https://tyson-wu.github.io/blogs/2021/07/08/Ronja_White_Noise/)文章介绍了如何在着色器中生成随机数。这里我想对前面生成的随机噪声图进行插值，得到一个光滑、渐变的噪声图。因为我们需要事先生成好的噪声图来进行插值，所以建议你先阅读[上一篇](https://tyson-wu.github.io/blogs/2021/07/08/Ronja_White_Noise/)来实现噪声图。本篇讲的值噪声和泊林噪声有点区别，首先两者都是在上一篇生成的随机噪声的基础上进一步平滑得到的，区别在于，本篇将上一篇得到的噪声当做值来处理，而泊林噪声则将其当做方向来处理。
![](https://www.ronja-tutorials.com/assets/images/posts/025/Result.gif)

## Show a Line

首先我们来实现一个简单的一维噪声可视化显示，这个是以[上一篇](https://tyson-wu.github.io/blogs/2021/07/08/Ronja_White_Noise/)中的噪声块为起点，然后改变这个色块的尺寸变量为标量，因为我们将处理的是一维数据。然后我们只通过世界坐标的`x`值来生成一维噪声。
```c++
Properties {
    _CellSize ("Cell Size", Range(0, 1)) = 1
}
```
```c++
float _CellSize;
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float value = floor(i.worldPos.x / _CellSize);
    o.Albedo = rand1dTo1d(value);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/1dValues.png)

这样我们可以看到我们的噪声图沿着`x`方向分布。接下来我们将这个分布图改为曲线表示，这样我们可以清楚地看到噪声的变化情况。首先我们将像素点的`y`坐标减去该像素的噪声值，然后取绝对值。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float value = floor(i.worldPos.x / _CellSize);
    float noise = rand1dTo1d(value);
    float dist = abs(noise - i.worldPos.y);
    o.Albedo = dist;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/CellDistance.png)

然后我们使用这个差值，并且设定一个阈值来剔除远离噪声值得点，这样我们就得到一根细细的线，不过这个阈值不好选定，线的粗细也不好控制。这里有一个更好的方法，通过计算像素值之间的梯度值，我们可以精确的到宽度为1个像素的细线。需要用到的函数就是`fwidth`，这个函数会自动比较相邻像素之间的值，然后返回梯度值。这里我们对世界坐标点的`y`求梯度，其物理含义是像素点的单位长度。所以用像素点的单位长度来作为阈值，就可以控制线的粗细了。

```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float value = floor(i.worldPos.x / _CellSize);
    float noise = rand1dTo1d(value);
    float dist = abs(noise - i.worldPos.y);
    float pixelHeight = fwidth(i.worldPos.y);
    float lineIntensity = smoothstep(0, pixelHeight, dist);
    o.Albedo = lineIntensity;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/CellLine.png)

## Interpolate Cells in one Dimension

为了实现色块之间插值功能，首先我们需要在片段着色器中执行两次噪声采样，分别是当前色块，和上一个色块。我们可以用`floor`和`ceil`两个函数，来计算上下两个色块的值。这里我们还可以直接使用坐标的小数部分作为插值位置。
```c++
float value = i.worldPos.x / _CellSize;
float previousCellNoise = rand1dTo1d(floor(value));
float nextCellNoise = rand1dTo1d(ceil(value));
float noise = lerp(previousCellNoise, nextCellNoise, frac(value));
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/LinearLine.png)

插值后得到一根连续的曲线，不过我希望曲线更光滑一点。因此，我们可以实现简单的缓动函数，后面我们还会深入的介绍缓动函数，不过这里使用简单的几种就够了。首先我们实现名为`easeIn`的缓入函数，这里我们执行平方操作，这样插值的边界值还是0-1，但是在接近0的时候变化更缓慢。然后我们将缓入函数应用到我们的插值中。
```c++
inline float easeIn(float interpolator){
    return interpolator * interpolator;
}
```

```c++
float interpolator = frac(value);
interpolator = easeIn(interpolator);
float noise = lerp(previousCellNoise, nextCellNoise, interpolator);
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/EaseIn.png)

使用缓入函数后，我们发现色块的开头位置更加平缓。下面我们再实现一个名为`EaseOut`的缓出函数，使得色块结尾位置更加平缓。在实现缓出函数时，我们利用了前面的缓入函数，不过将插值翻了个个，这样就是1附近的值变化更缓慢。然后我们还要将结果翻转一遍，这样可以保证缓入缓出同时应用的时候，连接部位是光滑的。
```c++
float easeOut(float interpolator){
    return 1 - easeIn(1 - interpolator);
}
```

最后一步是将两者结合，实现缓入缓出的效果。这里我们还是使用线性插值函数，当插值接近0时，我们倾向于使用缓入效果，当插值接近1时，我们倾向于使用缓出效果。
```c++
float easeInOut(float interpolator){
    float easeInValue = easeIn(interpolator);
    float easeOutValue = easeOut(interpolator);
    return lerp(easeInValue, easeOutValue, interpolator);
}
```
```c++
float interpolator = frac(value);
interpolator = easeInOut(interpolator);
float noise = lerp(previousCellNoise, nextCellNoise, interpolator);
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/SmoothLine.png)

有了缓入缓出函数，我们就可以对我们的一维噪声色块进行平滑处理了。

## Interpolate Cells in two Dimensions

实现两个维度的插值，我们可以选择相邻的四个色块。然后沿着`x`轴进行平滑处理、再沿着`y`轴进行平滑处理。
![](https://www.ronja-tutorials.com/assets/images/posts/025/2dInterpolationRules.png)

现在代码量比较多，所以我们单独封装一个函数来执行者两个维度的插值逻辑。这里我们使用`rand2dTo1d`分别计算四个色块的噪声值，然后分别计算`x`、`y`轴方向的缓入缓出插值，首先沿着`x`方向计算两两之间的插值，然后将结果沿着`y`方向计算最终的插值结果。

```c++
float ValueNoise2d(float2 value){
    float upperLeftCell = rand2dTo1d(float2(floor(value.x), ceil(value.y)));
    float upperRightCell = rand2dTo1d(float2(ceil(value.x), ceil(value.y)));
    float lowerLeftCell = rand2dTo1d(float2(floor(value.x), floor(value.y)));
    float lowerRightCell = rand2dTo1d(float2(ceil(value.x), floor(value.y)));

    float interpolatorX = easeInOut(frac(value.x));
    float interpolatorY = easeInOut(frac(value.y));

    float upperCells = lerp(upperLeftCell, upperRightCell, interpolatorX);
    float lowerCells = lerp(lowerLeftCell, lowerRightCell, interpolatorX);

    float noise = lerp(lowerCells, upperCells, interpolatorY);
    return noise;
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xy / _CellSize;
    float noise = ValueNoise2d(value);

    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/Grey2dNoise.png)

## Interpolate Cells in Three Dimensions and Loops

三个维度的插值方法和两个维度的方法基本一致。首先需要选择八个相邻的色块，然后沿着`x`轴进行两两插值，其结果再沿着`y`轴进行两两插值，其结果再沿着`z`轴两两插值，得到最终的插值结果。

但是为三维插值重写上面的方法，会产生很多行代码，不利于理解和管理。所以我们使用循环语句来避免这个问题。这里每个循环迭代两次，分别计算相邻两个需要插值的量。而最内层的循环将计算`x`方向相邻色块的值，然后存入临时数组中，待执行完后，计算`x`方向的插值。这里我们在循环语句前加`[unroll]`，表示我们的循环在编译时会被展开。因为GPU执行循环语句比较慢，而展开后便不再是循环语句了。
```c++
float interpolatorX = easeInOut(frac(value.x));

int y = 0, z = 0;

float cellNoiseX[2];
[unroll]
for(int x=0;x<=1;x++){
    float3 cell = floor(value) + float3(x, y, z);
    cellNoiseX[x] = rand3dTo1d(cell);
}
float interpolatedX = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
```
然后在外层再套一个循环，这个循环也会执行两次，然后将上一个循环中的插值结果存入临时数组，待结束后进行插值。这个操作和前面的二维插值效果基本一致。
```c++
float interpolatorX = easeInOut(frac(value.x));
float interpolatorY = easeInOut(frac(value.y));

int z = 0;

float cellNoiseY[2];
[unroll]
for(int y=0;y<=1;y++){
    float cellNoiseX[2];
    [unroll]
    for(int x=0;x<=1;x++){
        float3 cell = floor(value) + float3(x, y, z);
        cellNoiseX[x] = rand3dTo1d(cell);
    }
    cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
}
float interpolatedXY = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
```

最后在外层再套一个循环，和上一层循环类似，会将上一层循环执行两遍，在一个临时数组中记录上一层的插值结果。然后在循环结束后，对临时数组中的结果进行插值。
```c++
float ValueNoise3d(float3 value){
    float interpolatorX = easeInOut(frac(value.x));
    float interpolatorY = easeInOut(frac(value.y));
    float interpolatorZ = easeInOut(frac(value.z));

    float cellNoiseZ[2];
    [unroll]
    for(int z=0;z<=1;z++){
        float cellNoiseY[2];
        [unroll]
        for(int y=0;y<=1;y++){
            float cellNoiseX[2];
            [unroll]
            for(int x=0;x<=1;x++){
                float3 cell = floor(value) + float3(x, y, z);
                cellNoiseX[x] = rand3dTo1d(cell);
            }
            cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
        }
        cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
    }
    float noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
    return noise;
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos.xyz / _CellSize;
    float noise = ValueNoise3d(value);

    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/Grey3dNoise.png)

## 3d Output Values

上面实现了一维噪声值得平滑处理，把它改成三维噪声的平滑处理非常简单。只需要把随机函数从一维改为三维。然后将所有与噪声值相关的数据类型都改为三维。
```c++
float3 ValueNoise3d(float3 value){
    float interpolatorX = easeInOut(frac(value.x));
    float interpolatorY = easeInOut(frac(value.y));
    float interpolatorZ = easeInOut(frac(value.z));

    float3 cellNoiseZ[2];
    [unroll]
    for(int z=0;z<=1;z++){
        float3 cellNoiseY[2];
        [unroll]
        for(int y=0;y<=1;y++){
            float3 cellNoiseX[2];
            [unroll]
            for(int x=0;x<=1;x++){
                float3 cell = floor(value) + float3(x, y, z);
                cellNoiseX[x] = rand3dTo3d(cell);
            }
            cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
        }
        cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
    }
    float3 noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
    return noise;
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos.xyz / _CellSize;
    float3 noise = ValueNoise3d(value);

    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/025/Colorful3dNoise.png)

## Source

```c++
Shader "Tutorial/025_value_noise/1d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 1)) = 1
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;

		struct Input {
			float3 worldPos;
		};

		float easeIn(float interpolator){
			return interpolator * interpolator;
		}

		float easeOut(float interpolator){
			return 1 - easeIn(1 - interpolator);
		}

		float easeInOut(float interpolator){
			float easeInValue = easeIn(interpolator);
			float easeOutValue = easeOut(interpolator);
			return lerp(easeInValue, easeOutValue, interpolator);
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float value = i.worldPos.x / _CellSize;
			float previousCellNoise = rand1dTo1d(floor(value));
			float nextCellNoise = rand1dTo1d(ceil(value));
			float interpolator = frac(value);
			interpolator = easeInOut(interpolator);
			float noise = lerp(previousCellNoise, nextCellNoise, interpolator);

			float dist = abs(noise - i.worldPos.y);
			float pixelHeight = fwidth(i.worldPos.y);
			float lineIntensity = smoothstep(0, pixelHeight, dist);
			o.Albedo = lineIntensity;
		}
		ENDCG
	}
	FallBack "Standard"
}
```
```c++
Shader "Tutorial/025_value_noise/2d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 1)) = 1
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;

		struct Input {
			float3 worldPos;
		};

		float easeIn(float interpolator){
			return interpolator * interpolator;
		}

		float easeOut(float interpolator){
			return 1 - easeIn(1 - interpolator);
		}

		float easeInOut(float interpolator){
			float easeInValue = easeIn(interpolator);
			float easeOutValue = easeOut(interpolator);
			return lerp(easeInValue, easeOutValue, interpolator);
		}

		float ValueNoise2d(float2 value){
			float upperLeftCell = rand2dTo1d(float2(floor(value.x), ceil(value.y)));
			float upperRightCell = rand2dTo1d(float2(ceil(value.x), ceil(value.y)));
			float lowerLeftCell = rand2dTo1d(float2(floor(value.x), floor(value.y)));
			float lowerRightCell = rand2dTo1d(float2(ceil(value.x), floor(value.y)));

			float interpolatorX = easeInOut(frac(value.x));
			float interpolatorY = easeInOut(frac(value.y));

			float upperCells = lerp(upperLeftCell, upperRightCell, interpolatorX);
			float lowerCells = lerp(lowerLeftCell, lowerRightCell, interpolatorX);

			float noise = lerp(lowerCells, upperCells, interpolatorY);
			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float2 value = i.worldPos.xy / _CellSize;
			float noise = ValueNoise2d(value);

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```
```c++
Shader "Tutorial/025_value_noise/3d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 1)) = 1
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;

		struct Input {
			float3 worldPos;
		};

		float easeIn(float interpolator){
			return interpolator * interpolator;
		}

		float easeOut(float interpolator){
			return 1 - easeIn(1 - interpolator);
		}

		float easeInOut(float interpolator){
			float easeInValue = easeIn(interpolator);
			float easeOutValue = easeOut(interpolator);
			return lerp(easeInValue, easeOutValue, interpolator);
		}

		float3 ValueNoise3d(float3 value){
			float interpolatorX = easeInOut(frac(value.x));
			float interpolatorY = easeInOut(frac(value.y));
			float interpolatorZ = easeInOut(frac(value.z));

			float3 cellNoiseZ[2];
			[unroll]
			for(int z=0;z<=1;z++){
				float3 cellNoiseY[2];
				[unroll]
				for(int y=0;y<=1;y++){
					float3 cellNoiseX[2];
					[unroll]
					for(int x=0;x<=1;x++){
						float3 cell = floor(value) + float3(x, y, z);
						cellNoiseX[x] = rand3dTo3d(cell);
					}
					cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
				}
				cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
			}
			float3 noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos.xyz / _CellSize;
			float3 noise = ValueNoise3d(value);

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

本篇主要讲了使用线性插值来实现噪声图的平滑处理，希望能对你有所帮助。

你可以在以下链接找到源码：
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_1d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_1d.shader)
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_2d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_2d.shader)
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_3d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/025_Value_Noise/value_noise_3d.shader)

## 相关文章
- [Easing Functions for Animations](https://www.febucci.com/2018/08/easing-functions/)
- [Easing functions](https://easings.net/)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!