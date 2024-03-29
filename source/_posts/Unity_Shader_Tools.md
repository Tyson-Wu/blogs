---
title: Shader Tools
date: 2022-12-29 10:01:00
categories:
- [Unity]
- [Shader]
tags:
- Unity
- Shader
---

## 源码

```C#
// ShaderOpenTools.cs
using UnityEngine;
using UnityEditor;
using UnityEditor.Callbacks;
using System.Diagnostics;
using System.Text;
using UnityEditor.ProjectWindowCallback;
public partial class ShaderOpenTools : EditorWindow
{
    const string k_WorkDirectoryKey = "Shader_WorkDirectoryKey";
    static string m_keyPath = null;
    static string keyPath{
        get{
            if(m_keyPath == null)
                m_keyPath = $"{Application.dataPath}_{k_WorkDirectoryKey}";
            return m_keyPath;
        }
    }
    static string workDirectory
    {
        get
        {
            string path = EditorPrefs.GetString(keyPath, null);
            if(string.IsNullOrEmpty(path))
                return Application.dataPath;
            return path;
        }
        set
        {
            EditorPrefs.SetString(keyPath, value);
        }
    }
    [MenuItem("Tools/ShaderOpenTools")]
    static void CreateWindow()
    {
        Rect rect = new Rect();
        if(SceneView.lastActiveSceneView != null)
            rect = SceneView.lastActiveSceneView.position;
        rect.size = new Vector2(400, 200);
        rect.position = rect.position + new Vector2(10, 10);
        ShaderOpenTools win = GetWindowWithRect<ShaderOpenTools>(rect, true);
        win.titleContent = new GUIContent("ShaderOpenTools Setting");
        win.Show();
    }

    string path = null;
    void OnEnable()
    {
        path = workDirectory;
    }
    void OnGUI()
    {
        EditorGUILayout.BeginHorizontal();
        EditorGUILayout.LabelField(path);
        if(GUILaout.Button("o", GUILayout.Width(30)))
        {
            string temp = path;
            temp = EditorUtility.OpenFolderPanel("select shader project", temp, Application.dataPath);
            if(!string.IsNullOrEmpty(temp) && path != temp)
            {
                path = temp;
                workDirectory = path;
            }
        }
        EditorGUILayout.EndHorizontal();

        GUILayout.FlexibleSpace();
        if(GUILayout.Button("Open Project"))
        {
            OpenProject();
        }
    }
}

public partial class ShaderOpenTools
{
    static ProcessStartInfo CreateProcessStartInfo(string rootDirectory)
    {
        ProcessStartInfo start = new ProcessStartInfo("cmd.exe");
        start.CreateNoWindow = true;
        start.ErrorDialog = true;
        start.UseShellExecute = false;
        start.WorkingDirectory = rootDirectory;

        start.RedirectStandardOutput = true;
        start.RedirectStandardError = true;
        start.RedirectStandardInput = true;
        start.StandardOutputEncoding = Encoding.GetEncoding("GBK");
        start.StandardErrorEncoding = Encoding.GetEncoding("GBK");
        return start;
    }
    [OnOpenAsset(-100)]
    static bool OnShaderClick(int instanceID, int line)
    {
        string strFilePath = AssetDatabase.GetAssetPath(EditorUtility.InstanceIDToObject(instanceID));
        if(strFilePath.EndsWith(".shader")
        || strFilePath.EndsWith(".hlsl")
        || strFilePath.EndsWith(".cginc"))
        {
            string rootDirectory = workDirectory;
            string strFileName = Path.GetFullPath(strFilePath);
            ProcessStartInfo start = CreateProcessStartInfo(rootDirectory);
            Process process = Process.Start(start);
            using(var sw = process.StandardInput)
            {
                if(line > 0)
                {
                    sw.WriteLine($"code . && code -g \"{strFileName}\":{line} -r");
                }
                else
                {
                    sw.WriteLine($"code . && code \"{strFileName}\" -r");
                }
            }
            bool rst = true;
            using(var sr = process.StandardError)
            {
                string error;
                do
                {
                    error = sr.ReadLine();
                    if(!string.IsNullOrEmpty(error))
                    {
                        rst = false;
                    }
                }
                while(error != null);
            }
            process.Close();
            return rst;
        }
        return false;
    }
    static void OpenProject()
    {
        string rootDirectory = workDirectory;
        ProcessStartInfo start = CreateProcessStartInfo(rootDirectory);
        Process process = Process.Start(start);
        using(var sw = process.StandardInput)
        {
            sw.WroteLine($"code .");
        }
        using(var sr = process.StandardError)
        {
            string error;
            do
            {
                error = sr.ReadLine();
            }
            while(error!=null);
        }
        process.Close();
    }
}

public partial class ShaderOpenTools
{
    [SuppressMessage("Microsft.Performance", "CA1812")]
    internal class CreateHLSLAsset : EndNameEditAction
    {
        public override void Action(int instanceId, string pathName, string resourceFile)
        {
            FileInfo fileInfo = new FileInfo(pathName);
            string fileName = Path.GetFileNameWithoutExtension(pathName);
            using(var stream = fileInfo.CreateText())
            {
                string define = ObjectNames.NicifyVariableName(fileName)
                .Replace(" ", "_").ToUpper();
                stream.WriteLine($"#ifndef _{define}_INCLUDE");
                stream.WriteLine($"#define _{define}_INCLUDE");
                stream.WriteLine("\t");
                stream.WriteLine($"#endif //_{define}_INCLUDE");
            }
            AssetDatabase.Refresh();
        }
    }
    [MenuItem("Assets/Create/Shader/HLSL")]
    static void CreateHLSL()
    {
        ProjectWindowUtil.StartNameEditingIfProjectWindowExists(
            0,CreateInstance<CreateHLSLAsset>(), "NewHLSL.hlsl", null, null);
    }
}
```

## 参考

[VSCode Command line](https://code.visualstudio.com/docs/editor/command-line)
[VSCode settings](https://code.visualstudio.com/docs/getstarted/settings)
[VSCode GlobPattern](https://vshaxe.github.io/vscode-extern/vscode/GlobPattern.html):可以用于设置VSCode不显示`.meta`文件，在`Settings`界面，搜索`file:exclude`，然后增加`**/*.meta`。


## 使用配置文件

上面是通过路径打开VSCode文件，更实用的一种方法是直接通过配置文件打开。通过配置文件打开有几个好处，就是可以在同一个VSCode里面打开多个文件路径，并且保存设置。配置文件格式如下,文件名为`shaderworkspace.code-workspace`：
```json
{
    "folder":[
        {"path":"folder\\path1"},
        {"path":"folder\\path2"}
    ],
    "settings":{
        "files.exclude":{
            "**/*.meta":true
        }
    }
}
```

然后前面打开VSCode工程的命令可以改成：
```C#
if(line > 0)
{
    sw.WriteLine($"code shaderworkspace.code-workspace && code -g \"{strFileName}\":{line} -r");
}
else
{
    sw.WriteLine($"code shaderworkspace.code-workspace && code \"{strFileName}\" -r");
}
```

当然，上面命令有效的前提是`shaderworkspace.code-workspace`文件在当前工作目录下。