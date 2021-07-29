---
title: Voronoi Noise
date: 2021-07-12 19:01:00
categories:
- [Unity, Shader]
tags:
- Unity
- Shader
- 翻译
- ronja
---
原文：
[Voronoi Noise](https://www.ronja-tutorials.com/post/028-voronoi-noise/)

## Summary

还有一种噪声叫做维诺噪声。维诺噪声需要一堆随机点，然后基于这些相互最近的点之间构成的图案来生成噪声。维诺噪声和之前的噪声类似，也是基于噪声色块，只不过值类噪声对应色块的随机值表示的是灰度，而泊林噪声对应色块的随机值表示的是灰度变化趋势，维诺噪声对应色块的随机值表示的是色块中的某个随机位置。基于色块的计算相对简单，同时可重复，适用于并行计算。在阅读本文之前，建议你先对[着色器基础](https://tyson-wu.github.io/blogs/2021/07/01/Ronja_Basic_Shader/)有所了解，并且知道如何[在着色器中生成随机值](https://tyson-wu.github.io/blogs/2021/07/08/Ronja_White_Noise/)。
![](https://www.ronja-tutorials.com/assets/images/posts/028/Result.gif)

## Get Cell Values

在我们实现维诺噪声的过程中，每一个色块都对应一个随机点。首先我们来实现一个二维维诺噪声。首先我们照常将其分割为无数的小网格，然后生成随机向量，这个向量表示在对应网格中的位置。然后我们计算当前网格随机位置与当前处理位置的距离。当然整个计算过程都是基于世界坐标系，同时我们可以通过色块尺寸参数来控制色块的密集度。
```c++
Shader "Tutorial/028_voronoi_noise/2d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;

		struct Input {
			float3 worldPos;
		};

		float voronoiNoise(float2 value){
            float2 cell = floor(value);
            float2 cellPosition = cell + rand2dTo2d(cell);
            float2 toCell = cellPosition - value;
            float distToCell = length(toCell);
            return distToCell;
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float2 value = i.worldPos.xz / _CellSize;
			float noise = voronoiNoise(value);

			o.Albedo = noise;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

因为我们需要计算相邻近的随机点，所以我们不仅仅是要计算当前色块，还要计算相邻色块。因此我们使用`for`循环来遍历从-1到1的九宫格。在每次迭代中，我们都会计算色块随机点与输入值的距离，然后记录下最近距离的色块。用于记录最小距离的变量需要定义在循环外部，并且要有一个大于九宫格直径的默认值。然后我们使用`unroll`命令来将循环体展开，提升其执行效率。
```c++
float voronoiNoise(float2 value){
    float2 baseCell = floor(value);

    float minDistToCell = 10;
    [unroll]
    for(int x=-1; x<=1; x++){
        [unroll]
        for(int y=-1; y<=1; y++){
            float2 cell = baseCell + float2(x, y);
            float2 cellPosition = cell + rand2dTo2d(cell);
            float2 toCell = cellPosition - value;
            float distToCell = length(toCell);
            if(distToCell < minDistToCell){
                minDistToCell = distToCell;
            }
        }
    }
    return minDistToCell;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/MultiCellDistances.png)

当然，处理最近距离，我们还想知道最近的随机点是哪个。因此同样在循环外部定义一个坐标变量，然后在循环中更新。在得到随机点的坐标后，我们还可以使用随机函数为该随机点生成一个随机变量，来作为它的唯一标识。然后将函数返回类型改为二维向量，`x`值用来存最小距离，`y`存这个唯一标识。在表面着色器中我们就用这个唯一标识来做区域区分。
```c++
float voronoiNoise(float2 value){
    float2 baseCell = floor(value);

    float minDistToCell = 10;
    float2 closestCell;
    [unroll]
    for(int x=-1; x<=1; x++){
        [unroll]
        for(int y=-1; y<=1; y++){
            float2 cell = baseCell + float2(x, y);
            float2 cellPosition = cell + rand2dTo2d(cell);
            float2 toCell = cellPosition - value;
            float distToCell = length(toCell);
            if(distToCell < minDistToCell){
                minDistToCell = distToCell;
                closestCell = cell;
            }
        }
    }
    float random = rand2dTo1d(closestCell);
    return float2(minDistToCell, random);
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xz / _CellSize;
    float noise = voronoiNoise(value).y;
    o.Albedo = noise;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/GreyscaleCells.png)

重新划分的区域具有相同标识的表示同一区域。因此我们可以在表面着色器中根据这个唯一标识来生成彩色色块。这里我们只需要使用随机函数，将唯一表示转为为三维向量。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xz / _CellSize;
    float noise = voronoiNoise(value).y;
    float3 color = rand1dTo3d(noise);
    o.Albedo = color;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/ColorfulCells.png)

## Getting the distance to the border

上面通过计算点到随机点之间的距离，来得到以随机点为中心的色块。但是在很多情况下，例如显示色块之间的边界，我们可能希望计算点到边界的距离。一个常用的做法是计算最近随机点的距离、以及第二近的随机点的距离，然后两个距离作差。这个方法效率非常高，但是无法得到精确的边界信息。我们将要使用的方法是计算点到所有临近边界的距离，然后得到最短距离。

为了计算采样点到边界的距离，我们需要再一次遍历最近色块周围的色块。前面我们已经得到了最近色块的数据。然后我们来计算最近色块与其相邻色块边界到采样点的距离。首先我们计算这两个色块中心点，也就是随机点，连线的中心点的位置。然后构造由采样点到该连线中心点的向量。同时计算这两个随机点连线的单位向量。

得到这两个向量后，我们使用点乘，就得到采样点到边界的垂直距离。
![](https://www.ronja-tutorials.com/assets/images/posts/028/BorderDistanceExplanation.png)

在上面我们通过计算采样点到色块中心随机点的距离，得到了新的色块。然后现在我们创建一个新的变量来记录采样点到新生成的色块的边界距离。两次都需要使用到循环语句，并且两次循环都类似，都是从-1到1。所以我们这里对其重新命名，前一次循环的变量命名为`x1`和`y1`，后一次命名为`x2`和`y2`。

和第一次循环一样，在我们第二次循环的内循环中也要计算每个色块的随机中心点、以及采样点到随机中心点的向量。然后在条件语句下计算采样点到边界的距离。因为我们是要计算最近色块和其他色块边界到采样点的距离，而这里的最近色块也包含在九宫格中，所以需要将其剔除。当两个色块的随机中心点坐标相同时，说明他们是同一个色块，但是因为我们处理的是小数，存在精度问题，所以不能用等号来判断。

然后在条件语句中，我们处理的是不相同的两个色块，计算他们随机中心点所构成的方向向量，然后计算单位向量。并且计算这两个随机中心点连线的中心点，然后和采样点构造一个方向向量。这两个方向向量点乘便是采样点到边界的距离了。然后我们记录下采样点到周围边界的距离最小值。

在得到距离边界的最小值后，我们将函数返回值扩展为三维向量，然后将最小值储存到`z`值中。
```c++
float3 voronoiNoise(float2 value){
    float2 baseCell = floor(value);

    //查找最近色块
    float minDistToCell = 10;
    float2 toClosestCell;
    float2 closestCell;
    [unroll]
    for(int x1=-1; x1<=1; x1++){
        [unroll]
        for(int y1=-1; y1<=1; y1++){
            float2 cell = baseCell + float2(x1, y1);
            float2 cellPosition = cell + rand2dTo2d(cell);
            float2 toCell = cellPosition - value;
            float distToCell = length(toCell);
            if(distToCell < minDistToCell){
                minDistToCell = distToCell;
                closestCell = cell;
                toClosestCell = toCell;
            }
        }
    }

    //查找最近色块边界
    float minEdgeDistance = 10;
    [unroll]
    for(int x2=-1; x2<=1; x2++){
        [unroll]
        for(int y2=-1; y2<=1; y2++){
            float2 cell = baseCell + float2(x2, y2);
            float2 cellPosition = cell + rand2dTo2d(cell);
            float2 toCell = cellPosition - value;

            float2 diffToClosestCell = abs(closestCell - cell);
            bool isClosestCell = diffToClosestCell.x + diffToClosestCell.y < 0.1;
            if(!isClosestCell){
                float2 toCenter = (toClosestCell + toCell) * 0.5;
                float2 cellDifference = normalize(toCell - toClosestCell);
                float edgeDistance = dot(toCenter, cellDifference);
                minEdgeDistance = min(minEdgeDistance, edgeDistance);
            }
        }
    }

    float random = rand2dTo1d(closestCell);
    return float3(minDistToCell, random, minEdgeDistance);
}
```
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xz / _CellSize;
    float3 noise = implVoronoiNoise(value);
    o.Albedo = noise.z;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/EdgeDistances.png)

## Visualising vornoi noise

现在我们维诺噪声函数返回了三个值：采样点到色块随机中心点的距离、当前色块的唯一标识、采样点到色块边界的距离。前面已经介绍了使用唯一标识来生成彩色色块。我们还可以根据采样点到边界的距离来绘制色块边界。因此我们需要根据距离来判断哪些是边界，哪些不是。这个可以使用阶跃函数来实现，大于阈值则返回1，小于阈值则返回0。在得到边界判定值后，我们可以用来对边界颜色、和色块颜色进行插值。我们也可以将边界颜色暴露在材质面板上，这样方便我们后面调节。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float2 value = i.worldPos.xz / _CellSize;
    float3 noise = voronoiNoise(value);

    float3 cellColor = rand1dTo3d(noise.y); 
    float isBorder = step(noise.z, 0.05);
    float3 color = lerp(cellColor, _BorderColor, isBorder);
    o.Albedo = color;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/AliasedBorders.png)

这里有个问题，就是我们上面的边界计算是二选一的，非0即1，这样在颜色插值的时候也变成二选一，会产生明显的锯齿效果。我们可以提前对线条进行模糊操作，通过计算采样点周围边界距离的变化情况，然后将这个变化梯度，当作我们模糊区域的边界，在这个模糊区域外的，依然是二选一，但是在模糊区域以内的边界，则是进行颜色混合。

因为这里计算的边界距离和维诺噪声输入函数的输入值具有相同的缩放关系，所以我们以该输入值为参考，来计算比边界距离的梯度。这里我们还是使用`fwidth`来计算梯度，因为输入值是二维向量，所以梯度也是二维的，我们可以计算这个梯度向量的长度，然后以此来作为我们模糊区域的边界宽度。然后我们基于阈值进行上下偏移半个边界宽度。你也可以试着调整这个偏移宽度，看看会出现什么效果。

当我们计算出边界模糊区域的上下阈值，我们使用`smoothstep`来替代`step`函数，这样就可以应用边界模糊区域了。然后我们对结果翻转，这样我们的边界值就是1。

```c++
void surf (Input i, inout SurfaceOutputStandard o) {
	float2 value = i.worldPos.xz / _CellSize;
	float3 noise = voronoiNoise(value);

	float3 cellColor = rand1dTo3d(noise.y); 
	float valueChange = length(fwidth(value)) * 0.5;
	float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
	float3 color = lerp(cellColor, _BorderColor, isBorder);
	o.Albedo = color;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/BordersNoAliasing.png)

## 3d Voronoi

三维维诺噪声的实现方式类似，不过噪声函数的输入值要改成三维先向量，我们的色块也要变成三维的。另外原先平面上的九宫格要换成`3x3x3`的立方体。而函数返回值依然保持和二维维诺噪声一致。

在表面着色气函数中，我们将整个坐标值传入维诺噪声函数中。但是我们的边界梯度不能再用坐标值来求了，因为现在坐标是三维的，它的梯度方向也是三维的，而我们边界模糊区域应该是沿着二维屏幕方向，如果继续使用三维坐标来求梯度，会有部分区域的模糊区域过大或过小。所以我们这里直接使用边界距离来求解梯度。
```c++
float3 voronoiNoise(float3 value){
    float3 baseCell = floor(value);

    //查找最近色块
    float minDistToCell = 10;
    float3 toClosestCell;
    float3 closestCell;
    [unroll]
    for(int x1=-1; x1<=1; x1++){
        [unroll]
        for(int y1=-1; y1<=1; y1++){
            [unroll]
            for(int z1=-1; z1<=1; z1++){
                float3 cell = baseCell + float3(x1, y1, z1);
                float3 cellPosition = cell + rand3dTo3d(cell);
                float3 toCell = cellPosition - value;
                float distToCell = length(toCell);
                if(distToCell < minDistToCell){
                    minDistToCell = distToCell;
                    closestCell = cell;
                    toClosestCell = toCell;
                }
            }
        }
    }

    //查找最近色块边界
    float minEdgeDistance = 10;
    [unroll]
    for(int x2=-1; x2<=1; x2++){
        [unroll]
        for(int y2=-1; y2<=1; y2++){
            [unroll]
            for(int z2=-1; z2<=1; z2++){
                float3 cell = baseCell + float3(x2, y2, z2);
                float3 cellPosition = cell + rand3dTo3d(cell);
                float3 toCell = cellPosition - value;

                float3 diffToClosestCell = abs(closestCell - cell);
                bool isClosestCell = diffToClosestCell.x + diffToClosestCell.y + diffToClosestCell.z < 0.1;
                if(!isClosestCell){
                    float3 toCenter = (toClosestCell + toCell) * 0.5;
                    float3 cellDifference = normalize(toCell - toClosestCell);
                    float edgeDistance = dot(toCenter, cellDifference);
                    minEdgeDistance = min(minEdgeDistance, edgeDistance);
                }
            }
        }
    }

    float random = rand3dTo1d(closestCell);
    return float3(minDistToCell, random, minEdgeDistance);
}
```

```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos.xyz / _CellSize;
    float3 noise = voronoiNoise(value);

    float3 cellColor = rand1dTo3d(noise.y); 
    float valueChange = fwidth(value.z) * 0.5;
    float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
    float3 color = lerp(cellColor, _BorderColor, isBorder);
    o.Albedo = color;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/3dVoronoi.png)

## Scrolling noise

像前面介绍过的噪声一样，这里的噪声也不局限于使用空间坐标。我们可以将前面两个维度的输入值使用空间坐标，而第三个维度使用时间。这样随着时间的推移，我们可以看到噪声在不断发生变化。
```c++
void surf (Input i, inout SurfaceOutputStandard o) {
    float3 value = i.worldPos.xyz / _CellSize;
    value.y += _Time.y * _TimeScale;
    float3 noise = voronoiNoise(value);

    float3 cellColor = rand1dTo3d(noise.y); 
    float valueChange = fwidth(value.z) * 0.5;
    float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
    float3 color = lerp(cellColor, _BorderColor, isBorder);
    o.Albedo = color;
}
```
![](https://www.ronja-tutorials.com/assets/images/posts/028/Result.gif)

## Source 

### 2d Voronoi

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_2d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_2d.shader)
```c++
Shader "Tutorial/028_voronoi_noise/2d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_BorderColor ("Border Color", Color) = (0,0,0,1)
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;
		float3 _BorderColor;

		struct Input {
			float3 worldPos;
		};

		float3 voronoiNoise(float2 value){
			float2 baseCell = floor(value);

			//查找最近色块
			float minDistToCell = 10;
			float2 toClosestCell;
			float2 closestCell;
			[unroll]
			for(int x1=-1; x1<=1; x1++){
				[unroll]
				for(int y1=-1; y1<=1; y1++){
					float2 cell = baseCell + float2(x1, y1);
					float2 cellPosition = cell + rand2dTo2d(cell);
					float2 toCell = cellPosition - value;
					float distToCell = length(toCell);
					if(distToCell < minDistToCell){
						minDistToCell = distToCell;
						closestCell = cell;
						toClosestCell = toCell;
					}
				}
			}

			//查找最近色块边界
			float minEdgeDistance = 10;
			[unroll]
			for(int x2=-1; x2<=1; x2++){
				[unroll]
				for(int y2=-1; y2<=1; y2++){
					float2 cell = baseCell + float2(x2, y2);
					float2 cellPosition = cell + rand2dTo2d(cell);
					float2 toCell = cellPosition - value;

					float2 diffToClosestCell = abs(closestCell - cell);
					bool isClosestCell = diffToClosestCell.x + diffToClosestCell.y < 0.1;
					if(!isClosestCell){
						float2 toCenter = (toClosestCell + toCell) * 0.5;
						float2 cellDifference = normalize(toCell - toClosestCell);
						float edgeDistance = dot(toCenter, cellDifference);
						minEdgeDistance = min(minEdgeDistance, edgeDistance);
					}
				}
			}

			float random = rand2dTo1d(closestCell);
    		return float3(minDistToCell, random, minEdgeDistance);
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float2 value = i.worldPos.xz / _CellSize;
			float3 noise = voronoiNoise(value);

			float3 cellColor = rand1dTo3d(noise.y); 
			float valueChange = length(fwidth(value)) * 0.5;
			float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
			float3 color = lerp(cellColor, _BorderColor, isBorder);
			o.Albedo = color;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### 3d Voronoi

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_3d.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_3d.shader)

```c++
Shader "Tutorial/028_voronoi_noise/3d" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_BorderColor ("Border Color", Color) = (0,0,0,1)
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;
		float3 _BorderColor;

		struct Input {
			float3 worldPos;
		};

		float3 voronoiNoise(float3 value){
			float3 baseCell = floor(value);

			//查找最近色块
			float minDistToCell = 10;
			float3 toClosestCell;
			float3 closestCell;
			[unroll]
			for(int x1=-1; x1<=1; x1++){
				[unroll]
				for(int y1=-1; y1<=1; y1++){
					[unroll]
					for(int z1=-1; z1<=1; z1++){
						float3 cell = baseCell + float3(x1, y1, z1);
						float3 cellPosition = cell + rand3dTo3d(cell);
						float3 toCell = cellPosition - value;
						float distToCell = length(toCell);
						if(distToCell < minDistToCell){
							minDistToCell = distToCell;
							closestCell = cell;
							toClosestCell = toCell;
						}
					}
				}
			}

			//查找最近色块边界
			float minEdgeDistance = 10;
			[unroll]
			for(int x2=-1; x2<=1; x2++){
				[unroll]
				for(int y2=-1; y2<=1; y2++){
					[unroll]
					for(int z2=-1; z2<=1; z2++){
						float3 cell = baseCell + float3(x2, y2, z2);
						float3 cellPosition = cell + rand3dTo3d(cell);
						float3 toCell = cellPosition - value;

						float3 diffToClosestCell = abs(closestCell - cell);
						bool isClosestCell = diffToClosestCell.x + diffToClosestCell.y + diffToClosestCell.z < 0.1;
						if(!isClosestCell){
							float3 toCenter = (toClosestCell + toCell) * 0.5;
							float3 cellDifference = normalize(toCell - toClosestCell);
							float edgeDistance = dot(toCenter, cellDifference);
							minEdgeDistance = min(minEdgeDistance, edgeDistance);
						}
					}
				}
			}

			float random = rand3dTo1d(closestCell);
    		return float3(minDistToCell, random, minEdgeDistance);
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos.xyz / _CellSize;
			float3 noise = voronoiNoise(value);

			float3 cellColor = rand1dTo3d(noise.y); 
			float valueChange = fwidth(value.z) * 0.5;
			float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
			float3 color = lerp(cellColor, _BorderColor, isBorder);
			o.Albedo = color;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

### Scrolling Voronoi

[https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_scrolling.shader](https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/028_Voronoi_Noise/voronoi_noise_scrolling.shader)
```c++
Shader "Tutorial/028_voronoi_noise/scrolling" {
	Properties {
		_CellSize ("Cell Size", Range(0, 2)) = 2
		_BorderColor ("Border Color", Color) = (0,0,0,1)
		_TimeScale ("Scrolling Speed", Range(0, 2)) = 1
	}
	SubShader {
		Tags{ "RenderType"="Opaque" "Queue"="Geometry"}

		CGPROGRAM

		#pragma surface surf Standard fullforwardshadows
		#pragma target 3.0

		#include "Random.cginc"

		float _CellSize;
		float _TimeScale;
		float3 _BorderColor;

		struct Input {
			float3 worldPos;
		};

		float3 voronoiNoise(float3 value){
			float3 baseCell = floor(value);

			//查找最近色块
			float minDistToCell = 10;
			float3 toClosestCell;
			float3 closestCell;
			[unroll]
			for(int x1=-1; x1<=1; x1++){
				[unroll]
				for(int y1=-1; y1<=1; y1++){
					[unroll]
					for(int z1=-1; z1<=1; z1++){
						float3 cell = baseCell + float3(x1, y1, z1);
						float3 cellPosition = cell + rand3dTo3d(cell);
						float3 toCell = cellPosition - value;
						float distToCell = length(toCell);
						if(distToCell < minDistToCell){
							minDistToCell = distToCell;
							closestCell = cell;
							toClosestCell = toCell;
						}
					}
				}
			}

			//查找最近色块边界
			float minEdgeDistance = 10;
			[unroll]
			for(int x2=-1; x2<=1; x2++){
				[unroll]
				for(int y2=-1; y2<=1; y2++){
					[unroll]
					for(int z2=-1; z2<=1; z2++){
						float3 cell = baseCell + float3(x2, y2, z2);
						float3 cellPosition = cell + rand3dTo3d(cell);
						float3 toCell = cellPosition - value;

						float3 diffToClosestCell = abs(closestCell - cell);
						bool isClosestCell = diffToClosestCell.x + diffToClosestCell.y + diffToClosestCell.z < 0.1;
						if(!isClosestCell){
							float3 toCenter = (toClosestCell + toCell) * 0.5;
							float3 cellDifference = normalize(toCell - toClosestCell);
							float edgeDistance = dot(toCenter, cellDifference);
							minEdgeDistance = min(minEdgeDistance, edgeDistance);
						}
					}
				}
			}

			float random = rand3dTo1d(closestCell);
    		return float3(minDistToCell, random, minEdgeDistance);
		}

		void surf (Input i, inout SurfaceOutputStandard o) {
			float3 value = i.worldPos.xyz / _CellSize;
			value.y += _Time.y * _TimeScale;
			float3 noise = voronoiNoise(value);

			float3 cellColor = rand1dTo3d(noise.y); 
			float valueChange = fwidth(value.z) * 0.5;
			float isBorder = 1 - smoothstep(0.05 - valueChange, 0.05 + valueChange, noise.z);
			float3 color = lerp(cellColor, _BorderColor, isBorder);
			o.Albedo = color;
		}
		ENDCG
	}
	FallBack "Standard"
}
```

希望本文有助你理解什么是维诺噪声。

希望你能喜欢这个教程哦！如果你想支持我，可以关注我的[推特](https://twitter.com/totallyRonja),或者通过[ko-fi](https://ko-fi.com/ronjatutorials)、或[patreon](https://www.patreon.com/RonjaTutorials)给两小钱。总之，各位大爷，走过路过不要错过，有钱的捧个钱场，没钱的捧个人场:-)!!!
