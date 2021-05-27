---
title: Line
date: 2021-05-27 15:01:00
categories:
- [Math, Line]
tags:
- Math
- Line
---

## 线段相交

对于平面上给定的两个线段line0、line1，求解其交点
```
public static bool LinesIntersect(Vector2 line0_start, Vector2 line0_end, Vector2 line1_start, Vector2 line1_end, out float line0_cross_line1, out float along_line0, out along_line1, out Vector2 intersect_point)
{
	var dir_line0 = line0_end - line0_start;
	var dir_line1 = line1_end - line1_start;
	line0_cross_line1 = dir_line0.x * dir_line1.y - dir_line0.y * dir_line1.x;
	along_line0 = 0;
	along_line1 = 0;
	intersect_point = Vector2.zero;
	if(line0_cross_line1 == 0) return false;
	var dir_line_start01 = line0_start - line1_start;
	along_line0 = (dir_line1.x * dir_line_start01.y - dir_line1.y * dir_line_start01.x) / line0_cross_line1;
	along_line1 = (dir_line0.x * dir_line_start01.y - dir_line0.y * dir_line_start01.x) / line0_cross_line1;
	// if (along_line0 < 0 || along_line0 > 1 || along_line1 < 0 || along_line1 > 1) return false;
	return true;
}
```
相关链接：
[Math Open Reference](https://mathopenref.com/coordintersection.html)
[Polygon Clipping](https://sean.cm/a/polygon-clipping-pt1)
[Barend's Blog](https://barendgehrels.blogspot.com/2010/12/intersections-1.html)
[Concave Polygon Intersection - Algorithm](https://cs.stackexchange.com/questions/99927/concave-polygon-intersection-algorithm)
[Euclidean Shortest Paths](https://fribbels.github.io/shortestpath/writeup.html)