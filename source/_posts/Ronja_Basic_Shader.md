---
title: Basic Shader
date: 2021-07-01 12:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Basic Shader](https://www.ronja-tutorials.com/post/004-basic/)

## Summary

在前面三个教程中，我介绍了着色器工作的基本原理、与结构。接下来我进一步介绍如何修改其中的内容。

在此之前，我并没有介绍着色器的执行代码部分。因为作为入门介绍，我们需要从结构框架入手，而不应该拘泥于细节。在大致了解着色器的实现流程后，接下来让我们来发现更有趣的细节。
![](https://www.ronja-tutorials.com/assets/images/posts/004/Result.png)

## What we have so far

下面的着色器脚本，如果你觉得有前三章没有阐述到位的地方，都可以告诉我。我必将事事有会响！

```c++
Shader "Tutorial/001-004_Basic_Unlit"{
	//这些值将会显示在材质面板上
	Properties{
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		//当前的标签设置表示的是：不透明渲染，和其他不透明物体处以同一渲染队列
		Tags{ "RenderType"="Opaque" "Queue"="Geometry" }

		Pass{
			CGPROGRAM

			//这里包括一些工具函数、以及内置变量
			#include "UnityCG.cginc"

			//定义顶点、片段着色函数
			#pragma vertex vert
			#pragma fragment frag

			//材质所用的纹理、以及缩放偏移量
			sampler2D _MainTex;
			float4 _MainTex_ST;

			//材质的颜色，具体一点是：当纹理为白色图片时，材质的颜色
			fixed4 _Color;

			//模型网格、UI等输入数据，基本代表了模型内在属性
			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			//当经过顶点着色器处理后，处理的数据经过栅格化插值，然后传入片段着色器
			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			//顶点着色器函数，主要执行坐标的空间变换
			v2f vert(appdata v){
				v2f o;
				//将模型各顶点坐标，转换到裁剪空间
				o.position = UnityObjectToClipPos(v.vertex);
				//基于图片的缩放偏移参数，对UV进行变换
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return o;
			}

			//片段作色器，主要是计算像素点的颜色
			fixed4 frag(v2f i) : SV_TARGET{
			    //基于UV坐标，进行纹理采样
				fixed4 col = tex2D(_MainTex, i.uv);
				//将纹理颜色和材质颜色相乘
				col *= _Color;
				//返回最终的像素点颜色
				return col;
			}

			ENDCG
		}
	}
	Fallback "VertexLit"
}
```

## Setting up the shader stages

之前提到的顶点着色器、片段着色器，在着色器脚本中实际上就是`HLSL`函数。只不过，这些函数对应这渲染管线中的特定阶段。为了将这些函数关联到指定阶段，我们可以使用`#pragma`关键字来说明。前面经常谈到的顶点着色器、和片段着色器是非常重要的两个阶段。因为顶点着色器负责将模型数据转换到裁剪空间，然后经由栅格化处理，进入到片段着色器，最终有片段着色器计算出像素颜色，并写入渲染对象。可以说这两个着色器代表了渲染的基本流程。其中关联函数的操作如下：
```c++
//定义顶点、片段着色函数
#pragma vertex vert
#pragma fragment frag

//顶点着色器函数，主要执行坐标的空间变换
v2f vert(appdata v){
    //
}

//片段作色器，主要是计算像素点的颜色
fixed4 frag(v2f i) : SV_TARGET{
    //
}
```

## Vertex stage

在实现顶点着色器函数之前，我满需要定义好插值数据类型，前面提到过，这类数据是在顶点着色器中计算好，然后传递给片段着色器的。

顶点着色器的主要功能就是执行空间变换，将顶点数据从模型坐标系转换到裁剪坐标系。空间转换可以借助矩阵乘法来实现。但是，很多时候我们并不需要知道乘法的具体实现，因为Unity为我们提供了很多矩阵变换相关的函数。我们可以使用宏命令来引入Unity预先编写好的工具函数。例如`#include UnityCG.cginc`。其中`UnityObjectToClipPos`便是实现模型坐标系到裁剪坐标系的工具函数。而`UnityCG.cginc`文件中处了定义了很多常用的工具函数外，还定义了很多宏操作。宏操作的使用方法和函数的使用类似。例如，用于UV转换的宏`TRANSFORM_TEX`，使用顶点UV，以及纹理变量为参数，最终得到转换后的UV坐标。

编写好的顶点着色器函数如下：
```c++
//顶点着色器函数，主要执行坐标的空间变换
v2f vert(appdata v){
    v2f o;
    //将模型各顶点坐标，转换到裁剪空间
    o.position = UnityObjectToClipPos(v.vertex);
    //基于图片的缩放偏移参数，对UV进行变换
    o.uv = TRANSFORM_TEX(v.uv, _MainTex);
    return o;
}
```

## Fragment stage

在片段着色器中，我们使用由顶点着色器传来的插值数据、以及公共的纹理数据等，来进一步计算每个像素点的颜色。当然，我们可以直接返回白色，如`return float4(1,1,1,1);`。但是更实际的情况是，我们结合前面提到的数据，通过一些简单、或复杂的计算，来得到比较自然的颜色值。

我们把使用纹理数据的过程叫做纹理采样。因为纹理数据是一整张包含无数像素点的图片，而我们的片段着色器计算的是一个单独像素的颜色。因此我们只需要纹理中的一个像素值。采样的方法也很简单，直接使用`tex2D`函数就可以，当然还有其他一些复杂一点的采样函数。这里我们以`tex2D`为例，需要两个参数，第一个是纹理变量，第二个是UV坐标。下面我们除了使用采样后的像素颜色，同时也叠加了公共变量`_Color`的值。
```c++
//片段作色器，主要是计算像素点的颜色
fixed4 frag(v2f i) : SV_TARGET{
    //基于UV坐标，进行纹理采样
    fixed4 col = tex2D(_MainTex, i.uv);
    //将纹理颜色和材质颜色相乘
    col *= _Color;
    //返回最终的像素点颜色
    return col;
}
```

把上面的各个步骤组合起来，恭喜你，创建了属于你自己的着色器脚本。

## Source

- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/001-004_basic_unlit/basic_unlit.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/001-004_basic_unlit/basic_unlit.shader)

```c++
Shader "Tutorial/001-004_Basic_Unlit"{
	//这些值将会显示在材质面板上
	Properties{
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		//当前的标签设置表示的是：不透明渲染，和其他不透明物体处以同一渲染队列
		Tags{ "RenderType"="Opaque" "Queue"="Geometry" }

		Pass{
			CGPROGRAM

			//这里包括一些工具函数、以及内置变量
			#include "UnityCG.cginc"

			//定义顶点、片段着色函数
			#pragma vertex vert
			#pragma fragment frag

			//材质所用的纹理、以及缩放偏移量
			sampler2D _MainTex;
			float4 _MainTex_ST;

			//材质的颜色，具体一点是：当纹理为白色图片时，材质的颜色
			fixed4 _Color;

			//模型网格、UI等输入数据，基本代表了模型内在属性
			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			//当经过顶点着色器处理后，处理的数据经过栅格化插值，然后传入片段着色器
			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			//顶点着色器函数，主要执行坐标的空间变换
			v2f vert(appdata v){
				v2f o;
				//将模型各顶点坐标，转换到裁剪空间
				o.position = UnityObjectToClipPos(v.vertex);
				//基于图片的缩放偏移参数，对UV进行变换
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return o;
			}

			//片段作色器，主要是计算像素点的颜色
			fixed4 frag(v2f i) : SV_TARGET{
			    //基于UV坐标，进行纹理采样
				fixed4 col = tex2D(_MainTex, i.uv);
				//将纹理颜色和材质颜色相乘
				col *= _Color;
				//返回最终的像素点颜色
				return col;
			}

			ENDCG
		}
	}
	Fallback "VertexLit"
}
```

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
