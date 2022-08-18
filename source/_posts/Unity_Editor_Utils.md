---
title: Unity Editor Utils
date: 2022-08-18 10:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
---

## 
```C#
[MenuItem("Assets/Helper/Get Texture Path For Houdini Terrain",true,1)]
static bool _GetTexturePathForHoudiniTerrain()
{
    if(Selection.assetGUIDs.Length>0)
    {
        string path = AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[0]);
        string temp = "Resources/";
        int index = path.IndexOf(temp);
        if(index>-1)
        {
            return true;
        }
    }
    return false;
}
[MenuItem("Assets/Helper/Get Texture Path For Houdini Terrain",false,1)]
static void _GetTexturePathForHoudiniTerrain()
{
    string path = AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[0]);
    string temp = "Resources/";
    int index = path.IndexOf(temp);
    int end = path.lastIndexOf(',');
    index += temo.Length;
    GUIUtility.systemCopyBuffer = path.Substring(index,end-index);
}

[MenuItem("Assets/Helper/Trim File Name")]
static void TrimFileName()
{
    if(Selection.assetGUIDs.Length >0)
    {
        for(int i=0;i<Selection.assetGUIDs.Length;++i)
        {
            string path = AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[i]);
            string fileName = Path.GetFileName(path);
            int index = path.IndexOf(' ');
            if(index>-1)
            {
                string newFileName = fileName.Replace(' ','_');
                string rst = AssetDatabase.RenameAsset(path,newFileName);
            }
        }
        AssetDatabase.SaveAssets();
    }
}
```