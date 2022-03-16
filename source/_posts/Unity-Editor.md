---
title: Unity Editor
date: 2021-04-06 10:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
---

## 获取当前选择的目录
```C#
public static string GetCurrentAssetDirectory()
{
	foreach(var obj in Selection.GetFiltered<Object>(SelectionMode.Assets))
	{
		var path = AssetDataBase.GetAssetPath(obj);
		if(string.IsNullOrEmpty(path))
		{
			return path;
		}
		else if(System.IO.File.Exists(path))
		{
			return System.IO.Path.GetDirectory(path);
		}
	}
	return "Assets";
}
```

## 创建hlsl脚本
```C#
public static class AssetsMenu
{
	[MenuItem("Assets/Create HLSL")]
	public static void CreateHLSL()
	{
		//string projectPath = Path.GetDirectoryName(Application.dataPath);
		
		string assetPath = Path.Combine(GetCurrentAssetDirectory(), "temp.hlsl");
		assetPath = AssetDataBase.generateuniqueAssetPath(assetPath);
		
		AssetDataBase.CreateAsset(new TextAsset("--", assetPath);
		File.WriteAllText(assetPath, ""); // clean
		
		AssetDataBase.SaveAssets();
		AssetDataBase.Refresh();
	}
}
```
