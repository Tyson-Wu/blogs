---
title: URP 学习总结1
date: 2021-04-06 10:01:00
categories:
- [Unity, URP]
tags:
- Render
- Shader
---

## URP提供的工具函数

`VertexPositionInputs GetVertexPositionInputs(positionOS)`
计算出各个空间下的顶点坐标

`VertexNormalInputs GetVertexNormalInputs(normalOS)`
计算出各个空间下的顶点法向坐标

`half3 VertexLighting(positionWS, normalWS)`
获取指定顶点的顶点光照，当定义了`_ADDITIONAL_LIGHTS_VERTEX`，表明使用顶点光照，并返回光照值，否则返回零
`uint GetAdditionalLightsCount()`
获取作用在当前空间位置上的辅光源数量，其中`_AdditionalLightsCount.x`表示场景中所有被激活的灯光，在场景剔除的过程中，由渲染管线计算并提交。而`unity_LightData.y`应该是表示所有的光照数量。
`Light GetAdditionalLight(lightIndex, positionWS)`
获取指定辅光源作用在指定空间位置的光照值，`lightIndex`表示辅光源在当前空间点辅光源列表中的索引值。
`int GetPerObjectLightIndex(lightIndex)`
基于当前辅光源在当前空间点辅光源列表中的索引值，计算辅光源在所有激活光源中的索引值
`Light GetAttitionalPerObjectLight`(perObjectLightIndex, positionWS)`
场景中的每个物体受部分光源的影响，具体受哪些光源的影响根据光源排序确定，然后每个物体记录其受光源的索引值。需要清楚地是所有的光源信息是统一存放在一个公共Buffer里面。




shadowParams.x : 阴影强度
shadowParams.y : 1.0 表示软阴影， 0.0表示其他


unity_LightData.x ：表示当前模型灯光在灯光缓存中的起始索引偏移值

unity_LightData.z : 表示该光源是否可见，可见为1.0，不可见为0.0

float4 unity_LightIndices[2] : 每个模型最大辅光源个数为8， 这里面便存着实际光源的索引值


物理含义
NdotH : 表示观察视角和镜面反射的重合度，重合度越高，观察的光越强
LdotH : 表示入射光和观察视角的重合度

