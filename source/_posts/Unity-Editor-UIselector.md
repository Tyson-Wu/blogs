---
title: Scene UI Selection Filter
date: 2022-03-10 10:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
- UI Selector
---

## 功能
Unity中可以通过Scene窗口点击选择模型，但是UI之间是重叠关系，每次点击的时候选中的都是最前面的，即便这个UI并没有显示的内容。所以这里实现了一个UI过滤器，可以非常方便的快速选中看到的UI。

## 源码
```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
using System.Reflection;
using System;
using UnityEngine.UI;
using UnityEditor.SceneManagement;
using System.Linq;
using UnityEngine.SceneManagement;
 
public class SceneUISelectionFilter : EditorWindow
{
    [MenuItem("Tools/Scene UI Selection Filter")]
    public static void ShowWindow()
    {
		Rect rect = new Rect();
		if(SceneView.lastActiveSceneView != null)
			rect = SceneView.lastActiveSceneView.position;
		rect.size = new Vector2(230, 100);
		rect.position = rect.position + new Vector2(10, 10);
        var win = EditorWindow.GetWindowWithRect<SceneUISelectionFilter>(rect, true);
		win.titleContent = new GUIContent("UI Selector");
		win.Show();
    }
 
    private MethodInfo Internal_PickClosestGO;
 
    private void OnEnable()
    {
        Assembly editorAssembly = typeof(Editor).Assembly;
        System.Type handleUtilityType = editorAssembly.GetType("UnityEditor.HandleUtility");
 
        FieldInfo pickClosestDelegateInfo = handleUtilityType.GetField("pickClosestGameObjectDelegate", BindingFlags.Static | BindingFlags.NonPublic);
        Delegate pickHandler = Delegate.CreateDelegate(pickClosestDelegateInfo.FieldType, this, "OnPick");
        pickClosestDelegateInfo.SetValue(null, pickHandler);
 
        Internal_PickClosestGO = handleUtilityType.GetMethod("Internal_PickClosestGO", BindingFlags.Static | BindingFlags.NonPublic);
    }
 
	private void OnDisable()
	{
		Assembly editorAssembly = typeof(Editor).Assembly;
        System.Type handleUtilityType = editorAssembly.GetType("UnityEditor.HandleUtility");
 
        FieldInfo pickClosestDelegateInfo = handleUtilityType.GetField("pickClosestGameObjectDelegate", BindingFlags.Static | BindingFlags.NonPublic);
        pickClosestDelegateInfo.SetValue(null, null);
	}

	private void OnGUI()
	{
		GUILayout.Space(10);
		EditorGUILayout.LabelField("当前优先选中Image、Text等UI组件");
		GUILayout.Space(10);
		EditorGUILayout.HelpBox("点击Scene窗口中的UI会自动过滤掉哪些遮挡在前面看不见的UI，从而可以快速选中其中的Image、Text等组件。", MessageType.Inof);
	}

    private GameObject OnPick(Camera cam, int layers, Vector2 position, GameObject[] ignore, GameObject[] filter, out int materialIndex)
    {
        materialIndex = -1;
        filter = GetPickableObject();
 
        return (GameObject)Internal_PickClosestGO.Invoke(null, new object[] { cam, layers, position, ignore, filter, materialIndex });
    }

	private GameObject[] GetPickableObject()
	{
		List<GameObject> gameObjects = new List<GameObject>();
		for(int i = 0; EditorSceneManager.loadedSceneCount; ++i)
		{
			Scene scene = EditorSceneManager.GetSceneAt(i);
			foreach(var root in scene.GetRootGameObjects())
			{
				gameObjects.AddRange(root.GetComponentsInChildren<Graphics>().Select((a)=>{return a.gameObject;}));
			}
		}
		return gameObjects.ToArray();
	}
}
```C#
