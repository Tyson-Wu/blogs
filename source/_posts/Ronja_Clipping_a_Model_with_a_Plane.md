---
title: Clipping a Model with a Plane
date: 2021-07-07 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Clipping a Model with a Plane](https://www.ronja-tutorials.com/post/021-plane-clipping/)

## Summary

另一个比较炫酷的效果是，根据指定区域对模型进行裁剪。

我们这篇教程需要你掌握表面着色器的基本知识，所以建议你先阅读[这篇](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)教程。

![](https://www.ronja-tutorials.com/assets/images/posts/021/Result.gif)

## Define Plane

首先我们需要新建一个`C#`脚本，用来设置裁剪平面用的，然后将平面参数传递个着色器，还需要定义一个绑定该着色器的材质变量。我们使用`[ExecuteAlways]`标签让脚本在编辑模式下一直执行。当然，你可以根据你自身的需求考虑要不要加，毕竟就算不加，当程序运行时，该脚本也会正常执行。

我们在`Update`函数中新建一个类型为`Plane`的变量，并向其构造函数中传入平面的法向量，和平面上任意一个点的坐标。这里我们将选择脚本所在的物体的`Y`轴正方向为平面的法向量，脚本所在的物体的位置为平面上的任意一点。换句话说，我们这个平面就是脚本所在物体的局部坐标系的`O-XZ`平面。

然后我们创建一个四维向量，将平面的法向量传递给该向量的前三个元素，然后第四个元素存储平面到世界原点的距离。后面我会解释这些值得含义。

然后我们将这个四维变量传递给着色器。
```C#
using UnityEngine;

[ExecuteAlways]
public class ClippingPlane : MonoBehaviour {
    //用于裁剪的材质球
    public Material mat;

    //每帧都会执行一次
    void Update () {
        //创建一个平面
        Plane plane = new Plane(transform.up, transform.position);
        //平面参数
        Vector4 planeRepresentation = new Vector4(plane.normal.x, plane.normal.y, plane.normal.z, plane.distance);
        //传递给着色器中的_Plane变量
        mat.SetVector("_Plane", planeRepresentation);
    }
}
```
然后我们将该脚本绑定到物体上，该物体将被当做我们的裁剪工具。
![](https://www.ronja-tutorials.com/assets/images/posts/021/PlaneInspector.png)

## Clip Plane

接下来实现我们的着色器，这里我沿用[表面作色器](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)中的着色器。

首先我们需要在着色器中添加`_Plane`公共变量，因为这个变量是通过脚本赋值的，所以不需要出现在`Properties`块中。
```c++
float4 _Plane;
```

在表面着色器中我们可以计算模型表面上的点，到过原点与我们这个自定义平面平行的平面的距离。这个距离可以通过表面上的点坐标和平面法向量的点乘来计算。如果这个带你在我们的自定义平面上，那么这个距离值将会等于平面参数中的距离值。如果这个距离值大于平面参数中的距离值，那么这个点在平面上方，小于则在其下方。

要实现这个位置判断，我们需要计算模型顶点的世界坐标。在表面着色器中，我们只需要在输入结构体中加入`worldPos`变量，根据命名规则，表面作色器会自动给我们计算世界坐标。如果是其他着色器，那么我们需要使用模型矩阵自行计算。然后将距离传递给自发光变量。

```c++
//表面着色器中的输入数据，由其自动计算
struct Input {
    float2 uv_MainTex;
    float3 worldPos;
};
```
```c++
//表面作色器函数
void surf (Input i, inout SurfaceOutputStandard o) {
    //计算点到屏幕的距离
    float distance = dot(i.worldPos, _Plane.xyz);
    o.Emission = distance;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/021/OriginPlaneDistance.gif)

上图中平面的方向会影响亮度，但是位置却不会。这是因为我们还没有应用平面的位置参数。接下来我们将这个距离参数应用上。
```c++
//表面作色器函数
void surf (Input i, inout SurfaceOutputStandard o) {
    //计算点到屏幕的距离
    float distance = dot(i.worldPos, _Plane.xyz);
    distance = distance + _Plane.w;//应用平面距离参数
    o.Emission = distance;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/021/PlaneDistance.gif)

这里计算出来的距离是一个矢量，具有方向性，在平面上方的大于零，平面下方的小于零，因此上图中显示的平面两侧的亮度不一样。我们可以应用这个特性来将平面某一侧的模型剔除掉。例如这里不渲染平面上方的模型，只渲染平面下方的模型。

在前面的教程中有介绍，可以在片段着色器中使用`clip`函数来剔除某些像素。
```c++
//表面着色器函数
void surf (Input i, inout SurfaceOutputStandard o) {
    //计算点到面的距离
    float distance = dot(i.worldPos, _Plane.xyz);
    distance = distance + _Plane.w;
    clip(-distance);//剔除平面上方的模型
    o.Emission = distance;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/021/ClipWithDistance.png)

现在我们可以看到上面的模型已经被剔除了，这时候我们也不用自发光的亮度来表示哪里是平面的上方，可以恢复模型原本的纹理颜色。
```c++
//表面着色器
void surf (Input i, inout SurfaceOutputStandard o) {
    //计算点到面的距离
    float distance = dot(i.worldPos, _Plane.xyz);
    distance = distance + _Plane.w;
    o.Emission = distance;

    //纹理采样
    fixed4 col = tex2D(_MainTex, i.uv_MainTex);
    col *= _Color;
    o.Albedo = col.rgb;
    o.Metallic = _Metallic;
    o.Smoothness = _Smoothness;
    o.Emission = _Emission;
}
```

## Show Inside

虽然我们的模型成功的被平面裁剪为两半，但是剩下的这一部分看起来很怪异，好像缺了一部分，到处是孔洞。造成这种现象的原因是，默认情况下我们只渲染模型的正面，因为我们可以肯定模型背面，也就是其内部不会被渲染，这样起到一个优化的作用。但是，现在我们的模型被切开了，内部允许被看到，所以应该取消背面剔除功能。

要同时渲染模型的正反面，我们只需将着色器的`Cull`设置为`Off`。
```c++
SubShader{
    //不透明物体
    Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

    // 关闭剔除功能，这样模型的两个面都可以被渲染
    Cull Off

    //...

}
```

现在我们可以看到模型的内侧了，但是模型的法向量依然是指向外侧的，同时我们可能也不想看到内侧。这时候我们可以很容易的区分当前渲染的像素是内侧还是外侧。我们需要在表面着色器的输入结构体中加入一个表明像素朝向的变量。这个变量也是有表面着色器自行填充，1表示外侧，-1表示内侧。

这里我们还是用插值函数来实现内外侧的不同显示效果，所以我们需要将朝向变量映射到0-1之间。
```c++
//表面着色器的输入值
struct Input {
    float2 uv_MainTex;
    float3 worldPos;
    float facing : VFACE;
};
```
```c++
float facing = i.facing * 0.5 + 0.5;
o.Emission = facing;
```
![](https://www.ronja-tutorials.com/assets/images/posts/021/Facing.png)

上图可以看到我们已经可以清楚地区分内外侧了。我们可以给内侧指定一个颜色，然后外侧按照正常显示。我们需要一个内侧颜色，并且将这个颜色应用到自发光变量上。因为我们模型内侧的法向量是错误的，所以在计算光照候的结果也将是错误的，而自发光的颜色不受光照影响。另外，我们也将朝向值应用到其他光照参数上，这样保证内侧不会进行光照计算。

```c++
//材质面板
Properties{
    _Color ("Tint", Color) = (0, 0, 0, 1)
    _MainTex ("Texture", 2D) = "white" {}
    _Smoothness ("Smoothness", Range(0, 1)) = 0
    _Metallic ("Metalness", Range(0, 1)) = 0
    [HDR]_Emission ("Emission", color) = (0,0,0)

    [HDR]_CutoffColor("Cutoff Color", Color) = (1,0,0,0)
}
```
```c++
float4 _CutoffColor;
```
```c++
//为0时表示内侧， 1表示外侧
float facing = i.facing * 0.5 + 0.5;

//纹理采样
fixed4 col = tex2D(_MainTex, i.uv_MainTex);
col *= _Color;
o.Albedo = col.rgb * facing;
o.Metallic = _Metallic * facing;
o.Smoothness = _Smoothness * facing;
o.Emission = lerp(_CutoffColor, _Emission, facing);
```

![](https://www.ronja-tutorials.com/assets/images/posts/021/Result.gif)

在裁剪面上的颜色显示还是有些不正常，因为还会受到环境光的影响，但是这需要我们重写全局光才能排除其影响，本文并不打算深入这一块。

## Source

```c++
Shader "Tutorial/021_Clipping_Plane"{
    //材质面板
    Properties{
        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
        _Smoothness ("Smoothness", Range(0, 1)) = 0
        _Metallic ("Metalness", Range(0, 1)) = 0
        [HDR]_Emission ("Emission", color) = (0,0,0)

        [HDR]_CutoffColor("Cutoff Color", Color) = (1,0,0,0)
    }

    SubShader{
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        // 双面渲染
        Cull Off

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

        float4 _Plane;

        float4 _CutoffColor;

        //表面着色器输入数据
        struct Input {
            float2 uv_MainTex;
            float3 worldPos;
            float facing : VFACE;
        };

        //表面着色器
        void surf (Input i, inout SurfaceOutputStandard o) {
            //计算点到面的距离
            float distance = dot(i.worldPos, _Plane.xyz);
            distance = distance + _Plane.w;
            //移除平面上的点
            clip(-distance);

            float facing = i.facing * 0.5 + 0.5;

            //纹理采样
            fixed4 col = tex2D(_MainTex, i.uv_MainTex);
            col *= _Color;
            o.Albedo = col.rgb * facing;
            o.Metallic = _Metallic * facing;
            o.Smoothness = _Smoothness * facing;
            o.Emission = lerp(_CutoffColor, _Emission, facing);
        }
        ENDCG
    }
    FallBack "Standard" //后补着色器
}
```

本文的裁剪方案可以实现模型消失的效果，也可以实现简单的水在容器中的效果。希望这个教程让你有所收获。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/021_Clipping_Plane/ClippingPlane.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/021_Clipping_Plane/ClippingPlane.cs)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/021_Clipping_Plane/ClippingPlane.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/021_Clipping_Plane/ClippingPlane.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!


