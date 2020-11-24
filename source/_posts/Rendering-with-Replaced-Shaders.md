---
title: Rendering with Replaced Shaders
date: 2020-11-22 18:01:34
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- Replaced Shader
---

通常每个需要渲染的物体都自带材质以及相应的shader，但是有时候我们可能希望使用同一个Shader对场景中的进行渲染。例如，当我们对屏幕图像进行边缘检测时，需要知道每个像素点所对应三维空间点的法向量，然后根据法向量的差异来判断是否是边缘。还有一些情况我们需要场景的深度信息。无论是法向纹理还是深度散纹理等，都可以通过Relpace Shader来实现。
Replace Shader和普通shader没什么区别，我们只需要调用Camera.RenderWithShader或者`Camera.SetReplacementShader`函数，并指定需要用于替换的shader,以及shader的Tags值。
假如我们有以下三个shader，其中shaderA被设置为Relace Shader，且repalcementTag为"ReplacementTag"，其余两个都是挂载在模型上的shader：
``` bash
Shader "Shader A"
{
	...
	SubShader
	{
		Tags {"ReplacementTag"="true"}
		...
	}
	SubShader
	{
		Tags {"OtherTag"="whatever"}
		...
	}
}
```
``` bash
Shader "Shader B"
{
	...
	SubShader
	{
		Tags {"ReplacementTag"="false"}
		...
	}
	SubShader
	{
		Tags {"OtherTag"="whatever"}
		...
	}
}
```
``` bash
Shader "Shader C"
{
	...
	SubShader
	{
		Tags {"ReplacementTag"="true"}
		...
	}
	SubShader
	{
		Tags {"OtherTag"="whatever"}
		...
	}
}
```
这时候，只有挂载ShaderC的模型会被渲染。虽然ShaderB和ShaderC都含有“ReplacementTag",但是ShaderA中的`"ReplacementTag"="true"`，只有ShaderC中的第一个SubShader有匹配的Tag值。
## Lit Shader replacement
replacement渲染需要指定一个camera来实现，被指定camera的渲染设置都会应用到repalcement渲染过程中，例如render path、light、shadow等。
## Built-in scene depth/normals texture
camera本身内置了深度、法向纹理的渲染功能，只需要设置`Camera.depthTextureMode`，便可以开启这些功能，而渲染后的深度纹理可以用全局的`_CameraDepthTexture`变量来引用。
而这种内置的渲染功能的实际渲染方法与硬件相关。有些硬件就是通过replacement渲染的方式来实现的。因为ReplacementTag会对实际渲染的物体进行筛选，所以在编写Shader时，要正确使用“RenderType"这个标签。
## Code Example
在Start()函数中指定replacement shader:
``` bash
void Start()
{
	camera.SetReplacementShader(EffiectShader,"RenderType");
}
```
EffiectShader需要包含"RenderType"这个标签，如果没有"RenderType"这个，那么不会有任何物体被渲染。另外，EffiectShader可以有多个SubShader，每个SubShader对应一个"RenderType"。例如，我们需要渲染透明物体，可以指定两个SubShader分别渲染"RenderType"="Transparent"和"RenderType"="TransparentCutout"。
``` bash
Shader "EffectShader" {
     SubShader {
         Tags { "RenderType"="Opaque" }
         Pass {
             ...
         }
     }
     SubShader {
         Tags { "RenderType"="SomethingElse" }
         Pass {
             ...
         }
     }
 ...
 }
 ```
SetReplacementShader函数会查找场景中所有物体，这些物体自带的Shader的"RenderType"值和EffectShader中的"RenderType"值匹配的话，那么这些物体将会使用EffectShader中相对应的SubShader进行渲染。