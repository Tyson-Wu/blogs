---
title: Surface Shader Basics
date: 2021-07-01 13:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Surface Shader Basics](https://www.ronja-tutorials.com/post/005-simple-surface/)

## Summary

在Unity中，我们可以创建一个空的着色器脚本，然后手动一行一行的去实现整个脚本。当然这种方式比较费时，毕竟着色器有一套固定的流程，因此可以将一部分代码复用，例如光照模型。如果能够直接配置一些参数就能自动生成相关代码，你一定会喜欢吧。Unity就是这么会投其所好，它实现了一种名叫表面着色器的东东，刚好能够满足咱懒人的需求。懒-是推动科技发展的第一生产力，至理名言啊！

那什么是表面作色器呢？在进入正题之前，我建议你先了解最简单的无光照的着色器，如果你不清楚，可以参考我上一个[教程](/blogs/2021/07/01/Ronja_Structure/)。
![](https://www.ronja-tutorials.com/assets/images/posts/005/Result.png)

## Conversion to simple Surface Shader

相比于前面介绍的着色器实现方法，表面着色器的实现就显得更加简洁，原先需要处理的很多内容都可以剔除，因为Unity会自动帮我们生成相关代码。以上一个教程的着色器脚本为例，如果我们要用表面着色器来实现，那么前面提到的什么顶点着色器都可以不要了。与之相对应的宏命令也可以删除，两者之间的插值数据也可以不要了。甚至是`UnityCG.cginc`文件也可以不要，还有`MainTex_ST`等等。这些代码最终都会由Unity自动生成。一顿大刀阔斧，来看看我们的成果吧：
```c++
Shader "Tutorial/005_surface" {
	Properties {
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		sampler2D _MainTex;
		fixed4 _Color;

		fixed4 frag (v2f i) : SV_TARGET {
			fixed4 col = tex2D(_MainTex, i.uv);
			col *= _Color;
			return col;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

简洁的令人发指！等等，先别急着感叹，还有事没做完。上面的着色器脚本并不能执行，因为表面着色器有自己的一些要求。

首先，我们需要添加一个新的数据类型作为片段着色器的输入。这个数据类型将会包含所有与片段着色相关的必要数据。当然，我们这个简单的不能再简单的着色器只需要传递UV坐标。UV坐标还是二维向量。但是，这里的UV变量命名有特殊的规则。因为UV坐标是用来纹理采样的，上一章讲过UV变换，每一个纹理都有自己的缩放、偏移参数，所以必须将变换后的UV和对应的纹理相关联。这里采用命名规则来实现，首先UV变量必须以`uv`开头，然后后面跟随纹理变量的名称。这样，在后面自动生成代码的时候，程序就知道谁和谁配对了。
```c++
struct Input {
	float2 uv_MainTex;//此时配对的纹理是 _MainTex
};
```

接下来我们要对之前的片段着色其进行修改，使其编程表面着色器。为了区分两者，先把函数名改为`surf`。然后表面着色器函数是没有返回值的，所以函数返回类型改为`void`。

然后再拓展出两个参数。第一个参数正是我们刚刚定义好的`Input`结构，通过这个参数，表面着色器可以获取所有相关的必要参数；第二个参数是一个叫做`SurfaceOutputStandard`的结构体，从字面意思可以看出，这就是表面着色器的最终输出数据。当然，在函数结束之前，必须计算好、并赋值所有需要外传的参数。除了`SurfaceOutputStandard`结构外，Unity还定义了其他类似的结构，它们之间的区别在于适用于不同的光照模型。而这些结构中的成员变量就包含了后续光照处理所必须的数据，这些数据的具体含义将在后面介绍。

然后是删除`SV_Target`语义标识符，因为我们的`surf`函数的返回类型是`void`。

最后是把`return`语句删除。然后将计算所得的数据通过`SurfaceOutputStandard`传递出去。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
	fixed4 col = tex2D(_MainTex, i.uv_MainTex);
	col *= _Color;
	o.Albedo = col.rgb;
}
```

最后一步，我们需要引入光照模型，同时将写好的表面着色器函数与表面着色器相关联，这个和顶点着色器类似，都是通过`#pragma`来实现。因为我们这里的`surf`的输出结构是`SurfaceOutputStandard`，意味着我们使用标准的光照模型。所以在其后我们还得加上`Standard`来说明所用的光照模型，最后的`fullforwordshadows`是告诉Unity使用前向阴影。
```c++
Shader "Tutorial/005_surface" {
	Properties {
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows

		sampler2D _MainTex;
		fixed4 _Color;

		struct Input {
			float2 uv_MainTex;
		};

		void surf (Input i, inout SurfaceOutputStandard o) {
			fixed4 col = tex2D(_MainTex, i.uv_MainTex);
			col *= _Color;
			o.Albedo = col.rgb;
		}
		ENDCG
	}
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/005/SimpleAlbedo.png)

## Standard Lighting Properties






