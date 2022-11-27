---
title: Create Parent For Transforms
date: 2022-11-27 10:01:00
categories:
- [Unity, Extention, Editor, MenuUtility]
tags:
- Unity
- Editor
---

## 扩展命令
```C#
using UnityEngine;
using UnityEditor;

class CreateParentForTransforms : ScriptableObject
{
    //[MenuItem("MenuUtility/Create Parent For Selection _p")]
    [MenuItem("MenuUtility/Create Parent For Selection")]
    static void MenuInsertParent()
    {
        Transform[] selection = Selection.GetTransforms(
            SelectionMode.TopLevel | SelectionMode.Editable);
        GameObject newParent = ObjectFactory.CreateGameObject("Parent");
        Undo.RegisterCreatedObjectUndo(newParent, "Create Parent For Selection");
        foreach (Transform t in selection)
        {
            Undo.SetTransformParent(t, newParent.transform, "Create Parent For Selection 1");
        }
    }

    // Disable the menu if there is nothing selected
    //[MenuItem("MenuUtility/Create Parent For Selection _p", true)]
    [MenuItem("MenuUtility/Create Parent For Selection", true)]
    static bool ValidateSelection()
    {
        return Selection.activeTransform != null;
    }
}
```
