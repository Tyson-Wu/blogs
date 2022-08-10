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