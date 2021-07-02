---
title: Triplanar Mapping
date: 2021-07-02 13:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Triplanar Mapping](https://www.ronja-tutorials.com/post/010-triplanar-mapping/)

## Summary

在前面我们介绍过二维平面映射的实现方法，这里我们来讲讲三维平面的映射方法。
纳尼？平面本身是二维的叫二维平面还可以理解，你这来个三维平面，是欺负我读书少，想糊弄我？？？
稍安勿躁！首先专业名字本身依据其专业用途、含义来取的，很容易和我们习惯相冲突，比如数学领域各种眼花缭乱的术语。这里我们的三维平面更多的值得是三维空间上的平面，可以有三个维度的取值。之前提到的二维平面映射，是只从一个方向进行投影，换句话说，我们只用沿着其投影方向进行渲染，才能看到我们的纹理贴合在模型表面，如果换个角度，你可能就看不到了，即便看到了也可能是模糊不清的。而三维平面映射，是从分别从三个维度进行投影映射，然后将得到的三个纹理颜色进行混合，这样无论我们采用怎样刁钻的角度，也挑不出啥毛病。

当然，本文也是在之前的二维平面映射的基础上扩展的，在了解其原理后，你也可以使用表面着色器重写一遍。
![](https://www.ronja-tutorials.com/assets/images/posts/010/Result.gif)

## Calcualte Projection Planes

首先，为了得到三个不同方向的UV坐标，我们需要改变UV坐标的生成方式。在二维平面映射中，我们是在顶点着色器中进行UV变换。这里我们直接将顶点的世界坐标传递到片段着色器中，然后在片段着色器中进行uv变换。
```c++
struct v2f{
    float4 position : SV_POSITION;
    float3 worldPos : TEXCOORD0;
};

v2f vert(appdata v){
    v2f o;
    //计算裁剪空间坐标
    o.position = UnityObjectToClipPos(v.vertex);
    //计算世界坐标
    float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
    o.worldPos = worldPos.xyz;
    return o;
}
```

接下来我们对三个方向投影所对应的uv坐标进行UV变换。在这里我把世界坐标的`y`轴对应uv坐标的`v`，这样渲染出来的纹理就是正的。当然，你也可以随意尝试多种对应关系，看看会有什么不一样的效果。
```c++
fixed4 frag(v2f i) : SV_TARGET{
    //分别计算三个投影方向的uv变换
    float2 uv_front = TRANSFORM_TEX(i.worldPos.xy, _MainTex);
    float2 uv_side = TRANSFORM_TEX(i.worldPos.zy, _MainTex);
    float2 uv_top = TRANSFORM_TEX(i.worldPos.xz, _MainTex);
}
```

然后使用变换后的uv值进行纹理采样，并将三个不同的采样值进行平均。当然你也可以直接求和，不过最终结果会显得非常亮。
```c++
//分别对三个方向进行纹理采样
fixed4 col_front = tex2D(_MainTex, uv_front);
fixed4 col_side = tex2D(_MainTex, uv_side);
fixed4 col_top = tex2D(_MainTex, uv_top);

//求平均值
fixed4 col = (col_front + col_side + col_top) / 3;

//叠加材质颜色
col *= _Color;
return col;
```
![](https://www.ronja-tutorials.com/assets/images/posts/010/AllSides.png)

## Normals

到目前为止，你会发现整个材质表现的非常怪异，各种重影迭起，这是因为我们只是单纯的对三个方向的采样值进行平均。为了消除这种重影，我们可以根据不同的朝向，侧重显示对应朝向的采样值。表面朝向有个专业点的名称：法向向量。在我们的网格数据中就包含法向数据。因为一些特殊考虑，网格数据中的法向和顶点是一一对应的。

所以，我们首先要做的是在我们的输入结构体中加入法向变量，然后在顶点着色其中将其变换到世界坐标系，并且通过插值数据传入到片段着色器中参与后续的计算。这里之所以要变换到世界坐标系，是因为我们的纹理映射是基于世界坐标系的。换句话说，我们在进行计算时，应该保证空间数据的空间一致性。



