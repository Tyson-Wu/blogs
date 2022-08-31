---
title: GitLab and Jenkins and Unity
date: 2022-08-30 18:03:00
categories:
- [Docker]
tags:
- Jenkins
- GitLab
- Unity
---

在[这篇文章](https://tyson-wu.github.io/blogs/2022/08/29/Docker_GitLab_Jenkins/)讲了如何在Docker上安装GitLab，Jenkins，并且最终Jenkins从GitLab上拉取到项目文件。
我最终的目的是希望能够对Unity项目进行打包部署，所以我开始关联Jenkins和Unity。因为我的Unity是装在物理机上，而Jenkins是装在Docker上，所以Jenkins没办法调取Unity进行打包操作。
所以我后面采用的是GitLab部署在Docker上，而Jenkins和Unity都是安装在物理机上，这样Jenkins通过GitLab的Url拉取项目，然后通过Unity Puglin插件调用Unity进行打包，当然也可以使用windows的批处理命令`build.bat`的方式。采用Unity Plugin的方式的一个好处就是unity的log会直接显示到Jenkins的log界面上。

其中，虽然Jenkins和GitLab的安装位置不一样，但是他们之间的关联操作和之前的是一样的。这里我们讲一下Jenkins和Unity的关联步骤。
首先建一个Unity项目，然后在项目根目录下创建两个文件`build_image_url_link.bat`和`myqrcode.py`。
`build_image_url_link.bat`
```bash
set BASE_PATH=D:\ServicesData\Publish\%1\%2
set BASE_URL=http://192.168.31.186:8080/Publish/%1/%2
python myqrcode.py %BASE_URL%/%3_%4.apk %BASE_PATH%\qrcode.png
echo DESC_INFO:%BASE_URL%/qrcode.png,%BASE_URL%/%3_%4.apk
```
`myqrcode.py`
```python
import qrcode
import sys
data = sys.argv[1]
path=sys.argv[2]
img = qrcode.make(data)
img.save(path)
```
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-13-45.png)
然后在Editor下创建`BuildTools.cs`脚本。
```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;
using System;

public class BuildTools
{
    public static class BuildTargetName
    {
        public const string android = "Android";
        public const string window = "Window";
    }
    public static class EnvironmentParam
    {
        public const string projectName = "--projectName";
        public const string productName = "--productName";
        public const string version = "--version";
        public const string target = "--target";
    }
    public static bool GetEnvironmentParam(string paramName, out string paramValue)
    {
        paramValue = null;
        string[] args = System.Environment.GetCommandLineArgs();
        foreach (var s in args)
        {
            if (s.Contains(paramName))
            {
                if (s.Split(':').Length > 1)
                {
                    paramValue = s.Split(':')[1];
                    return true;
                }
            }
        }
        return false;
    }

    [MenuItem("Build/Build APK")]
    public static int BuildApk()
    {
        try
        {
            if (!GetEnvironmentParam(EnvironmentParam.productName, out string productName))
            {
                productName = "productName";
            }
            if (!GetEnvironmentParam(EnvironmentParam.version, out string bundleVersion))
            {
                bundleVersion = "bundleVersion";
            }
            if (!GetEnvironmentParam(EnvironmentParam.projectName, out string projectName))
            {
                projectName = "projectName";
            }
            if (!GetEnvironmentParam(EnvironmentParam.target, out string target))
            {
                target = BuildTargetName.android;
            }
            string extention = ".exe";
            PlayerSettings.productName = productName;
            PlayerSettings.bundleVersion = bundleVersion;
            BuildPlayerOptions opt = new BuildPlayerOptions();
            opt.scenes = new string[] { "Assets/Scenes/SampleScene.unity" };
            switch (target)
            {
                case BuildTargetName.android:
                    opt.target = BuildTarget.Android;
                    extention = ".apk";
                    break;
                case BuildTargetName.window:
                    opt.target = BuildTarget.StandaloneWindows64;
                    extention = ".exe";
                    break;
            }

            opt.locationPathName = $"D:/ServicesData/Publish/{projectName}/{target}/{productName}_{bundleVersion}{extention}";

            opt.options = BuildOptions.None;
            BuildPipeline.BuildPlayer(opt);
            Debug.Log("Build Done!");
        }
        catch(Exception e)
        {
            Debug.LogError(e.Message);
            return -1;
        }
        return 0;
    }
}

```
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-19-26.png)
到这一步Unity项目就建好了，然后将它提交的GitLab上。

创建一个打包项目，关联到GitLab，操作步骤参考上面的链接。进入当前打包项目的配置界面。按下图配置。
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_00-58-51.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_00-59-04.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-01-05.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-01-13.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-01-23.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-02-11.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-02-32.png)
`-quit -batchmode -nographics -executeMethod BuildTools.BuildApk --productName:$productName --version:$version --projectName:JenkinsUnityForAndroid --target:$target`
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-14-37.png)
`build_image_url_link.bat JenkinsUnityForAndroid %target% %productName% %version%`
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-02-57.png)
`DESC_INFO:(.*),(.*)`
`<img src="\1" height="200" width="200"  /> <a href="\2">点击下载</a>`

这样就基本配置好Jenkins和Unity项目了。但是上面配置中多了很多奇怪的操作，像什么Python脚本之类的。这是因为我想将打包好的包发布到一个本地服务，然后通过二维码下载。
所以这里我需要搭建一个本地服务，看网上有说用蒲公英什么的来当作发布平台，但是我就想本地局域网用，所以使用我比较熟悉的`HFS`来冲动服务。
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-39-11.png)
前面打包代码里面会将好的包放到这个服务路径下面，然后可以通过Url来访问，假设当前打好的包在这个路径下，那么手机输入对应的Url就可以下载了。不过每次都要输入Url很麻烦。所以这里考虑使用二维码。
[这里](https://www.cnblogs.com/rainboy2010/p/12007248.html)讲了使用`qrcode`和`Image`两个python插件来生成二维码，所以首先我们要安装Python，这里[Python 3.7.7](https://www.python.org/downloads/release/python-377/)下载，安装默认安装，并勾选写入环境变量。
然后打开命令行窗口，分别输入`pip install qrcode`和`pip install Image`两个命令，安装好两个插件。
然后回到Jenkins，设置Jenkins内部的环境变量。
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-31-34.png)
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-31-47.png)
这里因为我的python是安装在`C:\Users\10524\AppData\Local\Programs\Python\Python37`目录下的。都准备好后，就可以开始构建了，下面是最终效果。
![wwwwwww](/blogs/images/src/Snipaste_2022-09-01_01-50-28.png)

## 参考

[docker+gitlab+jenkins从零搭建自动化部署](https://blog.csdn.net/weboof/article/details/104491998)
[Jenkins生成APK链接的二维码](https://www.cnblogs.com/rainboy2010/p/12007248.html)
[Jenkins构建Python项目提示：‘python‘ 不是内部或外部命令，也不是可运行的程序](https://blog.csdn.net/qq_42281648/article/details/120057980)
[Python 3.7.7](https://www.python.org/downloads/release/python-377/)下载Windows x86-64 executable installer，安装默认安装，并勾选写入环境变量。
[HFS](http://www.rejetto.com/hfs/)下载，搭建简易服务。