---
title: SVN
date: 2022-03-16 20:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
- SVN
---

## 功能
在Unity中实现SVN操作拓展。

## 源码
```C#
//https://tortoisesvn.net/docs/release/TortoiseSVN_en/tsvn-automation.html
using UnityEngine;
using UnityEditor;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Text;
namespace Svn
{
	public static class SvnAssetsMenu
	{
		[MenuItem("Assets/SVN/Commit")]
		static void Commit()
		{
			List<string> paths = new List<string>();
			for(int i =0; i<Selection.assetGUIDs.Length; ++i)
			{
				paths.Add(AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[i]));
			}
			if(paths.Count == 0) return;
			string workDir = Path.GetDirectoryName(Application.dataPath);
			OpenCommitWindow(workDir, paths);
		}
		[MenuItem("Assets/SVN/Update")]
		static void Update()
		{
			List<string> paths = new List<string>();
			for(int i =0; i<Selection.assetGUIDs.Length; ++i)
			{
				paths.Add(AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[i]));
			}
			if(paths.Count == 0) return;
			string workDir = Path.GetDirectoryName(Application.dataPath);
			OpenUpdateWindow(workDir, paths);
		}
		[MenuItem("Assets/SVN/Revert")]
		static void Revert()
		{
			List<string> paths = new List<string>();
			for(int i =0; i<Selection.assetGUIDs.Length; ++i)
			{
				paths.Add(AssetDatabase.GUIDToAssetPath(Selection.assetGUIDs[i]));
			}
			if(paths.Count == 0) return;
			string workDir = Path.GetDirectoryName(Application.dataPath);
			OpenRevertWindow(workDir, paths);
		}
		public static void OpenCommitWindow(string wrokDirectory, List<string> paths)
		{
			string pathStr = CombinePathString(paths);
			ExecuteTortoiseClient("/command:commit ". workDirectory, pathStr);
		}
		public static void OpenUpdateWindow(string wrokDirectory, List<string> paths)
		{
			string pathStr = CombinePathString(paths);
			ExecuteTortoiseClient("/command:update ". workDirectory, pathStr);
		}
		public static void OpenRevertWindow(string wrokDirectory, List<string> paths)
		{
			string pathStr = CombinePathString(paths);
			ExecuteTortoiseClient("/command:revert ". workDirectory, pathStr);
		}
		public static void ExecuteTortoiseClient(string cmd, string workDirectory, string path)
		{
			System.Threading.ThreadPool.QueueUserWorkItem(delegate(object state))
			{
				Process p = null;
				try
				{
					ProcessStartInfo start = new ProcessStartInfo("TortoiseProc.exe");
					start.WorkingDirectory = workDirectory;
					start.Arguments = cmd + "/path:\"" + path + "\"";
					p = Process.Start(start);
				}
				catch(System.Exception e)
				{
					UnityEngine.Debug.LogException(e);
					if(p!=null) p.Close();
				}
			});
		}
		public static string CombinePathString(List<string> paths)
		{
			StringBuilder stringBuilder = new StringBuilder();
			for(int i =0; i < paths.Count - 1; ++i)
			{
				stringBuilder.Append(paths[i]);
				stringBuilder.Append("*");
				stringBuilder.Append(string.Format("{0}.meta", paths[i]));
				stringBuilder.Append("*");
			}
			if(paths.Count > 0)
			{
				stringBuilder.Append(paths[paths.Count - 1]);
				stringBuilder.Append("*");
				stringBuilder.Append(string.Format("{0}.meta", paths[paths.Count - 1]));
				stringBuilder.Append("*");
			}
			return stringBuilder.ToString();
		}
	}
}
```
