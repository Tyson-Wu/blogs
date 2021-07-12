---
title: Layered Noise
date: 2021-07-12 18:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Layered Noise](https://www.ronja-tutorials.com/post/027-layered-noise/)

## Layered Noise

目前为止，我们创建的噪声要么非常光滑，要么过于随机。我们可以将它们结合起来，从而实现一种多层次的噪声图。这样我们可以得到既平滑又细节丰富的噪声图。我们前面介绍过的值类噪声和泊林噪声都可以应用到本章的叠加噪声中。虽然叠加噪声产生的效果可能更符合你的预期，但是因为叠加了多个噪声图，所以其性能消耗也是叠加的。

![](https://www.ronja-tutorials.com/assets/images/posts/027/Result.gif)

## Layered 1d Noise

在前面的教程中，我们实现的噪声图可以通过控制色块的大小，进而控制噪声的频率。
![](https://www.ronja-tutorials.com/assets/images/posts/027/PerlinSizes.gif)

和之前一样，我们首先从一维开始做起。不过我们需要采集两次噪声，一次是和原来一样，一次是乘以2。乘以2的操作实际上就是提高它的频率。然后我们将高频噪声的值降低，高频低强度的噪声可以用来表示细节，然后将其叠加到低频噪声上。

为了便于大家理解，这里我还是照常贴出相关源码。
```c++
float sampleLayeredNoise(float value){
    float noise = gradientNoise(value);
    float highFreqNoise = gradientNoise(value * 6);
    noise = noise + highFreqNoise * 0.2;
    return noise;
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float value = i.worldPos.x / _CellSize;
    float noise = sampleLayeredNoise(value);
    
    float dist = abs(noise - i.worldPos.y);
    float pixelHeight = fwidth(i.worldPos.y);
    float lineIntensity = smoothstep(2*pixelHeight, pixelHeight, dist);
    o.Albedo = lerp(1, 0, lineIntensity);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/027/2Sample1dGradient.png)

上面简单的叠加已经能达到一些效果了，一次叠加获得比较粗糙的细节。不过我们还可以继续叠加。我们将叠加的层数称为阶，所以我们现在是二阶叠加噪声。

为了实现高阶叠加噪声，我们将使用循环语句。同时我们的阶数也可以使用宏命令来设置为常量，这样可以在材质面板上调节。当然我们也可以将阶数完全用变量来表示，但是使用变量控制的循环语句，是没办法进行展开优化的，所以应该尽量避免。这里每个层的频率变量乘以上一个层的实际频率，得到当前层的实际频率。我们叫这个频率变量为粗糙度，或者说相对于上一层的粗糙度。我们每一层之间的相对粗糙度取同一个值，所以所有层的相对粗糙度都为`_persistance`。然后还有一个从整体上控制粗糙度的量`_Routhness`。

```c++
Properties {
    _CellSize ("Cell Size", Range(0, 2)) = 2
    _Roughness ("Roughness", Range(1, 8)) = 3
    _Persistance ("Persistance", Range(0, 1)) = 0.4
}
```
```c++
//公共变量
#define OCTAVES 4 

float _CellSize;
float _Roughness;
float _Persistance;
```
```c++
float sampleLayeredNoise(float value){
    float noise = 0;
    float frequency = 1;
    float factor = 1;

    [unroll]
    for(int i=0; i<OCTAVES; i++){
        noise = noise + gradientNoise(value * frequency + i * 0.72354) * factor;
        factor *= _Persistance;
        frequency *= _Roughness;
    }

    return noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/027/Layered1dGradient.png)

## Layered multidimensional Noise

多维噪声图的实现方式类似，只不过相应的噪声函数输入参数要改为对应维度的向量。这里我们以二维为例。
```c++
float sampleLayeredNoise(float2 value){
    float noise = 0;
    float frequency = 1;
    float factor = 1;

    [unroll]
    for(int i=0; i<OCTAVES; i++){
        noise = noise + perlinNoise(value * frequency + i * 0.72354) * factor;
        factor *= _Persistance;
        frequency *= _Roughness;
    }

    return noise;
}
```
下面是三维。
```c++
float sampleLayeredNoise(float3 value){
    float noise = 0;
    float frequency = 1;
    float factor = 1;

    [unroll]
    for(int i=0; i<OCTAVES; i++){
        noise = noise + perlinNoise(value * frequency + i * 0.72354) * factor;
        factor *= _Persistance;
        frequency *= _Roughness;
    }

    return noise;
}

```
![](https://www.ronja-tutorials.com/assets/images/posts/027/NormalLayeredComparison.png)

## Special Use Case

另一个经常用到噪声的场景是高度图。通常我们处理纹理是在片段着色器中，但是当处理高度图时，我们是在顶点着色器中采样的，然后将采样结果叠加到顶点的`y`坐标。关于这部分的内容，你可以参考之前关于[顶点偏移](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Vertex_Displacement/)的介绍。

首先，我们修改表面着色器的宏命令部分。添加顶点处理函数，`#pragma surface surf Standard fullforwardshadows vertex:vert addshadow`。然后实现顶点处理函数，其中将采样噪声叠加到顶点的`y`坐标，然后计算顶点世界坐标。然后在表面着色器中，将光照参数`albedo`设置为白色。如果我们想让其表现的更细致，应该使用分辨率更高的网格数据，否者的话看到的将是很明显的多边形网格。
```c++
Properties {
    _CellSize ("Cell Size", Range(0, 10)) = 2
    _Roughness ("Roughness", Range(1, 8)) = 3
    _Persistance ("Persistance", Range(0, 1)) = 0.4
    _Amplitude("Amplitude", Range(0, 10)) = 1
}
```
```c++
//噪声强度
float _Amplitude;
```
```c++
void vert(inout appdata_full data){
    float4 worldPos = mul(unity_ObjectToWorld, data.vertex);
    float3 value = worldPos / _CellSize;
    //将噪声值映射到0-1区间
    float noise = sampleLayeredNoise(value) + 0.5;
    data.vertex.y += noise * _Amplitude;
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    o.Albedo = 1;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/027/HeightNoiseWrongNormals.png)

到目前为止，虽然我们修改了顶点位置，但是在阴影处理的时候还是被当做平面来看。我们可以让其根据新的顶点位置重新计算阴影，这一点在[顶点偏移](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Vertex_Displacement/)中有介绍。

在这里我遇到一个问题，就是传入的顶点齐次坐标的`w`值并不为1，也就是说该坐标是缩放后的坐标。因此我们需要执行齐次除法。
```c++
void vert(inout appdata_full data){
    //执行齐次除法，得到真实的顶点坐标
    float3 localPos = data.vertex / data.vertex.w;

    //计算新的顶点坐标
    float3 modifiedPos = localPos;
    float2 basePosValue = mul(unity_ObjectToWorld, modifiedPos).xz / _CellSize;
    float basePosNoise = sampleLayeredNoise(basePosValue) + 0.5;
    modifiedPos.y += basePosNoise * _Amplitude;
    
    //计算新坐标的切向量方向的临近点
    float3 posPlusTangent = localPos + data.tangent * 0.02;
    float2 tangentPosValue = mul(unity_ObjectToWorld, posPlusTangent).xz / _CellSize;
    float tangentPosNoise = sampleLayeredNoise(tangentPosValue) + 0.5;
    posPlusTangent.y += tangentPosNoise * _Amplitude;

    //计算新坐标的 bitangent方向的临近点
    float3 bitangent = cross(data.normal, data.tangent);
    float3 posPlusBitangent = localPos + bitangent * 0.02;
    float2 bitangentPosValue = mul(unity_ObjectToWorld, posPlusBitangent).xz / _CellSize;
    float bitangentPosNoise = sampleLayeredNoise(bitangentPosValue) + 0.5;
    posPlusBitangent.y += bitangentPosNoise * _Amplitude;

    //计算切向量和bitangent
    float3 modifiedTangent = posPlusTangent - modifiedPos;
    float3 modifiedBitangent = posPlusBitangent - modifiedPos;

    //计算新的法向
    float3 modifiedNormal = cross(modifiedTangent, modifiedBitangent);
    data.normal = normalize(modifiedNormal);
    data.vertex = float4(modifiedPos.xyz, 1);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/027/CorrectedNormals.png)

当然，我们也可以引入时间变量，从而达到滚动动画效果。
```c++
//材质属性
_ScrollDirection("Scroll Direction", Vector) = (0, 1)
```
```c++
//公共变量，滚动方向
float2 _ScrollDirection;
```
```c++
//计算水平位置
float2 basePosValue = mul(unity_ObjectToWorld, modifiedPos).xz / _CellSize + _ScrollDirection * _Time.y;
```
```c++
//计算切向临近坐标
float2 tangentPosValue = mul(unity_ObjectToWorld, posPlusTangent).xz / _CellSize + _ScrollDirection * _Time.y;
```
```c++
//计算bitangent方向临近坐标
float2 bitangentPosValue = mul(unity_ObjectToWorld, posPlusBitangent).xz / _CellSize + _ScrollDirection * _Time.y;
```
![](https://www.ronja-tutorials.com/assets/images/posts/027/Result.gif)

## Source

### 1d layered noise
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_1d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_1d.shader)
```c++
Shader "Tutorial/027_layered_noise/1d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_Roughness ("Roughness", Range(1, 8)) = 3
		_Persistance ("Persistance", Range(0, 1)) = 0.4
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		//公共变量
		#define OCTAVES 4 

		float _CellSize;
		float _Roughness;
		float _Persistance;

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

		float sampleLayeredNoise(float value){
			float noise = 0;
			float frequency = 1;
			float factor = 1;

			[unroll]
			for(int i=0; i<OCTAVES; i++){
				noise = noise + gradientNoise(value * frequency + i * 0.72354) * factor;
				factor *= _Persistance;
				frequency *= _Roughness;
			}

			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float value = i.worldPos.x / _CellSize;
			float noise = sampleLayeredNoise(value);
			
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

### 2d layered noise

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_2d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_2d.shader)
```c++
Shader "Tutorial/027_layered_noise/2d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_Roughness ("Roughness", Range(1, 8)) = 3
		_Persistance ("Persistance", Range(0, 1)) = 0.4
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		//公共变量
		#define OCTAVES 4 

		float _CellSize;
		float _Roughness;
		float _Persistance;

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
			//计算色块的四个顶点
			float2 lowerLeftDirection = rand2dTo2d(float2(floor(value.x), floor(value.y))) * 2 - 1;
			float2 lowerRightDirection = rand2dTo2d(float2(ceil(value.x), floor(value.y))) * 2 - 1;
			float2 upperLeftDirection = rand2dTo2d(float2(floor(value.x), ceil(value.y))) * 2 - 1;
			float2 upperRightDirection = rand2dTo2d(float2(ceil(value.x), ceil(value.y))) * 2 - 1;

			float2 fraction = frac(value);

			//计算四个顶点在当前位置的贡献
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

		float sampleLayeredNoise(float2 value){
			float noise = 0;
			float frequency = 1;
			float factor = 1;

			[unroll]
			for(int i=0; i<OCTAVES; i++){
				noise = noise + perlinNoise(value * frequency + i * 0.72354) * factor;
				factor *= _Persistance;
				frequency *= _Roughness;
			}

			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float2 value = i.worldPos.xz / _CellSize;
			//将噪声值映射到0-1区间
			float noise = sampleLayeredNoise(value) + 0.5;

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### 3d layered noise

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_3d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_perlin_noise_3d.shader)
```c++
Shader "Tutorial/027_layered_noise/3d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_Roughness ("Roughness", Range(1, 8)) = 3
		_Persistance ("Persistance", Range(0, 1)) = 0.4
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		//公共变量
		#define OCTAVES 4 

		float _CellSize;
		float _Roughness;
		float _Persistance;

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

		float sampleLayeredNoise(float3 value){
			float noise = 0;
			float frequency = 1;
			float factor = 1;

			[unroll]
			for(int i=0; i<OCTAVES; i++){
				noise = noise + perlinNoise(value * frequency + i * 0.72354) * factor;
				factor *= _Persistance;
				frequency *= _Roughness;
			}

			return noise;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos / _CellSize;
			//将噪声值映射到0-1区间
			float noise = sampleLayeredNoise(value) + 0.5;

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### Scrolling height noise

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_noise_special.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/027_Layered_Noise/layered_noise_special.shader)

```c++
Shader "Tutorial/027_layered_noise/special_use_case" {
	Properties {
		_CellSize ("Cell Size", Range(0, 16)) = 2
		_Roughness ("Roughness", Range(1, 8)) = 3
		_Persistance ("Persistance", Range(0, 1)) = 0.4
		_Amplitude("Amplitude", Range(0, 10)) = 1
		_ScrollDirection("Scroll Direction", Vector) = (0, 1, 0, 0)
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows vertex:vert addshadow
		#pragma target 3.0 

		#include "Random.cginc"

		//global shader variables
		#define OCTAVES 4 

		float _CellSize;
		float _Roughness;
		float _Persistance;
		float _Amplitude;

		float2 _ScrollDirection;

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
			//计算色块的四个顶点
			float2 lowerLeftDirection = rand2dTo2d(float2(floor(value.x), floor(value.y))) * 2 - 1;
			float2 lowerRightDirection = rand2dTo2d(float2(ceil(value.x), floor(value.y))) * 2 - 1;
			float2 upperLeftDirection = rand2dTo2d(float2(floor(value.x), ceil(value.y))) * 2 - 1;
			float2 upperRightDirection = rand2dTo2d(float2(ceil(value.x), ceil(value.y))) * 2 - 1;

			float2 fraction = frac(value);

			//计算四个顶点在当前位置的贡献
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

		float sampleLayeredNoise(float2 value){
			float noise = 0;
			float frequency = 1;
			float factor = 1;

			[unroll]
			for(int i=0; i<OCTAVES; i++){
				noise = noise + perlinNoise(value * frequency + i * 0.72354) * factor;
				factor *= _Persistance;
				frequency *= _Roughness;
			}

			return noise;
		}
		
        void vert(inout appdata_full data){
            //执行齐次除法，得到真实的顶点坐标
            float3 localPos = data.vertex / data.vertex.w;

            //计算新的顶点坐标
            float3 modifiedPos = localPos;
            float2 basePosValue = mul(unity_ObjectToWorld, modifiedPos).xz / _CellSize;
            float basePosNoise = sampleLayeredNoise(basePosValue) + 0.5;
            modifiedPos.y += basePosNoise * _Amplitude;
            
            //计算新坐标的切向量方向的临近点
            float3 posPlusTangent = localPos + data.tangent * 0.02;
            float2 tangentPosValue = mul(unity_ObjectToWorld, posPlusTangent).xz / _CellSize;
            float tangentPosNoise = sampleLayeredNoise(tangentPosValue) + 0.5;
            posPlusTangent.y += tangentPosNoise * _Amplitude;

            //计算新坐标的 bitangent方向的临近点
            float3 bitangent = cross(data.normal, data.tangent);
            float3 posPlusBitangent = localPos + bitangent * 0.02;
            float2 bitangentPosValue = mul(unity_ObjectToWorld, posPlusBitangent).xz / _CellSize;
            float bitangentPosNoise = sampleLayeredNoise(bitangentPosValue) + 0.5;
            posPlusBitangent.y += bitangentPosNoise * _Amplitude;

            //计算切向量和bitangent
            float3 modifiedTangent = posPlusTangent - modifiedPos;
            float3 modifiedBitangent = posPlusBitangent - modifiedPos;

            //计算新的法向
            float3 modifiedNormal = cross(modifiedTangent, modifiedBitangent);
            data.normal = normalize(modifiedNormal);
            data.vertex = float4(modifiedPos.xyz, 1);
        }

		void surf (Input i, inout SurfaceOutputStandard o) {
			o.Albedo = 1;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

高阶叠加噪声可以很好的模拟复杂的噪声图案。你也可以试着将不同类型的噪声进行叠加，然后看看会产生什么现象。总之，我希望你们能够理解其中的基本原理，然后肉会贯通。如果你发现有任何理解不了的地方，可以给我留言。

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!