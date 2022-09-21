---
title: Compiler Options
date: 2022-09-21 10:01:00
categories:
- [Unity, Editor]
tags:
- Unity
- Editor
- Compiler
---

## 功能
在用Unity开发项目时，每次编写或者修改C#脚本，Unity都会自动检查脚本，并进行自动编译。频繁的自动编译，无疑是会降低开发效率，基于这个出发点，我希望能够关闭自动编译功能。在Unity下有两个设置是和自动编译相关的，就是Preferences界面下的General里面的`Auto Refresh`和`Script Changes While Playing`。前者是自动刷新，每次刷新的时候就会检查脚本是否需要编译，这时候我们可以把自动刷新关掉，然后使用手动的方式进行刷新。后者主要是控制运行时脚本自动编译的时机，是否在运行时进行自动编译。
![wwwwwww](/blogs/images/src/Snipaste_2022-09-21_21-45-55.png)
上图中的设置项的位置可能会因Unity的版本不同而有所改变，这里我用的Unity版本是`Unity 2020.3.33f1c2 Personal`
这里写了一个编辑器脚本，将上面的设置暴露在菜单栏中。
```C#
using UnityEditor;

public class ScriptOptions
{
    //Auto Refresh


    //kAutoRefresh has two posible values
    //0 = Auto Refresh Disabled
    //1 = Auto Refresh Enabled


    //This is called when you click on the 'Tools/Auto Refresh' and toggles its value
    [MenuItem("Tools/Auto Refresh")]
    static void AutoRefreshToggle()
    {
        var status = EditorPrefs.GetInt("kAutoRefresh");
        if (status == 1)
            EditorPrefs.SetInt("kAutoRefresh", 0);
        else
            EditorPrefs.SetInt("kAutoRefresh", 1);
    }


    //This is called before 'Tools/Auto Refresh' is shown to check the current value
    //of kAutoRefresh and update the checkmark
    [MenuItem("Tools/Auto Refresh", true)]
    static bool AutoRefreshToggleValidation()
    {
        var status = EditorPrefs.GetInt("kAutoRefresh");
        if (status == 1)
            Menu.SetChecked("Tools/Auto Refresh", true);
        else
            Menu.SetChecked("Tools/Auto Refresh", false);
        return true;
    }


    //Script Compilation During Play


    //ScriptCompilationDuringPlay has three posible values
    //0 = Recompile And Continue Playing
    //1 = Recompile After Finished Playing
    //2 = Stop Playing And Recompile


    //The following methods assing the three possible values to ScriptCompilationDuringPlay
    //depending on the option you selected
    [MenuItem("Tools/Script Compilation During Play/Recompile And Continue Playing")]
    static void ScriptCompilationToggleOption0()
    {
        EditorPrefs.SetInt("ScriptCompilationDuringPlay", 0);
    }


    [MenuItem("Tools/Script Compilation During Play/Recompile After Finished Playing")]
    static void ScriptCompilationToggleOption1()
    {
        EditorPrefs.SetInt("ScriptCompilationDuringPlay", 1);
    }


    [MenuItem("Tools/Script Compilation During Play/Stop Playing And Recompile")]
    static void ScriptCompilationToggleOption2()
    {
        EditorPrefs.SetInt("ScriptCompilationDuringPlay", 2);
    }


    //This is called before 'Tools/Script Compilation During Play/Recompile And Continue Playing'
    //is shown to check for the current value of ScriptCompilationDuringPlay and update the checkmark
    [MenuItem("Tools/Script Compilation During Play/Recompile And Continue Playing", true)]
    static bool ScriptCompilationValidation()
    {
        //Here, we uncheck all options before we show them
        Menu.SetChecked("Tools/Script Compilation During Play/Recompile And Continue Playing", false);
        Menu.SetChecked("Tools/Script Compilation During Play/Recompile After Finished Playing", false);
        Menu.SetChecked("Tools/Script Compilation During Play/Stop Playing And Recompile", false);


        var status = EditorPrefs.GetInt("ScriptCompilationDuringPlay");


        //Here, we put the checkmark on the current value of ScriptCompilationDuringPlay
        switch (status)
        {
            case 0:
                Menu.SetChecked("Tools/Script Compilation During Play/Recompile And Continue Playing", true);
                break;
            case 1:
                Menu.SetChecked("Tools/Script Compilation During Play/Recompile After Finished Playing", true);
                break;
            case 2:
                Menu.SetChecked("Tools/Script Compilation During Play/Stop Playing And Recompile", true);
                break;
        }
        return true;
    }
}
```

回到开头的问题，为了解决频繁自动编译的问题，这里我禁用用自动刷新的功能，但是这就导致了我每次都需要手动刷新，如果没有手动刷新，那么修改的代码就不会生效。
所以这里换个思路来解决问题，我们只在运行前和退出运行的时候，进行代码编译，而其他时候都不会进行代码编译。这样上面提到的设置都不用管，仍然可以启用自动刷新功能，但是自动刷新不会进行自动编译。刚好Unity提供是否启用自动编译的接口。所以在不需要任何设置的情况下，只需要将下面的编辑器脚本导入到Unity的Editor目录下，在运行前和运行结束的时候会自动检查代码进行自动编译。
```C#
using UnityEditor;

[InitializeOnLoad]
public class CompilerOptionsEditorScript
{
    static CompilerOptionsEditorScript()
    {
        EditorApplication.playModeStateChanged += PlaymodeChanged;
    }

    static void PlaymodeChanged(PlayModeStateChange state)
    {
        //在切换运行模式的时候自动进行脚本编译
        if (state == PlayModeStateChange.ExitingPlayMode
            || state == PlayModeStateChange.ExitingEditMode)
        {
            EditorApplication.UnlockReloadAssemblies();
        }

        //在运行时或编辑模型下，关闭脚本自动编译，解决频繁编译降低开发效率的问题
        if (state == PlayModeStateChange.EnteredPlayMode
            || state == PlayModeStateChange.EnteredEditMode)
        {
            EditorApplication.LockReloadAssemblies();
        }
    }
}
```

## 参考
[How to stop automatic assembly compilation from script](https://support.unity.com/hc/en-us/articles/210452343-How-to-stop-automatic-assembly-compilation-from-script)
