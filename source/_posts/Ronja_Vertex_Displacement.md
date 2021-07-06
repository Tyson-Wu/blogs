---
title: Vertex Displacement
date: 2021-07-06 13:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Vertex Displacement](https://www.ronja-tutorials.com/post/015-wobble-displacement/)

目前为止，我们使用到最多的就是裁剪坐标系和世界坐标系，其实在顶点着色器中，我们能做的远远不止这些。接下来我将介绍如果将三角函数应用到模型上，从而实现模型抖动效果。

本篇的例子是采用表面着色器，如果你对表面着色器还不了解的话，建议你先从[这篇教程](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Surface_Shader_Basics/)看起。当然本篇介绍的思路可以用于到其他着色器上。
![](https://www.ronja-tutorials.com/assets/images/posts/015/Result.gif)

一般我们对顶点坐标的操作都是在顶点着色器中，而我们的表面着色器中似乎并没有顶点着色器函数，实际上表面着色器最终会被翻译为顶点、片段着色器，只不过这些都是由Unity来完成。而在表面着色器中其实还有一个和顶点着色器同名的函数，也是用来处理顶点数据的，只不过定义的时候是和表面着色器一起定义的。
```c++
    //表面着色器
    //表面着色器函数和标准光照模型
    //fullforwardshadows 使用所有的阴影Pass
    //vertex:vert 用来处理顶点变换
    #pragma surface surf Standard fullforwardshadows vertex:vert
```

然后我们需要去实现这个顶点处理函数。在无光照的着色器中，我们是在顶点着色器函数中处理裁剪变换。而在表面着色器中，这里的顶点处理函数并不需要处理裁剪变换，因为那些基础部分都由Unity自动生成。这我们只需要处理顶点坐标，然后将处理后的结果传给下一步由Unity自动生成的代码处理。

可以这么说，这里的顶点处理函数是在普通的顶点着色器函数之前执行的，所以顶点处理函数的输入参数也是模型网格数据。这里可以使用Unity提供给我们的`appadata_full`，也可以自定义。

和表面着色器函数一样，这里的顶点处理函数也不返回任何值，而是通过`inout`来向外部传递结果。

因为在表面着色器中，所有必要的顶点变换都是由Unity自动生成的，所以定义一个空的顶点处理函数并不会影响原先的表面着色器。
```c++
void vert(inout appdata_full data){

}
```

比较简单的顶点处理就是给所有的顶点乘以一个缩放因子，这样我们就可以控制模型变大变小。
```c+++
void vert(inout appdata_full data){
    data.vertex.xyz *= 2;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/ShadowRealm.png)

虽然模型变大了，但是整个显示却变得不正常了。这里的阴影还是基于原来的为改变的模型顶点来计算的。这是因为表面着色器并不会根据需求自动生成阴影Pass，而依然是复制已有的阴影Pass。为了解决这个问题，我们可以定义`addshadow`关键字，这样错误的阴影就会消失了。
```c++
//addshadows 是告诉表面着色器，基于顶点处理函数，重现创建一个阴影Pass
#pragma surface surf Standard fullforwardshadows vertex:vert addshadow
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/FixedShadows.png)

仅仅是缩放模型显得单调了，接下来我们可以实现更有趣的效果。通过计算顶点坐标的x值得三角函数，来改变器y值，从而产生一种波动的效果。
```c++
void vert(inout appdata_full data){
    data.vertex.y += sin(data.vertex.x);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/QueerMonkey.png)

上面的结果表明当前使用的三角函数波形较大、频率低，因此我们增加两个控制波形的变量。
```c++
//...

_Amplitude ("Wave Size", Range(0,1)) = 0.4
_Frequency ("Wave Freqency", Range(1, 8)) = 2

//...

float _Amplitude;
float _Frequency;

//...

void vert(inout appdata_full data){
float4 modifiedPos = data.vertex;
modifiedPos.y += sin(data.vertex.x * _Frequency) * _Amplitude;
data.vertex = modifiedPos;

//...
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/Sliders.png)
![](https://www.ronja-tutorials.com/assets/images/posts/015/WobblyMonkey.png)

现在我们可以很好地控制我们的模型波形了，但是在顶点处理函数中只处理了顶点坐标，而没有同时处理法向量，因此法向量相对应模型表面来说实际上是不匹配的。
![](https://www.ronja-tutorials.com/assets/images/posts/015/WrongNormalsExplanation.png)

这里最简单且最灵活的计算自定义模型表面法向量的方法是，通过采集模型表面上的点来重新计算法向量。

理论上来说，我们可以采集变形后的局部区域的任意点来计算切平面，进而计算法向量。但是我们需要充分利用已有数据来解析这个切平面。首先对于切向空间我们需要有所了解，在切向空间中，法向量叫`normal`，切向向量叫`tangent`，还有一个叫不出名字的`bitangent`，这三个向量相互垂直，构成切向空间的三个轴。如下图所示，蓝色是法向量，红色是切向量，黄色是`bitangent`。其中变形前的切向量和法向量都可以从模型网格数据中获取。所以变形前的`bitangent`可以通过前两者的叉乘来计算。
![](https://www.ronja-tutorials.com/assets/images/posts/015/ShowNormals.png)

在知道变形前的切向向量和`bitangent`就表示我们知道变形前的切平面，那么计算变形后的切平面我们同样可以先计算变形后的切向量和`bitangent`。因为这两个向量是沿着模型表面一同变形的，所以可以使用前面的波形函数计算两个向量变形后的方向，然后再通过叉乘来计算变形后的法向量。
```c++
//求解变形后的切向方向临近点的坐标
float3 posPlusTangent = data.vertex + data.tangent * 0.01;
posPlusTangent.y += sin(posPlusTangent.x * _Frequency) * _Amplitude;
//求解变形后的bitangent方向临近点的坐标
float3 bitangent = cross(data.normal, data.tangent);
float3 posPlusBitangent = data.vertex + bitangent * 0.01;
posPlusBitangent.y += sin(posPlusBitangent.x * _Frequency) * _Amplitude;
```

上面求解了两个临近点变形后的位置，加上前面计算好的顶点变形后的位置，我们就可以得到变性后的切向平面，然后求切平面的法向量。
```c++
void vert(inout appdata_full data){
    //求解变形后的顶点坐标
    float4 modifiedPos = data.vertex;
    modifiedPos.y += sin(data.vertex.x * _Frequency) * _Amplitude;
    //求解变形后的切向方向临近点的坐标
    float3 posPlusTangent = data.vertex + data.tangent * 0.01;
    posPlusTangent.y += sin(posPlusTangent.x * _Frequency) * _Amplitude;
    //求解变形后的bitangent方向临近点的坐标
    float3 bitangent = cross(data.normal, data.tangent);
    float3 posPlusBitangent = data.vertex + bitangent * 0.01;
    posPlusBitangent.y += sin(posPlusBitangent.x * _Frequency) * _Amplitude;
    //求解变形后的切平面
    float3 modifiedTangent = posPlusTangent - modifiedPos;
    float3 modifiedBitangent = posPlusBitangent - modifiedPos;
    //求解变形后切平面的法向量，也就是模型变形后的法向量
    float3 modifiedNormal = cross(modifiedTangent, modifiedBitangent);
    data.normal = normalize(modifiedNormal);
    data.vertex = modifiedPos;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/CorrectNormals.png)

最后我希望我们的波形抖动随着时间变化而变化。前面我们只采用了模型顶点坐标的x值作为波形函数的参数，从而得到变形后的坐标，在此基础上引入时间变量是非常简单的。

Unity向着色器中传递的时间变量是一个四维向量，其中第一个元素的值是时间处以20，第二是是时间，第三个是时间成以2，第四个是时间乘以三，这里的时间都是以秒为单位。这里我们选择第二个参数时间来控制波形，另外我们还需要控制波形动画速度的变量。
```c++
_AnimationSpeed ("Animation Speed", Range(0,5)) = 1

//...

float _AnimationSpeed;

//...

void vert(inout appdata_full data){
    float4 modifiedPos = data.vertex;
    modifiedPos.y += sin(data.vertex.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;
    
    float3 posPlusTangent = data.vertex + data.tangent * 0.01;
    posPlusTangent.y += sin(posPlusTangent.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;

    float3 bitangent = cross(data.normal, data.tangent);
    float3 posPlusBitangent = data.vertex + bitangent * 0.01;
    posPlusBitangent.y += sin(posPlusBitangent.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;

    float3 modifiedTangent = posPlusTangent - modifiedPos;
    float3 modifiedBitangent = posPlusBitangent - modifiedPos;

    float3 modifiedNormal = cross(modifiedTangent, modifiedBitangent);
    data.normal = normalize(modifiedNormal);
    data.vertex = modifiedPos;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/015/Result.gif)

上面计算临近点的时候我们是使用0.01个偏移来是变形更加平滑。这个值越小，其变形变越明显，越大，整个形变越光滑。
```c++
Shader "Tutorial/015_vertex_manipulation" {
    //材质面板
    Properties {
        _Color ("Tint", Color) = (0, 0, 0, 1)
        _MainTex ("Texture", 2D) = "white" {}
        _Smoothness ("Smoothness", Range(0, 1)) = 0
        _Metallic ("Metalness", Range(0, 1)) = 0
        [HDR] _Emission ("Emission", color) = (0,0,0)

        _Amplitude ("Wave Size", Range(0,1)) = 0.4
        _Frequency ("Wave Freqency", Range(1, 8)) = 2
        _AnimationSpeed ("Animation Speed", Range(0,5)) = 1
    }
    SubShader {
        //不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        CGPROGRAM

        //表面着色器
        //表面着色器函数和标准光照模型
        //fullforwardshadows 使用所有的阴影Pass
        //vertex:vert 用来处理顶点变换
        //addshadows 是告诉表面着色器，基于顶点处理函数，重现创建一个阴影Pass
        #pragma surface surf Standard fullforwardshadows vertex:vert addshadow
        #pragma target 3.0

        sampler2D _MainTex;
        fixed4 _Color;

        half _Smoothness;
        half _Metallic;
        half3 _Emission;

        float _Amplitude;
        float _Frequency;
        float _AnimationSpeed;

        //表面着色器的输入数据
        struct Input {
            float2 uv_MainTex;
        };

        void vert(inout appdata_full data){
            float4 modifiedPos = data.vertex;
            modifiedPos.y += sin(data.vertex.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;
            
            float3 posPlusTangent = data.vertex + data.tangent * 0.01;
            posPlusTangent.y += sin(posPlusTangent.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;

            float3 bitangent = cross(data.normal, data.tangent);
            float3 posPlusBitangent = data.vertex + bitangent * 0.01;
            posPlusBitangent.y += sin(posPlusBitangent.x * _Frequency + _Time.y * _AnimationSpeed) * _Amplitude;

            float3 modifiedTangent = posPlusTangent - modifiedPos;
            float3 modifiedBitangent = posPlusBitangent - modifiedPos;

            float3 modifiedNormal = cross(modifiedTangent, modifiedBitangent);
            data.normal = normalize(modifiedNormal);
            data.vertex = modifiedPos;
        }

        //表面着色器函数
        void surf (Input i, inout SurfaceOutputStandard o) {
            //纹理采样
            fixed4 col = tex2D(_MainTex, i.uv_MainTex);
            col *= _Color;
            o.Albedo = col.rgb;
            //设置标准光照参数
            o.Metallic = _Metallic;
            o.Smoothness = _Smoothness;
            o.Emission = _Emission;
        }
        ENDCG
    }
    FallBack "Standard"
}
```

希望本篇能启发你对模型顶点处理的思考，然后创造出美轮美奂的效果。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/015_VertexManipulation/vertexmanipulation.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/015_VertexManipulation/vertexmanipulation.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!