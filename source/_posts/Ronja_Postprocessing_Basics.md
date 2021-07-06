---
title: Postprocessing Basics
date: 2021-07-06 14:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Postprocessing Basics](https://www.ronja-tutorials.com/post/016-postprocessing-basics/)

## Summary

目前为止，我将所有我实现的着色器应用到了模型上，并将其渲染到屏幕上。着色器还有一个常用的用途就是处理图片、以及我们刚渲染好的上一帧画面。我们对前面渲染好的画面进行处理的操作就叫做后处理。

后处理所使用的着色器在语法和结构上和之间介绍的着色器一样。所以我建议你先了解前面关于着色器的[基础教程](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)。
![](https://www.ronja-tutorials.com/assets/images/posts/016/Result.jpg)

## Postprocessing Shader

作为后处理的入门教程，这里我将展示如何实现简单的颜色取反效果。

因为整个脚本和其他着色器类似，所以我将直接使用[着色器基础](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)教程中的着色器脚本，并在此基础上进行修改。

当然，即便是最基础的着色器也有一些后处理用不到的变量，我们可以将它删除。例如这里的材质颜色、渲染标签`Tags`、纹理参数。

然后我们还需要添加一些东西，使得我们的着色器更适用于后处理。例如在后处理中所有的属性变量都是通过脚本赋值的，所以在属性块中可以加入属性隐藏标签，另外后处理操作不应该对影响场景深度图，所以应该禁用深度写入等功能。

基于上述修改，最终的着色器如下：
```c++
Shader "Tutorial/016_Postprocessing"{
    //材质面板，这里所有属性都将隐藏
    Properties{
        [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
    }

    SubShader{
        // 不需要背面剔除
        // 禁用深度缓存
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

            //将要进行后处理的图片
            sampler2D _MainTex;

            //模型纹理数据，后处理会自动生成一个矩阵网格
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
                //变换到裁剪空间，后处理的渲染过程和普通着色器类似
                o.position = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            //片段着色器，后处理主要是在这里执行
            fixed4 frag(v2f i) : SV_TARGET{
                //原图片采样
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }

            ENDCG
        }
    }
}
```

## Postprocessing C# Script

上面我们已经准备好了用于后处理的着色器，接下来我们要实现C#脚本，来控制后处理过程。摄像机在执行后处理的时候会用到这个脚本。

新建的脚本是一个脚本组件，只有一个函数`OnRenderImage`。这个函数有Unity在特定时间调用的。其中传递两个参数，一是后处理原图，一是后处理结果图。将一张图中的数据复制到另一张图，可以使用`Blit`函数。
```c#
using UnityEngine;

//该脚本需要和摄像机绑定在同一物体
public class Postprocessing : MonoBehaviour {

	//当当前绑定的摄像机渲染完一帧画面后，会调用该函数
	void OnRenderImage(RenderTexture source, RenderTexture destination){
		//将原图写入到结果图
		Graphics.Blit(source, destination);
	}
}
```

到目前为止，整个后处理逻辑并不会产生什么特别的效果，因为从原图到结果图没有执行任何操作。我们可以再传第三个材质参数，那么在结果图写入前会执行该材质中的着色器脚本。所以这里我们给这个后处理脚本增加一个材质属性。
```c#
using UnityEngine;

//该脚本需要和摄像机绑定在同一物体
public class Postprocessing : MonoBehaviour {
    //用于后处理的材质
    [SerializeField]
    private Material postprocessMaterial;

	//当当前绑定的摄像机渲染完一帧画面后，会调用该函数
    void OnRenderImage(RenderTexture source, RenderTexture destination){
		//将原图按照材质着色器脚本逻辑，写入到结果图
        Graphics.Blit(source, destination, postprocessMaterial);
    }
}
```

当前面准备好后，我们创建需要的材质球，然后将该材质球和我们的后处理着色器绑定。
![](https://www.ronja-tutorials.com/assets/images/posts/016/EmptyMaterial.png)

然后将我们的后处理脚本绑定到我们的摄像机物体上，并且将前面的材质球赋值给这个后处理组件。
![](https://www.ronja-tutorials.com/assets/images/posts/016/PostprocessingComponent.png)

## Negative Colors Effect

做好这一切后，运行程序发现好像没啥特别的变化。要实现颜色取反的效果，我们还需要重新回到我们的后处理着色器中，在片段着色器函数中执行颜色取反的操作。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    //原图采样
    fixed4 col = tex2D(_MainTex, i.uv);
    //颜色取反
    col = 1 - col;
    return col;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/016/Result.jpg)

虽然颜色取反并不是我们常用的效果，但是它揭示了后处理的一般流程，为我们打开一个全新的后处理世界。在后面我会陆陆续续介绍其他常用的后处理效果。
```c++
Shader "Tutorial/016_Postprocessing"{
    //材质面板，这里所有属性都将隐藏
    Properties{
        [HideInInspector]_MainTex ("Texture", 2D) = "white" {}
    }

    SubShader{
        // 不需要背面剔除
        // 禁用深度缓存
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

            //将要进行后处理的图片
            sampler2D _MainTex;

            //模型纹理数据，后处理会自动生成一个矩阵网格
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
                //变换到裁剪空间，后处理的渲染过程和普通着色器类似
                o.position = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            //片段着色器，后处理主要是在这里执行
            fixed4 frag(v2f i) : SV_TARGET{
                //原图片采样
                fixed4 col = tex2D(_MainTex, i.uv);
                //颜色取反
                col = 1 - col;
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
public class Postprocessing : MonoBehaviour {
    //用于后处理的材质
    [SerializeField]
    private Material postprocessMaterial;

	//当当前绑定的摄像机渲染完一帧画面后，会调用该函数
    void OnRenderImage(RenderTexture source, RenderTexture destination){
		//将原图按照材质着色器脚本逻辑，写入到结果图
        Graphics.Blit(source, destination, postprocessMaterial);
    }
}
```

希望本篇教程能够让你了解如何使用简单的后处理效果、能够独立实现一些后处理效果。

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/016_Postprocessing/Postprocessing.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/016_Postprocessing/Postprocessing.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/016_Postprocessing/Postprocessing.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/016_Postprocessing/Postprocessing.cs)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!