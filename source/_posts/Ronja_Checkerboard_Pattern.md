---
title: Checkerboard Pattern
date: 2021-07-05 13:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Checkerboard Pattern](https://www.ronja-tutorials.com/post/011-chessboard/)

## Summary

我觉得使用着色器来生成图片比较有意思。下面我以棋盘格为例，像你们展示如何通过程序生成模型纹理的。
![](https://www.ronja-tutorials.com/assets/images/posts/011/Result.png)

## Stripes

参考前面关于[二维平面映射](https://tyson-wu.github.io/blogs/2021/07/02/Ronja_Planar_Mapping/)的教程，这里我们也是基于模型顶点的世界坐标来生成UV，这样的话，移动模型的过程中，其表面纹理在前后帧的渲染图可以无缝衔接。如果你希望生成的纹理跟随模型一起运动，那么可以选择基于模型坐标系进行计算。

首先，我们在顶点着色器中通过坐标变换，计算得到顶点的世界坐标，然后通过中间插值数据结构将计算结果传递到片段着色器。
```c++
struct v2f{
    float4 position : SV_POSITION;
    float3 worldPos : TEXCOORD0;
}

v2f vert(appdata v){
    v2f o;
    //计算裁剪坐标
    o.position = UnityObjectToClipPos(v.vertex);
    //计算世界坐标
    o.worldPos = mul(unity_ObjectToWorld, v.vertex);
    return o;
}
```

然后在片段着色器中，我们首先考虑一个维度上的棋盘格效果，也就是黑白相间的条纹效果。方法很简单，就是选取世界坐标中的一个轴向的值，然后取整，两个整数之间的数取整后的结果是一样的，也就是说取整后的值代表了两个整数之间的区域，正是我们这个里的条纹效果。

然后我们要区分条纹的奇偶顺序，因为两个相邻的条纹刚好可以用两个相邻的整数来表示，所以只需要求其整数的奇偶性。求一个整数的奇偶性，可以通过对2的求余操作来实现。然后通过奇偶性来判断其颜色。
```c++
fixed4 frag(v2f i) : SV_TARGET{
    //选择x轴的值取整
    float chessboard = floor(i.worldPos.x);
    //计算余数的一半
    chessboard = frac(chessboard * 0.5);
    //计算余数
    chessboard *= 2;
    return chessboard;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/011/1d.png)

## Checkerboard in 2d and 3d

接下来，我们处理两个轴向的棋盘格。按照前面的操作，另外在选一个轴向进行计算，这时候分别知道两个轴的奇偶性，然后两个奇偶值相加，再求一遍奇偶性，得到最终格子的颜色。实际上可以进一步优化，可以直接在取整之后就求和，然后后面的奇偶求解可以合并。
```c++
fixed4 frag(v2f i) : SV_TARGET{
    //合并两个轴向
    float chessboard = floor(i.worldPos.x) + floor(i.worldPos.y);
    //计算余数的一半
    chessboard = frac(chessboard * 0.5);
    //计算余数
    chessboard *= 2;
    return chessboard;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/011/OddEvenPattern.png)
![](https://www.ronja-tutorials.com/assets/images/posts/011/2d.png)

我们还可以进一步扩展到三个轴。
```c++
fixed4 frag(v2f i) : SV_TARGET{
    //合并三个轴向
    float chessboard = floor(i.worldPos.x) + floor(i.worldPos.y) + floor(i.worldPos.z);
    //计算余数的一半
    chessboard = frac(chessboard * 0.5);
    //计算余数
    chessboard *= 2;
    return chessboard;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/011/3d.png)

## Scaling

下面我们再给棋盘格增加一个缩放功能。我们需要在材质面板上引入缩放变量，方便后续调参。然后在片段着色器中，先将世界坐标处以这个缩放参数，然后在执行上面的操作。这样我们在材质面板上减小缩放变量时，棋盘格的大小也会变小。

除此之外，这里还有一个细微的改变，就是我们不再是对各个轴向分开取整，而是直接采样向量的方法，同时对所有轴向进行求整。
```c++
//...

//材质面板属性
Properties{
    _Scale ("Pattern Size", Range(0,10)) = 1
}

//...

float _Scale;

//...

fixed4 frag(v2f i) : SV_TARGET{
    //使用向量方法对各个轴向同时取整
    float3 adjustedWorldPos = floor(i.worldPos / _Scale);
    //各个轴向求和
    float chessboard = adjustedWorldPos.x + adjustedWorldPos.y + adjustedWorldPos.z;
    //计算余数的一半
    chessboard = frac(chessboard * 0.5);
    //计算余数
    chessboard *= 2;
    return chessboard;
}

//...
```
![](https://www.ronja-tutorials.com/assets/images/posts/011/Scaling.gif)

## Customizable Colors

最后，我们还可以增加两个变量来控制棋盘格的颜色。在片段着色器的最后我使用线性插值函数来实现两种颜色的二选一操作。
```c++
//...

//材质面板
Properties{
    _Scale ("Pattern Size", Range(0,10)) = 1
    _EvenColor("Color 1", Color) = (0,0,0,1)
    _OddColor("Color 2", Color) = (1,1,1,1)
}

//...

float4 _EvenColor;
float4 _OddColor;

//...

fixed4 frag(v2f i) : SV_TARGET{
    //同时取整
    float3 adjustedWorldPos = floor(i.worldPos / _Scale);
    //三个维度求和
    float chessboard = adjustedWorldPos.x + adjustedWorldPos.y + adjustedWorldPos.z;
    //计算余数的一半
    chessboard = frac(chessboard * 0.5);
    //计算余数
    chessboard *= 2;

    //二选一操作
    float4 color = lerp(_EvenColor, _OddColor, chessboard);
    return color;
}

//...
```
![](https://www.ronja-tutorials.com/assets/images/posts/011/colors.png)

下面是最终的棋盘格生成着色器。
```c++
Shader "Tutorial/011_Chessboard"
{
    //材质面板
    Properties{
        _Scale ("Pattern Size", Range(0,10)) = 1
        _EvenColor("Color 1", Color) = (0,0,0,1)
        _OddColor("Color 2", Color) = (1,1,1,1)
    }

    SubShader{
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}


        Pass{
            CGPROGRAM
            #include "UnityCG.cginc"

            #pragma vertex vert
            #pragma fragment frag

            float _Scale;

            float4 _EvenColor;
            float4 _OddColor;

            struct appdata{
                float4 vertex : POSITION;
            };

            struct v2f{
                float4 position : SV_POSITION;
                float3 worldPos : TEXCOORD0;
            };

            v2f vert(appdata v){
                v2f o;
                //计算裁剪坐标
                o.position = UnityObjectToClipPos(v.vertex);
                //计算世界坐标
                o.worldPos = mul(unity_ObjectToWorld, v.vertex);
                return o;
            }

            fixed4 frag(v2f i) : SV_TARGET{
                //同时取整
                float3 adjustedWorldPos = floor(i.worldPos / _Scale);
                //求和
                float chessboard = adjustedWorldPos.x + adjustedWorldPos.y + adjustedWorldPos.z;
                //计算余数的一半
                chessboard = frac(chessboard * 0.5);
                //计算余数
                chessboard *= 2;

                //二选一插值
                float4 color = lerp(_EvenColor, _OddColor, chessboard);
                return color;
            }

            ENDCG
        }
    }
    FallBack "Standard" //后补着色器
}
```

希望本篇对你有所帮助，能够让你知道如何通过程序实现模型纹理图。

你可以在以下链接找到源码：[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/011_ChessBoard/Chessboard.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/011_ChessBoard/Chessboard.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
