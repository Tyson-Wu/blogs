---
title: Variables
date: 2021-07-01 11:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Variables](https://www.ronja-tutorials.com/post/003-variables/)

## Summary

在了解`Shaderlab`语言的基本结构，以及着色器各个阶段的功能划分，接下来，让我们来学习一下着色器中所用到的变量，以及如何在代码中使用它们。在着色器中，变量可以分为材质变量、模型网格变量、以及各个着色阶段数值传递的中间变量。

## Object Data

模型数据。在介绍渲染过程的时候，我所提到的模型数据，实际上就是模型上面的网格数据。从底层角度来看，这些数据定义了模型的几何形状，决定了模型最终显示的形状。不过为了方便描述，我们直接将其归纳为模型数据、或者网格数据。通常情况下，模型数据包含模型中各个顶点的位置、以及三角面片序列。当然有些模型数据还包含的顶点法向、UV、颜色等数据。除了三角面片序列，其他的数据都是逐(个)顶点数据，也就是说顶点法向、UV、颜色、和顶点位置一一对应，具有相同的个数。因为顶点的位置数据是基于模型坐标系，所以，无论模型位置、朝向如何，都不会影响模型数据。所以对于同一种模型，我们可以使用同一个模型数据，然后通过对模型的缩放来实现一定的差异化。

在Unity着色器中， 模型数据首先是传递给顶点着色器，而模型数据通常也是以自定义数据类型表示，Unity也预先帮我们定义了一些类型，例如`struct appdata`。当然我们也可以按需自定义，类型的名字可以任意，只要不要和已有的重名就行。当然，因为着色器在执行的过程中有一套固定的流程，包括在各个节点使用什么样的数据。而我们定义的类型并不能传达这些信息，因此，需要在自定义类型的成员变量后面加上语义标识。如下所示,通过`POSITION`来表示我们的`vertex`是顶点坐标。
```c++
struct appdata{
  float4 vertex : POSITION;//顶点坐标
  float2 uv : TEXCOORD0;//UV坐标
};//别忘了加分号
```

关于其他定义的语义标识符，可以参考以下链接：
[https://docs.unity3d.com/Manual/SL-VertexProgramInputs.html](https://docs.unity3d.com/Manual/SL-VertexProgramInputs.html)

## Interpolators

插值数据。当顶点着色器将模型数据从模型空间转换到裁剪空间时，顶点着色阶段的任务就已经结束了。这时候需要将处理好的数据传递給下一个阶段，通常情况下是片段着色阶段。但是顶点着色器输出的结果是基于顶点的，但是片段着色器是基于像素点的，例如渲染一个三角形，顶点作色器只处理三个顶点，但是这个三角形投影到屏幕上就不止三个点了，一般会有更多的像素点构成。所以从顶点着色器到片段作色器，前后输出和输入参数个数不对对等，所以需要通过插值的方式来生成其他可能的像素点数据。

这里的插值过程又叫做栅格化处理，因为我们的屏幕是由一格一格的光栅构成，所以有此得名。栅格化处理是由硬件完成的，虽然这一步也属于整个渲染管线的一步，但是我们却不能对其进行修改。所以我们也需要通过语义标识符来告诉硬件，各个数据的用途。例如`SV_POSITION`就表示投影变换后的顶点坐标，后面也是根据它来进行插值操作，最终得到屏幕像素点。当然还有其他可选的语义标识符可以使用，例如顶点颜色、UV等，用法基本类似。

在习惯上人们通常会将插值数据命名为`v2f`，也就是`vertex to fragment`的缩写，表示是从顶点到像素片段的中间变量。具体例子如下：
```c++
//该数据是从顶点着色器，经过栅格化处理，传入到片段着色器中
struct v2f{
  float4 position : SV_POSITION;
  float2 uv : TEXCOORD0;
};
```

## Output color

最终输出的颜色值。片段着色器主要用于计算像素点的颜色，通常计算的颜色值由4维向量表示，分别对应红、绿、蓝、透明四个通道。这里也有一个语义标识符来表示输出的颜色`SV_Target`。

## Uniform data

公共数据。因为GPU的渲染过程是一个并行过程，模型数据传入后，在顶点着色器中，顶点之间属于并列关系，同一时间有多个顶点同时执行顶点着色器的逻辑。可以想象成一个军队，每个士兵拿着自己的武器在战场上做着同样的事情。但是这些数据有一些共性，它们同属于一个模型、引用同一张纹理贴图、受同一个光照影响。但是我们不可能为每一个顶点配置一份相同的数据。因此把这些数据抽象出来，形成一个公共部分，所有顶点都可以共享这些数据。在片段着色器中也类似，每个像素也都可以对同一张纹理进行采样。公共数据有很多，除了前面提到的，还有各种空间矩阵、以及一些自定义需求引入的数据。庆幸的是，大部分公共数据Unity都已经为我们定义好了，并且在程序执行时，会对其自动赋值。只有少数我们自己定义的公共数据需要我们初始化。

定义公共数据也很简单，直接向当前着色器代码中定义变量，不过这些变量必须定义在函数体外部。如下：
```c++
//材质所用的纹理、以及缩放偏移量
sampler2D _MainTex;
float4 _MainTex_ST;

//材质的颜色，具体一点是：当纹理为白色图片时，材质的颜色
fixed4 _Color;
```

只要定义了这些公共变量，那么就可以在`C#`程序中使用`Material.Set[Type]`接口来对其进行赋值。很多使用我们希望直接在材质面板上设置这些量，这时候只需要将需要暴露在材质面板的变量，在`Properties`块中重新声明一下，格式为`_Variable("材质面板上显示的名称", Type) = DefaultValue`。材质面板的显示也可以自定义，功能复杂点的需要重写编辑器脚本，简单点的也可以直接在着色器脚本中实现，只需要在`Properties`块中的变量前增加相应的[显示设置](https://docs.unity3d.com/ScriptReference/MaterialPropertyDrawer.html)。一般的来说，`Properties`块中的变量和公共变量是一对一的关系，但是纹理比较特殊，因为纹理数据比较复杂，除了纹理本身的数据外，还有纹理的缩放、偏移等参数。这时候公共变量中的纹理除了要声明纹理本身外，还要声明这些缩放、偏移参数。和纹理相关的参数的命名有一个规则，必须是纹理名称加相关参数的缩写符。如这里的缩放、偏移参数的缩写符就是`_ST`，`S`表示缩放，`T`表示偏移。例如下面例子中的`Properteis`块就和上面的公共变量相对应。
```c++
Properties{
  _Color ("Tint", Color) = (0, 0, 0, 1)
  _MainTex ("Texture", 2D) = "white" {}
}
```

## Spaces?

在着色器中，我们提到位置坐标，就一定会涉及模型、世界、观察、屏幕、裁剪坐标系。这时候，我们说的坐标，必须联系使用场景，来判断当前坐标是处以哪个坐标系。抛开坐标系谈坐标就是无根之木、无水之源。

模型空间坐标系，是以模型为中心，以模型自身为参考的坐标系。`(0,0,0)`在模型坐标系中表示的是模型的原点。如果我们旋转模型，那么模型坐标系也会跟着旋转，换句话说，我们对模型的空间操作，实际上是对模型坐标系的空间操作。我们的模型文件中存储的顶点坐标实际上就是模型空间坐标系的。在渲染时，传入顶点着色器的顶点坐标也是模型空间坐标系上的坐标。

世界空间坐标系，是一个绝对空间坐标系，有一个固定的参考点，不会因为某个局部影响而改变。世界空间坐标系也是所有模型的空间纽带。现实中我们描述我们的位置，大概率使用的就是世界坐标系。

观察坐标系是以摄像机为参考的坐标系。裁剪坐标系是在观察坐标系的基础上，经过投影变换后的坐标系。如果我们使用的是透视投影，那么模型在摄像机上的投影将会产生近大远小的效果。屏幕坐标系是在裁剪坐标系的基础上，进一步除处理得到的，其中需要经过栅格化处理、视口变换等，这一系列操作就是方便后面的渲染。而在片段着色器上处理的便是屏幕坐标系下的数据，因此我们很多时候可以忽略掉观察坐标系、和裁剪坐标系，同时，Unity也提供了很多工具函数来处理坐标系变换。

## Source

所有的教程都有配套源码，可以在教程结尾找到相关链接。因为目前我只是做了一些简单的分析介绍，所有源码在上面已经出现过了，这里直接简单的整理一下。
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
