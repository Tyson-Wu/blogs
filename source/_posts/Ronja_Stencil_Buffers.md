---
title: Stencil Buffers
date: 2021-07-07 20:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Stencil Buffers](https://www.ronja-tutorials.com/post/022-stencil-buffers/)

## Summary

深度缓存可以帮助我们对比模型之间的深度关系，确保模型之间正确遮挡。还有另一部分缓存用于模板操作，这个缓存叫做模板缓存。模板缓存就像一个印刷版，只有部分区域允许渲染到屏幕上。

Unity也有用到模板缓存来实现延迟渲染，所以如果你在执行延迟渲染时，会有一些限制。你可以阅读[官方文档](https://docs.unity3d.com/Manual/SL-Stencil.html)去了解这些具体的限制，深入了解如何使用模板。

本教程将会介绍模板缓存的基本使用，包括模板读写操作。这里也从[表面着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)中的着色器脚本开始，来实现模板缓存案例。当然使用方法适用于其他着色器，包括后处理操作。
![](https://www.ronja-tutorials.com/assets/images/posts/022/Result.gif)

## Reading form the Stencil Buffer

在使用模板的着色器中，着色器会读取模板中的值，然后以这个值按照一定条件来进行判断当前像素是否可以写入到帧缓存中，如果可以，那么再按一定条件刷新当前模板缓存值，如果不行，那么放弃后面所有操作。

所有的模板操作都是集中在一个叫做`Stencil`的块中。
```c++
SubShader {
    Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

    Stencil{
        //模板操作
    }

    //表面着色器代码

    //...
}
```

在模板参数中最重要的是`Ref`，这个参数是我们模板操作的参考值。在模板写入之前，模板缓存中的默认值是0。一般在所以操作之前我都会手动初始化模板缓存为0，这样可以让代码看起来逻辑更清晰。

下一个参数叫做`Comp`，定义了模板比较方法，什么情况下可以通过模板，什么时候不行，其默认值为`Always`，表示所有都无条件通过。在本文实现的着色器中，我们使用`Equal`这个比较方法，这意味着只有模板值等于参考值时，才能通过模板。
```c++
Stencil{
    Ref 0
    Comp Equal
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/022/NormalMaterial.png)

上面的模板设置并不会影响原本模型的显示，这是因为模板初始值为0，而参考值也为0，刚好所有值都通过模板。如果我们将参考值改为其他值，这时候模型会消失，因为所有模板值都未通过。
```c++
Stencil{
    Ref 1
    Comp Equal
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/022/Invisible.png)

为了方便后面调整，这里我将模板参考值暴露在材质面板上，这样我们可以通过材质面板来修改模板参考值。这里`IntRange`表示我们的滑块刻度是取整的。
```c++
Properties {
    _Color ("Tint", Color) = (0, 0, 0, 1)
    _MainTex ("Texture", 2D) = "white" {}
    _Smoothness ("Smoothness", Range(0, 1)) = 0
    _Metallic ("Metalness", Range(0, 1)) = 0
    [HDR] _Emission ("Emission", color) = (0,0,0)

    [IntRange] _StencilRef ("Stencil Reference Value", Range(0,255)) = 0
}
```

然后我们将模板操作中的参考值改为`_StancilRef`，这里将其放在中括号里面，表示我们这个值是属性块中的值，着色器会进行关联。这样修改之后好像我们的模型也还是只有显示我不显示两个状态，但是，使用材质面板上的滑块可以在两者之间进行快速切换。

## Writing to the Stencil Buffer

在实际应用中，我们除了需要根据模板来绝对哪些需要渲染，哪些不渲染，还要有能够向模板中写入新的数值得着色器。上面实现的是读取模板的着色器，下面我们实现第二个写入模板的着色器。第二个着色器的主要功能是对模板进行操作，所以不需要写入到帧缓存。这样在第一个着色器执行的时候，就可以使用第二个着色器写入的模板值进行渲染判断。

第二个着色器我们使用最[简单的着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)，因为我们只想对模板进行操作，不打算渲染其他东西。

那么对于第二个写入模板的着色器，我们将其片段着色器的返回值直接写为0，因为我们不想渲染它。然后设置混合模式为`Zero One`，这表示不会影响之前绘制好的像素。还有就是需要关闭深度写入功能，因为这个模型不渲染，说过写入深度的话，那么可能会遮挡后面的模型，这就会显得很诡异。最后是要保证第二个着色器比第一个着色器先执行，也就是先写后读，我们可以设置渲染队列顺序来实现。
同时我们还删除颜色变量，因为我们不需要。
```c++
fixed4 frag(v2f i) : SV_TARGET{
    return 0;
}
```
```c++
Blend Zero One
ZWrite Off
```
```c++
"Queue"="Geometry-1"
```
```c++
//清空材质面板
Properties{

}
```
![](https://www.ronja-tutorials.com/assets/images/posts/022/Invisible.png)

这样我们实现了一个完全不显示的着色器，这个着色器相比第一个着色器的优势是，它完全不受模板值得影响。因为不管模板值什么，它都不会显示出来。

然后我们将第一个着色器中的模板设置拷贝过来。然后将比较方法设置为`Always`，这表第二个着色器无条件通过模板。然后在加入一个`Pass`属性，它定义了当通过深度检测后，模板值将会如何。这里我们将它设为`Replace`，这表示当深度检测通过后，使用参考值替代原本的模板值。这里还有一个`Fail`的属性，是当检测失败后应该执行什么操作，默认是不做任何操作。
```c++
Stencil{
    Ref [_StencilRef]
    Comp Always
    Pass Replace
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/022/SphereAndQuad.png)

现在我们可以看到，当它们的参考值相同时，第一个材质物体和第二个材质物体重叠部分可见。

实现这两个着色器的过程中，我们已经知道了模板的基本用法。如果你想了解更多，你可以参考Unity的[官方文档](https://docs.unity3d.com/Manual/SL-Stencil.html)。
![](https://www.ronja-tutorials.com/assets/images/posts/022/WrongStencil.png)

在不断尝试后，我遇到一个问题。当有多个模板在对同一个像素点进行读写操作时，后面的(离摄像机更远)模板可能比前一个模板更晚执行，这样可能会覆盖原先的模板值。如果你也遇到类似的问题，那么你可以通过调整它们的渲染队列来保证它们的执行顺序。Unity中，当渲染队列值大于2500时，模型是从后往前渲染的。这样做的目的是为了保证透明物体正确渲染。所以我们同样可以通过渲染队列来控制模板的顺序。在我的例子中，我使用2501来作为写模板队列，而2052最为读渲染队列，这样保证写模板在读模板之前执行。还有一点就是我们的模板队列不要超过3000，因为超过3000为半透明物体的可用渲染队列值，它们的队列值混在一起可能会出问题。
![](https://www.ronja-tutorials.com/assets/images/posts/022/ReadInspector.png)
![](https://www.ronja-tutorials.com/assets/images/posts/022/Result.gif)

## Source

```c++
Shader "Tutorial/022_stencil_buffer/read" {
	Properties {
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
		_Smoothness ("Smoothness", Range(0, 1)) = 0
		_Metallic ("Metalness", Range(0, 1)) = 0
		[HDR] _Emission ("Emission", color) = (0,0,0)

		[IntRange] _StencilRef ("Stencil Reference Value", Range(0,255)) = 0
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        //模板操作
		Stencil{
			Ref [_StencilRef]
			Comp Equal
		}

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
```c++
Shader "Tutorial/022_stencil_buffer/write"{
	//材质面板
	Properties{
		[IntRange] _StencilRef ("Stencil Reference Value", Range(0,255)) = 0
	}

	SubShader{
		//将队列值设在读模板之前
		Tags{ "RenderType"="Opaque" "Queue"="Geometry-1"}

        //模板操作
		Stencil{
			Ref [_StencilRef]
			Comp Always
			Pass Replace
		}

		Pass{
            //不写入帧缓存、也不写入深入，只负责写入模板
			Blend Zero One
			ZWrite Off

			CGPROGRAM
			#include "UnityCG.cginc"

			#pragma vertex vert
			#pragma fragment frag

			struct appdata{
				float4 vertex : POSITION;
			};

			struct v2f{
				float4 position : SV_POSITION;
			};

			v2f vert(appdata v){
				v2f o;
				//计算裁剪坐标
				o.position = UnityObjectToClipPos(v.vertex);
				return o;
			}

			fixed4 frag(v2f i) : SV_TARGET{
				return 0;
			}

			ENDCG
		}
	}
}
```

希望通过本篇教程让你了解模板的使用。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/022_Stencil_Buffer/stencil_read.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/022_Stencil_Buffer/stencil_read.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/022_Stencil_Buffer/stencil_write.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/022_Stencil_Buffer/stencil_write.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
