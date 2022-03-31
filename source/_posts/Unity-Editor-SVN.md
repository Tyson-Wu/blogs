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