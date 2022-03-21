---
title: Project Linker
date: 2022-03-21 20:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
- Tree View
---

## 功能
在Unit内创建多个项目实例。

## 源码
```C#
namespace ProjectLinker
{
	static class CmdHelper
	{
		public static void LinkFolder(string orgPath, string linkPath)
		{
			ProcessStartInfo start = new ProcessStartInfo("cmd.exe");
			start.CreateNoWindow = true;
			start.UseShellExecute = false;
			start.RedirectStandardInput = true;
			var process = Process.Start(start);
			var sw = process.StandardInput;
			sw.WriteLine(@$"mklink /D ""{linkPath}"" ""{orgPath}""");
			sw.Close();
			process.WaitForExit();
			process.Close();
		}
		public static void LaunchUnityProject(string projectFullPath)
		{
			string editorPath = EditorApplication.applicationPath;
			System.Treading.ThreadPool.QueueUserWorkItem(delegate (object state)
			{
				Process p = null;
				try
				{
					string arg = $"-projectPath \"{projectFullPath}\"";
					ProcessStartInfo start = new ProcessStartInfo(editorPath, arg);
					start.CreateNoWindow = false;
					start.UseShellExecute = false;

					p = Process.Start(start);
				}
				catch(System.Exception e)
				{
					UnityEngine.Debug.LogException(e);
					if(p != null)
					{
						p.Close();
					}
				}
			})
		}
	}
}
```

```C#
using System;
using System.Collections.Generic;
namespace ProjectLinker
{
	[Serializable]
	class CacheDataTable
	{
		public List<CacheDataElem> elems = new List<CacheDataElem>();
		public CacheDataTable()
		{
			List<CacheDataElem> elems = new List<CacheDataElem>();
		}
	}
	[Serializable]
	class CacheDataElem
	{
		public string name;
		public string projectPath;
	}
}
```

```C#
using System;
using System.Collections.Generic;
using UnityEditor;
using UnityEditor.IMGUI.Controls;
using UnityEngine;
namespace ProjectLinker
{
	class ProjectTreeView : TreeView
	{
		private CacheDataTable _tableData;
		public event Action<int> OnSelected;
		public event Action OnDataChange;
		public ProjectTreeView(CacheDataTable table, TreeViewState state) : base(state)
		{
			_tableData = table;
			showAlternatingRowBackgrounds = false;
			ReLoad();
		}
		protected override TreeViewItem BuildRoot()
		{
			return new TreeViewItem {id = 0, depth = -1};
		}
		protected override IList<TreeViewItem> BuildRows(TreeviewItem root)
		{
			var rows = GetRows() ?? new List<TreeViewItem>(10);
			rows.Clear();
			var iconTex = EditorGUIUtility.FindTexture("Folder Icon");
			for(int i = 0; i < _tableData.elems.Count; ++i)
			{
				var item = new TreeViewItem {id = i + 1, depth = 0, displyName = _tableData.elems[i].name};
				item.icon = iconTex;
				root.AddChild(item);
				rows.Add(item);
			}
			SetupDepthsFromParentsAndChildren(root);
			return rows;
		}
		protected override void RowGUI(RowGUIArgs args)
		{
			base.RowGUI(args);
			float width = 16f;
			rect deleRect = new Rect(args.rowRect.width - width - 10, args.rowRect.y, width, width);
			Event evt = Event.current;
			if(evt.type == EventType.MouseDown && deleRect.Contains(evt.mousePosition))
				SelectionClick(args.item, false);
			if(GUI.Button(deleRect, "x"))
			{
				_tableData.elems.RemoveAt(args.item.id - 1);
				Reload();
				OnDataChange?.Invoke();
			}
		}
		protected override void SingleClickedItem(int id)
		{
			base.SingleClickedItem(id);
			int index = id - 1;
			OnSelected?.Invoke(index);
		}
	}
}
```

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
using System.IO;
using UnityEditor.IMGUI.Controls;
namespace ProjectLinker
{
	public class ProjectLinker : EditorWindow
	{
		[MenuItem("Window/Project Linker")]
		private static void Open()
		{
			var win = EditorWindow.GetWindow<ProjectLinker>("关联项目快捷启动");
			win.Show();
		}

		private const string _CACHE_TABLE_HEADER = "ProjectLinker_Table_";
		private string _copyProjectName = "";
		private string _defaultCopyProjectName = "";
		private string _curProjectPath = "";
		private string _rootPath = "";
		[SerializeField] TreeViewState _treeViewState;
		ProjectTreeView _treeView;

		CacheDataTable _cacheDataTable = null;
		int _selectedElemIndex = -1;

		CacheDataElem selectedElem
		{
			get
			{
				if(_selectedElemIndex > -1 && _selectedElemIndex < _cacheDataTable.elems.Count)
				{
					return _cacheDataTable.elems[_selectedElemIndex];
				}
				return null;
			}
		}
		private void OnEnable()
		{
			_curProjectPath = Path.GetDirectoryName(Application.dataPath);
			_rootPath = Path.GetDirectoryName(_curProjectPath);
			_defaultCopyProjectName = Path.GetFileName(_curProjectPath) + "_copy";
			_copyProjectName = _defaultCopyProjectName;
			LoadCache();

			if(_treeViewState == null)
				_treeViewState = new TreeViewState();
			_treeView = new ProjectTreeView(_cacheDataTable, _treeViewState);
			_treeView.OnSelected -= OnSelected;
			_treeView.OnSelected += OnSelected;
			_treeView.OnDataChange -= SaveCache;
			_treeView.OnDataChange += SaveCache;
		}
		private void OnSelected(int index)
		{
			_selectedElemIndex = index;
		}
		private void DrawTree()
		{
			Rect rect = GUILayoutUtility.GetRect(1000, 10000, 0, 10000);
			_treeView.OnGUI(rect);
		}
		private void DrawElemInfo()
		{
			EditorGUILayout.BeginVertical(EditorStyles.helpBox);
			GUILayout.Label("选中项目详情", EditorStyles.label);
			EditorGUILayout.BeginHorizontal(GUILayout.ExpandWidth(true));

			EditorGUILayout.BeginVertical();
			EditorGUI.BeginChangeCheck();
			selectedElem.name = EditorGUILayout.TextField("项目名称", selectedElem.name);
			EditorGUI.BeginDisabledGroup(true);
			EditorGUILayout.TextField("项目路径", selectedElem.projectPath);
			EditorGUI.EndDisabledGroup();
			if(EditorGUI.EndChangeCheck())
			{
				SaveCache();
				_treeView.Reload();
			}
			EditorGUILayout.EndVertical();

			EditorGUILayout.BeginVertical(GUILayout.Width(50));
			if(GUILayout.Button("删除"))
			{
				_cacheDataTable.elems.RemoveAt(_selectedElemIndex);
				SaveCache();
				_treeView.Reload();
			}
			if(GUILayout.Button("重置"))
			{
				selectedElem.name = Path.GetFileName(selectedElem.projectPath);
				SaveCache();
				_treeView.Reload();
			}
			EditorGUILayout.EndVertical();
			
			if(GUILayout.Button("打开", GUIlayout.Width(50), GUILayout.ExpandHeight(true)))
			{
				CmdHelper.LaunchUnityProject(selectedElem.projectPath);
				Close();
			}
			EditorGUILayout.EndHorizontal();
			EidtorGUILayout.EndVertical();
		}
		private void OnGUI()
		{
			DrawTree();
			if(selectedElem != null)
			{
				DrawElemInfo();
			}
			GUILayout.FlexibleSpace();
			DrawButton();
		}
		private void DrawButton()
		{
			EditorGUILayout.BeginHorizontal();
			if(GUILayout.Button("关联已有项目", GUILayout.Width(100), GUIlayout.ExpandHeight(true)))
			{
				var folder = EditorUtility.OpenFolderPannel("", _rootPath, "");
				if(!string.IsNullOrEmpty(folder))
				{
					folder = folder.Replace('/', '\\');
					if(folder == _curProjectPath)
					{
						Debug.LogError("不能关联当前项目本身");
					}
					else
					{
						DirectoryInfo folderInfo = new DirectoryInfo(folder);
						var childDirec = folderInfo.GetDirectories("Assets", SearchOption.TopDirectoryOnly);
						bool valide = childDirec.Lenght == 1;
						if(valid)
						{
							bool hasExisted = false;
							for(int i = 0;i < _cacheDataTable.elems.Count; ++i)
							{
								if(_cacheDataTable.elems[i].projectPath == folderInfo.FullName)
								{
									hasExisted = true;
									_selectedElemIndex = i + 1;
									_treeView.SetSelection(new List<int>{_selectedElemIndex});
									break;
								}
							}
							if(!hasExisted)
							{
								AddItem(folderInfo);
							}
						}
						else
						{
							Debug.LogError("关联的目录不是Unity项目");
						}
					}
				}
			}
			EditorGUILayout.BeginVertical(EditorStyles.helpBox);
			GUILayout.Label("当前项目复制版");
			_copyProjectName = EditorGUIlayout.TextField("项目名称:", _copyProjectName);
			EditorGUI.BeginDisabledGroup(string.IsNullOrEmpty(_copyProjectName));
			if(GUILayout.Button("创建"))
			{
				var folder = EditorUtility.OpenFolderPanel("", _rootPath, "");
				if(!string.IsNullOrEmpty(folder))
				{
					string path = Path.Combine(folder, _copyProjectName);
					if(Directory.Exists(path))
					{
						Debug.LogError("工程名已经存在");
					}
					else
					{
						DirectoryInfo folderInfo = new DirectoryInfo(path);
						try
						{
							folderInfo.Create();
							LinkProject(folderInfo);
							AddItem(folderInfo);
						}
						catch
						{
							Debug.LogError("Error");
						}
					}
				}
			}
			EditorGUI.EndDisabledGroup();
			EditorGUILayout.EndHorizontal();
			EditorGUILayout.EndHorizontal();
		}
		private void LinkProject(DirectoryInfo folderInfo)
		{
			string orgPath = Path.Combine(_curProjectPath, "Assets");
			string copyPath = Path.Combine(folderInfo.FullName, "Assets");
			CmdHelper.LinkFolder(orgPath, copyPath);
			string orgPath = Path.Combine(_curProjectPath, "ProjectSettings");
			string copyPath = Path.Combine(folderInfo.FullName, "ProjectSettings");
			CmdHelper.LinkFolder(orgPath, copyPath);
			string orgPath = Path.Combine(_curProjectPath, "Packages");
			string copyPath = Path.Combine(folderInfo.FullName, "Packages");
			CmdHelper.LinkFolder(orgPath, copyPath);
		}
		private void AddItem(DirectoryInfo folderInfo)
		{
			CacheDataElem elem = new CacheDataElem
			{
				name = folderInfo.name,
				projectPath = folderInfo.FullName
			};
			_cacheDataTable.elems.Add(elems);
			SaveCache();
			_treeView.Reload();
			_treeView.SetSelection(new List<int>{_cacheDataTable.elems.Count});
			_selectedElemIndex = _cacheDataTable.elems.Count - 1;
		}
		private void LoadCache()
		{
			var tableStr = EditorPrefs.GetString(_CACHE_TABLE_HEADER + Application.dataPath, null);
			if(string.IsNullOrEmpty(tableStr))
			{
				_cacheDataTable = new CacheDataTable();
			}
			else
			{
				_cacheDataTable = JsonUtility.FromJson<CacheDataTable>(tableStr);
			}
		}
		private void SaveCache()
		{
			var tableStr = JsonUtility.ToJson(_cacheDataTable);
			EditorPrefs.SetString(_CACHE_TABLE_HEADER + Application.dataPath, tableStr);
		}
	}
}
```