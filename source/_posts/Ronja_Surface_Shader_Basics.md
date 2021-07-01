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

那什么是表面作色器呢？在进入正题之前，我建议你先了解最简单的无光照的着色器，如果你不清楚，可以参考我上一个[教程](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)。
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

在`SurfaceOutputStandard`结构中包含很多材质相关的属性参数，具体如下：

- Albedo : 表示材质表面的颜色。而材质最终呈现的颜色受光照的影响，因为材质本身只能决定对吸收哪些波段的光。
- Normal : 表示材质表面法向向量。法向向量一般和顶点坐标一起，存储在模型数据中，这时候法向向量所在的坐标系是切向空间。什么是切向空间呢？可以简单的理解为沿着模型表面的坐标系。而法向是指垂直与模型表面的方向。法向向量一般要变换到世界坐标系再参与计算。
- Emission ：表示材质的自发光特性。一般的的模型渲染依赖于外部光源，如果关闭外部光源，那么模型表现为黑色。而具备自发光材质的模型即便没有外部光源也能显示出原本的颜色。一般场景中的灯具、或者熔岩会使用自发光属性。
- Metallic ：表示材质的金属特性。现实中，金属材料和非金属材料的光学特性不一样，例如即便黑色的金属也能在阳光下反射光芒，但是黑色的非金属就表现的黯淡无光。
- Smoothness : 表示材质的光滑度。光滑程度主要决定漫反射、和镜面反射之间的权重分布。玻璃的光滑度非常高，所以可以用来做镜子，而一般木材非常粗糙。
- Occlusion ：表示环境遮罩特性。举个例子，平坦的桌面上，光线能够达到每个角落。而崎岖不平的背包上，那些深深的褶皱显得格外阴暗。这些因为表面相互遮挡而产生的阴影就是我们这里的环境遮罩效果。
- Alpha : 表示材质的透明度。从字面可以很好理解，有些材质是透明的，如玻璃，有些不是，如木头。

上面提到的这些参数前三个是向量，后四个是标量。这些参数有些可以直接暴露在材质面板，方便美术编辑材质效果。

## Implement a few Lighting Properties

接下来我们把`emission`、`metallic`、`smoothness`三个参数为例，将其作为材质的可调参数。当然你也可以根据实际需求做调整。

首先，我们定义两个公共变量：光滑度和金属度。它们的类型我选择`half`。一般来说除了坐标采用`float`，其他的都选`half`类型。
```c++
half _Smoothness;
half _Metallic;
```

然后将这些公共变量添加到`Properties`块中，将其暴露在材质面板上。但是材质面板并不知道所显示的变量的类型，所以还需要在其名称后增加类型说明，如下所示：
```c++
Properties {
	_Color ("Tint", Color) = (0, 0, 0, 1)
	_MainTex ("Texture", 2D) = "white" {}
	_Smoothness ("Smoothness", float) = 0
	_Metallic ("Metalness", float) = 0
}
```

在`surf`函数中，我们可以直接将这些公共变量传递给`SurfaceOutputStandard`结构体，这样在后续的光照计算中，就可以使用这些参数了，同时我们修改材质面板上的值，便能立即改变材质的表现效果。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
	fixed4 col = tex2D(_MainTex, i.uv_MainTex);
	col *= _Color;
	o.Albedo = col.rgb;
	o.Metallic = _Metallic;
	o.Smoothness = _Smoothness;
}
```

目前为止，表面着色器已经基本完成了。但是还有些需要完善的地方，因为我们在材质面板上修改参数时，很容易设置到非法值，最终导致材质显示异常。我们可以在`Properties`做些小修改，将`float`修改为`Range(0,1)`就可以将这些参数限定在一个有效范围内。
```c++
Properties {
	_Color ("Tint", Color) = (0, 0, 0, 1)
	_MainTex ("Texture", 2D) = "white" {}
	_Smoothness ("Smoothness", Range(0, 1)) = 0
	_Metallic ("Metalness", Range(0, 1)) = 0
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/005/Inspector.png)

接下来我们添加自发光颜色。同样的我们也需要在公共变量、和`Properties`中加入`_Emission`的定义。
```c++
// ...

_Emission ("Emission", Color) = (0,0,0,1)

// ...

half3 _Emission;

// ...

o.Emission = _Emission;
```
![](https://www.ronja-tutorials.com/assets/images/posts/005/Emissive.png)

为了避免自发光材质过度曝光，我们只能将自发光颜色限定在`[0-1]`之间。当然，如果我们将自发光颜色定义为HDR类型的话，就可以不用担心过曝的问题了。如果我们使用纹理来将材质各个部位的自发光特性差别化，那么可以产生很炫的效果。例如怪兽的眼睛放光的效果。
```c++
[HDR] _Emission ("Emission", Color) = (0,0,0,1)
```
![](https://www.ronja-tutorials.com/assets/images/posts/005/HdrInspector.png)

## Minor Improvements

最后，让我们做一些小改动，让我们的材质看起来更自然。首先，我们在着色器脚本最后面加上`fallback shader`，这样我们可以复用其他着色器中的代码。这里我将标准着色器作为我们的`fallback shader`，然后复用其中的阴影渲染部分`shadow pass`的代码，这样就可以让我们的材质也表现出阴影效果。我们使用`fullfowardshadows`参数进而获得很好的阴影效果。另外我们也可以指定当前着色器的目标平台，例如设置`target`为3.0。其中`target`的值越高，表示可以使用的特性越多，但是支持的硬件平台会越少。这里`target`为3.0，已经可以使用大多数的高级特性了，所以能够实现更好的光照效果。
```c++
Shader "Tutorial/005_surface" {
	Properties {
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
		_Smoothness ("Smoothness", Range(0, 1)) = 0
		_Metallic ("Metalness", Range(0, 1)) = 0
		[HDR] _Emission ("Emission", color) = (0,0,0)
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		sampler2D _MainTex;
		fixed4 _Color;

		half _Smoothness;
		half _Metallic;
		half3 _Emission;

		struct Input {
			float2 uv_MainTex;
		};

		void surf (Input i, inout SurfaceOutputStandard o) {
			fixed4 col = tex2D(_MainTex, i.uv_MainTex);
			col *= _Color;
			o.Albedo = col.rgb;
			o.Metallic = _Metallic;
			o.Smoothness = _Smoothness;
			o.Emission = _Emission;
		}
		ENDCG
	}
	FallBack "Standard"
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/005/Result.png)

希望本章的介绍能让你有所收获！

你可以在一下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/005_Surface_Basics/simple_surface.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/005_Surface_Basics/simple_surface.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!

