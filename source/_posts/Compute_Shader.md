---
title: Compute Shader
date: 2021-06-21 15:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Compute Shader
---

原文：
[Compute Shader](https://www.ronja-tutorials.com/post/050-compute-shader/)

至此，我们已经学会了如何使用固定管线来渲染纹理，但是目前的显卡能做的远远不止这些。除了使用固定管线，我们还可以使用compute shader来实现。

你可能会问，为什么要用compute shader，目前的cpu其实已经很强大了，即便遇到大量的处理数据，我们也可以使用多线程来处理。是的，对于一些非图形任务，我们并不需要使用GPU。如果强行使用GPU来处理，很可能会产生各种未知错误，另外也无法使用常用的异常分析手段。优化就更麻烦了，因为需要考虑CPU与GPU之间的数据传输，而GPU并行处理也和CPU编程思路不一样。作为初学者，我们是否需要使用compute shader，首先要明白为什么要用compute shader，使用后是否能够达到比现有方法更好的效果。

如果你觉得你需要使用compute shader，那么继续读下去。在Unity中，你可以使用`SystemInfo.supportsComputeShaders`方法来查看你的GPU是否支持compute shader。
![](https://www.ronja-tutorials.com/assets/images/posts/050/result.gif)

## Basic Compute Shader

我们可以通过选中`Create>Shader>Compute Shader`来创建compute shader。默认创建的compute shader执行向图片中写入数据的操作。但是在本节，我将演示一个更简单的例子——写入一组坐标。

在compute shader中，我们可以向`RWStructuredBuffer`中写入一组数据，而`StructuredBuffer`是只读数据。在它们后面补充数据单元类型，类型包括`vector`或者`struct`。在本节中，我们使用`float3`。

我们把用于计算的方法块叫做`kernel`。我们必须在方法块前面添加`numthreads`标志符，同时方法块有一个输入参数，因为方法块是针对每一个元素进行并行计算的，所以需要一个索引值来表明当前方法块所对应的元素。在这里，我们定义x轴64线程，y、z轴都是1线程。只要支持compute shader的显卡，基本可以处理这种线程设置。因为整个线程设置是一维的，所以我们处理的数据也是一维的，而在并行处理中，我们只需要考虑单个元素的处理过程。因为我们的线程设置是基于三个维度，所以我们输入参数也是三个维度，参数中的值对应相应维度的线程。因为我们这里的数据都集中在x轴上，所以我们的参数也只需要考虑x轴的索引。和普通的Shader一样，我们需要为输入参数添加标志符，方便程序识别参数的语意，这里我们参数的标志符是`SV_DispatchThreadID`。

为了区分compute shader中的普通方法块和核方法块，我们需要使用`pragma`标识符，语法为`#pragma kernel <functionname>`。当然，在一个compute shader中可以拥有多个核函数。例如：
```c++
// 指定一个核函数，我们可以拥有多个核函数
#pragma kernel Spheres

// 输出
RWStructuredBuffer<float3> Result;


[numthreads(64, 1, 1)]
void Spheres(uint3 id : SV_DispatchThreadID)
{
	// compute shader 代码
}
```

首先，让我们输出坐标`(id, 0, 0)`到`Result`中：
```c++
[numthreads(64, 1, 1)]
void Spheres(uint3 id : SV_DispathcThreadID)
{
	Result[id.x] = float3(id.x, 0, 0);
}
```

## Executing Compute Shaders

和普通的Shader不同的是，compute shader并不是通过材质绑定的方式来执行，而是通过`C#`脚本来实现调用。

我们可以创建一个GameObject，并且在它上面创建一个`C#`脚本，在脚本上创建一个`ComputeShader`变量来引用前面创建的compute shader。同时我们创建一个整形变量，用来存储核函数的签名，这个签名是核函数在GPU中的索引ID。首先在`Start`函数中我们调用`FindKernel(<kernelname>)`来获取核函数的签名。在得到核函数签名后，我们可以使用该签名来获取该核函数的线程设置，也就是在compute shader中的`[numthreads(64, 1, 1)]`。我们只提取x轴的线程数，其他两个维度可以用`_`来表示忽略。

另外，我们在`C#`脚本中创建一个长度变量，用来指定compute shader中buffer的长度。知道buffer的长度，以及buffer中存储的数据类型`float3`，我们可以向GPU申请一块存储空间——`ComputeBuffer`。这个空间将会用来存储计算结果，也就是compute shader中的`Result`。创建`ComputeBuffer`需要两个参数，第一参数是元素的个数，第二个参数是元素的大小。我们的元素是`float3`，也就是大小为3个`float`。另外，我们需要在CPU中申请一块和`ComputeBuffer`同样大小的数组空间，以便于将计算后的结果转移到CPU，方便后续计算使用。在计算结束后，我们可以通过`ComputeBuffer.Dispose`方法来释放GPU申请的缓存空间。

当一切设置好后，我们可以在`Update`中使用compute shader。首先，我们需要将我们在GPU创建的`ComputeBuffer`和compute shader中的buffer相关联。在调用compute shader之前，我们还需要计算整个核函数需要执行多少遍，这个叫做线程组。因为核函数单次批处理的数量有限，所以需要分为多组，分批次处理。例如这里我们单次x轴处理量为64，而总的需要处理的数量为buffer长度，那么线程组的个数为后者处以前者。然后通过`dispatch`方法来启用核函数，最终计算结果通过`GetData`函数传回到CPU中。
```c++
public class BasicComputeSpheres : MonoBehaviour
{
    public int SphereAmount = 17;
    public ComputeShader Shader;

    ComputeBuffer resultBuffer;
    int kernel;
    uint threadGroupSize;
    Vector3[] output;

    void Start()
    {
        // 获取核函数签名
        kernel = Shader.FindKernel("Spheres");
		// 获取核函数线程设置
        Shader.GetKernelThreadGroupSizes(kernel, out threadGroupSize, out _, out _);

        //buffer on the gpu in the ram
        resultBuffer = new ComputeBuffer(SphereAmount, sizeof(float) * 3);
        output = new Vector3[SphereAmount];
    }

    void Update()
    {
		//绑定数据
        Shader.SetBuffer(kernel, "Result", resultBuffer);
        int threadGroups = (int) ((SphereAmount + (threadGroupSize - 1)) / threadGroupSize);
        Shader.Dispatch(kernel, threadGroups, 1, 1);
        resultBuffer.GetData(output);
    }

    void OnDestroy()
    {
        resultBuffer.Dispose();
    }
}
```

现在我们有了计算结果，但是并不能直观的去观察这些结果。有很多种方法可以直接在GPU中处理并显示这些结果，但是这并不是本文的重点。所以我选择使用生成一系列模型空间分布，来展示最终生成的结果。
在`Update`中直接将计算的坐标赋值给游戏物体空间坐标。
```c++
// in start method

//spheres we use for visualisation
instances = new Transform[SphereAmount];
for (int i = 0; i < SphereAmount; i++)
{
    instances[i] = Instantiate(Prefab, transform).transform;
}
```
```c++
//in update method
for (int i = 0; i < instances.Length; i++)
    instances[i].localPosition = output[i];
```
![](https://www.ronja-tutorials.com/assets/images/posts/050/Row.png)

## A tiny bit more complex Compute Shader

为了达到一个更好的视觉效果，请继续阅读，别担心这里涉及到的也只是基本的hlsl语法。

在compute shader中我加入了[randomness](https://www.ronja-tutorials.com/post/024-white-noise/)教程中关于噪声的函数，同时加入时间变量。在核函数中，我基于输入参数来构造一个长度为[0.1-1]的随机向量。然后采用叉乘的方法计算出与这些随机向量垂直的向量。然后使用时间变量的平方，加上一个比较大的奇数，得到一个关于时间的`sin`值和`cos`，最后将这两个值和两个随机向量相乘，并求和。在这个基础上乘以20，使得记过看起来更明显。
```c++
// Each #kernel tells which function to compile; you can have many kernels
#pragma kernel Spheres

#include "Random.cginc"

//variables
RWStructuredBuffer<float3> Result;
uniform float Time;

[numthreads(64,1,1)]
void Spheres (uint3 id : SV_DispatchThreadID)
{
    //generate 2 orthogonal vectors
    float3 baseDir = normalize(rand1dTo3d(id.x) - 0.5) * (rand1dTo1d(id.x)*0.9+0.1);
    float3 orthogonal = normalize(cross(baseDir, rand1dTo3d(id.x + 7.1393) - 0.5)) * (rand1dTo1d(id.x+3.7443)*0.9+0.1);
    //scale the time and give it a random offset
    float scaledTime = Time * 2 + rand1dTo1d(id.x) * 712.131234;
    //calculate a vector based on vectors
    float3 dir = baseDir * sin(scaledTime) + orthogonal * cos(scaledTime);
    Result[id.x] = dir * 20;
}
```

当然，我们很需要在`C#`脚本中向compute shader中传入时间变量。
```c++
Shader.SetFloat("Time", Time.time);
```

然后使用自发光材质，以及泛光后处理技术，最后呈现出下面绚烂的效果。
![](https://www.ronja-tutorials.com/assets/images/posts/050/result.gif)

## 源码

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/050_Compute_Shader/BasicCompute.compute](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/050_Compute_Shader/BasicCompute.compute)
```c++
// Each #kernel tells which function to compile; you can have many kernels
#pragma kernel Spheres

#include "Random.cginc"

//variables
RWStructuredBuffer<float3> Result;
uniform float Time;

[numthreads(64,1,1)]
void Spheres (uint3 id : SV_DispatchThreadID)
{
    //generate 2 orthogonal vectors
    float3 baseDir = normalize(rand1dTo3d(id.x) - 0.5) * (rand1dTo1d(id.x)*0.9+0.1);
    float3 orthogonal = normalize(cross(baseDir, rand1dTo3d(id.x + 7.1393) - 0.5)) * (rand1dTo1d(id.x+3.7443)*0.9+0.1);
    //scale the time and give it a random offset
    float scaledTime = Time * 2 + rand1dTo1d(id.x) * 712.131234;
    //calculate a vector based on vectors
    float3 dir = baseDir * sin(scaledTime) + orthogonal * cos(scaledTime);
    Result[id.x] = dir * 20;
}
```
[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/050_Compute_Shader/BasicComputeSpheres.cs](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/050_Compute_Shader/BasicComputeSpheres.cs)
```c++
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BasicComputeSpheres : MonoBehaviour
{
    public int SphereAmount = 17;
    public ComputeShader Shader;

    public GameObject Prefab;

    ComputeBuffer resultBuffer;
    int kernel;
    uint threadGroupSize;
    Vector3[] output;

    Transform[] instances;

    void Start()
    {
        //program we're executing
        kernel = Shader.FindKernel("Spheres");
        Shader.GetKernelThreadGroupSizes(kernel, out threadGroupSize, out _, out _);

        //buffer on the gpu in the ram
        resultBuffer = new ComputeBuffer(SphereAmount, sizeof(float) * 3);
        output = new Vector3[SphereAmount];

        //spheres we use for visualisation
        instances = new Transform[SphereAmount];
        for (int i = 0; i < SphereAmount; i++)
        {
            instances[i] = Instantiate(Prefab, transform).transform;
        }
    }

    void Update()
    {
        Shader.SetFloat("Time", Time.time);
        Shader.SetBuffer(kernel, "Result", resultBuffer);
        int threadGroups = (int) ((SphereAmount + (threadGroupSize - 1)) / threadGroupSize);
        Shader.Dispatch(kernel, threadGroups, 1, 1);
        resultBuffer.GetData(output);

        for (int i = 0; i < instances.Length; i++)
            instances[i].localPosition = output[i];
    }

    void OnDestroy()
    {
        resultBuffer.Dispose();
    }
}
```
希望你能喜欢这个教程 :-)