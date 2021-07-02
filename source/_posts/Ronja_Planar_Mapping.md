---
title: Planar Mapping
date: 2021-07-02 10:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Planar Mapping](https://www.ronja-tutorials.com/post/008-planar-mapping/)

## Summary

有时候我们的网格数据中并没有UV坐标，或者说器UV坐标并不适用于将要使用的纹理，或者我们想让纹理和模型表面根据某一规则对齐。或者还有其他什么原因，我们需要动态生成UV坐标。那么接下来的教程，我们将以最简单的方法，二维平面映射，来创建我们的UV坐标。

本教程是在上一个[图片着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Sprite_Shaders/)的基础上实现的。但是你也可以根据其基本思路，以表面着色器的形式重写其功能。
![](https://www.ronja-tutorials.com/assets/images/posts/008/Result.png)

## Basics

首先我们将输入结构体中的uv变量删除掉，因为我们打算通过脚本生成。
```c++
struct appdata{
    float vertex : POSITION;
};
```

因为片段着色器中的输入参数是由顶点着色器中的输出参数插值而得到的，因此我们选择在顶点着色器中计算新的uv值。首先我们将其顶点UV设置为顶点在模型坐标系下的`x`和`z`的值。这样足以让纹理出现在模型表面了，并且其效果看起来就好像是图片从上往下投影到模型表面一样。
```c++
v2f vert(appdata v){
    v2f o;
    o.position = UnityObjectToClipPos(v.vertex);
    o.uv = v.vertex.xz;
    return o;
}
```

## Adjustable Tiling

前面并没有考虑图片的缩放，或者我们可能希望图片显示不跟随模型一起旋转。

图片缩放的问题可以通过`TRANSFORM_TEX`宏来执行UV变换，这样最终用于纹理采样的UV可以跟随纹理缩放而相应改变。
```c++
v2f vert(appdata v){
    v2f o;
    o.position = UnityObjectToClipPos(v.vertex);
    o.uv = TRANSFORM_TEX(v.vertex.xz, _MainTex);
    return o;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/008/AdjustTilingOffset.gif)

## Texture Coordinates based on World Position

为了消除模型位置、和旋转对UV坐标的影响，我们需要将顶点坐标转换到世界坐标系，前面的例子是使用模型坐标系中的顶点坐标来生成UV坐标的。计算世界坐标的方法很简单，只需要将顶点坐标乘以模型空间矩阵。在我们得到世界坐标后，使用其世界坐标来生成UV坐标。
```c++
v2f vert(appdata v){
	v2f o;
	//计算裁剪空间下的顶点坐标
	o.position = UnityObjectToClipPos(v.vertex);
	//计算世界空间下的顶点坐标
	float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
	//应用纹理缩放，执行UV变换
	o.uv = TRANSFORM_TEX(worldPos.xz, _MainTex);
	return o;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/008/MoveSphere.gif)

从上面我们也可以看到基于世界坐标的二维平面映射也不完美，因为我们必须使用可重复的图片，否则无法覆盖整个空间区域。而且最终用渲染出来的纹理也会因为观察角度不同而发生扭曲。但是我们可以使用更牛的技术来改进，例如后面将会介绍的,三维平面映射。
```c++
Shader "Tutorial/008_Planar_Mapping"{
	//材质面板上显示的属性
	Properties{
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		//渲染不透明物体
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		Pass{
			CGPROGRAM

			#include "UnityCG.cginc"

			#pragma vertex vert
			#pragma fragment frag

			//定义公共变量
			sampler2D _MainTex;
			float4 _MainTex_ST;

			fixed4 _Color;

			struct appdata{
				float4 vertex : POSITION;
			};

			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			v2f vert(appdata v){
				v2f o;
				//计算裁剪空间下的坐标
				o.position = UnityObjectToClipPos(v.vertex);
				//计算世界空间下的坐标
				float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
				//执行UV变换
				o.uv = TRANSFORM_TEX(worldPos.xz, _MainTex);
				return o;
			}

			fixed4 frag(v2f i) : SV_TARGET{
				//纹理采样
				fixed4 col = tex2D(_MainTex, i.uv);
				//多个颜色叠加
				col *= _Color;
				return col;
			}

			ENDCG
		}
	}
	FallBack "Standard" //当当前着色器不支持时，选择后补着色器中的功能
}
```

你可以在以下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/008_Planar_Mapping/planar_mapping.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/008_Planar_Mapping/planar_mapping.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
