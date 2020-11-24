---
title: 顶点着色器中的坐标变化
date: 2020-11-12 19:01:34
---
{% blockquote %}
We are all in the gutter, but some of us are looking at the stars.
{% endblockquote %}

下图展示了顶点坐标在顶点着色器中的不同阶段的变化。
![wwwwwww](/blogs/images/src/13765939_10153839732515897_1876395612751424638_o.jpg)

其中NDC空间的z值，在DirectX和Opengl中的范围是不一样的：

- 在DirectX中，NDC的z值范围是[0,1];
- 在Opengl中，NDC的z值范围是[-1,1];

其中裁剪是发生在裁剪空间，而不是NDC空间。具体一点是在MVP转换之后，在透视除法之前。这样做可以减少除法执行的次数，提高硬件效率。因为先裁剪后做透视除法，可以剔除大量不会被显示的顶点，进而减少透视除法的次数。






