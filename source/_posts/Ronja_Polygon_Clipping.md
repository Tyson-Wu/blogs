---
title: Polygon Clipping
date: 2021-07-06 12:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Polygon Clipping](https://www.ronja-tutorials.com/post/014-polygon-clipping/)

## Summary

目前为止，我们接触的所有模型渲染都是基于多边形的。有人可能好奇，可不可以通过一系列顶点来实现对多边形的裁剪操作，这也是本文的重点。我将介绍如何在单`Pass`中的片段着色器函数中实现这个。当然关于裁剪还有其他的实现方案，例如通过将顶点构成的裁剪区域渲染到模板上，然后基于模板进行裁剪操作，但是本文不打算讨论它。

本文主要从技术角度来介绍器基本思路，并不会涉及到复杂的图形效果，所以这里我们以无光照的着色器来实现。关于无光照的着色器介绍可以参考[这里](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)
![](https://www.ronja-tutorials.com/assets/images/posts/014/Result.gif)

## Draw Line

首先我们需要顶点的世界坐标，就像前面关于[二维平面映射](https://tyson-wu.github.io/blogs/2021/07/02/Ronja_Planar_Mapping/)中介绍的一样，将传入的模型顶点坐标乘以模型矩阵，然后将结果传递给片段着色器。
```c++
//中间插值结构体
struct v2f{
    float4 position : SV_POSITION;
    float3 worldPos : TEXCOORD0;
};

//顶点着色器
v2f vert(appdata v){
    v2f o;
    //计算裁剪坐标
    o.position = UnityObjectToClipPos(v.vertex);
    //计算世界坐标
    float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
    o.worldPos = worldPos.xyz;
    return o;
}
```

然后在片段着色器中，我们需要计算点与线的关系。因为我们后面要使用点来构造线，而两点确定一条线正是我们这里用到的一条基本规律。

为了确定点线之间的关系，我们需要借助两个向量，一是从直线上任意一点到该点的向量，二是直线的法向量。单纯的谈直线的法向量并没有太大意义，因为法向具有方向性，在二维空间有两个，在三维空间有无数个。但是这里我们想判定点是在线的左侧还是右侧，所以我们这里将法向量定义为垂直与直线，并指向直线左侧的向量。

当我们得到这两个向量后，这两个向量的点乘就可以用来判断点与线的关系。如果结果为正，说明在左侧，如果为负，说明在右侧，如果为零，说明在直线上。
![](https://www.ronja-tutorials.com/assets/images/posts/014/Vectors.png)

那么在着色器脚本中，我们首先定义直线上的两个点，然后计算上图中的三个向量。首先我们计算直线的方向，我们将第一个点减去第二个点所得到的方向向量作为直线的方向，然后将该向量旋转90度。接下来我们将直线外的那个点减去直线内的任意一个点。

然后我们将直线的法向量和目标点的向量做点乘，并将结果显示在屏幕上。
```c++
float2 linePoint1 = float2(-1, 0);
float2 linePoint2 = float2(1, 1);

//计算这三个向量
float2 lineDirection = linePoint2 - linePoint1;
float2 lineNormal = float2(-lineDirection.y, lineDirection.x);
float2 toPos = i.worldPos.xy - linePoint1;

//计算点与线的关系
float side = dot(toPos, lineNormal);
//以0为分界线，大于零取1，否则取0
//side = step(0, side);

return side;
```
![](https://www.ronja-tutorials.com/assets/images/posts/014/Distance.png)

上图中出现一条灰色的过渡带。但是这并不是我们所想要的。因为所有小于零的区域显示为纯黑，0到1之间的显示为灰色区域，大于1的显示为百色。因此我们可以使用`step`函数来讲中间灰度区域去掉。

```c++
//以0为分界线，大于零取1，否则取0
float side = dot(toPos, lineNormal);
side = step(0, side);

return side;
```
![](https://www.ronja-tutorials.com/assets/images/posts/014/Line.png)

当我们再向其中加入一个点、两条线，就可以构成三角形了。因此我们将上面的判断逻辑抽成一个函数，方便复用。
```c++
//在左边返回1，否则返回0
float isLeftOfLine(float2 pos, float2 linePoint1, float2 linePoint2){
    //计算三个向量
    float2 lineDirection = linePoint2 - linePoint1;
    float2 lineNormal = float2(-lineDirection.y, lineDirection.x);
    float2 toPos = pos - linePoint1;

    //以0为分界线，大于零取1，否则取0
    float side = dot(toPos, lineNormal);
    side = step(0, side);
    return side;
}

//片段着色器
fixed4 frag(v2f i) : SV_TARGET{
    float2 linePoint1 = float2(-1, 0);
    float2 linePoint2 = float2(1, 1);

    side = isLeftOfLine(i.worldPos.xy, linePoint1, linePoint2);

    return side;
}
```

## Draw a Polygon of multiple lines

前面已经抽取点与线的位置判断函数，这样我们可以对多边形的每一条边界线进行判定。然后将所有的判定结果结合起来，判定点与多边形的位置关系。例如我们可以定义当点在所有线的左侧时，为true，反之为false。或者我们可以定义在所有线的右侧时，为false，反之为true。这里我们定义的三角形为顺时针，这意味着线的左侧为三角形的外侧，我们可以将所有线条的判定结果求和，当全为左侧时，为零，否则大于零。
```c++
//片段作色器
fixed4 frag(v2f i) : SV_TARGET{
    float2 linePoint1 = float2(-1, 0);
    float2 linePoint2 = float2(1, 1);
    float2 linePoint3 = float2(1, -1);

    float outsideTriangle = isLeftOfLine(i.worldPos.xy, linePoint1, linePoint2);
    outsideTriangle = outsideTriangle + isLeftOfLine(i.worldPos.xy, linePoint2, linePoint3);
    outsideTriangle = outsideTriangle + isLeftOfLine(i.worldPos.xy, linePoint3, linePoint1);

    return outsideTriangle;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/014/Triangle.png)

上面我们成功通过边界裁剪来实现多边形效果。现在我想通过材质面板来编辑这个裁剪区域。我们需要引入两个变量，一个是裁剪顶点列表，一个是顶点的个数。顶点列表记录的是我们裁剪区域的顶点位置。第二个是记录顶点的个数，因为着色器中不支持动态数组，所以我们需要定义一个固定数组，然后通过该变量来控制实际参与计算的顶点个数。
```c++
//顶点数组和个数
uniform float2 _corners[1000];
uniform uint _cornerCount;
```

## Filling the Corner Array

材质面板并不支持数组显示。所以我们需要创建一个脚本组件来管理这个数组。这里我给新创建的脚本添加两个属性，并且将该脚本定义为编辑模式可执行，这样我们就不需要每次都启动程序了。另外，下面的脚本要求必须和渲染的裁剪多边形绑定，这样我们可以访问其中的材质。
```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

[ExecuteInEditMode]
[RequireComponent(typeof(Renderer))]
public class PolygonController : MonoBehaviour {


}
```

下面是我们想脚本中加入两个变量，一个是裁剪着色器所需要的顶点数组，一个是绑定该着色器的材质。材质属性是私有的，因为我们是直接从当前绑定物体上获取的。顶点数组也是私有的，因为我们也不需要在外部访问，但是我们需要在组件面板上编辑它，所以需要`SerializeField`来标记它。
```c#
[SerializeField]
private Vector2[] corners;

private Material _mat;
```

然后我们实现一个函数将这些顶点数据传递给着色器。首先我们需要获取到当前物体的材质，这里我们使用`sharedmaterial`属性来获取材质，如果使用`material`属性的话，获取的将是该材质的副本。

然后我们再创建一个长度为1000的4维向量数组。之所以是4维而不是二维，因为Unity只支持向着色其中传递4为向量数组。而长度为1000是因为着色器中定义的是固定长度的数组。当然这里定义1000是假设我们会用到的顶点个数最大为1000，具有一定的随意性，你也可以根据实际需要调整。

当我们将二维向量赋值给四维向量时，为赋值的部分将会以0填充。

在准备好四维向量数组后，我们将其、以及实际的数组长度传递给着色器。
```c#
void UpdateMaterial(){
    //获取当前模型的材质
    if(_mat == null)
        _mat = GetComponent<Renderer>().sharedMaterial;

    //填充顶点位置数组
    Vector4[] vec4Corners = new Vector4[1000];
    for(int i=0;i<corners.Length;i++){
        vec4Corners[i] = corners[i];
    }

    //传递给着色器
    _mat.SetVectorArray("_corners", vec4Corners);
    _mat.SetInt("_cornerCount", corners.Length);
}
```

下一步是在Unity事件函数中调用上面的函数。这里我们选择`Start`和`OnValidate`两个函数，前者在游戏启动时会调用一次，后者在每次修改组件属性面板上的值时会调用一次。
```c#
void Start(){
    UpdateMaterial();
}

void OnValidate(){
    UpdateMaterial();
}
```

脚本编写完成后，将其以组件的形式赋给我们的裁剪物体。然后在组件属性面板上可以编辑需要的裁剪顶点。
![](https://www.ronja-tutorials.com/assets/images/posts/014/Inspector.png)

下面我们回到着色器脚本，然后在着色器中使用传入的顶点数组。

然后我们使用`for`循环来依次遍历这个顶点数组。因为在`hlsl`中，数组的起点是0，所以我们循环的起点也是0。循环的终止条件是超过实际的顶点个数。这里我们使用`for`循环来遍历。其实还可以将`for`循环展开，相比而言在显卡中的效率更高，但是我们需要提前知道循环次数才可以展开。这里我们的循环次数是可变的，所以只能使用`for`循环。

在循环中，我们计算所有的边界与点之间的关系，然后求和。而边界是由本次遍历的顶点和下次遍历的顶点构成的。但是这里有一个问题，就是最后的一个顶点的下一个顶点应该是第一个点的，所以这里我们可以使用求余来将索引值切回到起始点。
```c++
//片段着色器
fixed4 frag(v2f i) : SV_TARGET{

    float outsideTriangle = 0;

    [loop]
    for(uint index;index<_cornerCount;index++){
        outsideTriangle += isLeftOfLine(i.worldPos.xy, _corners[index], _corners[(index+1) % _cornerCount]);
    }

    return outsideTriangle;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/014/Hexagon.png)

最终我们得到由六个点构成的裁剪区域。

## Clip and Color the Polygon

前面的步骤实际上是一个区域标记的过程，在区域内外都会进行渲染，只不过使用不同的颜色而已。有人可能想问，如何只渲染其中一部分，这样就可以向其中添加其他模型一同渲染。在`hlsl`中有一个放弃渲染的函数叫做`clip`。如果向`clip`传入的值小于0，那么将不会执行颜色写入的操作，否则的话正常渲染。

前面我们已经计算出区域标记，不过要么是0要么是1，都不小于0，因此我们还需要做一些转换才能将其中一部分渲染剔除。
```c++
clip(-outsideTriangle);
return outsideTriangle;
```

![](https://www.ronja-tutorials.com/assets/images/posts/014/SuperHexagon.png)
![](https://www.ronja-tutorials.com/assets/images/posts/014/ConcaveBreaking.gif)

这种裁剪方式有很大的缺陷，就是只能用来渲染凸多边形。ps:其实我觉得凹多边形也不影响。
```c++
Shader "Tutorial/014_Polygon"
{
    //属性面板
    Properties{
        _Color ("Color", Color) = (0, 0, 0, 1)
    }

    SubShader{
        //渲染不透明物体
        Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

        Pass{
            CGPROGRAM

            //引入内置函数和变量
            #include "UnityCG.cginc"

            //定义顶点和片段着色器
            #pragma vertex vert
            #pragma fragment frag

            fixed4 _Color;

            //用于裁剪的顶点数组
            uniform float2 _corners[1000];
            uniform uint _cornerCount;

            //模型网格数据
            struct appdata{
                float4 vertex : POSITION;
            };

            //中间插值数据
            struct v2f{
                float4 position : SV_POSITION;
                float3 worldPos : TEXCOORD0;
            };

            //顶点着色器
            v2f vert(appdata v){
                v2f o;
                //计算裁剪坐标
                o.position = UnityObjectToClipPos(v.vertex);
                //计算世界坐标
                float4 worldPos = mul(unity_ObjectToWorld, v.vertex);
                o.worldPos = worldPos.xyz;
                return o;
            }

            //在左边返回1，否则返回0
            float isLeftOfLine(float2 pos, float2 linePoint1, float2 linePoint2){
                //计算所需的三个向量
                float2 lineDirection = linePoint2 - linePoint1;
                float2 lineNormal = float2(-lineDirection.y, lineDirection.x);
                float2 toPos = pos - linePoint1;

                //以0为分界线，大于零取1，否则取0
                float side = dot(toPos, lineNormal);
                side = step(0, side);
                return side;
            }

            //片段着色器
            fixed4 frag(v2f i) : SV_TARGET{

                float outsideTriangle = 0;

                [loop]
                for(uint index;index<_cornerCount;index++){
                    outsideTriangle += isLeftOfLine(i.worldPos.xy, _corners[index], _corners[(index+1) % _cornerCount]);
                }

                clip(-outsideTriangle);
                return _Color;
            }

            ENDCG
        }
    }
}
```

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

[ExecuteInEditMode]
[RequireComponent(typeof(Renderer))]
public class PolygonController : MonoBehaviour {
	[SerializeField]
	private Vector2[] corners;

	private Material _mat;

	void Start(){
		UpdateMaterial();
	}

	void OnValidate(){
		UpdateMaterial();
	}

	void UpdateMaterial(){
		//获取当前模型所用的材质
		if(_mat == null)
			_mat = GetComponent<Renderer>().sharedMaterial;

		//填充顶点数组
		Vector4[] vec4Corners = new Vector4[1000];
		for(int i=0;i<corners.Length;i++){
			vec4Corners[i] = corners[i];
		}

		//传递给着色器
		_mat.SetVectorArray("_corners", vec4Corners);
		_mat.SetInt("_cornerCount", corners.Length);
	}

}
```

希望你能从这边文章中了解到如何处理点、线等多边形问题。也希望我所介绍到刚好是你想知道的！

你可以在以下链接找到源码：
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/014_Polygon/Polygon.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/014_Polygon/Polygon.shader)
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/014_Polygon/PolygonController.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/014_Polygon/PolygonController.cs)

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!