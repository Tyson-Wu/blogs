---
title: Unity Shader 函数集
date: 2020-11-12 17:01:34
categories:
- [Unity, Shader, Command]
tags:
- Unity
- Shader
- CG
---

## DecodeDepthNormal(depthNormalSamp,depth,normal)

### description
depthNormalSamp是由[DepthTextureMode.DepthNormals](https://docs.unity3d.com/Manual/SL-CameraDepthTexture.html)模式创建的深度法向纹理。此方法将纹理数据解析为深度、法向量，并返回给depth、normal两个数据。得到的depth是一个范围为[0-1]的小数，normal是采用立体投影的方式编码，所以从三维向量变成二维向量。
### etc
``` bash
...
DecodeDepthNormal(tex2D(_CameraDepthNormalsTexture, i.scrPos.xy), depthValue, normalValues);
...
```

## UNITY_TRANSFER_DEPTH(o)

### description
参数是2维向量，计算顶点在摄像机空间下的深度值。如果需要获取深度图，应该在顶点着色器中使用该宏。有些具有本地深度图的平台上，该宏不起任何作用，因为这些平台会默认将深度信息存储在Z缓存中。和UNITY_OUTPUT_DEPTH(i)搭配使用
### etc

``` bash
Shader "Render Depth" {
    SubShader {
        Tags { "RenderType"="Opaque" }
        Pass {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct v2f {
                float4 pos : SV_POSITION;
                float2 depth : TEXCOORD0;
            };

            v2f vert (appdata_base v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                UNITY_TRANSFER_DEPTH(o.depth);
                return o;
            }

            half4 frag(v2f i) : SV_Target {
                UNITY_OUTPUT_DEPTH(i.depth);
            }
            ENDCG
        }
    }
}
```

## COMPUTE_EYEDEPTH (i)

### description
参数是顶点坐标，计算顶点在摄像机空间下的深度值，并直接返回。当我们不需要深度图时，但是需要获取顶点的深度信息，我们可以用这个宏。这个宏在顶点着色器中使用
### etc

## UNITY_OUTPUT_DEPTH(i)

### description
参数是2维向量，计算顶点在摄像机空间下的深度值。如果需要获取深度图，应该在片段着色器中使用该宏。有些具有本地深度图的平台上，该宏不起任何作用，因为这些平台会默认将深度信息存储在Z缓存中。和UNITY_TRANSFER_DEPTH(o)搭配使用
### etc

## COMPUTE_EYEDEPTH (i)

### description
参数是顶点坐标，计算顶点在摄像机空间下的深度值，并直接返回。当我们不需要深度图时，但是需要获取顶点的深度信息，我们可以用这个宏。这个宏在顶点着色器中使用
### etc

## DECODE_EYEDEPTH(i)/LinearEyeDepth(i)

### description
参数是深度图的值，从深度图中解析出实际的深度值。深度图中的深度值都是编码过的。
### etc
下面代码第一行是从深度图中采集深度值，这个深度值范围是[0-1],但是属于非均匀分布，同样一个单位的长度，在靠近摄像机的情况下远大于远离摄像机的情况。第二行用来解析深度值，解析后的值是正常的距离单位，假设是3，则表示距离摄像机3个单位。
``` bash
float existingDepth01 = tex2Dproj(_CameraDepthTexture, UNITY_PROJ_COORD(i.screenPosition)).r;
float existingDepthLinear = LinearEyeDepth(existingDepth01);
```

## Linear01Depth(i)

### description
参数是深度图的值，从深度图中解析出深度值。深度值均匀分布，但是进行缩放，深度范围0-1。0表示远平面，1表示近平面。
### etc

## UNITY_PROJ_COORD(a)

### description
参数是四维向量，然后返回一个适合投影贴图用的纹理坐标。在大多数平台，是直接返回向量a。通常情況下a是屏幕坐标
### etc

## tex2Dlod

### synopsis
``` bash
float4 tex2Dlod(sampler2D samp, float4 s)
float4 tex2Dlod(sampler2D samp, float4 s, int texelOff)

int4 tex2Dlod(isampler2D samp, float4 s)
int4 tex2Dlod(isampler2D samp, float4 s, int texelOff)

unsigned int4 tex2Dlod(usampler2D samp, float4 s)
unsigned int4 tex2Dlod(usampler2D samp, float4 s, int texelOff)
```
### parameters
samp : 纹理题图
s.xy : 纹理坐标
s.w : 多层次细节的图层层级
texelOff : 像素偏移量。
### description
基于纹理坐标获得纹理图片指定像素位置，然后在此基础上再做偏移，然后返回最终数据

## saturate

### synopsis
``` bash
float  saturate(float x);
float1 saturate(float1 x);
float2 saturate(float2 x);
float3 saturate(float3 x);
float4 saturate(float4 x);

half   saturate(half x);
half1  saturate(half1 x);
half2  saturate(half2 x);
half3  saturate(half3 x);
half4  saturate(half4 x);

fixed  saturate(fixed x);
fixed1 saturate(fixed1 x);
fixed2 saturate(fixed2 x);
fixed3 saturate(fixed3 x);
fixed4 saturate(fixed4 x);
```
### parameters
x : 值或向量
### description
x的值或其元素被限定在0-1中，例如，当x小于0时，返回0；当x大于1时，返回1；

## tex2Dproj

### synopsis
``` bash
float4 tex2Dproj(sampler2D samp, float3 s)
float4 tex2Dproj(sampler2D samp, float3 s, int texelOff)
float4 tex2Dproj(sampler2D samp, float4 s)
float4 tex2Dproj(sampler2D samp, float4 s, int texelOff)

int4 tex2Dproj(isampler2D samp, float3 s)
int4 tex2Dproj(isampler2D samp, float3 s, int texelOff)

unsigned int4 tex2Dproj(usampler2D samp, float3 s)
unsgined int4 tex2Dproj(usampler2D samp, float3 s, int texelOff)
```
### parameters
samp : 采样纹理
s : 采样坐标。采样坐标首先需要执行投影操作，也就是前2个坐标除以第3个，得到的两个坐标值用于纹理采样。如果还有第4个坐标，那么这个值是用于阴影比较。
texelOff ：像素偏移值。当使用s坐标计算出纹理像素坐标时，偏移一定的像素进行采样
### description
通过坐标s对纹理进行采样，采样坐标首先需要执行投影操作，也就是前2个坐标除以第3个，得到的两个坐标值用于纹理采样。如果还有第4个坐标，那么这个值是用于阴影比较。


