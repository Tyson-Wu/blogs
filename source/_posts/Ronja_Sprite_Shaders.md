---
title: Sprite Shaders
date: 2021-07-01 15:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Sprite Shaders](https://www.ronja-tutorials.com/post/007-sprite-shaders/)

## Summary 

在Unity中，图片的渲染和三维模型渲染非常相似。并且提供了`SpriteRender`的组件来实现图片渲染的功能。接下来我也会介绍这个组件、及其实现原理，然后通过我们的着色器来模拟这个组件的功能。

这个教程依赖于上一个[透明着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Transparency/)的教程。
![](https://www.ronja-tutorials.com/assets/images/posts/007/Result.png)

## Scene Setup

这里采用一个非常简单的场景来实验我们的图片着色器。首先，我将摄像机改为正交模式，并且将原先的小方块改成`sprite renderer`，同时将后面用到的图片格式改为`sprite`。
![](https://www.ronja-tutorials.com/assets/images/posts/007/Hierarchy.png)
![](https://www.ronja-tutorials.com/assets/images/posts/007/CameraInspector.png)
![](https://www.ronja-tutorials.com/assets/images/posts/007/SpriteInspector.png)
![](https://www.ronja-tutorials.com/assets/images/posts/007/SpriteImporter.png)

## Changing the Shader

在完成以上修改后，将上一章的透明材质放到`SpriteRenderer`组件的材质属性上，一切准备就绪！

`SpriteRenderer`组件会自动根据附加的图片自动生成一个网格数据，包括顶点、UV等。然后和普通的三维模型一样渲染到场景中。同时在`SpriteRenderer`上定义的颜色也会以顶点颜色的形式包含在网格数据中。另外还可以设置图片是否翻转。而且图片渲染和半透明渲染一样，是排在不透明渲染之后，甚至大多数渲染之后，而这个渲染队列是由`SpriteRenderer`自动帮我们设置的。

因为目前我们的半透明着色器还不支持翻转、和顶点颜色。所以接下来我们把这个添加上。

但是当我们翻转后，会发现图片不见了，然后再翻转，图片又出现了。这时因为从性能优化角度考虑，渲染单面比渲染双面更节约性能，这里叫做`backface culling`背面剔除。普通的不透明模型渲染使用背面剔除非常有效，因为其内部不可能被渲染。另外如果不使用背面剔除，在光照计算的时候会出现怪异的效果，因为参与计算的法向量只会指向正面方向，因此背面计算的结果无效。

在这里我们并不用考虑这个优化项，因为图片、半透明物体之类的并不存在哪个面朝内、哪个面朝外的问题，也不需要参与光照计算。因此我们这里直接关闭背面剔除公共，关闭的设置和之前混合模式、深度写入设置类似。
```c++
Cull Off
```

为了在最终的渲染中使用到前面说的顶点颜色数据。我们需要在半透明着色其中做一些修改。首先在输入数据结构和插值数据结构中加入一个四维向量，并标记为颜色类型。然后在顶点着色器中将输入数据结构中的顶点颜色传递给插值数据结构中的颜色值。最终在片段着色器中叠加到输出颜色上。
```c++
struct appdata{
    float4 vertex : POSITION;
    float2 uv : TEXCOORD0;
    fixed4 color : COLOR;
};

struct v2f{
    float4 position : SV_POSITION;
    float2 uv : TEXCOORD0;
    fixed4 color : COLOR;
};

v2f vert(appdata v){
    v2f o;
    o.position = UnityObjectToClipPos(v.vertex);
    o.uv = TRANSFORM_TEX(v.uv, _MainTex);
    o.color = v.color;
    return o;
}

fixed4 frag(v2f i) : SV_TARGET{
    fixed4 col = tex2D(_MainTex, i.uv);
    col *= _Color;
    col *= i.color;
    return col;
}
```

修改后的半透明材质就可以应用到`SpriteRenderer`组件上了，并且可以响应`SpriteRenderer`上的一些设置操作。在后面的教程中我们还可以在此基础上扩展出更加绚丽的效果。
![](https://www.ronja-tutorials.com/assets/images/posts/007/AdjustVariables.gif)
```c++
Shader "Tutorial/007_Sprite"{
	Properties{
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		Tags{ 
			"RenderType"="Transparent" 
			"Queue"="Transparent"
		}

		Blend SrcAlpha OneMinusSrcAlpha

		ZWrite off
		Cull off

		Pass{

			CGPROGRAM

			#include "UnityCG.cginc"

			#pragma vertex vert
			#pragma fragment frag

			sampler2D _MainTex;
			float4 _MainTex_ST;

			fixed4 _Color;

			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
				fixed4 color : COLOR;
			};

			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
				fixed4 color : COLOR;
			};

			v2f vert(appdata v){
				v2f o;
				o.position = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				o.color = v.color;
				return o;
			}

			fixed4 frag(v2f i) : SV_TARGET{
				fixed4 col = tex2D(_MainTex, i.uv);
				col *= _Color;
				col *= i.color;
				return col;
			}

			ENDCG
		}
	}
}
```

`SpriteRenderer`组件还会提供各种网格数据来实现图标、多边形图形、动画效果。

相比于Unity内置的`Sprite Shader`，我们目前还没有实现的功能有`instancing`、像素捕捉、`alpha`通道扩展，因为这些功能比较复杂，而且很少使用，因此我现在不做介绍。

你可以在以下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/007_Sprites/sprite.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/007_Sprites/sprite.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!



