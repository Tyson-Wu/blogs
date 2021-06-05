---
title: Graph
date: 2021-06-05 15:01:00
categories:
- [Gemotry, Graph]
tags:
- Gemotry
- Graph
---

## 

```
public static class SortHelp
{
	public delegate bool Compare<T>(T a, T b);
	public static void QuickSort<T>(List<T> array, int start, int end, Compare<T> compare)
	{
		if(start < end>)
		{
			int pIndex = Partition<T>(array, start, end, compare);
			QuickSort(array, start, pIndex - 1, compare);
			QuickSort(array, pIndex + 1, end, compare);
		}
	}

	static in Partition<T>(List<T> array, int start, int end, Compare<T> compare)
	{
		int pIndex = start;
		T pivot = array[end];
		for(int i = start; i< end; ++i>)
		{
			if(compare(array[i], pivot))
			{
				Swap<T>(array, i, pIndex);
				++i;
			}
		}
		Swap(array, end, pIndex);
		return pIndex;
	}
	static void Swap(List<T> array, int x, int y)
	{
		T t = array[x];
		array[x] = array[y];
		array[y] = t;
	}
}
```

```
public static class PolygonHelp
{
	public static float Cross(ref Vector2 edge1, ref Vector2 edge2)
	{
		return edge1.x * edge2.y - edge1.y * edge2.x;
	}
	public static float Cross(Vector2 edge1, Vector2 edge2)
	{
		return edge1.x * edge2.y - edge1.y * edge2.x;
	}
	public static float Dot(ref Vector2 edge1, ref Vector2 edge2)
	{
		return edge1.x * edge2.x + edge1.y * edge2.y;
	}
	public static float Dot(Vector2 edge1, Vector2 edge2)
	{
		return edge1.x * edge2.x + edge1.y * edge2.y;
	}
	public static float SignedAngle0_2PI(Vector2 dir1, Vector2 dir2)
	{
		dir1.Normalize();
		dir2.Normalize();
		float aCos = Dot(ref dir1, ref dir2);
		float angle = Mathf.Acos(aCos);
		float sign = Corss(ref dir1, ref dir2);
		if(sign < 0)
		{
			angle = 2 * Mathf.PI - angle;
		}
		return angle;
	}
	public static bool ClockWise(List<Vector2> polygon)
	{
		float area = 0;
		for(int pre = polygon.Count - 1, i =0; i < polygon.Count; pre = i, ++i)
		{
			area += Corss(polygon[pre], polygon[i] - polygon[pre]);
		}
		return area > 0;
	}
	public static bool PointInsideRegion(Vector2 pt, List<Vector2> region, float eps = float.Epsilon)
	{
		var x = pt[0];
		var y = pt[1];
		var last_x = region[region.Count -1][0];
		var last_y = region[region.Count -1][1];
		bool inside = false;
		for(int i=0; i<region.Count; ++i)
		{
			int curr_x = region[i][0];
			int curr_y = region[i][1];
			if((curr_y - y > eps) != (last_y -y > eps) && 
				(last_x - curr_x) * (y - curr_y) / (last_y - curr_y) + curr_x - x > eps)
				inside = !inside;
			last_x = curr_x;
			last_y = curr_y;
		}
		return inside;
	}
}
```

```
class Point
{
	public float x;
	public float y;
	public int index;
	public Edge edge1;
	public Edge edge2;
	public Point prePoint;
	public Point nextPoint;
	public float angle;
	public float maxAngle;
	public Point(){}
	public Point(Vector3 p, int i)
	{
		index = i;
		x = p.x;
		y = p.z;
	}
	public Point(Vector2 p, int i)
	{
		index = i;
		x = p.x;
		y = p.y;
	}
	public Point(float x, float y)
	{
		this.x = x;
		this.y = y;
	}
	public void Set(ref Vector2 v)
	{
		x = v.x;
		y = v.y;
	}
	public Vector3 ToVector3()
	{
		return new Vector3(x, 0, y);
	}
	public Vector2 ToVector2()
	{
		return new Vector2(x, y);
	}
	public static float Distance(Point p1, Point p2)
	{
		return Mathf.Sqrt((p1.x - p2.x) * (p1.x - p2.x) + (p1.y - p2.y) * (p1.y - p2.y));
	}
}
```

```
class Edge
{
	public Point point1;
	public Point point2;
	public Edge(Point p1, Point p2)
	{
		point1 = p1;
		point2 = p2;
	}
}
```

```
public class linesIntersect
{
	public int alongA;
	public int alongB;
	public Vector2 pt;
}
public class PathGraph
{
	private List<List<Point>> polygons = new List<List<Point>>();
	private List<Point> allPoints = new List<Point>();
	private List<List<Point>> graph = new List<List<Point>>();

	private Dictionary<int, Dictionary<int, float>> weightedGraph = new Dictionary<int, Dictionary<int, float>>();
	void Clear()
	{
		foreach(var t in polygons) t.Clear();
		polygons.Clear();
		foreach(var p in graph) p.Clear();
		graph.Clear();
		allPoints.Clear();
		foreach(var v in weightedGraph) v.Value.Clear();
		weightedGraph.Clear();
	}
	public void InitPolygons(PathBoundary boundary, List<PathBoundary> obstacles)
	{
		Clear();
		List<Point> points = null;
		int index = 0;
		points = new List<Point>();
		foreach(var point in boundary.polygon)
		{
			points.Add(new Point(point, index++));
		}
		this.polygons.Add(points);

		foreach(var obstacle in obstacles)
		{
			points = new List<Point>();
			foreach(var point in obstacle.polygon)
			{
				points.Add(new Point(point, index++));
			}
			this.polygons.Add(points);
		}
		Init();
	}
	void Init()
	{
		Point prePoint, curPoint;
		int pre;
		foreach(var polygon in polygons)
		{
			pre = polygon.Count - 1;
			for(int i =0; i<polygon.Count; ++i)
			{
				polygon[pre].nextPoint = polygon[i];
				polygon[i].prePoint = polygon[pre];
				Edge edge = new Edge(polygon[pre], polygon[i]);
				polygon[pre].edge2 = edge;
				polygon[i].edge1 = edge;

				allPoints.Add(polygon[i]);
				pre = i;
			}
		}
		foreach(var polygon in polygons)
		{
			foreach(var point in polygon)
			{
				Vector2 dir1 = point.edge1.point1.ToVector2() - point.ToVector2();
				Vector2 dir2 = point.edge2.point2.ToVector2() - point.ToVector2();
				point.maxAngle = PolygonHelp.SiginAngle0_2PI(dir1, dir2);
			}
		}
	}
	public void Graph(ref List<int> point1, ref List<int> point2, ref List<float> dist, ref bool reBuild)
	{
		if(reBuild)
		{
			CalculateGraph();
			point1.Clear();
			point2.Clear();
			dist.Clear();
			foreach(var links in weightedGraph)
			{
				foreach(var end in links.Value)
				{
					point1.Add(links.Key);
					point2.Add(end.Key);
					dist.Add(end.Value);
				}
			}
		}
		else
		{
			SetGraph(ref point1, ref point2, ref dist);
		}
	}
	void SetGraph(ref List<int> startPoints, ref List<int> endPoints, ref List<float> dist)
	{
		graph.Clear();
		weightedGraph.Clear();
		int startIndex = 0, endIndex = 0;
		float distance = 0;
		for(int i=0; i<startPoints.Count; ++i)
		{
			graph.Add(new List<Point>());
		}
		for(int i=0; i<startPoints.Count; ++i)
		{
			startIndex = startPoints[i];
			endIndex = endPoints[i];
			distance = dist[i];
			if(!weightedGraph.TryGetValue(startIndex, out var pointDist))
			{
				pointDist = new Dictionary<int, float>();
				weightedGraph[startIndex] = pointDist;
			}
			pointDist[endIndex] = distance;
			var edges = graph[startIndex];
			edges.Add(allPoints[endIndex]);
		}
	}
	void CalculateGraph()
	{
		List<Point> others = new List<Point>();
		for(int i= 0;i<allPoints.Count;++i)
		{
			Point point = allPoints[i];
			others.Clear();
			others.AddRange(allPoints);
			others.RemoveAt(i);
			var connected = RotationalSweep(point, ohters, allPoints);
			connected = RemoveNeedlessConnected(point, connected);
			graph.Add(connected);
		}
		for(int i=0;i<graph.Count; ++i)
		{
			Dictionary<int, float> pointDist = new Dictionary<int, float>();
			Point point = allPoints[i];
			foreach(var g in graph[i])
			{
				pintDist.Add(g.index, Point.Distance(point, g));
			}
			weightedGraph.Add(point.index, pointDist);
		}
	}
	Point tempP1 = new Point();
	Point tempP2 = new Point();
	public bool CandirectlyLink(Vector2 point1, Vector2 point2)
	{
		foreach(var p in allPoints)
		{
			var rst = LinesIntersect(point1, point2, p.edge1.point1, p.edge1.point2);
			if(rst != null)
			{
				if(rst.alongA ==0 && rst.alongB == 0) return false;
			}
		}
		return true;
	}
	public bool FindPath(Vector2 point1, Vector2 point2, ref List<Vector2> paths)
	{
		tempP1.Set(ref point1);
		tempP2.Set(ref point2);
		tempP1.maxAngle = 1000;
		tempP2.maxAngle = 1000;
		return FindPath(tempP1, tempP2, ref paths);
	}
	bool FindPath(Point point1, Point point2, ref List<Vector2> paths)
	{
		paths.Clear();
		List<Point> others = new List<Point>();
		others.AddRange(allPoints);
		List<Point> searched1 = RotationalSweep(point1, others, allPoints);
		if(searched1.Count == 0) return false;
		others.Clear();
		other.AddRange(allPoints);
		List<Point> searched2 = RotationalSweep(point2, others, allPoints);
		if(searched2.Count == 0) return false;

		InsertWeightGraph(point1, searched1);
		InsertWeightGraph(point2, searched2);

		ProorityQueue<int> frontier = new PriorityQueue<int>(true);
		frontier.Enqueue(0, point1.index);
		Dictionary<int, int> cameFrom = new Dictionary<int,int>();
		Dictionary<int, float> costSoFar = new Dictionary<int, float>();
		cameForm[point1.index]=-1;
		costSoFar[point1.index] = 0;
		while(frontier.Count != 0)
		{
			var current = frontier.Dequeue();
			if(current == point2.index) break;
			var neightbors = weightedGraph[current];
			foreach(var next in neightbors)
			{
				float newCost = costSorFar[current] + next.Value;
				if(!cameFrom.ContainsKey(next.Key) || newCost < costSorFar[next.Key])
				{
					costSorFar[next.Key] = new Cost;
					frontier.Enqueue(newCost, next.Key);
					cameFrom[next.Key] = current;
				}
			}
		}
		List<int> indices = new List<int>();
		int end = point2.index;
		while(end!=-1)
		{
			indices.Add(end);
			end = cameForm[end];
		}
		for(int i=indices.Count - 1;i>=0;--i)
		{
			paths.Add(allPoints[indices[i]].ToVector2());
		}
		RemoveWeightGraph(point2);
		RemoveWeightGraph(point1);
		return true;
	}

	void InsertWeightGraph(Point point, List<Point> linkedPoints)
	{
		point.index = allPoints.Count;
		allPoints.Add(point);
		Dictionary<int, float> pointDist = new Dictionary<int, float>();
		foreach(var s in linkedPoints)
		{
			float dist = Point.Distance(s, point);
			pointDist.Add(s.index, dist);
			weightedGraph[s.index].Add(point.index, dist);
		}
		weightedGraph.Add(point.index, pointDist);
	}

	void RemoveWeightGraph(Point point)
	{
		allPoints.RemoveAt(point.index);
		Dictionary<int, float> pointDist = weightedGraph[point.index];
		foreach(var v in pointDist)
		{
			weightedGraph[v.Key].Remove(point.index);
		}
		pointDist.Clear();
		weightedGraph.Remove(point.index);
	}
	public void GetMesh(ref List<Vector3> vertices, ref List<int> indices, bool clear = false)
	{
		if(clear)
		{
			vertices.Clear();
			indices.Clear();
		}
		int index = indices.Count;
		for(int i=0;i<allPoints.Count;++i)
		{
			Point point = allPoints[i];
			foreach(var g in graph[i])
			{
				vertices.Add(point.ToVector3());
				vertices.Add(g.ToVector3());
				indices.Add(index++);
				indices.Add(index++);
			}
		}
	}
	List<Point> RemoveNeedlessConnected(Point point, List<Point> points)
	{
		List<Point> temp = new List<Point>();
		foreach(var p in points)
		{
			if(point == p.edge1.point1 || point == p.edge2.point2)
			{
				temp.Add(p);
			}
			else if(!points.Contains(p.edge1.point1) || !points.Contains(p.edge2.point2))
			{
				if(!(point.maxAngle < Mathf.PI || p.maxAngle < Mathf.PI))
					temp.Add(p);
			}
		}
		points = temp;
		return points;
	}
	List<Point> RotationalSweep(Point point, List<Point> others, List<Point> all)
	{
		Point right = new Point(10000+point.x, point.y);
		foreach(var p in others)
		{
			Vector2 dir = new Vector(p.x - point.x, p.y - point.y).normalized;
			float dot = PolygonHelp.Dot(Vector.right, dir);
			float anlge = Mathf.Acos(dot);
			if(dir.y<0) angle = 2 * Mathf.PI - angle;
			p.angle = angle;
		}
		SortHelp.QuickSort<Point>(others, o, others.Count - , (a, b) =>
		{
			return a.angle < b.angle;
		});
		List<Edge> edges = new List<Edge>();
		foreach(var p in all)
		{
			var rst = LinesIntersect(point, right, p. edge1.point1, p.edge1.point2);
			if(rst!=null)
			{
				if(rst.angleA == 0 && rst.angleB ==0)
				{
					if(!edges.Contains(p.edge1))
					{
						edges.Add(p.edge1);
					}
				}
			}
		}
		bool visible = false;
		List<Point> visiblePoint = new List<Point>();
		foreach(var p in others)
		{
			if(point == p.edge1.point1 || point == p.edge2.point2){
				visible = true;
			}
			else
			{
				if(point.maxAngle < Mathf.PI || point.maxAngle<Mathf.PI)
				{
					visible = false;
				}
				else
				{
					visible = true;
					foreach(var edge in edges)
					{
						var rst = LinesIntersect(point, p, edge.point1, point2)
						if(rst != null)
						{
							if(rst.alongA ==0 && rst.alongB == 0)
							{
								visible = false;
								break;
							}
							else
							{
								if(point.edge1 != null)
								{
									Vertor2 dir1 = point.edge1.point1.ToVector2() - point.ToVector2();
									Vector2 dir2 = p.ToVector2() - point.ToVector2();
									float angle = PolygonHelp.SignedAngle0_2PI(dir1, dir2);
									if(angle > point.maxAngle)
									{
										visible = false;
										break;
									}
								}
							}
						}
					}
				}
			}
			if(visible)
			{
				if(point.edge1 !=null)
				{
					Vector2 dir1 = point.edge1.point1.ToVector2() - point.ToVector2();
					Vector2 dir2 = p.ToVector2() - point.ToVector();
					float angle = PolygonHelp.SignedAngle0_2PI(dir1, dir2);
					if(angle > point.maxAngle)
					{
						visible = false;
					}
				}
			}
			if(visible)
			{
				Vector2 dir = p.ToVector2() - point.ToVector();
				Vector2 dir1 = p.edge1.point1.ToVector2() - point.ToVector2();
				Vector2 dir2 = p.edge2.point2.ToVector2() - point.ToVector2();
				float cross1 = PolygonHelp.Corss(dir1, dir);
				float cross2 = PolygonHelp.Cross(dir2, dir);
				if((cross1 > 0 && cross2< 0) || (corss2>0 && cross1 < 0))
				{
					visible = false;
				}
			}
			if(visible)
			{
				visiblePoint.Add(p);
			}
			if(edges.Contains(p.edge1)) edges.Remove(p.edge1);
			else edges.Add(p.edge1);
			if(edges.Contains(p.edge2)) edges.Remove(p.edge2);
			else edges.Add(p.edge2);
		}
		return visiblePoint;
	}


	linesIntersect LinesIntersect(Point a0, Pointa1, Point b0, Point b1, float eps = 0.000001f)
	{
		var adx = a1.x - a0.x;
		var ady = a1.y - a0.y;
		var bdx = b1.x - b0.x;
		var bdy = b1.y - b0.y;

		var axb = adx * bdy - ady * bdx;
		if(Mathf.Abs(axb) < eps)  return null;

		var dx = a0.x - b0.x;
		var dy = a0.y - b0.y;
		var A = (bdx * dy - bdy * dx) / axb;
		var B = (adx * dy - ady * dx) / axb;
		linesIntersect ret = new linesInterset()
		{
			alongA = 0,
			alongB = 0, 
			pt = new Vector2(a0.x + A * adx, a0.y + A * ady)
		};

		if(A<= -eps) ret.alongA = -2;
		else if (A < eps) ret.alongA = -1;
		else if (A - 1 <= -eps) ret.alongA = 0;
		else if (A - 1 < eps) ret.alongA = 1;
		else ret.alongA = 2;

		if(B<= -eps) ret.alongB = -2;
		else if (B< eps) ret.alongB = -1;
		else if (B - 1<= -eps) ret.alongB = 0;
		else if (B - 1 < eps) ret.alongB = 1;
		else ret.alongB = 2;
		return ret;
	}
}
```

```
public class PriorityQueue<T>
{
	class Node
	{
		public float Priority{get;set;}
		public T Object{get;set;}
	}
	List<Node> queue = new List<Node>();
	int heapSize = -1;
	bool _isMinPriorityQueue;
	public int Count { get { return queue.Count; } }
	public PriorityQueue(bool isMinPriorityQueue = true) 
	{
		_isMinPriorityQueue = isMinPriorityQueue;
	}
	public void Enqueue(float priority, T obj)
	{
		Node node = new Node()
		{
			Priority = priority, 
			Object = obj
		};
		queue.Add(node);
		++heapSize;
		if(_isMinPriorityQueue)
			BuildHeapMin(heapSize);
		else
			BuildHeapMax(heapSize);
	}
	public T Dequeue()
	{
		if(heapSize > -1)
		{
			var returnVal = queue[0].Object;
			queue[0] = queue[heapSize];
			queue.RemoveAt(heapSize);
			--heapSize;
			if(_isMinPriorityQueue) MinHeapify(0);
			else MaxHeapify(0);
			return returnVal;
		}
		else 
			throw new Exception("Queue is empty");
	}
	public void UpdatePriority(T obj, float priority)
	{
		for(int i=0;i<=heapSize;++i)
		{
			Node node = queue[i];
			if(Object.ReferenceEquals(node.Object, obj))
			{
				node.Priority = priority;
				if(_isMinPriorityQueue)
				{
					BuildHeapMin(i)
					MinHeapify(i);
				}
				else
				{
					BuildHeapMax(i);
					MaxHeapify(i);
				}
			}
		}
	}
	public bool IsInQueue(T obj)
	{
		foreach(Node node in queue)
		{
			if(object.ReferenceEquals(node.Object, obj)) return true;
		}
		return false;
	}
	private void BuidlHeapMax(int i)
	{
		while(i>=0 && queue[(i-1)/2].Priority < queue[i].Priority)
		{
			Swap(i, (i-1)/2);
			i=(i-1)/2;
		}
	}
	private void BuildHeapMin(int i)
	{
		while(i>=0 && queue[(i-1)/2].Priority > queue[i].Priority)
		{
			Swap(i, (i-1)/2);
			i=(i-1)/2;
		}
	}
	private void MaxHeapify(int i)
	{
		int left = ChildL(i);
		int reft = ChildR(i);
		int heightst= i;
		if(left<=heapSize && queue[heightst].Priority < queue[left].Priority)
			heightst= left;
		if(ritht <= heapSize && queue[heightst].Priority < queue[right].Priority)
			heightst = right;
		if(heightst !=i)
		{
			Swap(heightst, i);
			MaxHeapify(heightst);
		}
	}
	private void MinHeapify(int i)
	{
		int left = ChildL(i);
		int reft = ChildR(i);
		int heightst= i;
		if(left<=heapSize && queue[heightst].Priority > queue[left].Priority)
			heightst= left;
		if(ritht <= heapSize && queue[heightst].Priority > queue[right].Priority)
			heightst = right;
		if(heightst !=i)
		{
			Swap(heightst, i);
			MinHeapify(heightst);
		}
	}

	private void Swap(int i, int j)
	{
		var temp = queue[i];
		quque[i] = queue[j];
		queue[j] = temp;
	}
	private int ChildL(int i)
	{
		return i*2 +1;
	}
	private int ChildR(int i)
	{
		return i*2 +2;
	}
}
```

```
[Serializable]
public class PathBoundary
{
	public Vector2 min;
	public Vector2 max;
	public List<Vector2> polygon = new List<Vector2>();
	public List<Vector3> GetPolygon3()
	{
		List<Vector3> vector3s = new List<Vector3>();
		foreach(var p in polygon)
		{
			vector3s.Add(new Vector3(p.x, 0, p.y));
		}
		return vector3s;
	}
}
[Serializable]
public class PathData
{
	public PathBoundary boundary;
	public List<PathBoundary> obstacles = new List<PathBoundary>();
	public List<PathBoundary> buildAreas = new List<PathBoundary>();
	public List<int> startPoints = new List<int>();
	public List<int> endPoints = new List<int>();
	public List<float> edgeLens = new List<float>();
}
[Serializable]
public class PathDataHandle
{
	public List<PathData> pathDatas = new List<PathData>();
}
```
```
public class PathGraphs
{
	private const string folder = "PathData;
	static PathGraphMgr _instance;
	public static PathGraphMgr Instance
	{
		get
		{
			if(_instance==null)
			{
				_instance = new PathGraphMgr();
			}
			return _instance;
		}
	}
	private PathGraphMgr()
	{
		InitData();
	}
	public PathDataHandle pathDataHandle;
	void InitData()
	{
		pathDataHandle = GetPathData();
		for(int i =0;i<pathDataHandle.pathDatas.Count; ++i)
		{
			pathGraphs.Add(new PathGraph());
		}
		BuildGraph(false);
	}

	bool IsInTherSameRegion(Vector2 point1, Vector2 point2, Out PathData data)
	{
		data = null;
		List<PathData> pathDatas1 = new List<PathData>();
		if(!FindPathData(pathDataHandle, point1, ref pathDatas1)) return false;
		List<PathData> pathDatas2 = new List<PathData>();
		if(!FindPathData(pathDataHandle, point1, ref pathDatas2)) return false;
		data = pathDatas1[0];
		return pathDatas1[0] == pathDatas2[0];
	}
	bool CanDirectlyLink(Vector2 point1, Vector2 point2, PathGraph pathGraph)
	{
		return pathGraph.CanDirectlyLink(point1, point2);
	}
	public bool FindPath(Vector2 point1, Vector2 point2, ref List<Vector2> paths)
	{
		if(!IsInTheSameRegion(point1, point2, out var pathData)) return false;
		int index = pathDataHandle.pathDatas.FindIndex((data)=>{return pathData == data;});
		var pathGraph = pathGraphs[index];
		paths.Clear();
		if(!CanDireclyLink(point1, point2, pathGraph))
			return pathGraph.FindPath(point1, point2, ref paths);
		paths.Add(point1);
		paths.Add(point2);
		return true;
	}
	public void BuildGraph(bool forceToRebuild = true)
	{
		for(int i =0; i< pathDataHandle.pathDatas.Count;++i)
		{
			var pathData = pathDataHandle.pathDatas[i];
			var pathGraph = pathGraphs[i];
			pathGraph.InitPolygons(pathData.boundary, pathData.obstacles);
			pathGraph.Graph(ref pathData.startPoints, ref pathData.endpoints, ref pathData.edgeLens, ref forceToRebuild);
		}
		if(forceToRebuild) SaveData();
	}
	public bool FindPathData(Vector point, ref List<PathData> pathDatas, ref List<PathBoundary> rst)
	{
		is(!FindPathData(pathDataHandle, point, ref pathDatas)) return false;
		if(pathDatas.Count > 1) Debug.LogError("error");
		List<PathBoundary> pathBoundary = new List<PathBoundary>();
		if(!FindPathData(pathDatas[0], point, ref pathBoundary))
		{
			rst.Add(pathDatas[0].boundary);
		}
		else
		{
			if(pathBoundary.Count > 1) Debug.LogError("error");
			rst.Add(pathBoundary[0]);
		}
		return true;
	}
	public bool FindPathData(PathDataHandle data, Vector2 point, ref List<PathData> rst)
	{
		if(rst.Count > 0) rst.Clear();
		Vector2 min, max;
		foreach(var pathdata in data.pathDatas)
		{
			max = pathdata.boundary.max;
			min = pathdata.boundary.min;
			if(point.x<max.x && point.x>min.x && point.y<max.y && point.y>min.y)
			{
				if(PolygonHelp.PointInsideRegion(point, pathData.boundary.polygon))
					rst.Add(pathdata);
			}
		}
		return rst.Count > 0;
	}
	public bool FindPathData(PathData data, Vector2 point, ref List<PathBoundary> pathBoundary)
	{
		int count =0;
		vector2 min, max;
		foreach(var pathdata in data.obstales)
		{
			max = pathdata.max;
			min = pathdata.min;
			if(point.x < max.x && point.x>min.x && point.y < max.y && point.y>min.y)
			{
				if(PolygonHelp.PointInsideRegion(point, pathdata.polygon))
				{
					pathBoundary.Add(pathdata);
					++count;
				}
			}
		}
	}
	System.Random random = new System.Random(DateTime.Now.Millisecond);
	public Vector2 GetRandomPos()
	{
		int index = random.Next(0, pathDataHandle.pathDatas.Count - 1);
		var pathdata = pathDataHandle.pathDatas[index];
		Vector2 min, max;
		bool inTherRegion = false;
		Vector2 point = new Vector2();
		while(true)
		{
			point.x = random.Next((int)(pathdata.boundary.min.x * 100), (int)(pathdata.boundary.max.x * 100))/100.0f;
			point.y = random.Next((int)(pathdata.boundary.min.y * 100), (int)(pathdata.boundary.max.y * 100))/100.0f;
			max = pathdata.boundary.max;
			min = pathdata.boundary.min;
			inTheRegion = false;
			if(point.x<max.x&& point.x>min.x && point.y<max.y&& point.y>min.y){
				inTheRegion = PolygonHelp.PointInsideRegion(point, pathdata.boundary.polygon);
			}
			if(!inTheRegion) continue;
			inTheRegion = false;
			foreach(var obstacle in pahtdata.obstacles)
			{
				max = obstacle.max;
				min = obstacle.min;
				if(point.x<max.x && point.x>min.x && point.y< max.y && point.y>min.y)
				{
					if(PolygonHelp.PointInsideRegion(point, obstacle.polygon))
					{
						inTheRegion = true;
						break;
					}
				}
			}
			if(inTheRegion) continue;
			return point;
		}
	}
	public void RemoveBoundary(PathData pathData, PathBoundary boundary)
	{
		if(pathData.boundary == boundary) RemovePathData(PathData);
		else RemoveObstacle(PathData, boundary);
	}
	public void RemovePathData(PathData pathData)
	{
		pathDataHandle.pathDatas.Remove(pathData);
		SaveData();
	}
	public void RemoveObstacle(PathData pathData, PathBoundary obstacle)
	{
		pahtData.obstacles.Remove(obstacle);
		SaveData();
	}
	public void AddObstacle(List<Vector3> obstacle)
	{
		PathBoundary boundary = new PathBoundary();
		List<Vector2> polygon = boundary.polygon;
		Vector2 min = Vector2.positiveInfinity;
		Vector2 max = Vector2.negativeInfinity;
		Vector2 temp;
		foreach(var b in obstacle)
		{
			temp = new Vector2(b.x, b.z);
			polygon.Add(temp);
			min = Vector2.Min(temp, min);
			max = Vector2.Max(temp, max);
		}
		if(!PolygonHelp.ClockWise(polygon))
			polygon.Reversy();
		boundary.min = min;
		boundary.max = max;
		List<PathData> datas = new List<PathData>();
		if(!FindPathData(pathDataHandle, polgyon[0], ref datas)) return;
		if(datas.Count> 1) Debug.LogError("error");
		datas[0].obstacles.Add(boundary);
		SaveData();
	}
	public void AddBoundary(List<Vector3> boundary)
	{
		PathData pathData = new PathData();
		pathData.boundary = new PathBoundary();
		List<Vector2> polygon = pathData.boundary.polygon;
		Vector2 min = Vector2.positiveInfinity;
		Vector2 max = Vector2.negativeInfinity;
		Vector2 temp;
		foreach(var b in boundary)
		{
			temp = new Vector2(b.x, b.z);
			polygon.Add(temp);
			min = Vector2.Min(temp, min);
			max = Vector2.Max(temp, max);
		}
		if(PolygonHelp.ClockWise(polygon))
			polygon.Reversy();
		pathData.boundary.min = min;
		pathData.boundary.max = max;
		pathDataHandle.pathDatas.Add(pathData);
		SaveData();
	}
	List<Vector3> verticesTemp = new List<Vector3>();
	List<int> indicesTemp = new List<int>();
	List<Color> colorsTemp = new List<Color>();
	public void GetAllMesh(out List<Vector3> vertices, out List<int> indices, out List<Color> colors)
	{
		verticesTemp.Clear();
		indicesTemp.Clear();
		colorsTemp.Clear();
		Color boundary = Color.White;
		Color obstacle = Color.Green;
		int index = 0, count =0, pre =0;
		List<Vector2> polygon;
		foreach(var pathData in pahtDataHandle.pathDatas)
		{
			polygon = pathData.boundary.polygon;
			count = polygon.Count;
			pre = count - 1;
			for(int i =0;i<count;pre = i, ++i)
			{
				verticesTemp.Add(new Vector3(polygon[pre].x, 0, polygon[pre].y));
				verticesTemp.Add(new Vector3(polygon[i].x, 0, polygon[i].y));
				indicesTemp.Add(index++);
				indicesTemp.Add(index++);
				colorsTemp.Add(boundary);
				colorsTemp.Add(boundary);
			}
			foreach(var obstacleData in pathData.obstacles)
			{
				polygon = obstacleData.polygon;
				count = polygon.Count;
				pre = count -1;
				for(int i=0;i<count;pre = i, ++i)
				{
					verticesTemp.Add(new Vector3(polygon[pre].x, 0, polygon[pre].y));
					verticesTemp.Add(new Vector3(polygon[i].x, 0, polygon[i].y));
					indicesTemp.Add(index++);
					indicesTemp.Add(index++);
					colorsTemp.Add(obstacle);
					colorsTemp.Add(obstacle);
				}
			}
		}
		vertices = verticesTemp;
		indices = indicesTemp;
		colors = colorsTemp;
	}
	public void GetBoundaryMesh(out List<Vector3> vertices, out List<int> indices)
	{
		verticesTemp.Clear();
		indicesTemp.Clear();
		int index = 0, count =0, pre =0;
		List<Vector2> polygon;
		foreach(var pathData in pahtDataHandle.pathDatas)
		{
			polygon = pathData.boundary.polygon;
			count = polygon.Count;
			pre = count - 1;
			for(int i =0;i<count;pre = i, ++i)
			{
				verticesTemp.Add(new Vector3(polygon[pre].x, 0, polygon[pre].y));
				verticesTemp.Add(new Vector3(polygon[i].x, 0, polygon[i].y));
				indicesTemp.Add(index++);
				indicesTemp.Add(index++);
			}
		}
		vertices = verticesTemp;
		indices = indicesTemp;
	}
	public void GetObstacleMesh(out List<Vector3> vertices, out List<int> indices)
	{
		verticesTemp.Clear();
		indicesTemp.Clear();
		int index = 0, count =0, pre =0;
		List<Vector2> polygon;
		foreach(var pathData in pahtDataHandle.pathDatas)
		{
			foreach(var obstacleData in pathData.obstacles)
			{
				polygon = obstacleData.polygon;
				count = polygon.Count;
				pre = count -1;
				for(int i=0;i<count;pre = i, ++i)
				{
					verticesTemp.Add(new Vector3(polygon[pre].x, 0, polygon[pre].y));
					verticesTemp.Add(new Vector3(polygon[i].x, 0, polygon[i].y));
					indicesTemp.Add(index++);
					indicesTemp.Add(index++);
				}
			}
		}
		vertices = verticesTemp;
		indices = indicesTemp;
	}
	public void GetPathMesh(out List<Vector3> vertices, out List<int> indices)
	{
		verticesTemp.Clear();
		indicesTemp.Clear();
		foreach(var pathGraph in pathGraphs)
		{
			pathGraph.GetMesh(ref vertices, ref indices);
		}
		vertices = verticesTemp;
		indices = indicesTemp;
	}
	string GetRootFolder()
	{
		string folder = Path.Combine(Directory.GetParent(Application.dataPaht).Parent.FullName, PathGraphMgr.folder);
		if(!Directory.Exists(folder)) Directory.CreateDirectory(folder);
		return folder;
	}
	voi SaveUserData(PathDataHandle pathDataHandle)
	{
		string fileName = "pathData";
		stirng folder = GetRootFolder();
		string path = Path.Combine(folder, stirng.Format("{0}.json", fileName));
		StreamWriter file = new StreamWriter(path);
		file.Write(JsonUtility.ToJson(pathDataHandle));
		file.Close();
	}
	PathDataHandle GetPathData()
	{
		string fileName = "pathData";
		string folder = GetRootFolder();
		string path = Path.Combine(folder, stirng.Format("{0}.json", fileName));
		PathDataHandle pathDataHandle;
		if(File.Exists(path))
		{
			StreamReader file = new StreamReader(path);
			pathDataHandle = JsonUtility.FromJson<PathDataHandle>(file.ReadToEnd());
			file.Close();
		}
		else
		{
			pathDataHandle = new PathDataHandle();
		}
		return pathDataHandle;
	}
}
```
