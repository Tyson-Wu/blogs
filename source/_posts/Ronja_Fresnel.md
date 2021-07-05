---
title: Fresnel
date: 2021-07-05 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Fresnel](https://www.ronja-tutorials.com/post/012-fresnel/)

## Summary

菲涅尔效果是渲染中常用的效果。使用菲涅尔效果我们可以很方便的对模型的边缘进行增亮、加黑等边缘效果，加强场景的深度感。

本篇将采用表面着色器的实现方法，所以如果你对表面着色器还不了解的话，建议你先看看[这篇](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)文章。当然你也可以将菲涅尔效果应用到其他类型的着色器，来增强模型平滑度、或突出重点。
![](https://www.ronja-tutorials.com/assets/images/posts/012/Result.jpg)

## Highlighting one Side of the Model

我们通过修改表面着色器来实现菲涅尔效果。菲涅尔效果是根据法向来计算效果的密度、或厚度。因为要在片段着色器中使用到世界坐标系中的法向量，所以我们首先在顶点着色器中计算好世界坐标系中的顶点法向，然后通过中间插值结构传递给片段着色器。当然，我们的输入结构体和中间插值结构体都需要定义法向量。

在[三维平面映射](https://tyson-wu.github.io/blogs/2021/07/02/Ronja_Triplanar_Mapping/)我们介绍了如何计算顶点的世界法向量。
```c++
//输入的模型网格数据，其中采用宏定义来定义部分数据
struct Input {
    float2 uv_MainTex;
    float3 worldNormal;
    INTERNAL_DATA
};
```

我们可以通过计算相邻向量之间的点乘，从而了解模型表明的平滑度、或者梯度。单位向量之间的点乘越大，说明它们方向越一致。
![](https://www.ronja-tutorials.com/assets/images/posts/012/DotSphere.png)
![](https://www.ronja-tutorials.com/assets/images/posts/012/DotDirection.png)

首先，我将所有法向量与一个固定向量做点乘，可以让我们更容易理解点乘的意义。然后将点乘的结果传递给`emission`变量，将点乘结果通过自发光的形式表现出来。
```c++
//表面着色器函数，主要用了计算光照模型的输入参数，然后由光照模型进行最中的颜色计算
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //两个向量之间的点乘
    float fresnel = dot(i.worldNormal, float3(0, 1, 0));
    //应用菲涅尔效果
    o.Emission = _Emission + fresnel;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/WhiteTopBlackBottom.png)

上图我们可以看到，越朝上的地方越亮、越朝下越暗。为了避免自发光出现负数，这里将菲涅尔值限定在`[0,1]`之间。这里有两个函数`staturate`和`clamp`都可以实现范围限定的功能，但是`staturate`在GPU上执行速度更快。下面是限定后的效果。
```c++
//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //计算菲涅尔值
    float fresnel = dot(i.worldNormal, float3(0, 1, 0));
    //限定为 0 - 1 之间
    fresnel = saturate(fresnel);
    //应用菲尼尔效果
    o.Emission = _Emission + fresnel;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/WhiteTopBlackBottom.png)

## Highlighting the outer Parts

接下来我们使用视角方向来计算菲涅尔值。视角放向可以直接定义输入结构体中，然后在表面着色器函数中就可以访问。

如果是在无光照着色器中实现菲涅尔效果，那么视角方向可以通过摄像机的位置和顶点世界坐标来计算。其中摄像机坐标存储在`_WorldSpaceCameraPos`这个内置变量中，由Unity来赋值。
```c++
//表面着色器的输入结构体
struct Input {
    float2 uv_MainTex;
    float3 worldNormal;
    float3 viewDir;
    INTERNAL_DATA
};

//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //金属度与光滑度
    o.Metallic = _Metallic;
    o.Smoothness = _Smoothness;

    //菲尼尔值
    float fresnel = dot(i.worldNormal, i.viewDir);
    //范围限定在 0 - 1
    fresnel = saturate(fresnel);
    //应用菲涅尔值
    o.Emission = _Emission + fresnel;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/WhiteFront.png)

总体来说实现了菲涅尔效果，但是整个材质是靠近中心区域更亮，而不是边缘更亮。如果我们想让边缘更亮，我们可以将1减去菲涅尔值。
```c++
//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //菲涅尔值
    float fresnel = dot(i.worldNormal, i.viewDir);
    //效果反转
    fresnel = saturate(1 - fresnel);
    //应用菲涅尔效果
    o.Emission = _Emission + fresnel;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/WhiteFresnel.png)

## Add Fresnel Color and Intensity

最后，我想增加一些自定义属性，比如菲涅尔颜色。
```c++
//...

_FresnelColor ("Fresnel Color", Color) = (1,1,1,1)

//...

float3 _FresnelColor;

//...

//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //菲涅尔值
    float fresnel = dot(i.worldNormal, i.viewDir);
    //效果反转
    fresnel = saturate(1 - fresnel);
    //使用菲涅尔自定义颜色
    float3 fresnelColor = fresnel * _FresnelColor;
    //应用菲涅尔效果
    o.Emission = _Emission + fresnelColor;
}

//...
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/Result.jpg)

然后再增加一个控制菲涅尔强度的属性。这里我们使用指数函数来调节菲涅尔强度。

因为指数函数计算消耗非常大，所以如果有其他方法能实现相同的效果，尽量不要使用指数函数。当然，指数函数用法简单、便捷，在设计效果的时候可以直接使用指数函数。
```c++
//...

[PowerSlider(4)] _FresnelExponent ("Fresnel Exponent", Range(0.25, 4)) = 1

//...

float _FresnelExponent;

//...

//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {

    //...

    //菲涅尔值
    float fresnel = dot(i.worldNormal, i.viewDir);
    //效果反转
    fresnel = saturate(1 - fresnel);
    //使用菲涅尔自定义颜色
    float3 fresnelColor = fresnel * _FresnelColor;
    //强度调节
    fresnel = pow(fresnel, _FresnelExponent);
    //应用菲涅尔效果
    o.Emission = _Emission + fresnelColor;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/012/ExponentAdjustment.gif)

菲涅尔效果除了用来突出轮廓外，还可以用来实现各种渐变效果，这里不展开，有兴趣可以自行尝试。
```c++
Shader "Tutorial/012_Fresnel" {
    //材质面板
    Properties {
        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
        _Smoothness ("Smoothness", Range(0, 1)) = 0
        _Metallic ("Metalness", Range(0, 1)) = 0
        [HDR] _Emission ("Emission", color) = (0,0,0)

        _FresnelColor ("Fresnel Color", Color) = (1,1,1,1)
        [PowerSlider(4)] _FresnelExponent ("Fresnel Exponent", Range(0.25, 4)) = 1
    }
    SubShader {
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        CGPROGRAM

        //这是表面着色器，unity会自动生成部分代码
        //surf 是被定义为表面着色器函数
        //fullforwardshadows 告诉Unity将所有阴影Pass都复制过来
        #pragma surface surf Standard fullforwardshadows
        #pragma target 3.0

        sampler2D _MainTex;
        fixed4 _Color;

        half _Smoothness;
        half _Metallic;
        half3 _Emission;

        float3 _FresnelColor;
        float _FresnelExponent;

        //表面着色器的输入结构
        struct Input {
            float2 uv_MainTex;
            float3 worldNormal;
            float3 viewDir;
            INTERNAL_DATA
        };

        //表面着色器函数，主要用来计算光照模型所需的参数
        void surf (Input i, inout SurfaceOutputStandard o) {
            //纹理采样
            fixed4 col = tex2D(_MainTex, i.uv_MainTex);
            col *= _Color;
            o.Albedo = col.rgb;

            //光滑度、和金属度
            o.Metallic = _Metallic;
            o.Smoothness = _Smoothness;

            //菲涅尔值
            float fresnel = dot(i.worldNormal, i.viewDir);
            //效果翻转
            fresnel = saturate(1 - fresnel);
            //控制菲涅尔强度
            fresnel = pow(fresnel, _FresnelExponent);
            //应用菲涅尔颜色
            float3 fresnelColor = fresnel * _FresnelColor;
            //应用菲涅尔
            o.Emission = _Emission + fresnelColor;
        }
        ENDCG
    }
    FallBack "Standard"
}
```

希望本篇能让你对菲涅尔效果有所了解。

你可以在以下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/012_Fresnel/Fresnel.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/012_Fresnel/Fresnel.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!