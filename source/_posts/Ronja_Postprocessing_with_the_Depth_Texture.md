---
title: Postprocessing with the Depth Texture
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
[Postprocessing with the Depth Texture](https://www.ronja-tutorials.com/post/017-postprocessing-depth/)

在上一篇教程中，我介绍了简单后处理效果的实现过程。但是在实际应用中，我们经常需要使用深度图来实现一些更高级的后处理效果。深度图是从摄像机视角采集的记录场景深度信息的纹理图。

在理解如何借助深度图来实现复杂的后处理之前，建议你先阅读[上一篇](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Postprocessing_Basics/)关于简单后处理效果的介绍。
![](https://www.ronja-tutorials.com/assets/images/posts/017/Result.gif)

## Read Depth

这里我们沿用[上一篇](https://tyson-wu.github.io/blogs/2021/07/06/Ronja_Postprocessing_Basics/)中实现的最简单的后处理脚本，然后在此基础上进行修改。

首先我们需要对后处理脚本进行扩展，保证后处理摄像机生成深度图，供这里的后处理使用。
```C#
private void Start(){
    Camera cam = GetComponent<Camera>();
    cam.depthTextureMode = cam.depthTextureMode | DepthTextureMode.Depth;
}
```

上面对后处理脚本修改完成后，接下来我们要对后处理着色器进行修改。

为了在着色器中访问深度图，我们首先需要定义一个名叫`_CameraDepthTexture`的纹理，这个名字是Unity内置的。深度图的采样和其他纹理一样，我们可以将采样结果渲染到屏幕上，看看深度图到底长啥样。因为深度图只有一个值有效，所以在纹理中深度值是存储在R通道，我们可以直接进行访问。
```c++
//深度图
sampler2D _CameraDepthTexture;
```
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度采样
    float depth = tex2D(_CameraDepthTexture, i.uv).r;

    return depth;
}
```

这一切都准备好后，启动游戏，不过这时候屏幕上显示的很可能是一片黑。这是因为深度值得存储位数有限，为了扩大深度值得记录范围，同时保证近景的深度精度，所以采用非线性编码，其中距离摄像机越近的区域深度值得精度越高，反之越低。当你将摄像机靠近物体时，你可能观察到更亮的颜色，这表明这个区域理摄像机很近。如果你将摄像机不断靠近，画面依然很黑，这时候你可以尝试将摄像机的近平面调大一点。
![](https://www.ronja-tutorials.com/assets/images/posts/017/Short.png)

前面的深度编码是考虑到存储的限制，而我们使用深度值之前必须对其进行解码。庆幸的是，Unity为我们提供了解码函数，解码后的深度值是线性的，范围在0-1之间，0表示在摄像机位置，1表示在远平面上。如果解码后的深度图显示除了天空盒区域是白色，其他地方基本是黑色，你可以将远平面调小，这样可以观察到更多的模型。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度采样
    float depth = tex2D(_CameraDepthTexture, i.uv).r;
    //深度解码，解码后的深度值是线性的，范围0-1， 0为摄像机位置，1为远平面上
    depth = Linear01Depth(depth);

    return depth;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/LinearDepth.png)

接下来的一步是基于摄像机参数，还原真实的深度值。这里有个`_ProjectionParams`是记录摄像机的投影参数，其中z值是远平面的大小。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度采样
    float depth = tex2D(_CameraDepthTexture, i.uv).r;
    //深度值解码
    depth = Linear01Depth(depth);
    //还原深度值，得到点到摄像机的真实距离
    depth = depth * _ProjectionParams.z;

    return depth;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/CorrectDepth.png)

因为场景中绝大多数的模型到摄像机的距离都大于一个单位，所以还原后的深度图显示在屏幕上将会是纯白色，但是这个深度值是与远平面无关的真实深度值，是点到摄像机的距离。

## Generate Wave

加下来我将基于这些信息来实现一种波效果，一种不断从玩家开始，向远处传播的效果。同时我们可以自定义某个时刻波距玩家的距离、以及波的拖尾长度、波的颜色。所以首先我们需要在着色器脚本中添加这些变量。这里我们使用`Header`属性标签来加粗标题，当然这只具有显示功能，不会影响着色器的实际使用。
```c++
//在编辑器上显示的属性
Properties{
    [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
    [Header(Wave)]
    _WaveDistance ("Distance from player", float) = 10
    _WaveTrail ("Length of the trail", Range(0,5)) = 1
    _WaveColor ("Color", Color) = (1,0,0,1)
}
```
```c++
//HLSL 内定义的变量
float _WaveDistance;
float _WaveTrail;
float4 _WaveColor;
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/Inspector.png)

我们这个波的一头是突然截断、另一头是渐变的拖尾效果。我们首先实现这个突然截断的头部效果。在前面的教程中谈到过`step`这个函数可以实现跳变的效果。
```c++
 //计算波的头部
float waveFront = step(depth, _WaveDistance);

return waveFront;
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/Cutoff.gif)

然后我们再使用`smoothstep`函数来实现尾部渐变效果，这个函数和`step`函数类似，只不过它有三个参数。如果第三个参数小于第一个参，那么返回0，如果大于第二个参数，那么返回1，其他情况返回一个0-1的值。
```c++
float waveTrail = smoothstep(_WaveDistance - _WaveTrail, _WaveDistance, depth);
return waveTrail;
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/Trail.gif)

你可能注意到上面两个波效果刚好相反，这正是我们想要的效果。因为我们将这两个波值相乘后，只有中间很窄的区域会为1，其他位置都将是0。
```c++
//计算前后波
float waveFront = step(depth, _WaveDistance);
float waveTrail = smoothstep(_WaveDistance - _WaveTrail, _WaveDistance, depth);
float wave = waveFront * waveTrail;

return wave;
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/WhiteWave.gif)

现在我们得到了想要的波，打算将其应用到最终的显示画面上。首先需要采集原始画面，然后和我们的波进行线性插值，插值的时候可以把我们的波颜色也应用上。
```c++
//和原图混合
fixed4 col = lerp(source, _WaveColor, wave);

return col;
```
![](https://www.ronja-tutorials.com/assets/images/posts/017/HitSky.gif)

上面的效果可以发现一些瑕疵，就是当波移动到远平面时，会突然高亮。虽然我们的天空盒就是在远平面处，但是我还是不想出现这种瑕疵。

要解决这个问题，我通过判断深度值是否达到远平面，如果达到，那么直接返回原始图。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //深度采样
    float depth = tex2D(_CameraDepthTexture, i.uv).r;
    //深度解码
    depth = Linear01Depth(depth);
    //还原深度值
    depth = depth * _ProjectionParams.z;

    //原图采样
    fixed4 source = tex2D(_MainTex, i.uv);
    //当达到远平面时，直接返回原图
    if(depth >= _ProjectionParams.z)
        return source;

    //计算波
    float waveFront = step(depth, _WaveDistance);
    float waveTrail = smoothstep(_WaveDistance - _WaveTrail, _WaveDistance, depth);
    float wave = waveFront * waveTrail;

    //波和原图混合
    fixed4 col = lerp(source, _WaveColor, wave);

    return col;
}
```

最后，我想扩展后处理脚本来实现自动设置波位置，并且让它缓慢远离摄像机。我想控制波速以及是否启用波后处理效果。所以我必须记住当前波的位置。下面是我添加的新变量。
```C#
[SerializeField]
private Material postprocessMaterial;
[SerializeField]
private float waveSpeed;
[SerializeField]
private bool waveActive;
```

然后我在后处理脚本中的`Update`函数中不断刷新波的位置。关闭波效将会重置波的位置，开启波效，波都会冲初始位置开始，慢慢的原理摄像机。
```C#
private void Update(){
    //启用时会不断移动波，关闭时会重置波的位置
    if(waveActive){
        waveDistance = waveDistance + waveSpeed * Time.deltaTime;
    } else {
        waveDistance = 0;
    }
}
```

然后我在`OnRenderImage`函数中的后处理操作执行之前，将波距离参数传递给着色器，保证每次渲染都是在正确的位置。
![](https://www.ronja-tutorials.com/assets/images/posts/017/AutoWave.gif)

```c++
Shader "Tutorial/017_Depth_Postprocessing"{
    //显示在编辑器上
    Properties{
        [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
        [Header(Wave)]
        _WaveDistance ("Distance from player", float) = 10
        _WaveTrail ("Length of the trail", Range(0,5)) = 1
        _WaveColor ("Color", Color) = (1,0,0,1)
    }

    SubShader{
        // 关闭剔除
        // 禁用深度缓存功能
        Cull Off
        ZWrite Off
        ZTest Always

        Pass{
            CGPROGRAM
            //引入内置函数和变量
            #include "UnityCG.cginc"

            //声明顶点、片段着色器
            #pragma vertex vert
            #pragma fragment frag

            //用于后处理的原图
            sampler2D _MainTex;

            //深度图
            sampler2D _CameraDepthTexture;

            //波参数
            float _WaveDistance;
            float _WaveTrail;
            float4 _WaveColor;


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
                o.uv = v.uv;
                return o;
            }

            //片段着色器，后处理操作主要是在这里执行
            fixed4 frag(v2f i) : SV_TARGET{
                //深度采样
                float depth = tex2D(_CameraDepthTexture, i.uv).r;
                //深度解码
                depth = Linear01Depth(depth);
                //深度还原
                depth = depth * _ProjectionParams.z;

                //原图采样
                fixed4 source = tex2D(_MainTex, i.uv);
                //当达到远平面时，直接返回原图
                if(depth >= _ProjectionParams.z)
                    return source;

                //计算波
                float waveFront = step(depth, _WaveDistance);
                float waveTrail = smoothstep(_WaveDistance - _WaveTrail, _WaveDistance, depth);
                float wave = waveFront * waveTrail;

                //原图与波混合
                fixed4 col = lerp(source, _WaveColor, wave);

                return col;
            }
            ENDCG
        }
    }
}
```
```c#
using UnityEngine;

//该脚本需要和摄像机绑定在同一物体
public class DepthPostprocessing : MonoBehaviour {
    //用于后处理的材质
    [SerializeField]
    private Material postprocessMaterial;
    [SerializeField]
    private float waveSpeed;
    [SerializeField]
    private bool waveActive;

    private float waveDistance;

    private void Start(){
        //设置当前摄像机为深度采集模式
        Camera cam = GetComponent<Camera>();
        cam.depthTextureMode = cam.depthTextureMode | DepthTextureMode.Depth;
    }

    private void Update(){
        //启用时会不断移动波，关闭时会重置波的位置
        if(waveActive){
            waveDistance = waveDistance + waveSpeed * Time.deltaTime;
        } else {
            waveDistance = 0;
        }
    }

	//当当前绑定的摄像机渲染完一帧画面后，会调用该函数
    private void OnRenderImage(RenderTexture source, RenderTexture destination){
        //同步当前波距到着色器
        postprocessMaterial.SetFloat("_WaveDistance", waveDistance);
		//将原图按照材质着色器脚本逻辑，写入到结果图
        Graphics.Blit(source, destination, postprocessMaterial);
    }
}
```

希望我的教程能够对你有所帮助。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/017_DepthPostprocessing/DepthPostprocessing.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/017_DepthPostprocessing/DepthPostprocessing.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/017_DepthPostprocessing/DepthPostprocessing.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/017_DepthPostprocessing/DepthPostprocessing.cs)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!