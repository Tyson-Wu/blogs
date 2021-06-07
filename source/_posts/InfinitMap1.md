---
title: Mesh
date: 2021-06-05 16:01:00
categories:
- [Gemotry, Mesh]
tags:
- Gemotry
- Mesh
---





























## 
```
public class Map : IDity
{
	WorldMapDataHandle worldMapData;
	MeshMap mapGrid;
	MapPathGenerator pathGenerator;
	bool dirty = false;
	public void SetDirty()
	{
		dirty = true;
	}
	public void ClearData()
	{
		worldMapData.worldData.Clear();
		SetDirty();
	}
	public event Action<Transform, Vector3, Vector3> UpdateBaseMapLayer;
	public event Action<Rect, float, int> UpdateGridMeshLayer;
	public event Action<List<Vector3>, List<int>> UpdateGridLayer;
	public event Action<Transform, Dictionary<int, List<TileData>>> UpdateOverlayDataLayer;
	Rect edge = new Rect(float.NegativeInfinity, float.NegativeInfinity, float.PositiveInfinity, float.PositiveInfinity);
	Camera camera;
	float zoomSpeed = 400;
	public Transform rootTransform {get; private set;}
	Transform baseMapLayerRoot = null;
	Transform overlayMapLayerRoot = null;
	float gridAngle = 45;
	float cameraHeight = 30;
	float cameraPitchAngle = 70;
	Vector3[] viewPortCorner;
	public void SetMapEdge(Rect edge)
	{
		this.edge = edge;
	}
	public void SetZoomSpeed(float zoomSpeed)
	{
		this.zoomSpeed = zoomSpeed;
	}
	public void SetRootTransform(Transform rootTransform)
	{
		this.rootTransform = rootTransform;
	}
	public void SetGridAngle(float angle)
	{
		gridAngle = Mathf.Clamp(angle, 0, 45);
	}
	public Void SetCameraHeight(float height)
	{
		cameraHeight = height;
	}
	public void SetCameraPitchAngle(float angle)
	{
		cameraPitchAngle = Mathf.Clamp(angle, 60, 90);
	}
	void InitRootTransform()
	{
		if(rootTransform == null)
			rootTransform = new GameObject("mapRoot").transform;
		baseMapLayerRoot = new GameObject("baseMapLayer").transform;
		overlayMapLayerRoot = new GameObject("overlayMapLayer").transform;
		baseMapLayerRoot.SetParent(rootTransform);
		overlayMapLayerRoot.SetParent(rootTransform);
		rootTransform.rotation = Quaternion.Euler(0, gridAngle, 0);
		baseMapLayerRoot.localPosition = Vector3.zero;
		baseMapLayerRoot.localRotation = Quaternion.Euler(0, -gridAngle, 0);
		baseMapLayerRoot.localScale = Vector3.one;
		overlayMapLayerRoot.localPosition = Vector3.zero;
		overlayMapLayerRoot.localRotation = Quaternion.Euler(0,0,0);
		OverlayMapLayerRoot.localScale = Vector3.one;
	}
	void InitCamera(Camera camera)
	{
		this.camera = camera;
		camera.transform.position = new Vector3(0,cameraHeight, 0);
		camera.transform.rotation = Quaterion.Euler(cameraPitchAngle, 0,0);
		viewPortCorner = CalcualteViewAreaInZeroPanel();
	}
	void InitGridMesh()
	{
		mapGrid = new MeshMap(edge);
		mapGrid.SetMapRotationAngle(gridAngle);
		mapGrid.SetViewAreaInZeroPanel(viewPoreCorner);
	}
	public void Init(Camera camera)
	{
		InitRootTransform();
		InitCamera(camera);
		InitGridMesh();
		worldMapData = new WorldMapDataHandel(this);
		pathGenerator = new MapPathGenerator();
	}
	Dictionary<int, List<TileData>> tileDataTempList = new Dictionary<int, List<TileData>>();
	public bool FindTile(Vector3 mapPos, out Dictionary<int, List<TileData>> tileDatas, uint layerMask)
	{
		var rst = worldMapData.SearchTile(mapPos, ref tileDataTempList, layerMask);
		tileDatas = tileDataTempList;
		return rst;
	}
	public bool FindTile(Quad region, out Dictionary<int, List<TileData>> tileDatas, uint layerMask)
	{
		var rst = worldMapData.SearchTile(region, ref tileDataTempList, layerMask);
		tileDatas = tileDataTempList;
		return rst;
	}
	public void AddTile(TileData tile)
	{
		worldMapData.AddTile(tile);
	}
	public RemoveTile(TileData tile)
	{
		worldMapData.RemoveTile(tile);
	}
	public bool IncludedInMapArea(vector3 mapPos)
	{
		return mapGrid.IncludedInMapArea(mapPos);
	}
	public void MoveTile(TileData tile, Vector2 endPos, float speed)
	{
		if(pathGenerator.TryGetPath(tile.position, endPos, out var posList))
		{
			worldMapData.MoveTile(tile, posList, speed);
		}
	}
	Vector3 selectedMapPos = Vector3.zero;
	Vector3 selectedUnityPos = Vector3.zero;
	float seledValue = 1;
	public Dictionary<int, List<TileData>> visibleTileData = new Dictionary<int, List<TileData>>();
	public void UpdateMapScale(float scale)
	{
		this.scaleValue = scale;
		SetDirty();
	}
	public void UpdateZoomScale(float detal)
	{
		this.scaledValue = (detal * zoomSpeed + 1) * scaledValue;
		SetDirty();
	}
	public void UpdateScreenMapCombinedPos(Vector2 screenPos, Vector3 mapPos)
	{
		selectedMapPos = mapPos;
		selectedUnityPos = screenPos;
		selectedUnityPos = ConvertScreen2UnityPos(screenPos);
		SetDirty();
	}
	public void UpdateMap()
	{
		if(!dirty) return;
		dirty = false;
		scaledValue = mapGrid.SetScale(scaledValue);
		mapGrid.BindLocalPosToWorld(selectedMapPos, selectedUnityPos);
		mapGrid.CalculateVisibleGrid();
		rootTransform.position = mapGrid.GetOrigionPos();
		rootTransform.rotation = mapGrid.Rotation;
		rootTransform.localScale = mapGrid.Scale;
		UpdateBaseMapLayer?.Invoke(baseMapLayerRoot, mapGrid._effectVisibleReginInBaseMapMin, mapGrid._effectVisibleReginInBaseMapMax);
		Rect rect = new Rect();
		if(mapGrid.Level == 0)
			rect = mapGrid._visibleGridRegion;
		UPdateGridMeshLayer?.Invoke(rect, mapGrid.UnitLen, MeshMap._tileGroupNum);
		UpdateGridLayer?.Invoke(mapGrid._visibleMeshOutlineVertexPosList, mapGrid._visibleMeshOutlineVertexIndices);

		worldMapData.SearchTile(mapGrid._visibleMeshSubQuad, ref visibleTileData);
		UpdateOverlayDataLayer?.Invoke(overlayMapLayerRoot, visibleTileData);
	}
	public Vector3 ConvertScreen2MapPos(vector3 screenPos)
	{
		return mapGrid.CvtWorldToLocalPos(ConvertScreen2UnityPos(screenPos));
	}
	public Vector3 ConvertMap2ScreenPos(Vector3 mapPos)
	{
		return ConvertUnity2ScreenPos(mapGrid.CvtLocalToWorldPos(mapPos));
	}
	public Vector3 ConvertScreen2UnityPos(Vector3 screen)
	{
		Ray ray = camera.ScreenPointToRay(screen);
		raturn RayCastUnityPosInZeroPanel(ray.origion, ray.direction);
	}
	public Vector3 ConvertUnity2ScreenPos(Vecotr3 unityPos)
	{
		return camera.WorldToScreenPos(unityPos);
	}
	public Vector3[] CalcualteViewAraInZeroPanle()
	{
		Vector3[] viewPortCorner = new Vector3[4];
		Vector3[] camraeNearPanelCorner = new Vector3[4];
		Vector3[] cameraFarPanelCorner = new Vector3[4];
		if(camera.othographic)
		{
			float halfWidth = aspect * camera.orthographicSize;
			cameraNearPanelCorner[0] = camera.transform.TransformPoint(new Vector3(-halfWidth, camera.orthographicSize, camera.nearClipPlane));
			cameraNearPanelCorner[1] = camera.transform.TransformPoint(new Vector3(halfWidth, camera.orthographicSize, camera.nearClipPlane));
			cameraNearPanleCorner[2] = camera.transform.TransformPoint(new Vector3(halfWidth, -camera.othergraphicSize, camera.nearClipPlane));
			cameraNearPanelCorner[3] = camera.transform.TransformPoint(new Vector3(-halfWidth, -camera.orthergraphicSize, camera.nearClipPlane));

			cameraFarPanelCorner[0] = camera.transform.TransformPoint(new Vector3(-halfWidth, camera.orthographicSize, camera.farClipPlane));
			cameraFarPanelCorner[1] = camera.transform.TransformPoint(new Vector3(halfWidth, camera.orthographicSize, camera.farClipPlane));
			cameraFarPanelCorner[2] = camera.transform.TransformPoint(new Vector3(halfWidth, -camera.othergraphicSize, camera.farClipPlane));
			cameraFarPanelCorner[3] = camera.transform.TransformPoint(new Vector3(-halfWidth, -camera.orthergraphicSize, camera.farClipPlane));
		}
		else
		{
			float halfNearHeight = camera.nearClipPlane * Mathf.Tan(Mathf.Deg2Rad * camera.fieldOfView * 0.5f);
			float halfNearWidth = halfNearHeight * aspect;
			float halfFarHeight =  camera.farClipPlane * Mathf.Tan(Mathf.Deg2Rad * camera.fieldOfView * 0.5f);
			flaot halfFarWidth = halfFarHeight * aspect;

			cameraNearPanelCorner[0] = camera.transform.TransformPoint(new Vector3(-halfNearWidth, halfNearHeight, camera.nearClipPlane));
			cameraNearPanelCorner[1] = camera.transform.TransformPoint(new Vector3(halfNearWidth, halfNearHeight, camera.nearClipPlane));
			cameraNearPanleCorner[2] = camera.transform.TransformPoint(new Vector3(halfNearWidth, -halfNearHeight, camera.nearClipPlane));
			cameraNearPanelCorner[3] = camera.transform.TransformPoint(new Vector3(-halfNearWidth, -halfNearHeight, camera.nearClipPlane));

			cameraFarPanelCorner[0] = camera.transform.TransformPoint(new Vector3(-halfFarWidth, halfFarHeight, camera.farClipPlane));
			cameraFarPanelCorner[1] = camera.transform.TransformPoint(new Vector3(halfFarWidth, halfFarHeight, camera.farClipPlane));
			cameraFarPanelCorner[2] = camera.transform.TransformPoint(new Vector3(halfFarWidth, -halfFarHeight, camera.farClipPlane));
			cameraFarPanelCorner[3] = camera.transform.TransformPoint(new Vector3(-halfFarWidth, -halfFarHeight, camera.farClipPlane));
		}
		for(int i =0;i<4; ++i)
		{
			viewPortCorner[i] = RayCastUnityPosInZeroPanel(cameraFarPlaneCorner[i], cameraNearPlaneCorner[i]  - cameraFarPlaneCorner[i]);
		}
		return viewPortCorner;
	}
	public Vector3 RayCastUnityPosInZeroPanel(Vector3 origin, Vector3 dir)
	{
		float t = -origin.y / dir.y;
		Vector3 centerPos = origin + t * dir;
		centerPos.y = 0;
		return centerPos;
	}
}
```

```
public class MapPathGenerator
{
	public bool TryGetPath(Vector2 startPos, Vector2 endPos, out List<Vector2> pathPoint)
	{
		pathPoint = new List<Vector2>();
		return PathGraphMgr.Instance.FindPath(startPos, endPos, ref pathPoint);
	}
}
```
```
public class WorldMapDataHandle
{
	public QuadTree<TileData> worldMap;
	Quad region;
	IDirty world;
	Dictionary<int, List<TileData>> tileCache = new Dictionary<int, List<TileData>>();
	public WorldMapDataHandle(IDirty world)
	{
		this.world = world;
		int halfSize = 1<< 20;
		region.MinX = -halfSize;
		region.Miny = -halfSize;
		reigon.MaxX = halfSize;
		region.MaxY = halfSize;
		worldData = new QuadTree<TileData>(ref region);
		WorldMapServer.Instance._heartbeat = HeartBeat;
	}
	private void HeartBeat(List<TileAddMessage> newAdd, List<TileDeleteMessage> delete)
	{
		TileData Tile;
		Quad quad;
		for(int i=0; i<delete.Count; ++i)
		{
			tile = delete[i].tile;
			worldData.Remove(tile);
		}
		for(int i = 0;i<newAdd.Count; ++i)
		{
			tile = new[i].tile;
			quad = new Quad(tile.min, tile.max);
			worldData.Insert(tile, ref quad);
		}
		if((newAdd.Count | delete.Count)>0)
		{
			world.SetDirty();
		}
	}
	public void AddTile(TileData tile)
	{
		TileAddMessage message = new TileAddMessage()
		{
			tile = tile
		};
		WorldMapServer.Instance.Add(message);
	}
	public void MoveTile(TileData tile, List<Vector2> posList, float speed)
	{
		TileMoveMessage message = new TileMoveMessage()
		{
			tile = tile,
			speed = speed,
			progress = 0,
		};
		messge.posList.AddRange(posList);
		WorldMapServer.Instance.Move(message);
	}
	public void RemoveTile(TileData tile)
	{
		TileDeleteMessage message = new TileDeleteMessage()
		{
			tile = tile
		};
		WorldMapServer.Instance.Delete(message);
	}
	Quad? newRequestBranch = null;
	void RequestNewBranch()
	{
		if(newRequestBranch == null) return;
		TileChunkData tileChunk = null;
		WorldMapServer.Instance.GetTileChunkData(newRequestBranch.Value, ref tileChunk);
		newRequestBranch = null;
		worldData.InsertBranch(tileChunk.tiles, ref tileChunk.quad);
		world.SetDirty();
	}
	public bool SearchTile(List<Quad> region, ref Dictionary<int, List<TileData>> data, uint layerMask = ~0u)
	{
		CollectionsEx.Clear(ref data);
		for(int i=0;i<region.Count; ++i)
		{
			bool rst = worldData.SearchArea(region[i], ref tileCache, 0, ref new RequestBranch, layerMask);
			foreach(var tiles in tileCache)
			{
				if(tiles.Value.Count > 0)
				{
					if(!data.TryGetValue(tile.Key, out var tileDatas))
					{
						tileDatas = new List<TileData>();
						data.Add(tiles.Key, tiles.Value);
					}
					foreach(var tile in tiles.Value)
					{
						int index = tileDatas.FindIndex((a)=> a.id == tile.id);
						if(index < 0) tileDatas.Add(tile);
					}
				}
			}
		}
		RequestNewBranch();
		return true;
	}
	public bool SearchTile(Quad region, ref Dictionary<int, List<TileData>> data, uint layerMask = ~0u)
	{
		bool rst = worldData.SearchArea(region, ref data, 0, ref newRequestBranch, layerMask);
		RequestNewBranch();
		return rst;
	}
	public bool SearchTile(Vector3 mapPos, ref Dictionary<int, List<TileData>> data, uint layerMask = ~0U)
	{
		return worldData.SearchPoint(mapPos.x, mapPos.z, ref data, layerMask);
	}
	public bool IncludedInMapArea(Quad region)
	{
		return this.regionContains(ref region);
	}
}
```
```
public class QuadTree<T> where T : ILeafValue
{
	internal static Stack<Branch> branchPool = new Stack<Branch>();
	internal static Stack<Leaf> leafPool = new Stack<Leaf>();

	protected Branch root;
	protected Quad region;
	internal Dictionary<int, Dictionary<int, Leaf>> leafLookup = new Dictionary<int, Dictionary<int, Leaf>>();
	static int leafCount = 0;
	public QuadTree(ref Quad region)
	{
		this.region  = region;
	}
	public QuadTree(Quad region):this(ref region)
	{

	}
	public void Clear()
	{
		root.Clear();
		root.Tree = this;
		foreach(var leafs in leafLookup)
		{
			leafs.Value.Clear();
		}
		leafLookup.Clear();
		leafCount = 0;
	}
	public static void ClearPools()
	{
		branchPool = new Stack<Branch>();
		leafPool = new Stack<Leaf>();
	}
	public void Remove(T value)
	{
		if(TryGetLeafsvalue.Layer, out var leaves)
		{
			if(leaves.TryGetValue(value.ID, out var leaf))
			{
				leaves.Remove(value.ID);
				Branch branch = leaf.Branch;
				RecycleLeaf(leaf);
				leaf.Branch = null;
				leaf.Value = default(T);
				return;
			}
		}
	}
	void GetLeafs(int layer, out Dictionary<int, Leaf> leafs)
	{
		if(!leafLookup.TryGetValue(layer, out leafs))
		{
			leafs = new Dictionary<int, Leaf>();
			leafLookup.Add(layer, leafs);
		}
	}
	bool TryGetLeafs(int layer, out Dictionary<int, Leaf> leafs)
	{
		return leafLookup.TryGetValue(layer, out leafs);
	}
	public void Insert(T value, ref Quad quad)
	{
		if(root == null) return;
		GetLeafs(value.Layer, out var leafs);
		if(!leafs.TryGetValue(value.ID, out var leaf))
		{
			leaf = CreateLeaf(value, ref quad);
			leafs.Add(value.ID, leaf);
		}
		else
		{
			leaf.Branch.Remove(leaf);
			leaf.Branch = null;
			leaf.Value = value;
			leaf.Quad = quad;
		}
		if(root.Insert(leaf)==null)
		{
			leafs.Remove(value.ID);
			RecycleLeaf(leaf);
			leaf.Branch = null;
			leaf.Value = default(T);
		}
	}
	public void Insert(T value, Quad quad)
	{
		Insert(value, ref quad);
	}
	public void InsertBranch(List<T> tiles, ref Quad quad)
	{
		Branch branch = CreateBranch(this, null, quad.MaxX - quad.Minx, ref quad);
		Leaf leaf;
		T tile;
		Quad rect;
		Directionary<int, Leaf> leafs;
		for(int i =0;i<tiles.Count;++i)
		{
			tile = tiles[i];
			rect = new Quad(tile.Min, tile.Max);
			leaf = CreateLeaf(tile, ref rect);
			leaf.Branch = branch;
			GetLeafs(tile.Layer, out leafs);
			leafs.Add(tile.ID, leaf);
			branch.AddLeaf(leaf);
		}
		if(root == null)
		{
			root = branch;
			return;
		}
		root.InsertBranch(branch);
	}
	public bool SearchArea(ref Quad quad, ref Dictionary<int,List<T>> valueLayers, float unitLen, ref Quad? new RequestBranch, uint layerMask = ~0u)
	{
		CollectionsEx.Clear(ref valueLayers);
		if(root ==null)
		{
			newRequestBranch = region;
			return false;
		}
		int count = 0;
		root.SearchQuad(ref quad, ref valueLayers, ref count, unitLen, ref newRequestBranch, layerMask);
		return count > 0;
	}
	public bool SearchArea(Quad, ref Dictionary<int, List<T>>valueLayers, float unitLen, ref Quad? newRequestBranch, uint layerMask = ~0u)
	{
		return SearchArea(ref quad, ref valueLayers, unitLen, ref newRequestBranch, layerMask);
	}
	public bool SearchPoint(float x, float y, ref Dictionary<int, List<T>> valueLayers, uint layerMask = ~0u)
	{
		CollectionsEx.Clear(ref valueLayers);
		if(root == null)
		{
			return false;
		}
		int count =0;
		root.SearchPoint(x, y, ref valueLayers, ref count, layerMask);
		return count > 0;
	}
	public bool FindCollision(T value, ref Dictionary<int, List<T>> valueLayers, ref Quad? newRequestBranch, uint layerMask = ~0u)
	{
		CollectionsEx.Clear(ref valueLayers);
		Leaf leaf ;
		if(!TryGetLeafs(value.Layer, out var leafs)) return false;
		if(!leafs.TryGetValue(value.ID, out leaf)) return false;
		int count = 0;
		var branch = leaf.Branch;
		foreach(var leaves in branch.Leaves)
		{
			if(!CheckLayer(leaves.Key, leyerMask)) continue;
			for(int i=0;i<leaves.Value.Count; ++i)
			{
				if(leaf != leaves.Value[i] && leaf.Quad.Intersects(ref leaves.Value[i].Quad))
				{
					CollectionsEx.Add(ref valueLayers, leaves.Value[i].Value.layer, leaves.Value[i].Value);
					++count;
				}
			}
		}
		if(branch.Split)
		{
			for(int i = 0;i<4; ++i)
			{
				if(branch.Branches[i]!=null)
				{
					branch.Branches.SearchQuad(ref reaf.Quad, ref vaueLayers, ref count, int.MaxValue, ref newRequestBranch, layerMask);
				}
			}
		}
		branch = branch.Parent;
		while(branch!=null)
		{
			foreach(var leaves in branch.Leaves)
			{
				if(!CheckLayer(leaves.Key, layerMask)) continue;
				for(int i=0;i<leaves.Value.Count;++i)
				{
					if(leaf.Quad.Intersects(ref leaves.Value[i].Quad))
					{
						CollectionsEx.Add(ref valueLayers, leaves.Value[i].Value.Layer,leaves.Value[i].Value);
						++count;
					}
				}
			}
			branch = branch.Parent;
		}
		return count > 0;
	}
	public int CountBranches()
	{
		int count = 0;
		CountBranches(root, ref count);
		return count;
	}
	void CountBranches(Branch branch, ref int count)
	{
		++count;
		if(branch.Split)
		{
			for(int i=0;i<4;++i)
			{
				if(branch.Branches[i]!=null)
				{
					CountBranches(branch.Branches[i], ref count);
				}
			}
		}
	}
	static Branch CreateBranch(QuadTree<T> tree, Branch parent, float branchLen, ref Quad quad)
	{
		var branch = branchPool.Count > 0? branchPool.Pop() : new Branch();
		branch.Tree = tree;
		branch.Parent = parent;
		branch.UnitLen = branchLen;
		float midX = quad.MinX + (quad.MaxX - quad.MinX) * 0.5f;
		float midY = quad.MinY + (quad.MaxY - quad.MinY) * 0.5f;
		branch.Quads[0].Set(quad.MinX, quad.MinY, midX, minY);
		branch.Quads[1].Set(minX, quad.MinY, quad.MaxX, midY);
		branch.Quads[2].Set(midX, midY, quad.MaxX, quad.MaxY);
		branch.Quads[3].Set(quad.MinX, midY, midX, quad.MaxY);
		return branch;
	}
	static Leaf CreateLeaf(T value, ref Quad quad)
	{
		var leaf = leafPool.Count > 0? leafPool.Pop() : new Leaf();
		leaf.Value = value;
		leaf.Quad = quad;
		++leafCount;
		return leaf;
	}
	static void RecycleLeaf(Leaf leaf)
	{
		leafPool.Push(leaf);
		--leafCount;
	}
	static bool CheckLayer(int layer, uint layerMask)
	{
		return ((1<<layer)& layerMask)>0;
	}
	protected internal class Branch
	{
		internal QuadTree<T> Tree;
		internal Branch Parent;
		internal Quad[] Quads = new Quad[4];
		internal Branch[] Branches = new Branch[4];
		internal Dictionary<int, List<Leaf>> leaves = new Dictionary<int, List<Leaf>>();
		internal bool Split = false;
		internal float UnitLen;
		public void AddLeaf(Leaf leaf)
		{
			List<Leaf> leaves;
			if(!Leaves.TryGetValue(leaf.Value.Layer, out leaves))
			{
				leaves = new List<Leaf>();
				Leaves.Add(leaf.Value.Layer, leaves);
			}
			leaves.Add(leaf);
		}
		public bool TryGetLeaves(int layer, out List<Leaf> leaves)
		{
			return Leaves.TryGetValue(layer, out leaves);
		}
		internal void Clear()
		{
			Tree = null;
			Parent = null;
			Split = false;
			for(int i=0;i<4;++i)
			{
				if(Branches[i]!=null)
				{
					branchPool.Push(Branches[i]);
					Branches[i].Clear();
					Branches[i] = null;
				}
			}
			foreach(var leaves in Leaves)
			{
				for(int i=0;i<leaves.Value.Count;++i)
				{
					RecycleLeaf(leaves.Value[i]);
					leaves.Value[i].Branch = null;
					leaves.Value[i].Value = default(T);
				}
			}
			CollectionsEx.Clear(ref Leaves);
		}
		internal void Remove(Leaf leaf)
		{
			if(TryGetLeaves(leaf.Value.Layer, out var leaves))
			{
				int index = leaves.FindIndex((x)=>{return x.Value.ID == leaf.Vallue.ID;});
				if(index < 0>) return;
				leaves[index] = leaves[leaves.Count - 1];
				leaves.RemoveAt(leaves.Count - 1);
			}
		}
		internal Branch Insert(Leaf leaf)
		{
			if(leaf.Value.UnitLen<UnitLen)
			{
				for(int i=0;i<4; ++i)
				{
					if(Quads[i].Contains(ref leaf.Quad))
					{
						if(Branches[i] == null)
						{
							return null;
						}
						return Branches[i].Insert(leaf);
					}
				}
				AddLeaf(leaf);
				leaf.Branch = this;
				return this;
			}
			else
			{
				AddLeaf(leaf);
				leaf.Branch = this;
				return this;
			}
		}
		internal Branch InsertBranch(Branch branch)
		{
			for(int i=0;i<4;++i)
			{
				if(Quads[i].Contains(ref branch.Quads[0]))
				{
					if(Quads[i].MaxX - Quads[i].MinX == branch.Quads[2].MaxX - branch.Quads[0].Minx){
						Branches[i] = branch;
						branch.Parent = this;
						branch.UnitLen = UnitLen / 2;
						Split = true;
						return this;
					}				
					else
					{
						return Branches[i].InsertBranch(branch);
					}
				}
			}
			return this;
		}
		internal void SearchQuad(ref Quad quad, ref Dictionary<int, List<T>> valueLayers, ref int count, float unitLen, ref Quad? newRequestBranch, uint layerMask)
		{
			if(UnitLen< unitLen) return ;
			foreach(var leaves in Leaves)
			{
				if(!CheckLayer(leaves.Key, layerMask)) continue;
				for(int i =0;i<leaves.Value.Count;++i)
				{
					var leaf = leaves.Value[i];
					if(leaf.Value.UnitLen >= unitLen && quad.Intersects(ref leaf.Quad))
					{
						CollectionsEx.Add(ref valueLayers, leaf.Value.Laer, leaf.Value);
						++count;
					}
				}
			}
			if(newRequestBranch ==null)
			{
				for(int i=0;i<4;++i)
				{
					if(Branches[i] == null && Quads[i].Contains(ref quad))
					{
						newRequestBranch = Quads[i];
						break;
					}
				}
			}
			for(int i=0;i<4;++i)
			{
				if(Branches[i]!=null && Quads[i].Contains(ref quad))
				{
					Branches[i].SearchQuad(ref quad, ref valueLayers, ref count, unitLen, ref newReuqestBranch, layerMask);
				}
			}
		}
		internal void SearchPoint(float x, float y, ref Dictionary<int, List<T>> valueLayers, ref int count, uint layerMask)
		{
			foreach(var leaves in Leaves)
			{
				if(!CheckLayer(leaves.Key, layerMask)) continue;
				for(int i=0;i<leaves.Value.Count;++i)
				{
					var leaf = leaves.Value[i];
					if(leaf.Quad.Contains(x,y))
					{
						CollectionsEx.Add(ref valueLayers, leaf.Value.Layer, leaf.Value);
						++count;
					}
				}
			}
			for(int i=0;i<4;++i)
			{
				if(Branches[i]!=null && Quads[i].Contains(x,y))
				{
					Branches[i].SearchPoint(x,y, ref valueLayers, ref count, layerMask);
				}
			}
		}
		internal void SearchPoint(float x, float y, float unitLen, ref Dictionary<int, List<T>> valueLayers, ref int count, uint layerMask)
		{
			if(UnitLen < unitLen) return;
			foreach(var leaves in Leaves)
			{
				if(!CheckLayer(leaves.Key, layerMask)) continue;
				for(int i=0;i<leaves.Value.Count;++i)
				{
					var leaf = leaves.Value[i];
					if(leaf.Quad.Contains(x,y))
					{
						CollectionsEx.Add(ref valueLayers, leaf.Value.Layer, leaf.Value);
					}
				}
			}
			for(int i=0;i<4;++i)
			{
				if(Branches[i]!=null && Quads[i].Contains(x, y))
				{
					Branches[i].SearchPoint(x,y, ref ValueLayers, ref count, layerMask);
				}
			}
		}
	}
	protected internal class Leaf
	{
		internal Branch Branch;
		internal T Value;
		interanl Quad Quad;
	}
}
```
```
[Serializable]
public struct Quad
{
	public const int unitLenLimit = 1;
	public float MinX;
	public float MinY;
	public float MaxX;
	public float MaxY;
	public Quad(float minx, float miny, float maxx, float maxy)
	{
		MinX = minx;
		MinY = miny;
		MaxX = maxx;
		MaxY = maxy;
	}
	public Quad(Vector2 min, Vector2 max)
	{
		MinX = min.x;
		MinY = min.y;
		MaxX = max.x;
		MaxY = max.y;
	}
	public void Set(float minx , float miny, float maxx, float maxy)
	{
		MinX = minx;
		MinY = miny;
		MaxX = maxx;
		MaxY = maxy;
	}
	public bool Intersects(ref Quad other)
	{
		return MinX < other.MaxX && MinY < other.MaxY && MaxX > other.MinX && MaxY > other.MinY;
	}
	public bool Contains(ref Quad other)
	{
		return other.MinX >= MinX && other.MinY >= MinY && other.MaxX <= MaxX && other.MaxY <= MaxY;
	}
	public bool Contains(float x, float y)
	{
		return x > MinX && y>MinY && x< MaxX && y< MaxY;
	}
	public void Include(ref Quad other)
	{
		MinX = MinX > other.MinX ? other.MinX : MinX;
		MinY = MinY > other.MinY ? other.MinY : MinY;
		MaxX = MaxX < other.MaxX ? other.MaxX : MaxX;
		MaxY = MaxY < other.MaxY ? other.MaxY : MaxY;
	}
	publicStatic Quad CreatePowerOfTowQuad(float width, float height)
	{
		int value = 1;
		while(value >= width && value >= height)
		{
			value = value << 1;
		}
		Quad rect = new Quad(0.0, value, value);
		return rect;
	}
	internal static bool SplitQuad(ref Quad quad, ref Quad[] temp)
	{
		float midX = quad.MinX + (quad.MaxX - quad.MinX) * 0.5f;
		if(unitLenLimit > midX - quad.Minx) return false;
		float midY = quad.MinY + (quad.MaxY - quad.MinY) * 0.5f;
		temp[0].Set(quad.MinX, quad.MinY, midX, midY);
		temp[1].Set(midX, quad.MinY, quad.MaxX, midY);
		temp[2].Set(midX, midY, quad.MaxX, quad.MaxY);
		temp[3].Set(quad.Miny, midY, midX, quad.MaxY);
		return true;
	}
	public override string ToString()
	{
		return string.Format("{0}, {1}, {2}, {3}", MinX, MinY, MaxX, MaxY);
	}
}

```
```
public interface ILeafValue
{
	float UnitLen{get;set;}
	Vector2 Min{get;set;}
	Vector2 Max{get;set;}
	int ID{get;set;}
	int Layer{get;set;}
}
```
```
[Serializable]
public class TileData:ILeafValue
{
	public int id;
	public string typy;
	public string name;
	public Vector2 position;
	public Vector2 min;
	public Vector2 max;
	public float priority;
	public int layer;
	public int userID;
	public List<Vector2> posList = new List<Vector2>();
	public float UnitLen {
		get { return priority;}
		set { priority = value;}
	}
	public Vector2 Min{
		get { return min;}
		set { min = value;}
	}
	public Value2 Max{
		get{ return max;}
		set {max = value;}
	}
	public int ID{
		get{return id;}
		set{id = value;}
	}
	public int Layer{
		get{return layer;}
		set{layer = value;}
	}
	public TileData Clone()
	{
		TileData tile = TileData()
		{
			id = id,
			type = type,
			name = name,
			position = position,
			min = min,
			max = max, 
			priority = priority,
			layer = layer,
			userID = userID,
			posList = posList
		};
		return tile;
	}
	public TileData Clone(Vector newPos)
	{
		TileData tile = TileData()
		{
			id = id,
			type = type,
			name = name,
			position = newPos,
			min = min - position + newPos,
			max = max - position + newPos, 
			priority = priority,
			layer = layer,
			userID = userID,
			posList = posList
		};
		return tile;
	}
	public override stirng ToString()
	{
		return id.ToString();
	}
	public override int GetHashCode()
	{
		return (int)id;
	}
}
```