---
title: Unity Configurable Joints
date: 2020-11-30 10:38:34
categories:
- [Unity,Physics,Joints]
tags:
- Unity
- Physics
- Joints
---
`ConfigurableJoint`是Unity物理引擎中非常重要且功能强大的部分，它涵盖了其他Joint的所有功能。拥有超过40个配置选项，通过这些配置项的灵活运用，我们几乎可以创建任何铰接的物理效果。

但是官方文档对它的介绍非常简短，我们通常需要不断的微调参数，来理解其内部机制。即便是不断尝试，也很难掌握。

在探讨`ConfigurableJoint`之前，我们先来看一下他的基本参数。

## anchors and connected body

`anchor`和`connectedAnchor`是Joint中最基本的属性。他们是铰接世界的罗密欧与朱丽叶-这两者永远是成对出现的。Unity会尝试移动铰接对象，直到铰接对象中的这两个点重合。通常情况下，这两个点最终都会重合在一起，但是有两个特例。一个是在Joint中使用弹力，这时候，在弹力的作用下两个点会保持一定的距离。另一个是，当我们设置某个轴为自由时，例如我们设置X轴自由，这两个点的只有Y、Z坐标会保持对齐。
需要注意的是，默认情况下，`anchor`是模型空间下的坐标，`connectedAnchor`是世界坐标系下的坐标。所以当模型移动的时候，`anchor`点会跟随移动，但是`connectedAnchor`点则保持不动。
在本文中。与`anchor`相关联的叫做`铰接体`，与`connectedAnchor`相关联的叫做`被铰接体`。

### 来个例子

如下图所示，Unity将铰接物体的`anchor`点移动到`connectedAnchor`点：
![](https://miro.medium.com/max/576/1*Epf1F9HAYOnsRa0c0Ap7sg.png)

我们还可以通过移动`connectedAnchor`点来将控制整个铰接物体移动。如果我们设置`connectedBody`，那么`connectedAnchor`点的坐标不在是世界坐标，而是属于`connectedBody`所关联物体下的模型坐标系。当我们移动`connectedBody`时，整个铰链物体会一起移动。并且在移动过程中`anchor`点会不断靠近`connectedAnchor`点。

### 铰接物体的相互作用

当我们移动被铰接物体中的任何一个物体时，另外一个物体也会跟随运动：
![](https://miro.medium.com/max/454/1*Ag7OwgAV6uax3kND7qRAkA.gif)

如果我们将`auotConfigureConnectedAnchor`设置为True时，Unity将会忽略掉面板上设置的`connectedAnchor`参数，同时自动计算`connectedAnchor`的坐标，使得`connectedAnchor`和`anchor`两个点保持重合。

### 再来个例子

下面的代码模拟了Joint的内部机制：

``` bash
public class OurOwnJoint : MonoBehaviour {
    // 这些是可配置参数
    public Rigidbody connectedBody;
    public Vector3 anchor;
    public bool autoConfigureConnectedAnchor;
    public Vector3 connectedAnchor;

    // 执行初始化
    void Start() {
        var ourBody = GetComponent<Rigidbody>();
        // 如果 autoConfigureConnectedAnchor 是 true, 
        // 我们应该忽略 connectedAnchor.
        if ( this.autoConfigureConnectedAnchor ) {
            // Anchor 是相对于 ourbody 的局部坐标 - 我们将他转换为世界坐标.
            var anchorWorldPosition = ourBody.transform.TransformPoint( this.anchor );
            // 如果我们设置了 connectedBody，那么将 anchor 坐标映射到connectedBody局部坐标系中.
            // 否则，我们直接使用 anchor 的世界坐标.
            this.connectedAnchor = this.connectedBody != null ?
                this.connectedBody.transform.InverseTransformPoint( anchorWorldPosition ) :
                anchorWorldPosition;
        }
    }

    // 这里我们不断将 anchor 向 connectedAnchor 点靠拢
    void FixedUpdate() {
        var ourBody = GetComponent<Rigidbody>();
        // 计算 anchor 的世界坐标
        var anchorWorldPosition = ourBody.transform.TransformPoint( this.anchor );
        // 计算connectedAnchor 的世界坐标. 注意, 如果connectedBody没有设置，那么
        // 这个坐标就存储在 connectedAnchor 配置中
        var connectedAnchorWorldPosition = this.connectedBody != null ?
            this.connectedBody.transform.TransformPoint( this.connectedAnchor ) :
            this.connectedAnchor;
        // 这里计算了 connectedAnchor 和 anchor 两个点的距离
        var positionError = connectedAnchorWorldPosition - anchorWorldPosition;
        // 移动被铰接的物体，使得两个点靠近
        ourBody.position += positionError;
    }
}
```

上面内容相对简单，来加点难度吧！

## 线性运动

这里我们介绍三个属性：xMotion 、 yMotion 、 zMotion。

假设`connectedAnchor`和`anchor`点在世界坐标系下的坐标分别是(1,2,3)、(-1,-2,-3)。很明显这两个点不在同一个位置，前面讲过，在Joint的作用下，这两个点会不断靠近。

为了达到这个目的，Joint会不断修改`connectedAnchor` 或 `anchor` 的坐标。而修改坐标的方式是，在(2,4,6)方向移动`铰接体`，或者在(-2.-4,-6)方向移动`被铰接体`。
我们还可以通过修改运动属性来控制这种相互靠近的过程。

### 运动参数的配置

每个维度都可以配置一下任何一种运动方式：
- Locked: 当我们希望`铰接体`和`被铰接体`靠近时。例如我们希望`铰接体`和`被铰接体`的两个锚点重合，那么我们可以设置yMotion为Locked；
- Limited: 当我们希望在`铰接体`和`被铰接体`之间的设置距离上限时。例如希望`铰接体`和`被铰接体`在Y方向的距离不超过10，那么可以设置yMotion为Limited；
- Free：当我们对`铰接体`和`被铰接体`之间的相对位置不做要求时。例如我们希望`铰接体`和`被铰接体`处于同一垂线，这时候我们对y轴没有要求，那么我们可以设置yMotion为Free；

### 来个例子

下图中，我们将xMotion设置为Free，将yMotion设置为Locked，因此`铰接体`和`被铰接体`的锚点在y轴上重合，但是x轴保持不变；

### 再来个代码例子

下面通过代码来表示Joint的内部机制。当我们将Motion设置为Free时，Joint是不会修改铰接体的坐标的。

``` bash
// ... 跳过无关代码
if ( this.xMotion == ConfigurableJointMotion.Free ) {
    positionError.x = 0;
}
if ( this.yMotion == ConfigurableJointMotion.Free ) {
    positionError.y = 0;
}
if ( this.zMotion == ConfigurableJointMotion.Free ) {
    positionError.z = 0;
}
ourBody.position += positionError;
```

如果我们将所有的Motion都设置为Free，然后移动`被铰接体`，这时候我们会发现`铰接体`不会移动，而是会旋转。
接下来的内容会解释这个神奇的现象/

## Axes

首先，需要明确的一点是，上面的运动约束，其被约束的轴向是相对于`铰接体`的。因此当我们想





























