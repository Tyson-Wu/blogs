---
title: InfinitMap
date: 2021-06-05 16:01:00
categories:
- [Gemotry, Mesh]
tags:
- Gemotry
- Mesh
---

## 
```
public struct MeshMapUnitCoord : IEquatable<MeshMapUnitCoord>
{
	public int x;
	public int y;
	public int level;
	public MeshMapUnitCoord(int x = 0, int y =0, int level =0)
	{
		this.x = x;
		this.y = y;
		this.level = leve;
	}
	public override string ToString()
	{
		return string.Format("({0}, {1}, {2})", x, y,level);
	}
	override public bool Equals(object obj)
	{
		return obj is MeshMapUnitCoord && Equals((MeshMapUnitCoord)obj);
	}
	public bool Equals(MeshMapUnitCoord other)
	{
		return x == other.x && y == other.y && level == other.level;
	}
	public override int GetHashCode()
	{
		return (level<<35) + (y<<20 +x);
	}
}
```

```
public class MeshMap
{
	public const int _tileGroupNum = 16;
	public const float _tileGroupLen = 16;
	public const float _tileUnit = _tileGroupLen / _tileGroupNum;

	private int _maxLevel = 1;
	private int _level = 0;
	public int Level
	{
		get{return _level;}
	}
	private float _minScale = 1;
	private float _bounds = 0.1f;
	Rect _effectVisibleRegion;
	private Vector3 _effectVisibleRegionInWorldMin;
	private Vector3 _effectVisibleRegionInWorldMax;
	public Vector3 _effectVisibleRegionInBaseMapMin;
	public Vector3 _effectVisibleRegionInBaseMapMax;
	private Vector3 _visibleAreaSize = Vector3.zero;
	private Vector3 _VisibleCenterPos = Vector3.zero;

	public Quaternion Rotation{get;private set;}
	private float _unitLen = _tileGroupLen;
	private float _unitLenInWorld = _tileGroupLen;
	public float UnitLen{get{return _unitLen;}}
	public float UnitLenInWorld{get{return _unitLenInWorld;}}
	private Vector3[] _mapRegionCornor = new Vector3[4];// lt rt rb lb 

	private float _scale = 1;
	public Vector3 Scale{get;private set;}
	private float _maxUnitDis = 1;
	private Vector3 _origionPos = Vector3.zero;
	public List<int> _visibleMeshCenterPosIndices = new List<int>(400);
	public List<Vector3> _visibleMeshCenterPosList = new List<Vector3>(400);
	public List<int> _visibleMeshOutLineVertexIndices = new List<int>(400);
	public List<Vector3> _visibleMeshOutlineVertexPosList = new List<Vector3>(400);
	public List<MeshMapUnitCoord> _visibleMeshUnitCoordList = new List<MeshMapUnitCoord>(400);
	public List<Quad> _visibleMeshSubQuad = new List<Quad>();
	public Quad _visibleMeshQuad = new Quad();

	private Matrix4x4 _translateMatrix = Matrix4x4.identity;
	private Matrix4x4 _rotateMatrix = Matrix4x4.identity;
	private Matrix4x4 _scaleMatrix = Matrix4x4.identity;
	private Matrix4x4 _localToWorldMatrix = Matrix4x4.identity;
	private Matrix4x4 _worldToLocalMatrix = Matrix4x4.identity;
	private Matrix4x4 _localToWorldMatrixWithoutRotation = Matrix4x4.identity;
	private Matrix4x4 _worldToLocalMatrixWithoutRotation = Matrix4x4.identity;

	private MeshMapUnitCoord _meshUnitCoord = new MeshMapUnitCoord();
	private Vector3[] _visibleMapAreaCornerPos = new Vector3[4];
	private Vector3[] _visibleAreaCornerPos = new Vector3[4];
	private Vector3[] _visibleAreaLineParam = new Vector3[4];
	private Action _onMapUpdated = null;

	Vector3 _mapReginMin = Vector3.zero;
	Vector3 _mapReginMax = Vector3.zero;
	private Vector3 _mapVisibleReginMinCornor;
	private Vector3 _mapVisibleReginMaxCornor;
	public event Action OnMapUpdatae{
		add {_onMapUpdated += value;}
		remove (_onMapUpdated -= value;)
	}
	public MeshMap(Rect mapRegin)
	{
		SetMapValidArea(mapRegin.min, mapRegin.max);
		SetUnitLen(_tileGroupLen);
	}
	public void SetMapValidArea(Vector2 min, Vector2 max)
	{
		_mapReginMin = new Vector3(min.x, 0, min.y);
		_mapReginMax = new Vector3(max.x, 0, max.y);
		Rect rect = new Rect(min.x, min.y, max.x - min.x, max.y - min.y);
		_mapRegionCornor[0] = new Vector3(rect.min.x, 0, rect.max.y);
		_mapRegionCornor[1] = new Vector3(rect.max.x, 0, rect.max.y);
		_mapRegionCornor[2] = new Vector3(rect.max.x, 0, rect.min.y);
		_mapRegionCornor[3] = new Vector3(rect.min.x, 0, rect.min.y);
		SetMinScale();
	}
	public bool IncludedInMapArea(Vector2 mapPos)
	{
		Vector3 pos = new Vector3(mapPos.x, 0, mapPos.y);
		pos = CvtLocalToWorldPos(pos);
		pos = _worldToLocalMatrixWithoutRotation.MultiplyPoint3x4(pos);
		if(pos.x<_mapReginMin.x) return false;
		if(pos.y < _mapReginMin.y) return false;
		if(pos.x > _mapReginMax.x) return false;
		if(pos.y > _mapReginMax.y) return false;
		return true;
	}
	public void SetMinScale()
	{
		float scale = _effectVisibleRegin.width / (_MapReginMax.m - _mapReginMin.x);
		float b = _effectVisibleRegin.height / (_mapReginMax.z - _mapReginMin.z);
		scale = Mathf.Max(scale, b);
		scale *= 1.1f;
		_minScale = scale;
		_maxLevel = Mathf.CeilToInt(Mathf.Log(_minScale, 0.5f));
	}
	void SetUnitLen(float len, bool needUpdate = false)
	{
		len = Mathf.Abs(len);
		_unitLen = len;
		_maxUnitDis = 1 * _unitLen * _sacle;
		_unitLenInWorld = _unitLen * _scale;
		if(needUpdate) CalcualteVisibleGrid();
	}
	public void SetMapRotationalAngle(float angle, bool needUpdate = false)
	{
		Rotation = Quaternion.Euler(0, angle, 0);
		_rotateMatrix = Matrix4x4.Rotate(Rotation);
		UpdateMatrix();
		if(needUpdate) CalcualteVisibleGrid();
	}
	public void SetOriginPosition(Vector3 pos, bool needUpdate = false)
	{
		_originPos = CorrectMapOriginPos(pos);
		_translateMatrix = matrix4x4.Translate(_originPos);
		UpdateMatrix();
		if(needUpdate) CalcualteVisibleGrid();
	}
	Vector3 CorrectMapOriginPos(Vector3 origion)
	{
		_localToWorldMatrixWithoutRotation = Matrix4x4.Translate(origin) * _scaleMatrix;
		_worldToLocalMatrixWithoutRotation = _localToWorldMatrixWithoutRotation.inverse;
		Vector3 max = _localToWorldMatrixWithoutRotation.MultiplyPoint3x4(_mapReginMax);
		Vector3 min = _localToWorldMatrixWithoutRotation.MultiplyPoint3x4(_mapReginMin);
		Vector3 center = (max + min)/2;
		if(_effectVisibleRegin.max.x - _effectVisibleRegin.min.x < max.x - min.x)
		{
			if(_effectVisibleRegin.max.x > max.x){
				origin.x += _effectVisibleRegin.max.x - max.x;
			}
			if(_effectVisibleRegin.min.x < min.x)
			{
				origin.x += _effectVisibleRegin.min.x - min.x;
			}
		}
		else
		{
			origion.x += _effectVisibleRegin.center.x - center.x;
		}
		if(_effectVisibleRegin.max.y - _effectVisibleRegin.min.y < max.z - min.z)
		{
			if(_effectVisibleRegin.max.y > max.z)
			{
				origin.z += _effectVisibleRegin.max.y - max.z;
			}
			else if(_effectVisibleRegin.min.y < min.z)
			{
				origin.z += _effectVisibleRegin.min.y -min.z;
			}
		}
		else
		{
			origin.z += _effectVisibleRegin.center.y - center.z;
		}
		_localToWorldMatrixWithoutRotation = Matrix4x4.Translate(origin) * _scaleMatrix;
		_worldToLocalMatrixWithoutRotation = _localToWorldMatrixWithoutRotation.inverse;

		_effectVisibleReginInBaseMapMin = worldToLocalMatrixWithoutRotation.MultiplyPoint3x4(_effectVisibleReginInWorldMin);
		_effectVisibleReginInBaseMapMax = worldToLocalMatrixWithoutRotation.MultiplyPoint3x4(_effectVisibleReginInWorldMax);
		return origin;
	}
	public float SetScale(float scale, bool needUpdate = false)
	{
		scale = Mathf.Abs(scale);
		scale = Mathf.Clamp(scale, _minScale, 1+0.1f);
		_scale = scale;
		Scale = new Vector3(scale, scale, scale);
		_scaleMatrix = Matrix4x4.Scale(Vector3.one * scale);
		_level = Mathf.CeilToInt(Mathf.Log(scale, 0.5f));
		float unit = Mathf.Pow(2, _level);;
		float unitLenInWorld = unit * _scale;
		if(unitLenInWorld > 1.5f)
		{
			--_level;
			_level = Mathf.Clamp(_level, 0, _maxLevel);
			unit = mathf.Pow(2, _level);
		}
		else if(unitLenInWorld<0.75f)
		{
			++_level;
			_level = Mathf.Clamp(_level, 0, _maxLevel);
			unit = Mathf.Pow(2, _level);
		}
		SetUnitLen(unit * _tileGroupLen);
		UpdateMatrix();
		if(needUpdate) CalcualteVisibleGrid();
		return _scale;
	}
	public Vector3 GetOriginPos()
	{
		return _originPos;
	}
	public Vector3 SetViewAreaInZeroPanel(Vector3[] posList)
	{
		CalculateEffectVisibleSize(posList);
		_visibleCenterPos = Vector3.zero;
		Vector3 min = Vector3.positiveInfinity;
		Vector3 max = Vector3.negativeInfinity;
		float(int i=0;i<4;++i)
		{
			Vector3 pos = posList[i];
			_visibleAreaCornerPos[i] = pos;
			_visibleAreaLineParam[i] = GetLineParam(pos, posList[(i+1)%4]);
			_visibleCenterPos += pos;
			min.x = min.x < pos.x ? min.x : pos.x;
			min.y = min.y < pos.y ? min.y : pos.y;
			min.z = min.z < pos.z ? min.z : pos.z;
			max.x = max.x > pos.x ? max.x : pos.x;
			max.y = max.y > pos.y ? max.y : pos.y;
			max.z = max.z > pos.z ? max.z : pos.z;
		}
		_visibleAreaSize = max - min;
		_visibleCenterPos /= 4;
		return _visibleCenterPos;
	}

	void CalculateEffectVisibleSize(Vector3[] polList)
	{
		_effectVisibleRegin.min = new Vector2(posList[3].x, posList[3].z);
		_effectVisibleRegin.max = new Vector2(posList[2].x, posList[1].z);
		_effectVisibleReginInWorldMin = new Vector3(PosList[0].x, _originPos.y, posList[3].z);
		_effectVisibleReginInWorldMax = new Vector3(PosList[1].x, _originPos.y, posList[1].z);

		_visibleMapAreaCornerPos[0] = PosList[0];
		_visibleMapAreaCornerPos[1] = PosList[1];
		_visibleMapAreaCornerPos[2] = PosList[2];
		_visibleMapAreaCornerPos[3] = PosList[3];
		SetMinScale();
	}
	public Static Vector3 GetLineParam(Vector3 point1, Vector3 point2)
	{
		Vector3 param = Vector3.Cross(Vector3.up, point1 - point2);
		param.y = param.z;
		param.z = -point1.x * param.x - point1.z * param.y;
		param /= Mathf.Sqrt(param.x * param.x + param.y * param.y);
		return param;
	}
	public Static Vector2 FloorGridPos(Vector2 mapPos)
	{
		mapPos.x = Mathf.Floor(mapPos.x / _tileUnit) * _tileUnit;
		mapPos.y = Mathf.Floor(mapPos.y / _tileUnit) * _tileUnit;
		return mapPos;
	}
	public void BindLocalPosToWorld(Vector3 localPos, Vector3 worldPos, bool needUpdate = false)
	{
		Vector3 offset = worldPos - CvtLocalToWorldPos(lcoalPos);
		SetOriginPosition(_originPos + offset, needUpdate);
	}
	public Vector3 CvtLocalToWorldPos(Vector3 lcoalPos)
	{
		return _localToWorldMatrix.MultiplyPoint3x4(localPos);
	}
	public Vector3 CvtWorldToLocalPos(Vector3 worldPos)
	{
		return _worldToLocalMatrix.MultiplyPoint3x4(worldPos);
	}
	float tilePovitOffset = 0; //[0-1]
	public MeshMapUnitCoord PickMeshUnitCoord(Vector3 woldPos)
	{
		worldPos.y = 0;
		Vector3 posInMeshMap = _worldToLocalMatrix.MultiplyPoint3x4(worldPos);
		int x = Mathf.FloorToInt(posInMeshMap.z / _unitLen + tilePovitOffset);
		int y = Mathf.FloorToInt(posInMeshMap.x / _unitLen + tilePovitOffset);
		return new MeshMapUnitCoord(x, y, _level);
	}
	public Vector3 RayCastMeshMapPos(Vector3 origin, Vector3 dir)
	{
		Vector3 centerPos = Vector3.zero;
		float t = (_origionPos.y - origion.y)/dir.y;
		centerPos = origion + t* dir;
		centerPos.y = _origionPos.y;
		return centerPos;
	}
	void UpdateMatrix()
	{
		_lcoalToWorldMatrix = _translateMatrix * rotateMatrix * _scaleMatrix;
		_worldToLocalMatrix = _localToWorldMatrix.inverse;
	}
	public Rect _visibleGridRigion {get;private set;}
	Vector3[] cornerPos = new Vector3[4];
	public void CalcualteVisibleGrid()
	{
		_visibleMeshQuad = new Quad(Vector2.positiveInfinity, Vector2.negativeInfinity);
		_visibleMeshSubQuad.Clear();
		_visibleMeshCenterPosList.Clear();
		_visibleMeshCenterPosIndices.Clear();
		_visibleMeshOutlineVertexIndices.Clear();
		_visibleMeshOutlineVertexPosList.Clear();
		_visibleMeshUnitCoordList.Clear();
		CalcCenterMeshUnitCoord();
		float radius = Mathf.Sqrt(_visibleAreaSzie.x * _visibleAreaSzie.x + _visibleAreaSize.z * _visibleAreaSize.z)/2;
		float unitLenInWorld = _unitLen * _scale;
		int halfCount = Mathf.CeilToInt(radius / unitLenInWorld) + 1;
		float yStart = (-halfCount + _meshUnitCoord.y) * _unitLen;
		int outlineIndicesCount = 0;
		int centerIndicesCount =0;
		Vector3 centerOffset = new Vector3(_unitLen *(0.5f - tilePovitOffset), 0, _unitLen * (0.5f - tilePovitOffset));
		_visibleGridRigion = new Rect((-halfCount + _meshUnitCoord.x) * _unitLen,
			(-halfCount + _meshUnitCoord.y) * _unitLen,
				halfCount * _unitLen * 2, halfCount * _unitLen * 2);
		for(int j = -halfCount + _meshUnitCoord.y; j<= halfCount + _meshUnitCoord.y; ++j)
		{
			float xStart =(-halfCount + _meshUnitCoord.x) * _unitLen;
			for(int i=-halfCount + _meshUnitCoord.x; i<=halfCount+_meshUnitCoord.x; ++i)
			{
				Vector3 posOnMesh = new Vector3(xStart, 0, yStart);
				xStart += _unitLen;

				Vector3 posOnWorld = _localToWorldMatrix.MultiplyPoint3x4(posOnMesh + centerOffset);
				if(!MeshIsVisible(posOnWorld)) continue;

				Quad quad = new Quad();
				quad.MinX = posOnMesh.x - _unitLen * tilePovitOffset;
				quad.MinY = posOnMesh.z - _unitLen * tilePovitOffset;
				quad.MaxX = quad.MinX + _unitLen;
				quad.MaxY = quad.MinY + _unitLen;
				_visibleMeshSubQuad.Add(quad);
				_visibleMeshQuad.Include(ref quad);
				_visibleMeshCenterPosList.Add(posOnWorld);
				_visibleMeshCenterPosIndices.Add(centerIndicesCount++);
				_visibleMeshUnitCoordList.Add(new MeshMapUnitCoord(i,j,_level));

				cornerPos[0] = new Vector3(posOnMesh.x - _unitLen * tilePovitOffset, posOnMesh.y, posOnMesh.z - _unitLen * tilePovitOffset);
				cornerPos[1] = cornerPos[0] + new Vector3(_unitLen, 0,0);
				cornerPos[2] = cornerPos[1] + new Vector3(0,0, _unitLen);
				cornerPos[3] = cornerPos[2] + new Vector3(-_unitLen, 0,0);

				_visibleMeshOutlineVertexIndices.Add(outlineIndicesCount++);
				_visibleMeshOutlineVertexPosList.Add(cornerPos[0]);
				_visibleMeshOutlineVertexIndices.Add(outlineIndicesCount++);
				_visibleMeshOutlineVertexPosList.Add(cornerPos[3]);
				_visibleMeshOutlineVertexIndices.Add(outlineIndicesCount++);
				_visibleMeshOutlineVertexPosList.Add(cornerPos[3]);
				_visibleMeshOutlineVertexIndices.Add(outlineIndicesCount++);
				_visibleMeshOutlineVertexPosList.Add(cornerPos[2]);
			}
			yStart += _unitLen;
		}
		_onMapUpdated?.Invoke();
	}
	bool MeshIsVisible(Vector3 pos)
	{
		float distance = 0;
		Vector3 param;
		for(int i=0;i<4;++i>)
		{
			param = _visibleAreaLineParam[i];
			distance = param.x * pos.x + param.y * pos.z + param.z;
			if(distance > _maxUnitDis) return false;
		}
		return true;
	}
	void CalcCenterMeshUnitCoord()
	{
		_meshUnitCoord = PickMeshUnitCoord(_visibleCenterPos, false);
	}
}
```
