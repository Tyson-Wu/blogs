---
title: Houdini Debug List
date: 2022-08-10 10:01:00
categories:
- [Unity, Houdini]
tags:
- Unity
- Houdini
---

## HEU_BaseSync.cs
```
terrainData.size = new Vector3(terrainBuffers[t]._terrainSizeX, heightRange, terainBuffers[t]._terrainSizeY);
terrain.Flush();
UnityEditor.AssetDatabase.SaveAssets();  // add this line
```

## HEU_HAPIUtility.cs(H19.0.434)
```
// Do not process the main display geo twice!
if (editGeoInfo.isDisplayGeo) continue;

// We only handle editable curves for now
//if (editGeoInfo.type != HAPI_GeoType.HAPI_GEOTYPE_CURVE) continue; 
if (editGeoInfo.type != HAPI_GeoType.HAPI_GEOTYPE_CURVE && editGeoInfo.type != HAPI_GeoType.HAPI_GEOTYPE_INTERMEDIATE) continue;   // modify this
//session.CookNode(editNodeID, HEU_PluginSettings.CookTemplatedGeos);

// Add this geo to the geo info array
editableGeoInfos.Add(editGeoInfo);
```