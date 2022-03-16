---
title: GUIStyle
date: 2022-03-16 20:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
- GUIStyle
---

## 功能
Unity自定义编辑器时，需要使用GUIStyle自定义UI组件的显示（包括区域大小、位置、以及字体等），这里自定义一个GUIStyle参数调节预览窗口，辅助自定义编辑器。

## 源码
```C#
using UnityEngine;
using UnityEditor;
 
public class GUIStyleEditor : EditorWindow
{
	[MenuItem("Window/GUIStyleEditor")]
	static void Open()
	{
		var win = GUIStyleEditor.GetWindow<GUIStyleEditor>();
		win.titleContent = new GUIContent("GUIStyleEditor");
		win.Show();
	}
    GUIStyle _GUIStyle = new GUIStyle();
	GUIStyle _GUIContent = new GUIStyle();

	GUIContent[] _GUIContents = null;
	GUIStyle _GUIStyleBg = new GUIStyle();
	Vector2 _scrollPos = Vector2.zero;
 
    private void OnEnable()
    {
        _GUIStyleBg.margin = new RectOffset(-10, -10, 10, 10);
		_GUIStyleBg.padding = new RectOffset(10, 10, 5, 5);
		_GUIStyleBg.border = new RectOffset(10, 10, 10, 10);
		_GUIStyleBg.normal.background = EditorGUIUtility.IconContent("0L box@2x").image as Texture2D;
		_GUIContents = new GUIContent[]{_GUIContent, _GUIContent, _GUIContent};

		_GUIStyle.border = new RectOffset(10, 10, 10, 10);
		_GUIStyle.margin = new RectOffset(10, 10, 10, 10);
		_GUIStyle.padding = new RectOffset(10, 10, 10, 10);
		_GUIStyle.overflow = new RectOffset(5, 5, 0, 0);
		_GUIStyle.normal.background = EditorGUIUtility.IconContent("0L box@2x").image as Texture2D;
		_GUIContent.image = EditorGUIUtility.IconContent("BuildSettings.Editor").image;
    }

	private void OnGUI()
	{
		GUILayout.BeginVertical(_GUIStyleBg);
		GUILayout.Label("GUIContent");
		EditorGUILayout.BeginHorizontal();
		EditorGUILayout.PrefixLabel("GUIContent.text");
		_GUIContent.text = EditorGUILayout.TextField(_GUIContent.text);
		EditorGUILayout.EndHorizontal();
		_GUIContent.image = EditorGUILayout.ObjectField("GUIContent.image", _GUIContent.image, typeof(Texture), false, GUILayout.Height(40)) as Texture;
		GUILayout.Space(10);

		GUILayout.Label("GUIStyle");
		_GUIStyle.fontStyle = (FontStyle)EditorGUILayout.EnumPopup("GUIStyle.fontStyle", _GUIStyle.fontStyle);
		_GUIStyle.alignment = (TextAnchor)EditorGUILayout.EnumPopup("GUIStyle.alignment", _GUIStyle.alignment);
		_GUIStyle.imagePosition = (ImagePosition)EditorGUILayout.EnumPopup("GUIStyle.imagePosition", _GUIStyle.imagePosition);
		_GUIStyle.contentOffset = EditorGUILayout.Vector2Field("GUIStyle.contentOffset", _GUIStyle.contentOffset);
		_GUIStyle.border = DrawRectOffset("GUIStyle.border", _GUIStyle.border);
		_GUIStyle.margin = DrawRectOffset("GUIStyle.margin", _GUIStyle.margin);
		_GUIStyle.padding = DrawRectOffset("GUIStyle.padding", _GUIStyle.padding);
		_GUIStyle.overflow = DrawRectOffset("GUIStyle.overflow", _GUIStyle.overflow);
		_GUIStyle.normal.textColor = EditorGUILayout.ColorField("textColor", _GUIStyle.normal.textColor);
		_GUIStyle.normal.backgrount = EditorGUILayout.ObjectField("GUIContent.image", _GUIStyle.normal.background, typeof(Texture2D), false, GUILayout.Height(40)) as Texture2D;
		GUILayout.EndVertical();


		GUILayout.Space(2);
		_scrollPos = GUILayout.BeginScrollView(_scrollPos, _GUIStyleBg);
		GUILayout.Button(_GUIContent, _GUIStyle);
		GUILayout.Label(_GUIContent, _GUIStyle);
		GUILayout.Box(_GUIContent, _GUIStyle);
		GUILayout.TextField(_GUIContent.text, _GUIStyle);
		GUILayout.TextArea(_GUIContent.text, _GUIStyle);
		GUILayout.Toggle(ture, _GUIContent, _GUIStyle);
		GUILayout.Toolbar(1, _GUIContents, _GUIStyle);
		GUILayout.SelectionGrid(1, _GUIContents, 2, _GUIStyle);
		GUILayout.RepeatButton(_GUIContent, _GUIStyle);
		GUILayout.PaswordField(_GUIContent.text, '*', _GUIStyle);
		GUILayout.BeginVertical(_GUIStyle);
		GUILayout.Space(100);
		GUILayout.EndScrollView();
		GUILayout.EndVertical();
	}
	Vector4 RectOffsetToVector4(RectOffset rectOffset)
	{
		return new Vector4(rectOffset.left, rectOffset.right, rectOffset.top, rectOffset.bottom);
	}
	RectOffset Vector4ToRectOffset(Vector4 vector)
	{
		return new RectOffset((int)vector.x, (int)vector.y, (int)vector.z, (int)vector.w);
	}
	RectOffset DrawRectOffset(string text, RectOffset rectOffset)
	{
		Vector4 vec = RectOffsetToVector4(rectOffset);
		vec = EditorGUILayout.Vector4Field(text, vec);
		return Vector4ToRectOffset(vec);
	}
}
```C#
