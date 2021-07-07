---
title: Hull Outlines
date: 2021-07-07 13:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Hull Outlines](https://www.ronja-tutorials.com/post/020-hull-outline/)

到目前为止，我们基本上是一个着色器只会执行一次将模型绘制到屏幕上。实际上在一个着色器中是允许对一个模型绘制多次。比如说我们接下来的轮廓实现方案就需要对模型绘制多次。首先按往常一样渲染一遍模型，然后将模型顶点沿着法线方向移动一点，然后再次进行绘制，而这第二次绘制的模型会出现在上一次绘制的边缘处，也就是我们想得到的轮廓。

为了能够更好的理解本文，建议你先了解什么是[表面着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)，以及[无光照着色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)。
![](https://www.ronja-tutorials.com/assets/images/posts/020/Result.png)

## Outlines for Unlit Shaders

沿用之前无光照着色器脚本，我们只需要将其中的`Pass`复制一遍就可以。现在有两个完全相同的`Pass`，所以即便是绘制两遍，最终的结果也是一样的。
```c++
//复制出来的，用于绘制轮廓的Pass
Pass{
    CGPROGRAM

    //引入内置函数和变量
    #include "UnityCG.cginc"

    //声明顶点、片段着色器
    #pragma vertex vert
    #pragma fragment frag

    //模型表面纹理
    sampler2D _MainTex;
    float4 _MainTex_ST;

    //模型颜色
    fixed4 _Color;

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
        //计算裁剪坐标
        o.position = UnityObjectToClipPos(v.vertex);
        o.uv = TRANSFORM_TEX(v.uv, _MainTex);
        return o;
    }

    //片段着色器
    fixed4 frag(v2f i) : SV_TARGET{
        fixed4 col = tex2D(_MainTex, i.uv);
        col *= _Color;
        return col;
    }

    ENDCG
}
```

然后我们需要对上面这个`Pass`的变量进行修改，因为轮廓不需要纹理，只需要轮廓颜色、轮廓宽度，所以我们删除纹理变量，然后增加轮廓颜色、和轮廓宽度变量，并且在`Properties`块中添加这两个属性。
```c++
//材质面板
Properties{
    _OutlineColor ("Outline Color", Color) = (0, 0, 0, 1)
    _OutlineThickness ("Outline Thickness", Range(0,.1)) = 0.03

    _Color ("Tint", Color) = (0, 0, 0, 1)
    _MainTex ("Texture", 2D) = "white" {}
}
```
```c++
//轮廓颜色
fixed4 _OutlineColor;
//轮廓宽度
float _OutlineThickness;
```

接下来是修改片段着色器，直接返回我们的轮廓颜色。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    return _OutlineColor;
}
```

因为我们没有使用纹理，所以与纹理相关的UV变量也不需要，所以可以将其从那些结构体中删除。
```c++
//模型网格数据
struct appdata{
    float4 vertex : POSITION;
};

//中间插值数据
struct v2f{
    float4 position : SV_POSITION;
};

//顶点着色器
v2f vert(appdata v){
    v2f o;
    //计算裁剪坐标
    o.position = UnityObjectToClipPos(position);
    return o;
}
```

![](https://www.ronja-tutorials.com/assets/images/posts/020/DarkMonkey.png)

上图是修改后的显示效果，我们的物体最终显示为轮廓色，这是因为我们第二个`Pass`将第一个`Pass`渲染的图完全覆盖了。我们接下来处理这个问题。

为了保证我们的第二个`Pass`超出第一个`Pass`的显示范围，从而形成轮廓。我们需要将模型的顶点沿着其法向量的方向偏移。因此我们需要在模型网格数据中传入法向量，
```c++
//模型网格数据
struct appdata{
    float4 vertex : POSITION;
    float3 normal : NORMAL;
};
```
```c++
//顶点着色器
v2f vert(appdata v){
    v2f o;
    //顶点沿着法向偏移
    float3 normal = normalize(v.normal);
    float3 outlineOffset = normal * _OutlineThickness;
    float3 position = v.vertex + outlineOffset;
    //计算裁剪坐标
    o.position = UnityObjectToClipPos(position);

    return o;
}
```

现在可以可以通过`_OutlineThinckness`来控制边缘的宽度，但是我们第一个`Pass`渲染的画面还是被遮挡了。为了修复这个问题，我们将第二个`Pass`改为正面剔除。这样可以保证第二个`Pass`渲染的画面永远在第一个`Pass`之后。

```c++
//第二个Pass， 用来绘制轮廓
Pass{
    Cull Front

    //...
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/020/SimpleOutlines.png)

上图就是我们得到的轮廓了。

### Source

```c++
Shader "Tutorial/19_InvertedHull/Unlit"{
    //材质面板
    Properties{
        _OutlineColor ("Outline Color", Color) = (0, 0, 0, 1)
        _OutlineThickness ("Outline Thickness", Range(0,.1)) = 0.03

        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
    }

    SubShader{
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        //第一个Pass， 用来渲染模型本身
        Pass{
            CGPROGRAM

            //引入内置函数和变量
            #include "UnityCG.cginc"

            //声明顶点和片段着色器
            #pragma vertex vert
            #pragma fragment frag

            //模型纹理
            sampler2D _MainTex;
            float4 _MainTex_ST;

            //模型颜色
            fixed4 _Color;

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
                //计算裁剪坐标
                o.position = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            //片段着色器
            fixed4 frag(v2f i) : SV_TARGET{
                fixed4 col = tex2D(_MainTex, i.uv);
                col *= _Color;
                return col;
            }

            ENDCG
        }

        //第二个Pass，用来绘制轮廓
        Pass{
            Cull front

            CGPROGRAM

            //引入内置函数和变量
            #include "UnityCG.cginc"

            //声明顶点和片段着色器
            #pragma vertex vert
            #pragma fragment frag

            //轮廓颜色
            fixed4 _OutlineColor;
            //轮廓宽度
            float _OutlineThickness;

            //模型网格数据
            struct appdata{
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            //中间插值数据
            struct v2f{
                float4 position : SV_POSITION;
            };

            //顶点着色器
            v2f vert(appdata v){
                v2f o;
                //沿着法向移动顶点
                float3 normal = normalize(v.normal);
                float3 outlineOffset = normal * _OutlineThickness;
                float3 position = v.vertex + outlineOffset;
                //计算裁剪坐标
                o.position = UnityObjectToClipPos(position);

                return o;
            }

            //片段着色器
            fixed4 frag(v2f i) : SV_TARGET{
                return _OutlineColor;
            }

            ENDCG
        }
    }

    //后补着色器
    FallBack "Standard"
}
```

## Outlines with Surface Shaders

前面是在普通的顶点、片段着色其中应用轮廓效果，在表面着色器中其实也一样。对于表面着色器，Unity会自动生成部分代码，但是不会改动我们写入的代码，因此我们可以直接将前面的第二个轮廓`Pass`直接复制过来，并且可以实现同样的轮廓效果。

![](https://www.ronja-tutorials.com/assets/images/posts/020/Result.png)

### Source

```c++
Shader "Tutorial/020_InvertedHull/Surface" {
    Properties {
        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
        _Smoothness ("Smoothness", Range(0, 1)) = 0
        _Metallic ("Metalness", Range(0, 1)) = 0
        [HDR] _Emission ("Emission", color) = (0,0,0)

        _OutlineColor ("Outline Color", Color) = (0, 0, 0, 1)
        _OutlineThickness ("Outline Thickness", Range(0,1)) = 0.1
    }
    SubShader {
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        CGPROGRAM
        //表面着色器
        //表面着色器函数和光照模型
        //fullforwardshadows 使用所有的阴影Pass
        #pragma surface surf Standard fullforwardshadows
        #pragma target 3.0

        sampler2D _MainTex;
        fixed4 _Color;

        half _Smoothness;
        half _Metallic;
        half3 _Emission;

        //表面着色器的输入数据
        struct Input {
            float2 uv_MainTex;
        };

        //表面着色函数，主要计算光照模型所需的参数
        void surf (Input i, inout SurfaceOutputStandard o) {
            //纹理采样
            fixed4 col = tex2D(_MainTex, i.uv_MainTex);
            col *= _Color;
            o.Albedo = col.rgb;
            //光照模型相关参数
            o.Metallic = _Metallic;
            o.Smoothness = _Smoothness;
            o.Emission = _Emission;
        }
        ENDCG

        //第二个轮廓Pass
        Pass{
            Cull Front

            CGPROGRAM

            //引入内置函数和变量
            #include "UnityCG.cginc"

            //声明顶点、片段着色器
            #pragma vertex vert
            #pragma fragment frag

            //轮廓颜色、粗细
            fixed4 _OutlineColor;
            float _OutlineThickness;

            //模型网格数据
            struct appdata{
                float4 vertex : POSITION;
                float4 normal : NORMAL;
            };

            //中间插值数据
            struct v2f{
                float4 position : SV_POSITION;
            };

            //顶点着色器
            v2f vert(appdata v){
                v2f o;
                //计算裁剪坐标
                o.position = UnityObjectToClipPos(v.vertex + normalize(v.normal) * _OutlineThickness);
                return o;
            }

            //片段着色器
            fixed4 frag(v2f i) : SV_TARGET{
                return _OutlineColor;
            }

            ENDCG
        }
    }
    FallBack "Standard"
}
```

本篇轮廓实现方案和上一篇后处理轮廓方案的区别在于，本文所有的作色器是应用到个体模型上，所以可以根据需要选择哪些模型显示轮廓，并且还可以调节轮廓的宽度，整体的表现效果也有很大的差异。如果说哪个方案好，我觉得我们应该掌握这两种轮廓方案，然后根据实际情况进行选择。

希望你能通过本篇，了解如何在一个着色器中使用多个`Pass`，并且知道如何利用它们来实现轮廓效果。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/020_Inverted_Hull/UnlitOutlines.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/020_Inverted_Hull/UnlitOutlines.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/020_Inverted_Hull/SurfaceOutlines.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/020_Inverted_Hull/SurfaceOutlines.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
