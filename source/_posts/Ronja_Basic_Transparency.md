---
title: Basic Transparency 
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
[Basic Transparency](https://www.ronja-tutorials.com/post/006-simple-transparency/)

## Summary

对于不透明物体的渲染，是直接将计算所得到的像素颜色覆盖掉原有的颜色值。而透明物体的渲染方法恰恰相反，是保留原有颜色值，然后两者通过混合后，达到一种看是半透明的效果。为了阐明关键思想，这里以最简单的不受光材质为例。

如果你还不知道如果编写着色器，这里建议你先阅读我之前的[教程](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)
![](https://www.ronja-tutorials.com/assets/images/posts/006/SemitransparentCube.png)

为了达到正确的渲染效果，首先我们需要告诉Unity：我们打算渲染一个透明物体。因此，我们可以将`SubShader`块中的`Tags`块的渲染类型改为`Transparent`，以及将其中的渲染队列改为`Transparent`。渲染队列的设置是为了保证我们的半透明物体一定是在不透明物体之后渲染。如果不这么设置，根据上面说到的不透明渲染的方法，很可能有不透明物体直接覆盖在半透明物体区域，即便我们的半透明物体在不透明物体的前方。
```c++
Tags{ "RenderType"="Transparent" "Queue"="Transparent"}
```

接下来我们要定义混合模式，前面提到需要将半透明的像素颜色和已经渲染的像素颜色相混合，就是通过这个混合模式来决定的。混合模式的定义由两个关键字构成，第一个和半透明颜色相乘，第二个和已有的颜色相乘，然后两者相加。

当我们渲染不透明物体时，我们将第一个参数设为1，第二个参数设为0，这样新的颜色值便会完全替换掉原先的颜色值。而在透明材质中，我们通常使用新颜色的透明通道作为第一个参数，而第二个参数为1减该透明通道值，这样我们就可以调整其透明通道来实现不同程度的透明效果。

混合模式可以定义在`SubShader`块中，也可以定义在`Pass`块中，位置的不同决定其作用域的大小。但是必须定义在`hlsl`代码以外，因为这属于`Shaderlab`拓展的特性。
```c++
Blend SrcAlpha OneMinusSrcAlpha
```

你可以从下面网址找到更全面的混合模式设置介绍：[https://docs.unity3d.com/Manual/SL-Blend.html](https://docs.unity3d.com/Manual/SL-Blend.html)

为了突出重点，精简逻辑，这里举两个小例子：

- 当我们的片段着色器返回的透明通道值为`0.5`，那么根据前面设置的混合模式，将会将新的颜色值得一半和旧的颜色值得一一半进行混合。如果新颜色是白色，旧颜色是黑色，那么混合后的将是灰色；
- 当我们的片段着色器返回的透明通道值为`0.9`，那么混合后，新颜色将占有90%的比例，而旧颜色只占有%10；

因此，在前面不透明着色器脚本的基础上，做以上调整，该着色器就成功变成一个半透明着色器。因为公共变量`_Color`参与了片段着色器中的颜色计算，所以我们在材质面板修改其透明度，将会影响最终的混合效果。如下：
![](https://www.ronja-tutorials.com/assets/images/posts/006/AdjustTint.gif)

另一个需要修改的地方是关闭透明着色器的深度写入功能。一般来说，当模型被渲染到画面上是，其距离摄像机的深度信息会被记录到一张深度纹理是上。然后后面渲染的模型只需要与这张深度图相比较，就可以正确判断其相互之间的遮挡关系，其中被遮挡的部分将不会渲染到画面上。但是这并不适用于半透明物体，因为即便被遮挡也会被渲染。所以我们只能从其渲染队列方面考虑，将半透明物体限定在不透明物体之后渲染，并且半透明物体之间的渲染排序是由远及近。深度值写入的设置如下，和混合模式设置一样，都可以定义在`SubShader`或`Pass`块中。
```c++
Blend SrcAlpha OneMinusSrcAlpha
ZWrite Off
```

如果我们的纹理贴图也存在半透明通道，那么最终渲染出来的效果将是各个部位的透明程度有差异。
![](https://www.ronja-tutorials.com/assets/images/posts/006/TextureTransparentCube.png)

```c++
Shader "Tutorial/006_Basic_Transparency"{
	Properties{
		_Color ("Tint", Color) = (0, 0, 0, 1)
		_MainTex ("Texture", 2D) = "white" {}
	}

	SubShader{
		Tags{ "RenderType"="Transparent" "Queue"="Transparent"}

		Blend SrcAlpha OneMinusSrcAlpha
		ZWrite off

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
			};

			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			v2f vert(appdata v){
				v2f o;
				o.position = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return o;
			}

			fixed4 frag(v2f i) : SV_TARGET{
				fixed4 col = tex2D(_MainTex, i.uv);
				col *= _Color;
				return col;
			}

			ENDCG
		}
	}
}
```
你可以在一下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/006_Transparency/transparent.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/006_Transparency/transparent.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!