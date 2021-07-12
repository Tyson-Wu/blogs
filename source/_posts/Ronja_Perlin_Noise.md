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

泊林噪声也是非常常用的一种噪声。和上一篇值类噪声非常相似，泊林噪声也是基于噪声色块，是“梯度噪声”实现方式的一种，所以容易产生重复、光滑的效果。它们之间的区别在于本篇将上上一篇得到的噪声当做方向来处理，而值类噪声则将其当做值来处理。因为噪声部分内容复杂，建议你先从噪点、和值类噪声开始学习。
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

正如开篇提到的，泊林噪声是将初始的随机噪声当做方向来处理，然后在此基础上进行进一步平滑操作。因此我们首先要计算梯度。梯度方向可上可下，因此我们需要将噪声值从[0,1]区间变换到[-1,1]区间。

在前面值类噪声处理中，我们很容易得到每个色块的噪声值，但是现在我们记录的并不是色块的噪声值，而是该色块噪声变化的一种趋势，也就是前面说的方向，梯度方向。但是在平滑处理的插值运算中，我们仍然需要知道色块具体位置的噪声值，所以我们要根据这个梯度方向来重新构造一个噪声分布曲线。因为梯度的存在，这里构造的是倾斜的一段一段的直线。在前面的值类噪声中的线段是水平的，这也是值类和方向类噪声处理的实际差别。
```c++
float gradientNoise(float value){
    float fraction = frac(value);

    float previousCellInclination = rand1dTo1d(floor(value)) * 2 - 1;
    float previousCellLinePoint = previousCellInclination * fraction;
    
    return previousCellLinePoint;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/Inclinations.png)

上面的线段都是沿着色块中心点沿着一个方向伸展的，色块之间没有重叠区域，所以需要去下一个色块，沿着反方向延伸。这样，两个相邻色块之间有了重叠区域，可以做插值平滑处理。而正反方向的延伸要保持在同一条直线上。因为前面定义正向延伸的区间是0到1，所以反向延伸的区间就是-1到0。
```c++
float nextCellInclination = rand1dTo1d(ceil(value)) * 2 - 1; //下一个色块中心
float nextCellLinePoint = nextCellInclination * (fraction - 1);// -1 到 0
```

接下来是前后两端的插值，为了保证线条连续，我们设定距离哪个色块越近，其插值权重越大。为了保证线条光滑，我们仍然使用缓动函数来平滑。
```c++
float gradientNoise(float value){
    float fraction = frac(value);
    float interpolator = easeInOut(fraction);

    float previousCellInclination = rand1dTo1d(floor(value)) * 2 - 1;
    float previousCellLinePoint = previousCellInclination * fraction;

    float nextCellInclination = rand1dTo1d(ceil(value)) * 2 - 1;
    float nextCellLinePoint = nextCellInclination * (fraction - 1);

    return lerp(previousCellLinePoint, nextCellLinePoint, interpolator);
}
```

另一个小改动是，在调用一维随机数函数时，我将其中不再使用乘积方式，而是改成加一个常数，这样当输入为0的时候，结果也不会永远是零。这个我在随机数那一篇文章应该也仅仅做了修改。

```c++
float rand1dTo1d(float3 value, float mutator = 0.546){
	float random = frac(sin(value + mutator) * 143758.5453);
	return random;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/1dGradient.png)

## 2d Perlin Noise

对于多维泊林噪声，处理起来就比较麻烦了。首先我们噪声的需要在多个维度进行插值，并且我们的噪声梯度方向也是多维的。
```c++
float perlinNoise(float2 value){
    //...
}
```

而且相邻点也有两个变为四个，也就是需要在色块的四个角点上进行噪声值采样，将采样值转换为方向向量。
```c++
float2 lowerLeftDirection = rand2dTo2d(float2(floor(value.x), floor(value.y))) * 2 - 1;
float2 lowerRightDirection = rand2dTo2d(float2(ceil(value.x), floor(value.y))) * 2 - 1;
float2 upperLeftDirection = rand2dTo2d(float2(floor(value.x), ceil(value.y))) * 2 - 1;
float2 upperRightDirection = rand2dTo2d(float2(ceil(value.x), ceil(value.y))) * 2 - 1;
```

得到四个角点的梯度向量后，计算四个角点在当前点的贡献值。
```c++
float2 fraction = frac(value);

float lowerLeftFunctionValue = dot(lowerLeftDirection, fraction - float2(0, 0));
float lowerRightFunctionValue = dot(lowerRightDirection, fraction - float2(0, 1));
float upperLeftFunctionValue = dot(upperLeftDirection, fraction - float2(1, 0));
float upperRightFunctionValue = dot(upperRightDirection, fraction - float2(1, 1));
```

然后对他们进行二次插值。
```c++
float interpolatorX = easeInOut(fraction.x);
float interpolatorY = easeInOut(fraction.y);

float lowerCells = lerp(lowerLeftFunctionValue, lowerRightFunctionValue, interpolatorX);
float upperCells = lerp(upperLeftFunctionValue, upperRightFunctionValue, interpolatorX);

float noise = lerp(lowerCells, upperCells, interpolatorY);
return noise;
```

现在我们可以在片段函数中使用我们的泊林噪声函数了，因为生成的泊林噪声值是在-0.5到0.5之间，所以我们需要将其映射到0到1之间。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xz / _CellSize;
    float noise = perlinNoise(value) + 0.5;

    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/2dPerlin.png)

## 3d Perlin Noise

三维泊林噪声和上一遍的值类噪声很类似，只不过在最内层循环中，我们不能直接得到噪声值，而是需要通过梯度方向来计算噪声值。
```c++
float perlinNoise(float3 value){
    float3 fraction = frac(value);

    float interpolatorX = easeInOut(fraction.x);
    float interpolatorY = easeInOut(fraction.y);
    float interpolatorZ = easeInOut(fraction.z);

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
                float3 cellDirection = rand3dTo3d(cell) * 2 - 1;
                float3 compareVector = fraction - float3(x, y, z);
                cellNoiseX[x] = dot(cellDirection, compareVector);
            }
            cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
        }
        cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
    }
    float noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
    return noise;
}
```

然后在片段着色器中，我们使用三维向量就可以生成三维泊林噪声了。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos / _CellSize;
    //将噪声值映射到0-1区间
    float noise = perlinNoise(value) + 0.5;

    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/3dPerlin.png)

## Special Use Case

泊林噪声看起来像行为怪异的云朵，但是可以实现一些有趣的效果。

首先，我们可以将相同值的泊林噪声点相连接，形成类似地图等高线的图案。首先我们将噪声值放大，然后取其小数部分。
```c++
float3 value = i.worldPos / _CellSize;
//将噪声值映射到0-1区间
float noise = perlinNoise(value) + 0.5;

noise = frac(noise * 6);

o.Albedo = noise;
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/fracNoise.png)

然后我们来实现光滑的等高线。首先我们需要计算一个像素内的噪声变化情况，这里我们可以使用`fwidth`函数。然后使用`smoothstep`函数来剔除小于1的像素，因为像素是离散分布的，所以有些像素只能近似等于1，所以需要以一个像素内的变化值来做近似处理。同样的我们再剔除大于0的像素，两次剔除得到的边界相叠加。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos / _CellSize;
    //将噪声值映射到0-1区间
    float noise = perlinNoise(value) + 0.5;

    noise = frac(noise * 6);

    float pixelNoiseChange = fwidth(noise);

    float heightLine = smoothstep(1-pixelNoiseChange, 1, noise);
    heightLine += smoothstep(pixelNoiseChange, 0, noise);

    o.Albedo = heightLine;
}
```

![](https://www.ronja-tutorials.com/assets/images/posts/026/heightLines.png)

还有一个使用多维噪声的技巧，就是将其中一个不用的维度抽出来，加上时间变量，这样随着时间的改变，我们的噪声图也会随着变化。
```c++
Properties {
    _CellSize ("Cell Size", Range(0, 1)) = 1
    _ScrollSpeed ("Scroll Speed", Range(0, 1)) = 1
}
```
```c++
//公共变量
float _CellSize;
float _ScrollSpeed;
```
```c++
float3 value = i.worldPos / _CellSize;
value.y += _Time.y * _ScrollSpeed;
//将噪声值映射到0-1区间
float noise = perlinNoise(value) + 0.5;
```
![](https://www.ronja-tutorials.com/assets/images/posts/026/AnimatedLines.gif)

## Source

### 1d gradient noise
```c++
Shader "Tutorial/026_perlin_noise/1d" {
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
			return interpolator * interpolator * interpolator * interpolator * interpolator;
		}

		float easeOut(float interpolator){
			return 1 - easeIn(1 - interpolator);
		}

		float easeInOut(float interpolator){
			float easeInValue = easeIn(interpolator);
			float easeOutValue = easeOut(interpolator);
			return lerp(easeInValue, easeOutValue, interpolator);
		}

		float gradientNoise(float value){
			float fraction = frac(value);
			float interpolator = easeInOut(fraction);

			float previousCellInclination = rand1dTo1d(floor(value)) * 2 - 1;
			float previousCellLinePoint = previousCellInclination * fraction;

			float nextCellInclination = rand1dTo1d(ceil(value)) * 2 - 1;
			float nextCellLinePoint = nextCellInclination * (fraction - 1);

			return lerp(previousCellLinePoint, nextCellLinePoint, interpolator);
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float value = i.worldPos.x / _CellSize;
			float noise = gradientNoise(value);
			
			float dist = abs(noise - i.worldPos.y);
			float pixelHeight = fwidth(i.worldPos.y);
			float lineIntensity = smoothstep(2*pixelHeight, pixelHeight, dist);
			o.Albedo = lerp(1, 0, lineIntensity);
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### 2d perlin noise
```c++
Shader "Tutorial/026_perlin_noise/2d" {
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
		float _Jitter;

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

		float perlinNoise(float2 value){
			//计算色块四个顶点的梯度向量
			float2 lowerLeftDirection = rand2dTo2d(float2(floor(value.x), floor(value.y))) * 2 - 1;
			float2 lowerRightDirection = rand2dTo2d(float2(ceil(value.x), floor(value.y))) * 2 - 1;
			float2 upperLeftDirection = rand2dTo2d(float2(floor(value.x), ceil(value.y))) * 2 - 1;
			float2 upperRightDirection = rand2dTo2d(float2(ceil(value.x), ceil(value.y))) * 2 - 1;

			float2 fraction = frac(value);

			//计算四个顶点在当前点的贡献值
			float lowerLeftFunctionValue = dot(lowerLeftDirection, fraction - float2(0, 0));
			float lowerRightFunctionValue = dot(lowerRightDirection, fraction - float2(1, 0));
			float upperLeftFunctionValue = dot(upperLeftDirection, fraction - float2(0, 1));
			float upperRightFunctionValue = dot(upperRightDirection, fraction - float2(1, 1));

			float interpolatorX = easeInOut(fraction.x);
			float interpolatorY = easeInOut(fraction.y);

			//二次插值
			float lowerCells = lerp(lowerLeftFunctionValue, lowerRightFunctionValue, interpolatorX);
			float upperCells = lerp(upperLeftFunctionValue, upperRightFunctionValue, interpolatorX);

			float noise = lerp(lowerCells, upperCells, interpolatorY);
			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float2 value = i.worldPos.xz / _CellSize;
			//将噪声值映射到0-1区间
			float noise = perlinNoise(value) + 0.5;

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### 3d perlin noise
```c++
Shader "Tutorial/026_perlin_noise/3d" {
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
		float _Jitter;

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

		float perlinNoise(float3 value){
			float3 fraction = frac(value);

			float interpolatorX = easeInOut(fraction.x);
			float interpolatorY = easeInOut(fraction.y);
			float interpolatorZ = easeInOut(fraction.z);

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
						float3 cellDirection = rand3dTo3d(cell) * 2 - 1;
						float3 compareVector = fraction - float3(x, y, z);
						cellNoiseX[x] = dot(cellDirection, compareVector);
					}
					cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
				}
				cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
			}
			float noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos / _CellSize;
			//将噪声值映射到0-1区间
			float noise = perlinNoise(value) + 0.5;

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### special use tircks

```c++
Shader "Tutorial/026_perlin_noise/special" {
	Properties {
		_CellSize ("Cell Size", Range(0, 1)) = 1
		_ScrollSpeed ("Scroll Speed", Range(0, 1)) = 1
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;
		float _ScrollSpeed;

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

		float perlinNoise(float3 value){
			float3 fraction = frac(value);

			float interpolatorX = easeInOut(fraction.x);
			float interpolatorY = easeInOut(fraction.y);
			float interpolatorZ = easeInOut(fraction.z);

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
						float3 cellDirection = rand3dTo3d(cell) * 2 - 1;
						float3 compareVector = fraction - float3(x, y, z);
						cellNoiseX[x] = dot(cellDirection, compareVector);
					}
					cellNoiseY[y] = lerp(cellNoiseX[0], cellNoiseX[1], interpolatorX);
				}
				cellNoiseZ[z] = lerp(cellNoiseY[0], cellNoiseY[1], interpolatorY);
			}
			float noise = lerp(cellNoiseZ[0], cellNoiseZ[1], interpolatorZ);
			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos / _CellSize;
			value.y += _Time.y * _ScrollSpeed;
			//将噪声值映射到0-1区间
			float noise = perlinNoise(value) + 0.5;

			noise = frac(noise * 6);

			float pixelNoiseChange = fwidth(noise);

			float heightLine = smoothstep(1-pixelNoiseChange, 1, noise);
			heightLine += smoothstep(pixelNoiseChange, 0, noise);

			o.Albedo = heightLine;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

我花了很长时间来理解泊林噪声的工作原理，这里做了一些总结，希望能对你理解泊林噪声有所帮助。

你可以在以下链接找到源码：
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_1d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_1d.shader)
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_2d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_2d.shader)
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_3d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_3d.shader)
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_special.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/026_Perlin_Noise/perlin_noise_special.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!