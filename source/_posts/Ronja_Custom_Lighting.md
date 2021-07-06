---
title: Custom Lighting
date: 2021-07-05 22:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Custom Lighting](https://www.ronja-tutorials.com/post/013-custom-lighting/)

## Summary

表面着色器真的非常方便，特别是在处理光照时，表面着色器可以使用PBR模型快速实现物理光照效果。但是有时候我们可能想实现其他一些光照效果，例如卡通风格的。这时候我们可以使用自定义光照函数来满足我们的需求。

本篇主要介绍表面着色器特有的一些功能。但是光照处理的基本原理可以应用到其他着色器中。只不过Unity会为表面着色器生成一个基本的可复用的光照框架，如果我们使用其他着色器，那么必须手动补全这部分代码，但是这些并不是本文的重点，所以不做赘述。

如果你是一个初学者，建议你从[第一章](https://tyson-wu.github.io/blogs/2021/06/25/Ronja_Structure/)开始看。

![](https://www.ronja-tutorials.com/assets/images/posts/013/Result.png)

## Use Custom Lighting Function

首先我们需要将光照模型设置为我们自定义的光照函数。
```c++
//表面着色器
//表面着色函数以及我们的自定义光照函数
//fullforwardshadows 应用所有的阴影Pass
#pragma surface surf Custom fullforwardShadows
```

然后我们添加我们的自定义光照函数。光照函数的名字结构是`LightingX`，其中`X`是上面定义的光照函数，这里是`Custom`。在下面的自定义光照函数中使用的`SurfaceOutput`结构体，实际上是由`surf`表面着色器函数返回的数据，两者最终的数据结构必须一致。另外还有光照方向、以及光照衰减度，衰减度在后面会介绍。
```c++
//自定义光照函数，针对每个光源都会处理一遍
float4 LightingCustom(SurfaceOutput s, float3 lightDir, float atten){
    return 0;
}
```

`SurfaceOutput`和`SurfaceOutputSstandard`都是Unity预定义的数据结构，前者是针对非物理渲染的，后者是提供了物理渲染的基本参数。当然你也可以把他们当做是一个参考模板，然后实现自己的数据结构。使用的方法是，先在表面着色器函数中对该结构赋值，然后在自定义光照函数中使用。因为`SurfaceOutput`不包含光滑度、以及金属度的属性，所以在表面着色器删除对这两者的赋值。

//表面着色器
void surf (Input i, inout SurfaceOutput o) {
    //纹理采样
    fixed4 col = tex2D(_MainTex, i.uv_MainTex);
    col *= _Color;
    o.Albedo = col.rgb;

    //o.Emission = _Emission;
}

现在我们有了自己的光照函数，但是该函数目前返回的是0，所以模型渲染后看不到光照效果。

按理说光照为零的话，整个模型应该是纯黑色，但是我们现在却能看清模型的基本轮廓。这是因为，光照函数处理的是直接光源，但是在模型渲染的时候除了直接光源，还有间接光源作用。其中环境光就是一种间接光源，Unity会自动将天空盒的颜色来当成环境光的一部分，所以我们在环境光的作用下还能看清物体。如果你在Unity编辑器的场景窗口上方选择关闭环境光的按钮，那么整个模型最终就会变成纯黑色，并且其最终显示都完全受我们的自定义光照函数控制。当然这里我保留Unity编辑器的默认设置。

## Implement Lighting Ramp

下面我们来实现简单的自定义光照函数。第一步我们先求解表面法向和入射光线方向向量的点乘，这两个参数正好在自定义光照函数的参数列表中。
```c++
//自定义光照函数
float4 LightingCustom(SurfaceOutput s, float3 lightDir, float atten){
    //法向和光照入射方向的点乘，可以用来表示光照密度
    float towardsLight = dot(s.Normal, lightDir);
    return towardsLight;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/013/DotLight.png)

下面我们实现的光照函数非常简单，涵盖了一般通用结构。我们使用光线入射向量来最为纹理纹理采样的参数，并将采样结果当成光源在该点的亮度。

因此我们必须将点乘结果的取值范围从`[-1, 1]`映射为`[0, 1]`，因为后者是UV的取值范围。

然后我们创建一个叫`ramp`的纹理变量。按照前面说的采样方法进行采样。然后在材质面板上设置纹理为半黑半白的图片。
```c++
//材质面板
Properties {
    _Color ("Tint", Color) = (0, 0, 0, 1)
    _MainTex ("Texture", 2D) = "white" {}
    [HDR] _Emission ("Emission", color) = (0,0,0)

    _Ramp ("Toon Ramp", 2D) = "white" {}
}

//...

sampler2D _Ramp;
```

下面是我们所用的`ramp`纹理。
![](https://www.ronja-tutorials.com/assets/images/posts/013/HardRamp.png)

```c++
//自定义光照函数
float4 LightingCustom(SurfaceOutput s, float3 lightDir, float atten){
    //入射光强
    float towardsLight = dot(s.Normal, lightDir);
    //数值区间映射
    towardsLight = towardsLight * 0.5 + 0.5;

    //纹理采样
    float3 lightIntensity = tex2D(_Ramp, towardsLight).rgb;

    return float4(lightIntensity, 1);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/013/DrawRamp.png)

在上图中，模型背光面也能看清模型表面纹理。这还是因为环境光的影响。

为了让效果看起来更好，我们将模型颜色和光照密度相乘、同时应用衰减因子，最终的到一个具有颜色、且强度随距离变化的光源。模型最终的表现也和光源的颜色、和距离相关。
```c++
//自定义光照
float4 LightingCustom(SurfaceOutput s, float3 lightDir, float atten){
    //法向和光入射方向点乘
    float towardsLight = dot(s.Normal, lightDir);
    //数值范围映射
    towardsLight = towardsLight * 0.5 + 0.5;

    //光强采样
    float3 lightIntensity = tex2D(_Ramp, towardsLight).rgb;

    //最终的颜色
    float4 col;
    //应用光照
    col.rgb = lightIntensity * s.Albedo * atten * _LightColor0.rgb;
    //赋值透明通道
    col.a = s.Alpha; 

    return col;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/013/CorrectRampLighting.png)

上面已经介绍完了一个完整的自定义光照着色器的实现。我们可以使用不同的`ramp`纹理来得到完全不同的渲染风格。比如下面，我们采样冷暖色条纹，实现的卡通效果。图片源于[这里](https://docs.unity3d.com/Manual/SL-SurfaceShaderLightingExamples.html)
![](https://www.ronja-tutorials.com/assets/images/posts/013/HotColdRamp.png)
![](https://www.ronja-tutorials.com/assets/images/posts/013/HotColdRampModel.png)

在着色器我们还有一个`emission`没有用到，但是模型最终的显示中却受自发光这个变量的影响。

上面这种卡通风格的着色器非常有用，在很多地方可以灵活运用。

自定义光照函数非常强大，但是只能用在前向渲染中。即便是你把渲染路径改为延迟渲染，最终渲染管线依然会把这部分的渲染转移到前向渲染路径中。
```c++
Shader "Tutorial/013_CustomSurfaceLighting" {
    //材质面板
    Properties {
        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
        [HDR] _Emission ("Emission", color) = (0,0,0)

        _Ramp ("Toon Ramp", 2D) = "white" {}
    }
    SubShader {
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        CGPROGRAM

        //表面着色器
        //自定义表面着色器函数、自定义光照函数
        //fullforwardshadows 使用所有的阴影Pass
        #pragma surface surf Custom fullforwardshadows
        #pragma target 3.0

        sampler2D _MainTex;
        fixed4 _Color;
        half3 _Emission;

        sampler2D _Ramp;

        //自定义光照函数
        float4 LightingCustom(SurfaceOutput s, float3 lightDir, float atten){
            //点乘
            float towardsLight = dot(s.Normal, lightDir);
            //数值范围映射
            towardsLight = towardsLight * 0.5 + 0.5;

            //采样
            float3 lightIntensity = tex2D(_Ramp, towardsLight).rgb;

            //最终的颜色
            float4 col;
            //计算光照
            col.rgb = lightIntensity * s.Albedo * atten * _LightColor0.rgb;
            //使用透明通道
            col.a = s.Alpha; 

            return col;
        }

        //输入结构体
        struct Input {
            float2 uv_MainTex;
        };

        //表边着色器函数
        void surf (Input i, inout SurfaceOutput o) {
            //纹理采样
            fixed4 col = tex2D(_MainTex, i.uv_MainTex);
            col *= _Color;
            o.Albedo = col.rgb;

            //o.Emission = _Emission;
        }
        ENDCG
    }
    FallBack "Standard"
}
```

希望本篇能帮助你理解表面着色器和自定义光照。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/013_CustomSurfaceLighting/CustomLighting.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/013_CustomSurfaceLighting/CustomLighting.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!