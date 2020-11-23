---
title: Camera’s Depth Texture
date: 2020-11-22 17:01:34
categories:
- [Unity, Shader,Depth Texture]
tags:
- Unity
- Camera
- Shader
- Depth Texture
---
[Camera](https://docs.unity3d.com/Manual/class-Camera.html)可以创建深度图、深度+法向纹理、运动向量图。这算是一种简化版的G-buffer，这些纹理图可以用在post-processing中，从而实现自定义的光照模型。我们也可以使用[Shader Replacement](https://tyson-wu.github.io/myblog/2020/11/20/Rendering-with-Replaced-Shaders/)创建这类纹理图。
Camera虽然内置了创建深度图的功能，但是通常该功能是关闭的，需要在脚本中通过[Camera.depthTextureMode](https://docs.unity3d.com/ScriptReference/Camera-depthTextureMode.html)来启用该功能。
Unity提供了三种深度图纹理模式：
- DepthTextureMode.Depth：纯[深度图](https://docs.unity3d.com/Manual/SL-DepthTextures.html)。默认情况下生成的深度值的范围是[0-1]，且是非线性分布。
- DepthTextureMode.DepthNormals：同时包含深度和法向量信息。
- DepthTextureMode.MotionVectors：在屏幕空间内，每个像素点在当前帧的运动向量。这是一张RG16的纹理图，只有两个通道，每个通道深度为16。

这几种模式采用的是位标记，也就是说可以两两组合。例如，我需要深度图和运动向量图，可以执行以下操作：

``` bash
camera.depthTextureMode = DepthTextureMode.Depth | DepthTextureMode.MotionVectors;
```

## DepthTextureMode.Depth texture
此模式会创建一张屏幕大小的[深度图](https://docs.unity3d.com/Manual/SL-DepthTextures.html)。
深度图渲染是在ShadowCaster pass中执行的，ShadowCaster pass也是用来实现阴影采集的Shader pass。因此，如果游戏物体所挂载的Shader不支持Shadow casting，也就是Shader中没有shadow caster pass，那么这个游戏物体不会被渲染到深度图中。
因此，当我们需要渲染某些物体的深度图时，需要保证：
- 这些物体所挂载的Shader或者其fallback中的Shader具有shadow casting pass。或者
- 使用[surface shader](https://docs.unity3d.com/Manual/SL-SurfaceShaders.html)，然后加入addshadow标识符，使得表面着色器能够生成shadow pass。

需要注意的是，只有"RenderType"="Opaque"的物体会被渲染到深度图中。这些物体被认为是实体，在渲染队列中的编号<=2500。

## DepthTextureMode.DepthNormals texture
此模式会创建一张屏幕大小、深度为8、32位的纹理图。其中基于观察坐标系的法向量被存储到R&G通道中，而深度被存到B&A通道中。其中法向量是采用立体投影的方式进行编码，而深度是16位精度的浮点数。
在[UnityCG.cginc]文件中有一个DecodeDepthNormal可以用来对这里的深度以及法向量进行解码。其中解码后的深度值是范围为[0,1]的小数。
可以参考[屏幕空间环境遮罩](https://docs.unity3d.com/Manual/PostProcessing-AmbientOcclusion.html)的图像后处理教程。

## DepthTextureMode.MotionVectors texture
此模式会创建一张屏幕大小的、两通道的、RG16的纹理图，每个通道占16位。基于屏幕空间的运动向量就是被编码到这两个通道中，分别表示U、V两个坐标值。
当采样值的范围在[-1,1]时，这表示的是上一帧与当前帧的UV偏移值，因为UV坐标的坐标值范围是[0-1]。

## Tips&Tricks
当Camera启用渲染深度图或深度+法向图时，Camera inspactor会有相应的提示语。
使用Camera.depthTextureMode方式获取深度图，当我们想关闭该功能时，需要手动设置：
``` bash
camera.depthTextureMode = DepthTextureMode.None;
```
当我们在实现复杂Shader或者图像效果时，我们要考虑到[不同平台之间的差异性](https://docs.unity3d.com/Manual/SL-PlatformDifferences.html)。

## Shader variables
当我们想在Shader中使用由camera.depthTextureMode方法生成深度图时，在Shader中可以声明_CameraDepthTexture变量：
``` bash
sampler2D _CameraDepthTexture;
```
_CameraDepthTexture是全局Shader属性，任何Shader都可以声明并访问该变量。_CameraDepthTexture总是指向摄像机的主深度图(我理解的是当前帧第一张深度纹理)。还有_LastCameraDepthTexture，用来指向最后一个深度图(好绕。。。)。例如我们在脚本中使用第二个Camera来获取分辨率减半的纹理图。
如果生成的是运动向量图，那么可以使用_CameraMotionVectorsTexture这个全局属性。
这里强调这些纹理图是使用camera.depthTextureMode方式生成的。因为除此之外，还可以使用[Relapcement Shader](/_posts/Rendering-with-Replaced-Shaders.md)的方式，但是这种方式生成的纹理不会自动写入到上面这几个全局变量中。

## Under the hood
本节所讲的深度图，是Camera的内置功能，不需要使用者考虑如何实现。但是这种内置的深度图生成方法，却是与所使用的rendering path和硬件平台有关。通常情况下，Deferred Shading或者Legacy Deferred Lighting这两种rendering path，会自动生成深度图并存放在G-Buffer中。另外，有一些平台在渲染的时候，会自动将深度信息缓存到Depth buffer。如果前面两种情况都不满足的话，那么引擎会额外开一个Pass来生成深度图，这种额外的方式就是[Shader Replacement](https://docs.unity3d.com/Manual/SL-ShaderReplacement.html)，因此我们需要准确的使用RenderType这个标签。例如"RenderType"="Transparent"的物体的深度信息可能会采集不到。
MotionVectors纹理图是通过额外开启一个Pass生成的。只有运动的物体会被采集，通过对别前后帧物体的运动来计算像素运动向量。