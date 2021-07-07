---
title: Postprocessing with Normal Texture
date: 2021-07-06 15:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Postprocessing with Normal Texture](https://www.ronja-tutorials.com/post/018-postprocessing-normal/)

## Summary

处理深度图外，场景的法向纹理也是后处理中可能会用到的数据，同样也是通过简单的配置摄像机就可以获得。法向纹理记录了屏幕上每个像素点所对应模型表面的法向向量。

法向纹理的操作和深度图类似，所以如果你对深度图还不了解，建议你先从[上一篇](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Postprocessing_with_the_Depth_Texture/)教程看起，有助于你对本篇的理解。
![](https://www.ronja-tutorials.com/assets/images/posts/018/Result.png)

## Read Depth and Normals

本篇教程延续并使用[上一篇](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Postprocessing_with_the_Depth_Texture/)教程的着色器脚本，然后在此基础上进行扩展。

首先我们将后处理脚本中的内容清空，在上一个教程中，这个脚本主要用来刷新波的位置。然后我们将摄像机的深度模式改为深度法向模式。这样摄像机会同时采集场景的深度和法向信息。

```c#
private void Start(){
    //将摄像机的深度模式改为深度法向模式
    cam = GetComponent<Camera>();
    cam.depthTextureMode = cam.depthTextureMode | DepthTextureMode.DepthNormals;
}
```

设置完成后，我们就可以在着色器中访问法向图了，那么接下来修改我们的着色器。

在着色器中，我们也将和波有关的代码删除，然后将`_CameraDepthTexture`改为`_CameraDepthNormalsTexture`。
```c++
//编辑器上显示的属性
Properties{
    [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
}
```

```c++
//深度法向纹理
sampler2D _CameraDepthNormalsTexture;
```

设置好这些后，我们可以在片段着色器中使用我们的深度法向图。如果将其显示在屏幕上，你会发现有趣的现象。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度法向纹理采样
    float4 depthnormal = tex2D(_CameraDepthNormalsTexture, i.uv);

    return depthnormal;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/EncodedNormals.png)

但是上面的画面并不是真正的法向纹理，我们只看到近处的红绿颜色，和远处的蓝色。这是因为`_CameraDepthNormalsTexture`是中存储的深度和法向数据是经过编码的，所以使用之前需要对其解码。Unity也为我们提供了相应的解码函数。该解码函数有三个参数，第一个参数是采样值，后两个参数分别是解码后的深度、和法向。

和之前的深度图不同的是，这里解码后的深度值已经是线性的了，所以我们可以直接还原深度值。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度法向采样
    float4 depthnormal = tex2D(_CameraDepthNormalsTexture, i.uv);

    //深度法向解码
    float3 normal;
    float depth;
    DecodeDepthNormal(depthnormal, depth, normal);

    //深度值还原
    depth = depth * _ProjectionParams.z;

    return depth;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/Depth.png)

继续回到我们的主题法向纹理，我们可以将其显示到屏幕上。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度法向采样
    float4 depthnormal = tex2D(_CameraDepthNormalsTexture, i.uv);

    //深度法向解码
    float3 normal;
    float depth;
    DecodeDepthNormal(depthnormal, depth, normal);

    //还原深度值
    depth = depth * _ProjectionParams.z;
    //显示法向
    return float4(normal, 1);
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/DecodedNormals.gif)

但是当我们转动摄像机时，你会发现模型表面的法向一直在变，这是因为我们的法向纹理是基于摄像机空间生成的。所以如果我们需要额外的一步来将其转换到世界坐标系。从摄像机坐标系转换到世界坐标系很简单，但是Unity并没有提供相应的函数。因此我们需要实现，并且将转换矩阵传递给着色器。

回到后处理脚本中，我们获取到用于后处理的摄像机组件，并把它存储为脚本的成员属性，然后在`OnRenderImage`函数中，将变换矩阵传递给着色器，这样在我们转动摄像机的时候，都能及时刷新着色器中的变换矩阵。
```C#
using UnityEngine;

//挂载到后处理摄像机上
public class NormalPostprocessing : MonoBehaviour {
	//后处理材质球
	[SerializeField]
	private Material postprocessMaterial;

	private Camera cam;

	private void Start(){
		//设置摄像机
		cam = GetComponent<Camera>();
		cam.depthTextureMode = cam.depthTextureMode | DepthTextureMode.DepthNormals;
	}

	//场景渲染完后会调用该函数
	private void OnRenderImage(RenderTexture source, RenderTexture destination){
		//观察坐标系到世界坐标系的变换矩阵
		Matrix4x4 viewToWorld = cam.cameraToWorldMatrix;
		postprocessMaterial.SetMatrix("_viewToWorld", viewToWorld);
		//执行后处理
		Graphics.Blit(source, destination, postprocessMaterial);
	}
}
```

然后在着色器中使用观察坐标系到世界坐标系的变换矩阵，将法向量转换到世界坐标系。
```c++
//着色器中的变换矩阵
float4x4 _viewToWorld;
```
```c++
//将法向量转换到世界坐标系
normal = normal = mul((float3x3)_viewToWorld, normal);
return float4(normal, 1);
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/WorldspaceNormals.gif)

## Color the Top

知道了世界坐标系下的法向量，我们可以实现一些简单的效果，使得模型看起来有层次感。这里我们给模型顶部上色，也就是法向朝上的区域。

因此，我们将法向量和向上的向量相比较。通过两者的点乘，可以知道两个向量之间的关系，为1使同向，为0时垂直，为-1时反向。
```c++
float up = dot(float3(0,1,0), normal);
return up;
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/Topness.png)

上面的图可能还不是很明显，为了凸出向上的区域，我们可以使用`step`将表面区域绝对划分为向上和非向上。下面我们将这个划分阈值设置为0.5，阈值越大被认定的顶部区域越小。
```c++
float up = dot(float3(0,1,0), normal);
up = step(0.5, up);
return up;
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/TopCutoff.png)

接下来我们将原图和我们生成的顶部图融合，其中非顶部区域采用原图颜色，顶部区域采用白色。前面我们讲过很多次了，这种效果可以使用插值。
```c++
float up = dot(float3(0,1,0), normal);
up = step(0.5, up);
float4 source = tex2D(_MainTex, i.uv);
float4 col = lerp(source, float4(1,1,1,1), up);
return col;
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/WhiteTop.png)

最后，我们可以将其中的阈值、和顶部颜色放到材质面板上，这样我们可以更灵活的控制我们的后处理效果。
```c++
_upCutoff ("up cutoff", Range(0,1)) = 0.7
_topColor ("top color", Color) = (1,1,1,1)
```
```c++
//阈值和顶部颜色
float _upCutoff;
float4 _topColor;
```

然后我们将片段着色器中的阈值和顶部颜色固定值改为上面定义的变量，同时我们还可以将顶部颜色的透明通道应用上。通过调节其透明通道，可以实现顶部颜色和原图的混合效果。
```c++
 float up = dot(float3(0,1,0), normal);
up = step(_upCutoff, up);
float4 source = tex2D(_MainTex, i.uv);
float4 col = lerp(source, _topColor, up * _topColor.a);
return col;
```
![](https://www.ronja-tutorials.com/assets/images/posts/018/Result.png)

以上展示了深度法向纹理的使用方法。当然如果你想实现雪覆盖在模型上的效果，你可以直接在模型着色器上实现，而不是通过后处理的方式。

## Source

```C#
using UnityEngine;

//挂载到后处理摄像机上
public class NormalPostprocessing : MonoBehaviour {
    //后处理材质球
    [SerializeField]
    private Material postprocessMaterial;

    private Camera cam;

    private void Start(){
        //设置后摄像机
        cam = GetComponent<Camera>();
        cam.depthTextureMode = cam.depthTextureMode | DepthTextureMode.DepthNormals;
    }

    //当场景渲染完后，会执行该函数
    private void OnRenderImage(RenderTexture source, RenderTexture destination){
        //观察坐标系到世界坐标系的变换矩阵
        Matrix4x4 viewToWorld = cam.cameraToWorldMatrix;
        postprocessMaterial.SetMatrix("_viewToWorld", viewToWorld);
        //执行后处理
        Graphics.Blit(source, destination, postprocessMaterial);
    }
}
```
```c++
Shader "Tutorial/018_Normal_Postprocessing"{
    //材质面板
    Properties{
        [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
        _upCutoff ("up cutoff", Range(0,1)) = 0.7
        _topColor ("top color", Color) = (1,1,1,1)
    }

    SubShader{
        // 关闭剔除
        // 禁用深度缓存
        Cull Off
        ZWrite Off 
        ZTest Always

        Pass{
            CGPROGRAM
            //引入内置函数和变量
            #include "UnityCG.cginc"

            //声明顶点和片段着色器
            #pragma vertex vert
            #pragma fragment frag

            //用于后处理的原图
            sampler2D _MainTex;
            //观察坐标系到世界坐标系的变换矩阵
            float4x4 _viewToWorld;
            //深度法向纹理
            sampler2D _CameraDepthNormalsTexture;

            //自定义的可调节参数
            float _upCutoff;
            float4 _topColor;


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
                //变换到裁剪坐标系
                o.position = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            //片段着色器
            fixed4 frag(v2f i) : SV_TARGET{
                //深度法向纹理采样
                float4 depthnormal = tex2D(_CameraDepthNormalsTexture, i.uv);

                //深度法向量解码
                float3 normal;
                float depth;
                DecodeDepthNormal(depthnormal, depth, normal);

                //还原深度值
                depth = depth * _ProjectionParams.z;

                normal = mul((float3x3)_viewToWorld, normal);

                float up = dot(float3(0,1,0), normal);
                up = step(_upCutoff, up);
                float4 source = tex2D(_MainTex, i.uv);
                float4 col = lerp(source, _topColor, up * _topColor.a);
                return col;
            }
            ENDCG
        }
    }
}
```

希望我的教程能够对你有所帮助。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/018_NormalPostprocessing/NormalPostprocessing.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/018_NormalPostprocessing/NormalPostprocessing.cs)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/018_NormalPostprocessing/NormalPostprocessing.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/018_NormalPostprocessing/NormalPostprocessing.shader)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!

