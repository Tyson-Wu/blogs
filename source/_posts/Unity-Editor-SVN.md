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

```C#
using System.Diagnostics;
namespace Svn
{
	static class SvnClientHelper
	{
		public static void OpenCommitWindow(string workDirectory)
		{
			ExecuteTortoiseClient("/command:commit ", workDirectory);
		}
		public static void OpenUpdateWindow(string workDirectory)
		{
			ExecuteTortoiseClient("/command:update ", workDirectory);
		}
		public static void OpenRevertWindow(string workDirectory)
		{
			ExecuteTortoiseClient("/command:revert ", workDirectory);
		}
	    private static void ExecuteTortoiseClient(string cmd, string workDirectory)
		{
			System.Threading.ThreadPool.QueueUserWorkItem(delegate(object state))
			{
				Process p = null;
				try
				{
					ProcessStartInfo start = new ProcessStartInfo("TortoiseProc.exe");
					start.Arguments = cmd + "/path:\"" + workDirectory + "\"";
					p = Process.Start(start);
				}
				catch(System.Exception e)
				{
					UnityEngine.Debug.LogException(e);
					if(p!=null) p.Close();
				}
			});
		}
	}
}
```

```C#
//https://svnbook.red-bean.com/en/1.6/svn.advanced.changelists.html
//https://tortoisesvn.net/docs/release/TortoiseSVN_en/tsvn-cli-main.html#tsvn-cli-addignore
//https://blog.stone-head.org/svn-changelist/
//https://www.visualsvn.com/support/svnbook/ref/svn/#svn.ref.svn.sw.targets
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Text.RegularExpressions;
using UnityEndgine;
using System.Text;
using Debug = UnityEngine.Debug;
namesapce Svn
{
	public static class SvnCmdHelper
	{
		enum TagType
		{
			Added,
			UnTrack,
			Modified,
			Deleted
		}
		static string[] _tags = new string[] {"A", "?", "M", "!"};
		private const string _defaultSvnGroupName = "_default_";

		static string _tempFilePath = null;
		static strng _tempFileName = "_svnTemp.txt";
		static string GetTempFilePath();
		{
			if(_tempFilePath = null)
			{
				_tempFilePath = Path.Combine(Application.temporaryCachePath, _tempFileName);
			}
			return _tempFilePath;
		}
		static StreamReader OpenText()
		{
			//https://www.open.collab.net/scdocs/SVNEncoding.html
			FileInfo fileInfo = new FileInfo(GetTempFilePath());
			return new StreamReader(fileInfo.OpenRead(), Encoding.GetEncoding("gb2312"));
		}
		static StreamWriter CreateText()
		{
			FileInfo fileInfo = new FileInfo(GetTempFilePath());
			return new StreamWriter(fileInfo.Create(), Encoding.GetEncoding("gb2312"));
		}
		static void CreateProcessStart(string workDirectory)
		{
			ProcessStartInfo start = new ProcessStartInfo("cmd.exe");
			start.CreateNoWindow = true;
			start.ErrorDialog = true;
			start.UseShellExecute = false;
			start.WorkingDirectory = workDirectory;

			start.RedirectStandardOutput = true;
			start.RedirectStandardError = true;
			start.RedirectStandardInput = true;
			start.StandardOutputEncoding = Endcoding.GetEncoding("GBK");
			start.StandardErrorEncoding = Encoding.GetEncoding("GBK");
			return start;
		}
		public static string GetSvnVertsion()
		{
			string workDirectory = Path.GetDirectoryName(Application.dataPath);
			ProcessStartInfo start = CreateProcessStart(workDirectory);
			Process process = Process.Start(start);
			using(var sw = process.StandardInput)
			{
				sw.WriteLine("svn --version");
			}
			using(var sr = process.StandardError)
			{
				string line;
				do
				{
					line = sr.ReadLine();
					if(!string.IsNullOrEmpty(line)) return null;
				}
				while(line != null);
			}
			string rst = "svn";
			using(var sr = process.StandardOutput)
			{
				string line;
				const string vStart = "svn, version";
				do
				{
					line = sr.ReadLine();
					if(line.StartsWith(vStart))
					{
						rst = line.Sbustring(vStart.Length);
						break;
					}
				}
				while(line != null);
			}
			process.Close();
			return rst;
		}
		public static string GetSvnWorkDir()
		{
			string workDirectory = Path.GetDirectoryName(Application.dataPath);
			ProcessStartInfo start = CreateProcessStart(workDirectory);
			Process process = Process.Start(start);
			using(var sw = process.StandardInput)
			{
				sw.WriteLine("svn info");
			}
			using(var sr = process.StandardError)
			{
				string line;
				do
				{
					line = sr.ReadLine();
					if(!string.IsNullOrEmpty(line)) return null;
				}
				while(line != null);
			}
			string rst = "svn";
			using(var sr = process.StandardOutput)
			{
				string line;
				const string vStart = "Working Copy Root Path: ";
				do
				{
					line = sr.ReadLine();
					if(line.StartsWith(vStart))
					{
						rst = line.Sbustring(vStart.Length);
						break;
					}
				}
				while(line != null);
			}
		}
		static void FillEmptyStatus(Dictionary<string, Dictionary<string, List<string>>> status)
		{
			if(!status.ContainsKey(_defaultSvnGroupName))
				status[_defaultSvnGroupName] = new Dictionary<string, List<string>>();
			foreach(var kv in status)
			{
				for(int i = 0; i < _tags.Length; ++i)
				{
					if(!kv.Value.ContainsKey(_tags[i]))
						kv.Value[_tags[i]] = new List<string>();
				}
			}
		}
		public static Dictionary<string, Dictionary<string, List<string>>> GetStatus()
		{
			string workDirectory = Path.GetDirectoryName(Application.dataPath);
			ProcessStartInfo start = CreateProcessStart(workDirectory);
			Process process = Process.Start(start);
			using(var sw = process.StandardInput)
			{
				sw.WriteLine("svn status");
			}
			Dictionary<string, Dictionary<string, List<string>>> rst = new Dictionary<string, Dictionary<string, List<string>>>();
			Dictionary<string, List<string>> levOne = null;
			List<string> levTow = null;
			Regex stateRegex = new Regex(@"^([M?A!])\s*(.*)$");
			Regex stateRegex2 = new Regex(@"--- Changelist\s*'(.*)':$");
			int state = 0;
			try
			{
				using(var sr = process.StandardError)
				{
					string line;
					do
					{
						line = sr.ReadLine();
						if(state == 0)
						{
							if(line.StartsWith(workDirectory))
							{
								state = 1;
								levOne = new Dictionary<string, List<string>>();
								rst[_defaultSvnGroupName] = levOne;
							}
						}
						else if(state == 1)
						{
							if(!string.IsNullOrEmpty(line))
							{
								var match = stateRegex.Match(line);
								if(match.Groups.Count > 2)
								{
									string key = match.Groups[1].ToString();
									string value = match.Groups[2].ToString();
									string dir = value.Replace("\\", "/");
									if(!levOne.ContainsKey(key))
									{
										levTow = new List<string>();
										levOne[key] = levTow;
									}
									else
									{
										levTow = LevOne[key];
									}
									levTow.Add(dir);
								}
							}
							else
							{
								state = 2;
							}
						}
						else if(state == 2)
						{
							if(!string.IsNullOrEmpty(line))
							{
								var match = stateRegex2.Match(line);
								if(match.Groups.Count > 1)
								{
									string value = match.Groups[1].ToString();
									state = 1;
									levOne = new Dictionary<string, List<string>>();
									rst[value] = levOne;
								}
							}
						}
					}
					while(line != null);
				}
			}
			finally
			{
				process.Close();
				FillEmptyStatus(rst);
				return rst;
			}
		}
		public static void SetChangelistForFilterData(SvnFilterData data)
		{
			int ignoreListLimitCount = 10000;
			var workDir = GetSvnWrokDir();
			var status = GetStatus(workDir);
			ClearAllChangeList(workDir, ref status);
			AddAllUnTracked(status, workDir);
			List<List<string>> ignoreList = new List<List<string>>();
			List<string> list;
			foreach(var kv in status)
			{
				if(kv.Key == "ignore-on-commit") continue;
				list = kv.Value[_tags[(int)TagType.Added]];
				Filter(ignoreListLimitCount, list, data.ignorePatternAdd, data.ignoreFilesAdd, data.ignoreFoldersAdd, ref ignoreList);
				list = kv.Value[_tags[(int)TagType.Deleted]];
				Filter(ignoreListLimitCount, list, data.ignorePatternDelete, data.ignoreFilesDelete, data.ignoreFoldersDelete, ref ignoreList);
				list = kv.Value[_tags[(int)TagType.Modified]];
				Filter(ignoreListLimitCount, list, data.ignorePatternModify, data.ignoreFilesModify, data.ignoreFoldersModify, ref ignoreList);
			}
			foreach(var v in ignoreList)
			{
				if(v.Count > 0)
					AddChangeList("ignore-on-commit", v, workDir, ref status);
			}
		}
		static void Filter(int limitCount, List<string> list, List<string> ignorePattern, List<string> ignoreFiles, List<string> ignoreFolders, ref List<List<string>> outList)
		{
			if(outList == null) outList = new List<List<string>>();
			List<string> temp = null;
			if(outList.Count > 0)
			{
				temp = outList[outList.Count - 1];
				if(temp.Count >= limitCount) temp =null;
			}
			if(temp == null)
			{
				temp = new List<string>();
				outList.Add(temp);
			}
			bool find = false;
			Regex stateRegex = new Regex(@"(\.[^\.]+)+$");
			foreach(var v in list)
			{
				find = false;
				if(!find)
				{
					foreach(var subFix in ignorePattern)
					{
						var match = stateRegex.Match(v);
						if(match.Success)
						{
							var group = match.Groups[match.Groups.Count - 1]
							int caCount = group.Captures.Count;
							for(int k = 0; k <caCount; ++k)
							{
								if(group.Captures[k].Value == subFix)
								{
									find = true;
									break;
								}
							}
							if(find) break;
						}
					}
				}
				if(!find)
				{
					foreach(var subFix in ignoreFiles)
					{
						if(v == subFix)
						{
							find = true;
							break;
						}
					}
				}
				if(!find)
				{
					foreach(var subFix in ignoreFolders)
					{
						if(v.StartsWith(subFix))
						{
							find = true;
							break;
						}
					}
				}
				if(find)
				{
					//https://jeremy.hu/svn-at-character-peg-revision-is-not-allowed-here/
					string path = v;
					if(path.Contains("@"))
						path = string.Format("{0}@", path);
					if(temp.COunt < limitCount)
					{
						temp.Add(path);
					}
					else
					{
						temp = new List<string>();
						outList.Add(temp);
						temp.Add(path);
					}
				}
			}
		}
		static void AddAllUnTracked(Dictionary<string, Dictionary<string, List<string>>> status, string wrokDir)
		{
			var defaultGroup = status[_defaultSvnGroupName];
			var newGroup = defaultGroup["?"];
			if(newGroup.Count == 0) return;
			List<string> files = new List<string>();
			foreach(var path in newGroup)
			{
				if(Directory.Exists(string.Format("{0}/{1}", wrokDir, path)))
				{
					files.Add(path);
				}
				else
				{
					files.Add(path);
				}
			}
			if(files.Count > 0)
			{
				newGroup.Clear();
				AddFile(files, workDir, ref status);
			}
		}
		public static void AddChangeList(string changelist, List<string> paths, string workDir, ref Dictionary<string, Dictionary<string, List<string>>> rst)
		{
			ProcessStartInfo start = CreateProcessStart(workDir);
			Process process = Process.Start(start);
			using(var sw = process.StandardInput)
			{
				using(var write = CreateText())
				{
					for(int i = 0; i<paths.Count; ++i)
					{
						write.WriteLine(paths[i]);
					}
				}
				sw.WriteLine(string.Format("svn changelist {0} --targets {1} -R", changelist, GetTempFilePath());
			}
			try
			{
				using(var sr = process.StandardOutput)
				{
					string line = sr.ReadToEnd();
				}
			}
			finally
			{
				process.Close();
			}
		}
		public static void ClearAllChangeList(string workDir, ref Dictionary<string, Dictionary<string, List<string>>> rst)
		{
			ProcessStartInfo start = CreateProcessStart(workDir);
			Process process = Process.Start(start);
			List<string> changelist = new List<string>();
			var defaultList = rst[_defaultSvnGroupName];
			foreach(var kv in rst)
			{
				if(_defaultSvnGroupName != kv.Key)
				{
					changelist.Add(kv.Key);
					for(int i = 0; i<_tags.Length; ++i)
					{
						defaultList[_tags[i]].AddRange(kv.Value[_tags[i]]);
						kv.Value[_tags[i]].Clear();
					}
				}
			}
			rst.Clear();
			rst.Add(_defaultSvnGroupName, defaultList);
			using(var sw = process.StandardInput)
			{
				for(int i= 0;i< changelist.Count; ++i)
				{
					sw.WriteLine(string.Format("svn changelist --remove --changelist {0} --depth infinity .", changelist[i]);
				}
			}
			try
			{
				using(var sr = process.StandardOutput)
				{
					string line = sr.ReadToEnd();
				}
			}
			finally
			{
				process.Close();
			}
		}
		public static void AddFile(List<string> filePath, string workDir, ref Dictionary<string, Dictionary<string, List<string>>> rst)
		{
			ProcessStartInfo start = CreateProcessStart(workDir);
			Process process = Process.Start(start);
			using(var write = CreateText())
			{
				for(int i= 0;i<filePath.Count; ++i)
				{
					write.WriteLine(filePath[i]);
				}
			}
			using(var sw = process.StandardInput)
			{
				sw.WriteLine(string.Format("svn add --targets {0} --depth infinity", GetTempFilePath()));
			}
			Dictionary<string, List<string>> LevOne = rst[_defaultSvnGroupName];
			List<string> levTow = null;
			Regex stateRegex = new Regex(@"^(A)(\s\(bin\))?\s*(.*)$");
			try
			{
				using(var sr = process.StandardOutput)
				{
					string line;
					do
					{
						line = sr.ReadLine();
						if(!string.IsNullOrEmpty(line))
						{
							var math = stateRegetx.Match(line):
							if(match.Groups.Count > 2)
							{
								string key = match.Groups[1].ToString();
								string value = match.Groups[match.Groups.Count - 1].ToString();
								string dir = value.Replace("\\","/");
								if(!levOne.ContainsKey(key))
								{
									levTow = new List<string>();
									levOne[key] = levTow;
								}
								else
								{
									levTow = levOne[key];
								}
								levTow.Add(dir);
							}
						}
					}
					while(line != null);
				}
			}
			finally
			{
				process.Close();
			}
		}
	}
}
```

```C#
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;
namespace Svn
{
	[System.Serializable]
	public class SvnFilterData
	{
		public int careerIndex = 0;
		public List<string> ignorePatternModify = new List<string>();
		public List<string> ignoreFilesModify = new List<string>();
		public List<string> ignoreFoldersModify = new List<string>();
		public List<string> ignorePatternAdd = new List<string>();
		public List<string> ignoreFilesAdd = new List<string>();
		public List<string> ignoreFoldersAdd = new List<string>();
		public List<string> ignorePatternDelete = new List<string>();
		public List<string> ignoreFilesDelete = new List<string>();
		public List<string> ignoreFoldersDelete = new List<string>();
		public SvnFilterData()
		{
			careerIndex = 0;
			ignorePatternModify = new List<string>();
			ignoreFilesModify = new List<string>();
			ignoreFoldersModify = new List<string>();
			ignorePatternAdd = new List<string>();
			ignoreFilesAdd = new List<string>();
			ignoreFoldersAdd = new List<string>();
			ignorePatternDelete = new List<string>();
			ignoreFilesDelete = new List<string>();
			ignoreFoldersDelete = new List<string>();
		}
		public static void Save(SvnFilterData data)
		{
			string jsonStr = JsonUtility.ToJson(data);
			EditorPrefs.SetString(SaveKey, jsonStr);
		}
		public static bool Load(out SvnFilterData data)
		{
			string jsonStr = EditorPrefs.GetString(SaveKey, null);
			bool hasData = !string.IsNullOrEmpty(jsonStr);
			if(hasData)
			{
				data = JsonUtility.FromJson<SvnFilterData>(jsonStr);
			}
			else
				data = new SvnFilterData();
			return hasData;
		}
		public static string SaveKey
		{
			get {return string.Format("{0}_svnFilterData", Application.dataPath);}
		}
	}
}
```

```C#
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
using System.Text;
namespace Svn
{
	public class SVNSettingWindow : EditorWindow
	{
		const string _MenuPath = "Svn";
		const string _tip = "";
		string[] _career = {"Artist", "Tester", "Programmer", "Designer"};
		GUIContent[] _tabContents = new GUIContent[]{
			new GUIContent("Artist"),
			new GUIContent("Tester"),
			new GUIContent("Programmer"),
			new GUIContent("Designer"),
		};
		[MenuItem(_MenuPath + "Update", priority = 0, validate = false)]
		public static void UpdateSVN()
		{
			SvnClientHelper.OpenUpdateWindow(SvnCmdHelper.GetSvnWorkDir());
		}
		[MenuItem(_MenuPath + "Revert", priority = 0, validate = false)]
		public static void RevertSVN()
		{
			SvnClientHelper.OpenRevertWindow(SvnCmdHelper.GetSvnWorkDir());
		}
		[MenuItem(_MenuPath + "CommitAll", priority = 0, validate = false)]
		public static void CommitAllSVN()
		{
			SvnClientHelper.OpenCommitWindow(SvnCmdHelper.GetSvnWorkDir());
		}
		[MenuItem(_MenuPath + "CommitAllWithFilter", priority = 0, validate = true)]
		public static bool CommitAllWithFilterValidate()
		{
			if(string.IsNullOrEmpty(SvnCmdHelper.GetSvnVerison())) return false;
			if(string.IsNullOrEmpty(SvnCmdHelper.GetSvnWorkDir())) return false;
			return true;
		}
		[MenuItem(_MenuPath + "CommitAllWithFilter", priority = 0, validate = false)]
		public static void CommitAllWithFilter()
		{
			if(SvnFilterData.Load(out var data))
			{
				SvnCmdHelper.SetChangelistForFilterData(data);
				SvnClientHelper.OpenCommitWindow(SvnCmdHelper.GetSvnWorkDir());
			}
			else
				Open();
		}
		[MenuItem(_MenuPath + "Settings", false, 1000)]
		public static void Open()
		{
			SVNSettingWindow window = GetWindow<SVNSettingWindow>("Setting");
			window._svnVersion = SvnCmdHelper.GetSvnVersion();
			window._workDir = SvnCmdHelper.GetSvnWorkDir();
			if(!string.IsNullOrEmpty(window._workDir))
				window._workDir = window._workDir.Replace("\\","/");
			window.minSize = new Vector2(520, 520);
			window.InitConfigData();
			window.Show();
		}

		[SerializeField] private string _svnVersion = null;
		[SerializeField] private string _workDir = null;

		[SerializeField] private int careerSelectedIdx = -1;
		[SerializeField] private string ignorePatternModify = "";
		[SerializeField] private List<string> ignoreFilesModify = new List<string>();
		[SerializeField] private List<string> ignoreFoldersModify = new List<string>();
		[SerializeField] private string ignorePatternAdd = "";
		[SerializeField] private List<string> ignoreFilesAdd = new List<string>();
		[SerializeField] private List<string> ignoreFoldersAdd = new List<string>();
		[SerializeField] private string ignorePatternDelete = "";
		[SerializeField] private List<string> ignoreFilesDelete = new List<string>();
		[SerializeField] private List<string> ignoreFoldersDelete = new List<string>();

		[SerializeField] private Vector2 scrollPos = Vector2.zero;
		private GUIContent content = new GUIContent();
		private StringBuilder stringBuilder = new StringBuilder();
		private GUIContent GetGUIContent(string text, Texture image, string tooltip)
		{
			content.text = text;
			content.image = image;
			content.tooltip = tooltip;
			return content;
		}
		private void InitConfigData()
		{
			if(SvnFilterData.Load(out var data))
			{
				careerSelectedIdx = data.careerIndex;
				AddWorkDir(_workDir, data.ignoreFilesModify, ref ignoreFilesModify);
				AddWorkDir(_workDir, data.ignoreFoldersModify, ref ignoreFoldersModify);
				PatternCombine(data.ignorePatternModify, ref ignorePatternModify);
				AddWorkDir(_workDir, data.ignoreFilesAdd, ref ignoreFilesAdd);
				AddWorkDir(_workDir, data.ignoreFoldersAdd, ref ignoreFoldersAdd);
				PatternCombine(data.ignorePatternAdd, ref ignorePatternAdd);
				AddWorkDir(_workDir, data.ignoreFilesDelete, ref ignoreFilesDelete);
				AddWorkDir(_workDir, data.ignoreFoldersDelete, ref ignoreFoldersDelete);
				PatternCombine(data.ignorePatternDelete, ref ignorePatternDelete);
			}
		}
		void PatternSplit(string pattern, ref List<string> patternList)
		{
			patternList.Clear();
			var slpits = pattern.Split(' ");
			foreach(var v in splits)
			{
				if(string.IsNullOrEmpty(v))
					patternList.Add(v);
			}
		}
		var PatternCombine(List<string> patternList, ref string pattern)
		{
			StringBuilder stringBuilder = new StringBuilder();
			const string space = " ";
			foreach(var v in patternList)
			{
				stringBuilder.Append(v);
				stringBuilder.Append(space);
			}
			pattern = stringBuilder.ToString();
		}
		void RemoveWorkDir(string workDir, List<string> org, ref List<string> dst)
		{
			dst.Clear();
			workDir = string.Format("{0}/", workDir);
			workDir = workDir.Replace("\\","/");
			for(int i = 0;i<org.Count; ++i)
			{
				dst.Add(org[i].Replace("\\", "/").Replace(workDir, ""));
			}
		}
		void AddWorkDir(string workDir, List<string> org, ref List<string> dst)
		{
			dst.Clear();
			for(int i = 0;i<org.Count; ++i)
			{
				if(org[i].StartsWith(workDir))
					dst.Add(org[i]);
				else
					dst.Add(string.Format("{0}/{1}", workDir, org[i]));
			}
		}
		private void OnGUI()
		{
			bool show = false;
			EditorGUILayout.LabelField("Base Info");
			if(_svnVersion == null)
			{
				EditorGUILayout.HelpBox("never find svn!", MessageType.Error, true);
			}
			else if(_workDir == null)
			{
				EditorGUI.indentLevel += 1;
				GUI.enabled = false;
				EditorGUILayout.TextField("svn version:", _svnVersion);
				GUI.enabled = true;
				EditorGUI.indentLevel -= 1;
				EditorGUILayout.HelpBox("current project not in svn control", MessageType.Warning, true);
			}
			else
			{
				GUI.enabled = false;
				EditorGUI.indentLevel += 1;
				EditorGUILayout.TextField("svn version:", _svnVersion);
				EditorGUILayout.TextField("workDir:", _workDir);
				EditorGUI.indentLevel -= 1;
				GUILayout.Space(10);
				GUI.enabled = true;
				show = true;
			}
			GUI.enabled = show;
			DrawBody();
			GUI.enabled = true;
		}
		void DrawBody()
		{
			EditorGUILayout.HelpBox(_tip, MessageType.Info, true);
			EditorGUI.BeginChangeCheck();
			careerSelectedIdx = GUILayout.Toolbar(careerSelectedIdx, _tabContents);
			if(EditorGUI.EndChangeCheck())
			{
				PresetConfig(careerSelectedIdx);
				EditorGUIUtility.editingTextField = false;
			}

			GUI.enbaled = careerSelectedIdx != -1;
			GUILayout.BeginHorizontal();
			EditorGUILayout.LabelField("Ignore Setting");
			if(GUILayout.Button(GetGUIContent("save", nul, "save"), GUILayout.Width(50)))
			{
				SaveConfig(careerSelectedIdx);
			}
			GUILayout.EndHorizontal();

			scrollPos = GUILayout.BeginScrollView(scrollPos, EditorStyles.helpBox);
			Panel("Modify Ignore", ref ignorePatternModify, ref ignoreFilesModify, ref ignoreFoldersModify);
			Panel("Add Ignore", ref ignorePatternAdd, ref ignoreFilesAdd, ref ignoreFoldersAdd);
			Panel("Delete Ignore", ref ignorePatternDelete, ref ignoreFilesDelete, ref ignoreFoldersDelete);
			GUILayout.EndScrollView();
			GUILayout.Space(10);
		}
		private void Panel(string name, ref string ignPat, ref List<string> ignFiles, ref List<string> ignFolders)
		{
			GUILayout.Space(10);
			EditorGUILayout.LabelField(name);
			EditorGUI.indentLevel += 1;
			ignPat = EditorGUILayout.TextField(GetGUIContent("Ignore Pattern:", null, ""), ignPat);

			EditorGUILayout.LabelField("Ignore Files");
			EditorGUI.indentLevel += 1;
			int removeIdx = -1;
			for(int i = 0;i<ignFiles.Count; ++i)
			{
				EditorGUILayout.BeginHorizontal();
				GUI.enabled = false;
				EditorGUILayout.TextTield(GetGUIContent("Ignore Files:", null, ""), ignFiles[i]);
				GUI.enabled = true;
				if(GUILayout.Button("-", GUILayout.Width(20)))
					removeIdx = i;
				EditorGUILayout.EndHorizontal();
			}
			if(removeIdx != -1)
			{
				ignFiles.RemoveAt(removeIdx);
			}
			EditorGUILayout.BeginHorizontal();
			GUILayout.Label("", GUILayout.Width(24));
			if(GUILayout.Button(GetGUIContent("Add File", null, "")))
			{
				string file = EditorUtility.OpenFilePanel("Select File", _workDir, null);
				if(file.Contens(_workDir) &&!ignFiles.Contents(file))
					ignFiles.Add(file);
			}
			EditorGUILayout.EndHorizontal();
			EditorGUI.indentLevel -= 1;

			EditorGUILayout.LabelField("Ignore Folders");
			EditorGUI.indentLevel += 1;
		 	removeIdx = -1;
			for(int i = 0;i<ignFolders.Count; ++i)
			{
				EditorGUILayout.BeginHorizontal();
				GUI.enabled = false;
				EditorGUILayout.TextTield(GetGUIContent("Ignore Folders:", null, ""), ignFolders[i]);
				GUI.enabled = true;
				if(GUILayout.Button("-", GUILayout.Width(20)))
					removeIdx = i;
				EditorGUILayout.EndHorizontal();
			}
			if(removeIdx != -1)
			{
				ignFolders.RemoveAt(removeIdx);
			}
			EditorGUILayout.BeginHorizontal();
			GUILayout.Label("", GUILayout.Width(24));
			if(GUILayout.Button(GetGUIContent("Add Folder", null, "")))
			{
				string file = EditorUtility.OpenFolderPanel("Select Folder", _workDir, null);
				if(file.Contens(_workDir) &&!ignFolders.Contents(file))
					ignFolders.Add(file);
			}
			EditorGUILayout.EndHorizontal();
			EditorGUI.indentLevel -= 1;
		}

		private void PresetConfig(int careerIndex)
		{
			string career = _careers[careerIndex];
			stringBuilder.Clear();
			stringBuilder.Append(defaultPattern);
			switch(career)
			{
				case "Artist":
					break;
				case "Tester":
				case "Programmer":
				case "Designer":
					{
						stringBuilder.Append(".mat .anim .png .jpg");
					}
			}
			ignorePatternModify = stringBuilder.ToString();
			ignoreFilesModify.Clear();
			ignoreFoldersModify.Clear();
			ignorePatternAdd = stringBuilder.ToString();
			ignoreFilesAdd.Clear();
			ignoreFoldersAdd.Clear();
			ignorePatternDelete = stringBuilder.ToString();
			ignoreFilesDelete.Clear();
			ignoreFoldersDelete.Clear();
		}

		private void SaveConfig(int careerIndex)
		{
			SvnFilterData data = new SvnFilterData();
			data.careerIndex = careerIndex;
			RemoveWorkDir(_workDir, ignoreFilesModify, ref data.ignoreFilesModify);
			RemoveWorkDir(_workDir, ignoreFoldersModify, ref data.ignoreFoldersModify);
			PatternSplit(ignorePatternModify, ref data.ignorePatternModify);
			RemoveWorkDir(_workDir, ignoreFilesAdd, ref data.ignoreFilesAdd);
			RemoveWorkDir(_workDir, ignoreFoldersAdd, ref data.ignoreFoldersAdd);
			PatternSplit(ignorePatternAdd, ref data.ignorePatternAdd);
			RemoveWorkDir(_workDir, ignoreFilesDelete, ref data.ignoreFilesDelete);
			RemoveWorkDir(_workDir, ignoreFoldersDelete, ref data.ignoreFoldersDelete);
			PatternSplit(ignorePatternDelete, ref data.ignorePatternDelete);
		}
	}
}
```