---
title: Partial Derivatives (fwidth)
date: 2021-07-08 20:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Partial Derivatives (fwidth)](https://www.ronja-tutorials.com/post/046-fwidth/)

`ddx`、`ddy`、`fwidth`这三个偏导函数，我们平时很少用到，而且刚接触的时候很难理解。但是我却很喜欢它们，因为我觉得有很多适合它们直接使用的场景。所以在这里我也想向你们介绍它们。因此下面主要是针对函数进行讲解，所以并不要求你对渲染有多深的了解。但是你还需要掌握一些基本的渲染相关的知识，所以如果你完全没有基础，建议你先从[着色器基础](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)看起。
![](https://www.ronja-tutorials.com/assets/images/posts/046/fire.gif)

## DDX and DDY

"导数“的意义是表示函数在某个点的变化情况。使用导数的概念，我们可以计算屏幕上任意点与临近点之间的变化情况。在上面三个函数中，`ddx`、和`ddy`是最简单的两个。它们分别用来计算水平、和垂直两个方向的相邻两个像素点之间的变化情况。这个计算过程并不涉及到什么复杂大函数，也没有说需要什么复杂的GPU结构支持，它就是一个简单的、像素之间的差值计算。但是，有一点你必须清楚，当我们计算某个像素点的导数时，并不是单独地、针对每一个像素都计算一遍。而是将`2x2`的像素块组成一个独立的处理单元，我们的片段着色器也是以像素块为单元，进行并行处理的。可以这么理解，片段着色器一次性串行的执行像素块中的四个像素的逻辑，所以一个片段着色器可以同时访问四个像素值，因此可以一次性计算出它们的偏导。其中`ddx`函数是计算水平方向的两个相邻像素的差值，`ddy`是计算垂直方向的两个相邻像素的差值。这也意味着，同一个像素块中，相邻的两个像素的`ddx`、或`ddy`相同。
![](https://www.ronja-tutorials.com/assets/images/posts/046/ddx_ddy.png)

这里我实现一个简单的着色器来做个测试。在片段着色器中传入了uv坐标，然后我们使用`ddx`来计算水平方向相邻两个像素`u`坐标的变化情况，然后再乘以一个缩放因子便于观察，然后将结果以颜色值得形式显示在屏幕上。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
  //计算uv坐标中的u值在水平相邻像素之间的差异
  float derivative = ddx(i.uv.x) * _Factor;
  //将偏导转换为灰度值
  fixed4 col = float4(derivative.xxx , 1);
  col *= _Color;
  return col;
}
```

然后我们在场景中观察其表现，我们发现模型的灰度值受uv坐标和屏幕坐标之间的关系的影响。开始我们拉进、或者放大观察，发现灰度慢慢变暗，这是因为放大后，相邻两个像素对应的UV坐标的u值差减小了。当我们旋转90度后发现其完全变黑，这是因为旋转后u方向和x方向垂直，所以相邻两个像素对应的UV坐标的u值完全相同，差值为零。

![](https://www.ronja-tutorials.com/assets/images/posts/046/ddx_transform_change.gif)

仅仅这些就已经可以实现很多功能了，例如我们可以使用深度图粗略的计算出法向纹理，然后`tex2D`函数会基于该法向值的变化来选择`mipmap`的层级。但是更多时候我们需要得到所有的变化情况，而不是某一个方向的变化情况，而`fwidth`的作用正在于此。

## fwidth

如果你想将`ddx`和`ddy`两个函数结合起来使用，最直接的方法就是取各自结果的绝对值求和。下面就是我们自定义的`fwidth`函数。
```c++
float fwidth(float value){
  return abs(ddx(value)) + abs(ddy(value));
}
```

如果我们将前面的着色器中的`ddx`函数用`fwidth`来替换，你会发现缩放的时候灰度变化和之前一样，但是旋转的时候灰度变化相对更亮，在90度时也不是全黑了。当然我们使用余弦公式来替代前面的绝对值求和，这样颜色变化的估计值会更准确一点，但是很多时候并不是使用越高级的函数，消耗更多的性能，就可以得到更好的效果。
![](https://www.ronja-tutorials.com/assets/images/posts/046/fwidth_transform_change.gif)

## Non-aliased step

`fwidth`的第一个应用场景，至少对于我来说，就是在不处理锯齿的情况下，计算出梯度值然后使用指定阈值进行区域划分。它会以不同的形式应用到火焰、水、卡通光照等场景中。进行梯度划分最简单的方法就是使用`step`函数--也叫阶跃函数，然后将梯度值和阈值传入`step`函数中，得到的结果在应用到线性插值函数`lerp`上，可以对不同区域进行上色。当然这里`step`函数会引入锯齿等边缘问题。这里我们可以实现一个逆插值函数来替代`step`函数，逆插值函数通过计算单个像素点上的变化情况，来实现区域划分，同时保证中间有渐变过渡带，这样就解决了锯齿问题。

首先我们使用`fwidth`来计算像素梯度，然后在执行逆插值之前，我们先要计算我们的边界区间，也就是我们的过渡带。下面的0.5就是我们的边界阈值，`halfChange`是我们的半带宽，当然边界阈值和带宽都可以根据应用场景来定义。重点是后面的逆插值函数，会将当前像素值和过度带进行比较，在过渡带区间的值将映射为0-1之间的值，也就是灰色区域，而不在过渡带区间的值将会映射为小于-1或大于1的值。然后我们使用`saturate`函数对其进行裁剪，就可以得到无锯齿的区域划分图案。

通过上面的描述我们可能有些属性，这不就是`smoothstep`函数的功能吗。是的，但是作为内置函数的`smoothstep`，它在过渡带区域还使用了一些平滑操作，相应的计算量会大一点。但是本身边界区域就很窄，平滑效果并不会很明显，所以我们这里并不需要多余的平滑。使用我们的逆插值函数就足够了。
```c++
//片段函数
fixed4 frag(v2f i) : SV_TARGET{
  //选取用于求梯度的像素值
  float gradient = i.uv.x;
  //计算梯度
  float halfChange = fwidth(gradient) / 2;
  //计算过渡带上下边界
  float lowerEdge = 0.5 - halfChange;
  float upperEdge = 0.5 + halfChange;
  //使用你插值函数，将像素值划分为过渡带上侧、中间、下侧三个区域
  float stepped = (gradient - lowerEdge) / (upperEdge - lowerEdge);
  stepped = saturate(stepped);
  //将划分结果转换为颜色显示
  fixed4 col = float4(stepped.xxx, 1);
  return col;
}
```

下面是我实现的三种边界效果，左边第一个是使用`step`实现的，边界具有明显锯齿效应。第二个是使用`smoothstep`实现的，进行一定的边界平滑。最后一个是通过上面的逆插值方法实现的。后面两个的抗锯齿效果几乎一般无二，所以我建议你在使用`smoothstep`函数前可以考虑一下是否可以使用逆插值方法，可以节省一部分性能开销。

## A better step?

上面我们介绍了一种更好的边界平滑的技术，但是步骤多，写起来比较复杂。虽然这个方法也有一定的固定开销，不可能继续优化，而且99%的性能瓶颈问题都不是函数固定开销引起的。就像前面提到的`tex2d`，在执行的时候也会调用这些函数，但是这些函数在其中的消耗占比并不高。但是，我们可以将这个方法进行封装，这样我们在使用的时候就可以方便的调用了。

像`step`函数有两个参数，第一个是边界值，第二个是用来比较的值，当后者小于前者时，返回0，否则，返回1。同样，我们可以将上面的方法封装成和`step`类似的函数，只不过除了0和1，还会返回中间过渡区域的值。
```c++
//我们自定义的加强版的边界划分函数
float aaStep(float compValue, float gradient){
  float halfChange = fwidth(gradient) / 2;
  //计算过渡带上下边界
  float lowerEdge = compValue - halfChange;
  float upperEdge = compValue + halfChange;
  //使用你插值函数，将像素值划分为过渡带上侧、中间、下侧三个区域
  float stepped = (gradient - lowerEdge) / (upperEdge - lowerEdge);
  stepped = saturate(stepped);
  return stepped;
}

//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
  float stepped = aaStep(0.5, i.uv.x); 
  //显示边界
  fixed4 col = float4(stepped.xxx, 1);
  return col;
}
```

## An example

通过程序实现火焰效果，是阶跃函数一个比较好的应用方向，这里我大致参考[Febucci](https://twitter.com/febucci)的[火焰着色器](https://www.febucci.com/2019/05/fire-shader/)来实现我们的火焰效果。

这里我们随着时间不断对UV坐标进行偏移，然后用偏移后的UV对噪声图进行采样，采样结果当做是当前位置火焰的强度。另外我对将uv坐标的v值取反然后平方，这样所求得的梯度值就会沿着`y`方向成递减的趋势，从而使得火焰的形状下密上疏。这里我们使用的噪声图是泊林噪声，是在[之前教程](https://www.ronja-tutorials.com/post/030-baking-shaders/)中实现的。然后我们将噪声图中的采样值当做是阶跃阈值，这样就得到一个火焰的基本轮廓。为了模拟更逼真的火焰分层效果，这里我们通过偏移，产生多个火焰轮廓，然后使用线性插值函数来对这些分层区域上色。

然后我们将`aaStep`函数中的梯度值除以2的操作去掉，这样我们就扩大了其过渡区域的宽度。你可以试着修改这个值，然后观察一下产生的变化，选一个比较好的效果。

```c++
//强化版的阶跃函数
float aaStep(float compValue, float gradient){
  float change = fwidth(gradient);
  //计算过渡边界
  float lowerEdge = compValue - change;
  float upperEdge = compValue + change;
  //使用逆插值函数
  float stepped = (gradient - lowerEdge) / (upperEdge - lowerEdge);
  stepped = saturate(stepped);
  //最终结果近似于 `smoothstep(lowerEdge, upperEdge, gradient)`
  return stepped;
}

//片段着色函数
fixed4 frag(v2f i) : SV_TARGET{
  //使用平方值，让火焰变得更加旺盛
  float fireGradient = 1 - i.uv.y;
  fireGradient = fireGradient * fireGradient;
  //滑动uv，产生动画效果
  float2 fireUV = TRANSFORM_TEX(i.uv, _MainTex);
  fireUV.y -= _Time.y * _ScrollSpeed;
  //噪声采样
  float fireNoise = tex2D(_MainTex, fireUV).x;
  
  //划分火焰区域
  float outline = aaStep(fireNoise, fireGradient);
  float edge1 = aaStep(fireNoise, fireGradient - _Edge1);
  float edge2 = aaStep(fireNoise, fireGradient - _Edge2);
  
  //定义火焰外层颜色
  fixed4 col = _Color1 * outline;
  //其他层的颜色
  col = lerp(col, _Color2, edge1);
  col = lerp(col, _Color3, edge2);
  
  //输出结果
  return col;
}
```

下面我还对比了普通阶跃函数和我们这里加强版的阶跃函数，差别看起来不大，但是如果你的游戏审美要求是像素级别的，那么这个差别还是很明显的。所以我觉得你可以从现在开始，使用这里的方法，让你的游戏画面开起来更加柔和、更加平滑，即便是在低分辨率的情况下。
![](https://www.ronja-tutorials.com/assets/images/posts/046/FireComparison.png)

## Sources

- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/testing.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/testing.shader)
```c++
Shader "Tutorial/046_Partial_Derivatives/testing"{
	//材质面板
	Properties{
		_Factor("Factor", Range(0, 100)) = 1
	}

	SubShader{
		//不透明物体
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}
		
		Cull Off

		Pass{
			CGPROGRAM

			//引入内置函数和变量
			#include "UnityCG.cginc"

			//声明顶点、片段着色器
			#pragma vertex vert
			#pragma fragment frag

			float _Factor;

			//模型网格数据
			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			//中间插值数据
			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			//顶点着色器
			v2f vert(appdata v){
				v2f o;
				//裁剪坐标
				o.position = UnityObjectToClipPos(v.vertex);
				o.uv = v.uv;
				return o;
			}

			//片段着色器
			fixed4 frag(v2f i) : SV_TARGET{
                //计算梯度
			    float derivative = fwidth(i.uv.x) * _Factor;
			    //可视化梯度
				fixed4 col = float4(derivative.xxx , 1);
				return col;
			}

			ENDCG
		}
	}
}
```
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/aa_step.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/aa_step.shader)
```c++
Shader "Tutorial/046_Partial_Derivatives/aaStep"{
	//材质面板
	Properties{
	
	}

	SubShader{
		//不透明物体
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}
		
		Cull Off

		Pass{
			CGPROGRAM

			//引入内置函数和变量
			#include "UnityCG.cginc"

			//声明顶点、片段着色器
			#pragma vertex vert
			#pragma fragment frag

			//模型网格数据
			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			//中间插值数据
			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			//顶点着色器
			v2f vert(appdata v){
				v2f o;
				//裁剪坐标
				o.position = UnityObjectToClipPos(v.vertex);
				o.uv = v.uv;
				return o;
			}
			
			//强化版阶跃函数
			float aaStep(float compValue, float gradient){
			    float halfChange = fwidth(gradient) / 2;
			    //计算边界
			    float lowerEdge = compValue - halfChange;
			    float upperEdge = compValue + halfChange;
			    //使用逆插值函数
			    float stepped = (gradient - lowerEdge) / (upperEdge - lowerEdge);
			    stepped = saturate(stepped);
			    //计算结果近似于 `smoothstep(lowerEdge, upperEdge, gradient)`
			    return stepped;
			}

			//片段着色器
			fixed4 frag(v2f i) : SV_TARGET{
                float stepped = aaStep(0.5, i.uv.x); 
			    //梯度可视化
				fixed4 col = float4(stepped.xxx, 1);
				return col;
			}
			
			

			ENDCG
		}
	}
}
```
- [https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/Fire.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/046_Partial_Derivatives/Fire.shader)
```c++
Shader "Tutorial/046_Partial_Derivatives/fire"{
	//材质面板
	Properties{
	    _MainTex ("Fire Noise", 2D) = "white" {}
	    _ScrollSpeed("Animation Speed", Range(0, 2)) = 1
	
		_Color1 ("Color 1", Color) = (0, 0, 0, 1)
		_Color2 ("Color 2", Color) = (0, 0, 0, 1)
		_Color3 ("Color 3", Color) = (0, 0, 0, 1)
		
		_Edge1 ("Edge 1-2", Range(0, 1)) = 0.25
		_Edge2 ("Edge 2-3", Range(0, 1)) = 0.5
	}

	SubShader{
		//不透明物体
		Tags{ "RenderType"="transparent" "Queue"="transparent"}
		
		Cull Off
		Blend SrcAlpha OneMinusSrcAlpha
		ZWrite Off

		Pass{
			CGPROGRAM

			//引入内置函数和变量
			#include "UnityCG.cginc"

			//声明顶点、片段着色器
			#pragma vertex vert
			#pragma fragment frag

			//火焰颜色
			fixed4 _Color1;
			fixed4 _Color2;
			fixed4 _Color3;
			
			float _Edge1;
			float _Edge2;
			
			float _ScrollSpeed;
			
			sampler2D _MainTex;
			float4 _MainTex_ST;

			//模型网格数据
			struct appdata{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			//中间插值数据
			struct v2f{
				float4 position : SV_POSITION;
				float2 uv : TEXCOORD0;
			};

			//顶点着色器
			v2f vert(appdata v){
				v2f o;
				//裁剪坐标
				o.position = UnityObjectToClipPos(v.vertex);
				o.uv = v.uv;
				return o;
			}
			
			//强化版阶跃函数
			float aaStep(float compValue, float gradient){
			    float change = fwidth(gradient);
			    //计算边界
			    float lowerEdge = compValue - change;
			    float upperEdge = compValue + change;
			    //使用逆插值函数
			    float stepped = (gradient - lowerEdge) / (upperEdge - lowerEdge);
			    stepped = saturate(stepped);
			    //结果近似于 `smoothstep(lowerEdge, upperEdge, gradient)`
			    return stepped;
			}

			//片段着色器
			fixed4 frag(v2f i) : SV_TARGET{
			    //平方使得火焰更旺盛
			    float fireGradient = 1 - i.uv.y;
			    fireGradient = fireGradient * fireGradient;
			    //滑动uv值，产生动画效果
			    float2 fireUV = TRANSFORM_TEX(i.uv, _MainTex);
			    fireUV.y -= _Time.y * _ScrollSpeed;
			    //噪声采样
			    float fireNoise = tex2D(_MainTex, fireUV).x;
			    
			    //计算火焰轮廓
                float outline = aaStep(fireNoise, fireGradient);
                float edge1 = aaStep(fireNoise, fireGradient - _Edge1);
                float edge2 = aaStep(fireNoise, fireGradient - _Edge2);
			    
			    //外层火焰颜色
			    fixed4 col = _Color1 * outline;
			    //其他层火焰颜色
			    col = lerp(col, _Color2, edge1);
			    col = lerp(col, _Color3, edge2);
			    
			    //输出结果
				return col;
			}

			ENDCG
		}
	}
}
```

## 相关文章
- [An introduction to shader derivative functions](http://www.aclockworkberry.com/shader-derivative-functions/)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
