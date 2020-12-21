---
title: OepnGL Overview
date: 2020-12-19 15:01:00
categories:
- [OpenGL, Overview]
tags:
- OpenGL
---

## OpenGL Introduction

OpenGL 是图形硬件的软件接口，接口功能与硬件无关，可以在不同的硬件平台上使用。OpenGL程序也可以通过网络的方式执行，例如客户端-服务器这种分布方式。OpenGL的客户端是指执行OpenGL程序的计算机，而服务端是指执行绘图操作的计算机。

OpenGL使用gl前置符来标记OpenGL内核命令，使用glu来标记OpenGL Utility库命令。另外，OpenGL所有以GL_开头的命令全部是大写。OpenGL也会用后置标志符来指明命令参数的个数以及类型。

``` markdown
glColor3f(1,0,0);		//使用三个浮点数设置颜色
glColor4d(0,1,0,0.2);	//使用双精度浮点数设置颜色，并具有Alpha通道
glVertex3fv(vertex);	//使用三维向量设置顶点坐标
```

## State Machine

OpenGL是一个状态机。其中设置的各种状态参数，包括模式、属性等，会一直有效，直到我们对其手动修改。绝大多数的状态值可以直接使用glEnable() 或 glDisable()来开启和禁用。我们也可以使用gllsEnabled()来获取某个状态值当前的开启状态。我们也可以使用glPushAttrib()和glPopAttrib()来保存或提取一组状态值。GL_ALL_ATTRIB_BITS参数表示保存或提取所有的状态值。在标准的OpenGL中，栈的个数不少于16个。我们可以使用[glinfo](http://www.songho.ca/opengl/files/glinfo.zip)来获取栈的大小信息。

``` markdown
glPushAttrib(GL_LIGHTING_BIT);			//
	glDisable(GL_LIGHTING);
	glEnable(GL_COLOR_MATERIAL);
glPushAttrib(GL_COLOR_BUFFER_BIT);
	glDisable(GL_DITHER);
	glEnable(GL_BLEND);
	
...//其他代码

glPopAttrib();		//提取GL_COLOR_BUFFER_BIT
glPopAttrib();		//提取GL_LIGHTING_BIT
```

## glBegin()和glEnd()

我们可以直接在glBegin()和glEnd()方法之间，指定一组顶点数据来绘制几何图元，例如点、线、三角形等等。这种绘制方式称为及时模式。我们也可以使用其他的方式进行绘制，例如[vertex array](http://www.songho.ca/opengl/gl_vertexarray.html)。

``` markdown
glBegin(GL_TRIANGLES);
	glColor3f(1,0,0);		//设置颜色为红色
	glVertex3fv(v1);		//设置三角形的三个顶点坐标
	glVertex3fv(v2);
	glVertex3fv(v3);
glEnd();
```

OpenGL中包含十种图元类型：GL_POINTS, GL_LINES, GL_LINE_STRIP, GL_LINE_LOOP, GL_TRIANGLES, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN, GL_QUADS, GL_QUAD_STRIP, GL_POLYGON。

这里需要注意的是，不是所有的命令都可以放在glBegin()和glEnd()之间，只有一小部分命令可以，例如glVertex\*(), glColor\*(), glNormal\*(), glTexCoord\*(), glMaterial\*(), glCallList()等等。

## glFlush()和glFinish()

类似于计算机的IO缓存，OpenGL命令也不是立刻执行的。所有的命令首先是存储在缓存中，包括网络缓存以及图形加速器，都是在缓存满了之后，再一并执行缓存中的命令。举个例子，我们的应用程序是通过网络连接的，那么最有效的数据传输方式是将所有数据打包到一起，然后一并发送。相比于每一个命令发送一次，这种批处理的方式效率更高。

glFlush()是用来清空命令缓存，同时强制立即执行缓存中的所有命令。也就是说，使用glFlush()是一种手动执行的操作，它不需要等到命令缓存满了才执行。虽然glFlush()会立即执行缓存中的命令，但是其实际操作应该是将缓存中的命令插入到GPU已有命令的队列里面，当然这种插入是优先插入到最前面，等当前执行的操作完成之后，就会立即开始执行这些命令，然后这些命令执行完后，glFlush()会立即返回到CPU程序，而原先被强制插队的操作会依次执行。所以对glFlush()更为准确的描述应该是，在调用glFlush()后，缓存中的命令会在短暂的等待后立即执行，然后在这些命令执行完成后，glFlush()会立即返回到CPU程序，而原先在GPU中排队的操作会继续执行。

glFinish()和glFlush()方法类似，都是清空命令缓存并强制立即执行缓存中的命令。但是glFinsh()会打断已经开始执行的操作，然后等命令缓存中的所有操作都执行完，再恢复原先被打断的操作。这里面打断，是指GPU中正在执行的操作被暂停。因此，glFinsh()方法，在命令缓存中的命令全部执行完后，不会立即返回到CPU程序，而是等之前未完成的操作全部执行完后，再返回到CPU程序。所以，我们可以是在一些同步任务(也就是GPU的执行操作和CPU的执行操作是串在一起的)中使用glFinsh(),也可以用于测量原先操作的执行时间，因为glFinsh()会等原先的操作执行完再返回。