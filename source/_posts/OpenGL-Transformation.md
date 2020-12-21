---
title: OepnGL Transformation
date: 2020-12-19 15:01:00
categories:
- [OpenGL, Transformation]
tags:
- OpenGL
---

## Overview

顶点、法向这些几何数据的坐标变换是发生在Vertex Operation，而[OpenGL Pipeline](http://www.songho.ca/opengl/gl_pipeline.html)图元拼装是发生在栅格化操作之前。
![](http://www.songho.ca/opengl/files/gl_transform02.png)
OpenGL vertex transformation

### Object Coordinates

模型坐标系表示的是模型的初始位置和朝向，也是模型在所有坐标变换之前的初始状态。我们可以使用glRotatef(), glTranslatef(), glScalef()来对模型进行坐标变换。

### Eye Coordinates

模型坐标经过GL_MODELVIEW矩阵变换后，进入摄像机坐标系。GL_MODELVIEW矩阵结合了模型矩阵和观测矩阵(Mview \* Mmodel)。模型矩阵变换是从模型坐标系到世界坐标系，而观察矩阵变换是从世界坐标系到观察坐标系。
![](http://www.songho.ca/opengl/files/gl_transform07.png)

在OpenGL中，没有单独的观察矩阵(Mview)。因此，要模拟摄像机变换，也就是移动摄像机，我们通常是对场景中的物体做一个反向变换，例如，我们控制摄像机右移，实际上是将整个场景左移。更具体一点就是，在OpenGL中，摄像机的位置永远都是在原点，面向观察坐标系下的-z方向。关于GL_MODELVIEW更详细的介绍请参考[ModelView Matrix](http://www.songho.ca/opengl/gl_transform.html#modelview)。

模型的法向主要是用来进行光照计算的，在OpenGL中，用于光照计算的法向是在观察坐标系下的法向。也就是我们先使用GL_MODELVIEW对法向进行变换，然后再进行光照计算。需要注意的是，法向的变换和坐标点的变换操作有所区别。我们这里坐标变换是直接将坐标左乘GL_MODELVIEW变换矩阵。但是法向变换是左乘GL_MODELVIEW变换矩阵逆矩阵的转置矩阵。关于法向变换请参考[Normal Vector Transformation](http://www.songho.ca/opengl/gl_normaltransform.html)。
![](http://www.songho.ca/opengl/files/gl_normaltransform01.png)

### Clip Coordinates

使用GL_PROJECTION投影矩阵，从观察坐标系变换到裁剪坐标系。其中GL_PROJECTION定义了裁剪区域，以及投影方式，是透视投影还是正交投影呢。之所以称作裁剪坐标系，是因为我们可以通过变换后的坐标来实现裁剪操作。更具体一点就是，变换后的坐标(x,y,z,w)。其中w表示的裁剪边界，当(x,y,z)都在[-w,w]这个范围内时，说明这个点在裁剪区域内；而不在这个范围的点将会被剔除。如果某个图元的所有点都不在裁剪区域内，那么整个图元都会被剔除，如果部分顶点在裁剪区域内，那么不在区域内的点将会被剔除，但是会基于裁剪边界对图元进行一个缝合操作。关于GL_PROJECTION的更多介绍请参考[Projection Matix](http://www.songho.ca/opengl/gl_transform.html#projection)。
![](http://www.songho.ca/opengl/files/gl_transform08.png)

### Normalized Device Coordinates (NDC)

经过裁剪变换后，在裁剪区域内的坐标点的x,y,z的范围在[-w,w]之间，将x,y,z除以w后，得到齐次设备空间坐标。这个操作叫做投影除法。齐次设备空间坐标的x,y,z的范围现在是[-1,1]。
![](http://www.songho.ca/opengl/files/gl_transform12.png)

### Window Coordinates (Screen Coordinates

使用视窗变换(ViewPort)，从齐次设备空间变换到屏幕空间。通过对齐次设备空间进行缩放、平移，来对齐屏幕渲染窗口区域。这个时候，虽然属于屏幕坐标系，但是整个空间还是属于几何意义上的、连续的空间，和像素表示的离散的空间有本质的区别。因为我们的图形还是通过点、线、多边形图元表示，而离散的像素空间是使用单个像素表示。
屏幕空间下的坐标、法向等几何信息，最终通过栅格化处理，得到离散化的片段数据，坐标变换和栅格化操作之间的关系请参考[OpenGL pipeline](http://www.songho.ca/opengl/gl_pipeline.html)中的管线图。glViewport()方法是用来定义屏幕空间哪个矩形区域用来显示最终的渲染画面。glDepthRange()方法是用来确定屏幕坐标系的z值。最终得到的屏幕坐标是基于这两个方法所设置的参数决定的：

- glViewport(x,y,w,h);
- glDepthRange(n,f);

![](http://www.songho.ca/opengl/files/gl_transform13.png)
![wwwwwww](/blogs/images/src/d626436913e4116bd6112677aa7b026.jpg)

视窗变换只是简单的线性映射：

![](http://www.songho.ca/opengl/files/gl_transform14.png)


## OpenGL Transformation Matrix

![](http://www.songho.ca/opengl/files/gl_transform04.png)
OpenGL Transformation Matrix

OpenGL使用[4X4的矩阵](http://www.songho.ca/opengl/gl_matrix.html)作为变换矩阵。值得注意的是，这16个元素是按列主序存储在一维数组里面。我们可以通过转置操作将其变为行主序。行主序是标准公约。

OpenGL有4中不同类型的矩阵：GL_MODELVIEW, GL_PROJECTION, GL_TEXTURE, GL_COLOR。我们可以通过glMatrixMode()方法来切换当前矩阵类型。例如，我们想选择GL_MODELVIEW矩阵，那么可以使用`glMatrixMode(GL_MODELVIEW)`。

### Model-View Matrix (GL_MODELVIEW)

GL_MODELVIEW将观察矩阵和模型矩阵结合到一起。如果我们想从模型坐标系变换到观察坐标系，那么我们需要GL_MODELVIEW的逆矩阵进行变换。gluLookAt()主要用来设置观察矩阵。

![](http://www.songho.ca/opengl/files/gl_anglestoaxes01.png)
GL_MODELVIEW矩阵

上图中，最右边的三个元素(m12,m13,14)，表示的是平移变换--glTranslatef()。元素m15表示的是[齐次坐标](http://www.songho.ca/math/homogeneous/homogeneous.html)，主要用来做透视变换用的。

另外9个元素(m0,m1,m2),(m4,m5,m6),(m8,m9,10)是用来进行欧式变换以及仿射变换，相关的函数是glRotatef()和glScalef()。另外值得指出的是，这三组向量表示的是相互正交的轴。

- (m0,m1,m2) : +X轴,   left 向量，默认是(1,0,0)
- (m4,m5,m6) : +Y轴,    up  向量，默认是(0,1,0)
- (m8,m9,m10): +Z轴, forward向量，默认是(0,0,1)

除了使用变换函数，如glRotatef()，来修改GL_MODELVIEW矩阵，我们也可以根据摄像机偏转角或者朝向来手动计算GL_MODELVIEW矩阵，例如以下几种方法：

- [Angles to Axes](http://www.songho.ca/opengl/gl_anglestoaxes.html)
- [Lookat to Axes](http://www.songho.ca/opengl/gl_lookattoaxes.html)
- [Rotation About Arbitrary Axis](http://www.songho.ca/opengl/gl_rotate.html)
- [Matrix4 class](http://www.songho.ca/opengl/gl_matrix.html)

