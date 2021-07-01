---
title: Structure of Shader 
date: 2021-06-25 11:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Structure](https://www.ronja-tutorials.com/post/001-structure/)

## Shader Structure

着色器编程难度较大，在学习初期阶段，我们首先需要学习它的基本结构，以便于后面灵活的修改、应用它们。

现代着色器采用的是可变渲染管线，其中顶点着色器、和片段着色器是其基本组成。除此之外，还有可选部分，几何着色器、曲面细分着色器，但是它们的应用场景比较少。顶点着色器的作用是将模型网格，通过矩阵变换，投影到屏幕(实际是投影到裁剪空间)。同时，顶点着色器的一个非常有用的操作是，执行顶点动画。当顶点坐标变换到屏幕空间后，其所构成的三角面片将被栅格化。为了保证三角面片能够正确显示，从顶点着色器到片段着色器的过程中，需要对顶点进行插值，从而得到三角面片中各个片段的位置、颜色值等。
![](https://www.ronja-tutorials.com/assets/images/posts/001/pipeline.png)

上面简单介绍了着色器的基本流程。接下来我将阐述如何编写着色器、这些“空间坐标系”的具体含义、不同着色器之间的数据传递。但是，我相信了解其基本流程，有助于我们对着色器不同阶段之间的关系有一个直观的理解。因为，在大多数着色器语言中，基本采用了这种基本流程。即便是那些更为高级的、可以通过节点拼接的着色器语言，最终也是将其翻译为这种基本流程。

## ShaderLab

在Unity中，着色器实际上就是一个以`.shader`结尾的文本文件。我们可以在资源面板下，依次选择`Create > Shader >`中的着色器模板，当然模板中的内容可能并不是我们想要的，不过没关系，下面我将介绍如何自定义着色器。为了易于上手，这里我以模板着色器`Create > Shader > Unlit`为参考，编写我们自己的着色器。当然这里我写的和`Unlit`之间最大的区别在于，我们这个是不会处理雾效，同时又增加了一个颜色属性，这样可以从整体对模型颜色进行调整。接下来我也会一步一步的讲解其实现逻辑。本教程目标是着色器小白，所以你如果有哪里不理解的地方，可以告诉我，促使我对其进行调整，以便于后续学习者能够更加顺畅。
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

## Whats ShaderLab?

`Shaderlab`是Unity内置的着色器语言，其中定义了绝大多数渲染模型所需的数据。但是着色器执行渲染逻辑的部分，实际上是采用`hlsl`、`glsl`、`CG`这三种着色语言中的一种。这里`hlsl`是微软开发的底层着色器语言，`glsl`是英伟达开发的底层着色器语言，`CG`是更为高级的作色器语言。而这些执行部分在`Shaderlab`中占有一个独立的区域。具体一点，`Shaderlab`是在执行`hlsl`、`glsl`、`CG`的基础上，扩展了一些属于自身的语法，其中包括`Properties`属性块，用来关联外部输入参数。
![](https://www.ronja-tutorials.com/assets/images/posts/002/LanguageAreas.png)

从上图可以看到，实际`ShaderLab`扩展的仅占着色器很小区域。其中一部分原因是：`Shaderlab`不是可执行语言，而是一种抽象的描述性语言，定义了着色器有哪些输入参数、需要执行哪些渲染操作。而Unity便会识别这些描述性语言，然后将其翻译为GPU可执行的作色器语言，同时关联渲染所需的数据。对于一些简单的渲染需求，参考上图的例子，然后对`CG`部分进行简单的调整就行了。

## Shader/SubShader/Pass

在上图中，你可能也发现了，其中有很多个`{}`花括号，将程序分为很多个块。下面我们来看看，这些块到底是干嘛的。

首先，最外层的`Shader`块，代表了整个作色器。在我们创建材质球后，需要在材质球面板上选择所需的着色器，从而得到我们所需的材质球。那些在材质面板上的着色器名称，实际上就是紧跟在`Shader`块后面的字符串。在这个字符串中，我们可以使用`/`反斜杠来对着色器进行有效的组织分类，这很像我们文件目录的组织形式。在本教程中，我将所有的着色器划分到`Tutorial`这个大类中，其中还会细分出一些小类。当然，这些分类可以根据需要随意改动，只要达到有效组织的目的就行。
可以看到，所谓的着色器，实际上也就是一连串描述的文本文件。需要注意的是，一个文本文件，只能定义一个着色器。但是，一个着色器却可以复用另一个着色器的功能。例如，在`Shader`块中，也就是最外层花括号中，我们可以定义`fallback shader`，当Unity将其翻译为更底层的着色器代码时，会将`fallback shader`中的`SubShader`块复制过来。

在`Shader`块中，可以定义多个`SubShader`块。在模型渲染时，只会从中选择一个`SubShader`来执行，而具体选择哪个，依赖于实际运行的平台。可惜的是，关于如何定义`SubShader`的说明文档极其匮乏，根据我多年的经验，在很多情况下，一个`Shader`块中只定义一个`SubShader`能满足基本需求，减少很多不必要的麻烦。凡事皆有特例，当我们想实现阴影效果时，需要在当前`SubShader`块中实现相应的`ShadowPass`，每次都实现一遍很麻烦。因为阴影着色流程基本固定，所以Unity提供的现成的便可以使用，这时候，我们可以使用包含阴影着色逻辑的`fallback shader`，当渲染时，从该`SubShader`中未找到可以使用的`ShadowPadd`时，便会从`fallback shader`中去查找。大多数情况下，我们使用`VertexLit`着色器，来作为我们的`fallback shader`，因为`VertexLit`中的逻辑简单、性能消耗低、基本上能够兼容所有的显卡，也可能是大家相互Copy，从而形成`VertexLit`流行的假象:-)。另外，在`SubShader`块中，我们可以定义[Subshader tags](https://docs.unity3d.com/Manual/SL-SubShaderTags.html)；还可以定义多个`Pass`，例如前面说的`Shadow pass`；以及属性，在`SubShader`块中定义的属性是由所有`Pass`共享的。

一个`Pass`包含一套完整的渲染流程，从底层着色语言的角度来看，一个`Pass`才是实际上的着色器，它将模型数据转换为五彩斑斓的画面。在内置渲染管线中，如果我们在`SubShader`块中定义了多个`Pass`，当该`SubShader`被平台选定时，其中的`Pass`将会被一个接着一个的执行(而最新的URP渲染管线，目前仅支持单个光照`Pass`)。对于具有多个`Pass`的`SubShader`，我们可以将其公共属性等数据定义在`SubShader`中，而`Pass`中定义一些独有的数据或逻辑。

## Properties and Tags

你们可能注意到，在上图中还有两个块`Properties`、`Tags`未被提及，那我们继续吧。

在很多编程语言中有字典的概念，顾名思义，就是类似汉语字典一样，可以通过拼音、笔画等关键信息进行快速检索。这里的`Tags`就可以类比到字典，在`Tags`中可以定义多个关键字，以及关键字所对应的值，它们公共构成了着色器的配置参数。其中关键字表示了参数的类型，值表示了参数的实际设置。在`SubShader`中，可以通过`Tags`定义着色器的材质表现、渲染顺序、以及其他操作；而`Pass`中的`Tags`主要定义了光照模式。详细的说明可以参考[SubShader Tags](https://docs.unity3d.com/Manual/SL-SubShaderTags.html)和[Pass Tags](https://docs.unity3d.com/Manual/SL-PassTags.html)。

`Properties`属性块，主要用来定义材质面板中的属性显示。通过材质面板来调整材质参数具有一定的局限性，因为整个调整是以材质球为单位，也就是说，如果多个模型使用同一个材质球，那么就无法做到材质差异化。这时候需要创建多个材质球，分别对应于不同的模型。在接下来得教程中我也会详细讨论`Properties`的使用。

下面我们对`Shaderlab`的基本结构进行一个总结：
```c++
Shader "Category/Name"{
	Properties{
		//用于材质面板显示、与配置的属性
	}
	Subshader{
		Tags{
			//一些公共配置，涉及渲染类型、渲染顺序等设置
		}

		//公共设置、属性、方法可以写在Subshader中

		Pass{
			Tags{
				//主要是光照模式的配置
			}

			//单个Pass的设置， 例如剔除、模板等

			CGPROGRAM
			//实际执行的渲染程序、以及所用到的属性参数
			ENDCG
		}
	}
}
```

## Source

所有的教程都有配套源码，可以在教程结尾找到相关链接。因为目前我只是做了一些简单的分析介绍，所有源码在上面已经出现过了，这里直接简单的整理一下。
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/001-004_basic_unlit/basic_unlit.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/001-004_basic_unlit/basic_unlit.shader)

```c++
Shader "Tutorial/001-004_Basic_Unlit"{
	//这些值将会显示在材质面板上
	Properties{
//	_Color ("Tint", Color) = (0, 0, 0, 1)
//	_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		//当前的标签设置表示的是：不透明渲染，和其他不透明物体处以同一渲染队列
		Tags{ "RenderType"="Opaque" "Queue"="Geometry" }

		Pass{
			CGPROGRAM
//
//		//这里包括一些工具函数、以及内置变量
//		#include "UnityCG.cginc"
//
//		//定义顶点、片段着色函数
//		#pragma vertex vert
//		#pragma fragment frag
//
//		//材质所用的纹理、以及缩放偏移量
//		sampler2D _MainTex;
//		float4 _MainTex_ST;
//
//		//材质的颜色，具体一点是：当纹理为白色图片时，材质的颜色
//		fixed4 _Color;
//
//		//模型网格、UI等输入数据，基本代表了模型内在属性
//		struct appdata{
//			float4 vertex : POSITION;
//			float2 uv : TEXCOORD0;
//		};
//
//		//当经过顶点着色器处理后，处理的数据经过栅格化插值，然后传入片段着色器
//		struct v2f{
//			float4 position : SV_POSITION;
//			float2 uv : TEXCOORD0;
//		};
//
//		//顶点着色器函数，主要执行坐标的空间变换
//		v2f vert(appdata v){
//			v2f o;
//			//将模型各顶点坐标，转换到裁剪空间
//			o.position = UnityObjectToClipPos(v.vertex);
//			//基于图片的缩放偏移参数，对UV进行变换
//			o.uv = TRANSFORM_TEX(v.uv, _MainTex);
//			return o;
//		}
//
//		//片段作色器，主要是计算像素点的颜色
//		fixed4 frag(v2f i) : SV_TARGET{
//			//基于UV坐标，进行纹理采样
//			fixed4 col = tex2D(_MainTex, i.uv);
//			//将纹理颜色和材质颜色相乘
//			col *= _Color;
//			//返回最终的像素点颜色
//			return col;
//
//		}
			ENDCG
		}
	}
	Fallback "VertexLit"
}
```

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!