---
title: Unity Package Menu Item
date: 2022-11-27 10:01:00
categories:
- [Unity, Extention, Editor, AssetsMenuUtility]
tags:
- Unity
- Editor
- Package
---

## Package扩展命令
```C#
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEditor;
using UnityEditor.PackageManager.Requests;
using UnityEditor.PackageManager;
using UnityEngine;
using System.Text.RegularExpressions;

namespace Unity.Editor.Extention
{
    static class PackageMenuItem
    {
        static String targetPackage;
        static EmbedRequest Request;
        static ListRequest LRequest;

        [MenuItem("Assets/Package/Embed Installed Package")]
        static void EmbedInstalledPackage()
        {
            if(Selection.assetGUIDs.Length > 0)
            {
                string path = AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[0]);
                Match match = Regex.Match(path, "^Packages/([^/]+)$");
                if(match.Success)
                {
                    string targetPackage = match.Groups[1].Value;
                    //Debug.LogError(targetPackage);
                    Embed(targetPackage);
                }
            }
        }
        [MenuItem("Assets/Package/Embed Installed Package", validate = true)]
        static bool Check_EmbedInstalledPackage()
        {
            if (Selection.assetGUIDs.Length > 0)
            {
                string path = AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[0]);
                Match match = Regex.Match(path, "^Packages/([^/]+)$");
                return match.Success;
            }
            return false;
        }
        static void Embed(string inTarget)
        {
            // Embed a package in the project
            Debug.Log("Embed('" + inTarget + "') called");
            Request = Client.Embed(inTarget);
            EditorApplication.update += Progress;

        }

        static void Progress()
        {
            if (Request.IsCompleted)
            {
                if (Request.Status == StatusCode.Success)
                    Debug.Log("Embedded: " + Request.Result.packageId);
                else if (Request.Status >= StatusCode.Failure)
                    Debug.Log(Request.Error.message);

                EditorApplication.update -= Progress;
            }
        }


        #region Test
        [MenuItem("Assets/Package/Test/Get Package Name", priority = 100)]
        static void GetPackageName()
        {
            // First get the name of an installed package
            LRequest = Client.List();
            EditorApplication.update += LProgress;
        }

        static void LProgress()
        {
            if (LRequest.IsCompleted)
            {
                if (LRequest.Status == StatusCode.Success)
                {
                    foreach (var package in LRequest.Result)
                    {
                        // Only retrieve packages that are currently installed in the
                        // project (and are neither Built-In nor already Embedded)
                        if (package.isDirectDependency && package.source
                            != PackageSource.BuiltIn && package.source
                            != PackageSource.Embedded)
                        {
                            targetPackage = package.name;
                            Debug.LogError(targetPackage);
                            //break;
                        }
                    }

                }
                else
                    Debug.Log(LRequest.Error.message);

                EditorApplication.update -= LProgress;

                //Embed(targetPackage);

            }
        }
        #endregion
    }
}
```
