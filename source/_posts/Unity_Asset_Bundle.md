---
title: Unity AssetBundles
date: 2022-12-7 10:01:00
categories:
- [Unity, AssetBundles]
tags:
- Unity
- AssetBundles
- Shader
---

接着上一篇[Unity资源逆向解析](https://tyson-wu.github.io/blogs/2022/11/10/Unity_Asset_Extractor/)，这里再来谈一谈AssetBundles。

## AssetBundles 使用

使用`AssetBundles`总体上分成两步，第一步是创建`AssetBundles`资源，第二部是使用`AssetBundles`资源。

### AssetBundles 创建

Unity提供内置的[BuildPipeline](https://docs.unity3d.com/2020.3/Documentation/ScriptReference/BuildPipeline.BuildAssetBundles.html)管线和可编程的[Scriptable Build Pipeline](https://docs.unity3d.com/Manual/com.unity.scriptablebuildpipeline.html),这里仅以内置的`BuildPipeline`管线为例进行讲解。

首先内置的`BuildPipeline`管线提供了`BuildPipeline.BuildAssetBundles()`函数来帮助创建资源，有两种调用方式。
- 方式一：
`public static AssetBundleManifest BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);`

- 方式二：
`public static AssetBundleManifest BuildAssetBundles(string outputPath, AssetBundleBuild[] builds, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);`

方式一是针对项目中所有被标记为`AssetBundles`的资源，根据其标记的`AssetBundles`名称对这些资源进行打包的。
方式二是忽略`AssetBundles`的资源标记，而是通过纯代码的形式，向`BuildAssetBundles`传入所需要构建的资源，以及`AssetBundles`资源名称。

如果想实现自动化构建资源的话，偏向于使用方式二，可以完全通过脚本进行控制。
但是这里为了方便，采用方式一进行构建资源。
因此资源构建代码如下：
```c#
using UnityEngine;
using UnityEditor;
using System.IO;
public class CreateAssetBundles
{
    [MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles()
    {
        string assetBundleDirectory = "./assetBundleDirectory";// 这里传你需要的导出的路径
        if(!Directory.Existes(assetBundleDirectory))
            Directory.CreateDirectory(assetBundleDirectory);
        BuildPipeline.BuildAssetBundles(assetBundleDirectory,BuildAssetBundleOptions.None, BuildTarget.StandaloneWindows);
        Debug.Log("build assetbundles done!");
    }
}
```

然后要手动编辑哪些资源需要被打包成`AssetBundles`，根据官方文档，可以对Assets目录下的文件、以及文件夹进行标记，如果标记的是文件夹，那么相当于标记了文件夹下所有子文件。当然文件的标记优先级高于文件夹的优先级。
这里我的目录结构如下：
```
资源                          AssetBundleName         资源依赖
[Assets]
    [texture]                 texture
        mogutexture.png
        planks.png
        stone 2.png
    2_ZJZ_zhiwu09.fbx         mogu
    mogucolor.prefab          mogucolor              [2_ZJZ_zhiwu09.fbx|mogumaterial.mat]
    mogumaterial.mat          mogumaterial           [MoguShader.shader|mogutexture.png]
    MoguShader.shader         mogushader              
    ShaderVar.shadervariants  mogushader
```

上面第一列是资源目录结构，第二列是针对资源的`AssetBundles`资源名称，第三列表示的是该资源所依赖的其他资源。

一切准备就绪后，在Assets菜单下点击`Build AssetBundles`开始构建资源。
在`assetBundleDirectory`目录下得到如下资源：
```
[assetBundleDirectory]
    assetBundleDirectory
    assetBundleDirectory.manifest
    texture
    texture.manifest
    mogu
    mogu.manifest
    mogucolor
    mogucolor.manifest
    mogumaterial
    mogumaterial.manifest
    mogushader
    mogushader.manifest
```

其中以manifest为后缀的文件是给人读的，展示了一些基本信息。加载`AssetBundles`时只需要读取无后缀的文件就行。而`assetBundleDirectory`这个资源文件记录了其他所有资源的manifest文件信息。
上面打包出来的资源文件可以使用[Unity资源逆向解析](https://tyson-wu.github.io/blogs/2022/11/10/Unity_Asset_Extractor/)中提到的解包工具打开。

### AssetBundles 加载

本文加载的基本思路是读取`assetBundleDirectory`资源文件中的信息，然后去构建一个资源目录表，然后在使用资源时，首先访问资源目录表，看看该资源是否已经加载，如果没有加载的话就对该资源进行加载，加载时也会判断资源依赖，待所有依赖资源以及该资源都加载成功后，触发资源加载成功回调函数。
代码如下：
```C#
using System.Collections.Generic;
using System.IO;
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.Networking;
using System;

pulic class ResurceData
{
    public enum LoadState
    {
        Null,
        Waiting,
        Loading,
        Loaded,
    }
    public bool isReadly = false; // 该资源以及其依赖资源是否都成功加载
    public string assetBundleName = string.Empty; // 该资源的assetBundle名称
    public int serialId = 0; // 给资源的一个唯一id
    public List<string> dependanceName = new List<string>(); // 其直接依赖的资源名称列表
    public List<string> parentName = new List<string>(); // 运行时引用了该资源的资源名称列表
    public AssetBundle assetBundle = null; // 加载得到的AssetBundle
    public LoadState loadState = LoadState.Null; // 加载状态
    public referenceCount = 0; // 该资源被外部引用的次数（不包括资源与资源之间的引用）
    public bool hasRequest {get{return loadState != LoadState.Null;}} // 是否已经执行过资源下载命令
    public bool hasError {get{ return loadState == LoadState.Loaded && assetBundle == null;}} // 该资源是否加载失败
}
[Serializable]
public class OnFinishLoadAssetBundle : UnityEvent<int, bool>{}
[Serializable]
public class OnFinishLoadManifest : UnityEvent<bool>{};
public class LoadAndUnloadRemoteAssets : Singleton<LoadAndUnloadRemoteAssets>
{
    private in m_MaxId = 0;
    private string m_Uri;

    private bool m_Inited = false;
    public bool inited => m_Inited;
    private UnityWebRequest m_MainfestRequest = null;
    AssetBundleManifest m_AssetBundleManifest = null;
    AssetBundle m_ManifestAssetBundle = null;

    Queue<int> m_WaitToCheck = new Queue<int>();

    Dictionary<int, ResourceData> allResource = new Dictionary<int, ResourceData>();
    Queue<int> m_WaitToLoad = new Queue<int>();
    Dictionary<int, UnityWebRequest> m_UnityWebRequests = new Dictionary<int, UnityWebRequest>();
    Dictionary<string, int> m_NameToId = new Dictionary<string, int>();
    private List<int> m_TORemoveRequest = new List<int>();

    public OnFinishLoadMainfest OnFinishLoadMainfest = new OnFinishLoadMainfest();
    public OnFinishLoadManifest OnFinishLoadManifest = new OnFinishLoadManifest();

    // url 是`assetBundleDirectory`目录的路径，以http:等开头的形式
    public void Init(string url)
    {
        if(m_ManifestRequest != null) return;
        m_Inited = false;
        m_Uri = url;
        int index = url.LastIndexOf('/');
        string manifestName = url.Substring(index + 1);
        url = Path.Combine(url, manifestName);
        m_ManifestRequest = UnityWebRequestAssetBundle.GetAssetBundle(url, 0);
        m_ManifestRequest.SendWebRequest();
    }
    private int GetMaxId()
    {
        return ++m_MaxId;
    }
    public AssetBundle GetAssetBundle(int id)
    {
        if(allResource.TrayGetValue(id, out var resourceData))
            return resourceData.assetBundle;
        return null;
    }
    public void UnloadAllAssets()
    {
        foreach(var resource in allResource)
        {
            var resourceData = resource.Value;
            if(resourceData.assetBundle)
            {
                resourceData.assetBundle.Unload(true);
            }
            resourceData.assetBundle = null;
            resourceData.loadState = ResourceData.LoadState.Null;
            resourceData.isReady = false;
            resourceData.referencCount = 0;
            resourceData.parentName.Clear();
        }
    }
    public bool UnloadAsset(int id)
    {
        // 暂时没时间写这块逻辑，感兴趣的话可以把这部分代码补上。
        // ToDo
    }
    public int LoadAsset(string assetbundleName)
    {
        if(!m_NameToId.TryGetValue(assetbundleName, out in id))
            return -1;
        return LoadAssetInternal(assetbundleName, null);
    }
    private int LoadAssetInternal(string assetbundleName, string parent)
    {
        m_NameToId.TryGetValue(assetbundleName, out var id);
        allResource.TryGetValue(id, out var resourceData);
        if(!resourceData.hasRequest)
        {
            foreach(var dep in resourceData.dependanceName)
                LoadAssetInternal(dep, assetbundleName);
            m_WaitToLoad.Enqueue(id);
            resourceData.loadState = ResourceData.LoadState.Waiting;
        }
        if(parent == null)
            ++resourceData.referenceCount;
        else
            resourceData.parentName.Add(parent);
        return id;
    }
    private void Update()
    {
        if(!m_Inited)
        {
            if(m_ManifestRequest != null && m_ManifestRequest.isDone)
            {
                OnManifestDownloaded(m_ManifestRequest);
                m_ManifestRequest.Dispose();
                m_ManifestRequest = null;
            }
        }
        else
        {
            foreach(var request in m_UnityWebRequests)
            {
                if(request.Value.isDone)
                {
                    if(string.IsNullOrEmpty(request.Value.error))
                    {
                        var resource = allResource[request.Key];
                        resour.loadState = ResourceData.LoadState.Loaded;
                        resource.assetBundle = DownloadHandlerAssetBundle.GetContent(request.Value);
                        var isReady = true;
                        // 这个循环其实被包含在下面的循环逻辑中，可以因此该段代码
                        foreach(var dep in resource.dependanceName)
                        {
                            if(!allResource[m_NameToId[dep]].isReady)
                            {
                                isReady = false;
                            }
                        }
                        if(isReady)
                        {
                            int curId = request.Key;
                            m_WaitToCheck.Enqueue(curId);
                            while(m_WaitToCheck.Count > 0)
                            {
                                curId = m_WaitToCheck.Dequeue();
                                resource = allResource[curId];
                                if(resource.assetBundel == null)
                                    continue;
                                
                                isReady = true;
                                // 判断依赖文件
                                foreach(var dep in resource.dependanceName)
                                {
                                    if(!allResource[m_NameToId[dep]].isReady)
                                    {
                                        isReady = false;
                                    }
                                }
                                if(isReady)
                                {
                                    resource.isReady = true;
                                    OnFinishLoadAssetBundle.Invoke(curId, true);
                                    // 往上迭代，检查哪些资源引用了该资源，并更新其状态
                                    foreach(var parent in resource.parentName)
                                    {
                                        m_WaitToCheck.Enqueue(m_NameToId[parent]);
                                    }
                                }
                            }
                        }
                    }
                    else
                    {
                        allResource[request.Key].loadState = ResourceData.LoadState.Loaded;
                        OnFinishLoadAssetBundle.Invoke(request.Key, false);
                        Debug.LogError($"fail to load {allResource[request.Key].assetBundleName}. Because:{request.Value.error}");
                    }
                    m_ToRemoveRequest.Add(request.Key);
                }
            }
            foreach(var id in m_ToRemoveRequest)
                m_UnityWebRequests.Remove(id);
            m_ToRemoveRequest.Clear();

            // 暂时设置为最多同时加载4个资源
            while(m_UnityWebRequests.Count < 4)
            {
                if(m_WaitToLoad.Count > 0)
                {
                    var id = m_WaitToLoad.Dequeue();
                    allResource[id].loadState = ResourceData.LoadState.Loading;
                    var url = Path.Combine(m_Uri, allResource[id].assetBundleName);
                    var request = UnityWebRequestAssetBundle.GetAssetBundle(url, 0);
                    request.SendWebRequest();
                    m_UnityWebRequest.Add(id, requeset);
                }
                else
                {
                    break;
                }
            }
        }
    }
    private void OnManifestDownloaded(UnityWebRequest manifestRequest)
    {
        AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(manifestRequest);
        var assetBundleManifest = bundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");
        m_ManifestAssetBundle = bundle;
        if(AssetBundleManifest != null)
        {
            m_Inited  = true;
            m_AssetBundleManifest = assetBundleManifest;
            var allAssetBundles = assetBundleManifest.GetAllAssetBundles();
            foreach(var assbundle in allAssetBundles)
            {
                var dirctD = assetBundleManifest.GetDirectDependencies(assbundle);

                ResourceData resourceData = new ResourceData();
                resourceData.assetBundleName = assbundle;
                resourceData.dependanceName.AddRange(dirctD);
                resourceData.serialId = GetMaxId();
                allResource.Add(resourceData.serialId, resourceData);
                m_NameToId.Add(assbundle, resourceData.serialId);
            }
            OnFinishLoadManifest.Invoke(true);
            Debug.Log("successfully load AssetBundleManifest");
        }
        else
        {
            OnFinishLoadManifest.Invoke(false);
            Debug.LogError("fail to load AssetBundleManifest");
        }
    }
}

//=======================
//使用顺序
//先调用Init函数初始化AssetBundleManifest
//然后调用LoadAsset异步加载，每次调用引用加1
//OnFinishLoadManifest，和OnFinishLoadAssetBundle两个回调监听初始化和加载结果
//加载成功后使用GetAssetBundle获取资源包
//用完后使用UnloadAsset卸载资源包，每次调用引用减1，和LoadAsset配套使用
//=======================
```

## AssetBundles 内存分析

这里以实例化`mogucolor.prefab`为例。总共有以下几个步骤：
- 调用Init函数初始化`AssetBundleManifest`
- LoadAsset异步加载`mogucolor`，因为其直接以及间接依赖的资源有`texture`、`mogu`、`mogumaterial`、`mogushader`,所以一共会有5个AssetBundle会被加载
- 然后调用`AssetBundle.LoadAsset<GameObject>("mogucolor)`，提取其中的`mogucolor.prefab`资源供Unity后面使用。
- 最后使用`GameObject.Inistatiate()`函数将`mogucolor.prefab`实例化出来。

在Profiler监控窗口中，选中`Momory`,然后在下方选择`Detailed`。执行实例化加载流程，然后每一步点击`Task Sample Editor`进行性能数据采样，你会发现下方有以下5大类：
* Other
* Assets
* Not Saved
* Builtin Resources
* Scene Momory

通过记录发现，每一步各个类型数据下都各有变化，以下是变化情况：
```
第一步：调用Init函数初始化`AssetBundleManifest`
Other +1 []
NotSaved +2 [AssetBundle+1, AssetBundleManifest+1]
Assets +0 []
Builtin Resources +0 []
SceenMemory +0 []

第二步：LoadAsset异步加载`mogucolor`
Other +5 []
NotSaved +5 [AssetBundle+5]
Assets +0 []
Builtin Resources +0 []
SceenMemory +0 []

第三步：然后调用`AssetBundle.LoadAsset<GameObject>("mogucolor)`
Other +0 []
NotSaved +0 []
Assets +8 [Shader+1, Texture+1, Mesh+1, Material+1, MeshRenderer+1, GameObject+1, MeshFilter+1]
Builtin Resources +0 []
SceenMemory +0 []

第四步：最后使用`GameObject.Inistatiate()`
Other +1 []
NotSaved +2 [RenderTexture+2, Material+1]
Assets +0 []
Builtin Resources +1 [Shader+1]
SceenMemory +4 [Transform+1, GameObject+1, MeshRenderer+1, MeshFilter+1]
```

可以发现，第一和第二步，主要是增加了NotSaved里面的AssetBundle。第三步增加的是Assets里面的资源数据。第四步增加的是场景里面的数据，至于NotSaved和Builtin Resources数据也被动增加，主要是因为新入场景的物体可能会导致渲染流程发生改变，所以内置的一些材质或者RenderTexture会有所增加。
至于Builtin Resources里面的数据，从分析上来看主要是GUI相关的的一些内置数据，以及异常Shader等。

## AssetBundles 与 Shader

Shader也是一种资源，和模型、贴图资源一样，可以打包进AssetBundle。但是Shader又是一种程序，只不过是运行在GPU上的，因此为了节约性能，需要对Shader进行裁剪，将不需要的变体剔除掉。而我们自己使用AssetBundles对Shader进行打包时，也会进行剔除。以下是官方文档中的[原文](https://docs.unity3d.com/2020.3/Documentation/Manual/AssetBundles-Troubleshooting.html)，介绍了如何根据需要将需要的变体打包。
```
Interactions between Shaders and AssetBundles
When you build an AssetBundle, Unity uses information in that bundle to select which shader
 variants to compile. This includes information about the scenes
, materials, Graphics Settings, and ShaderVariantCollections in the AssetBundle.

Separate build pipelines compile their own shaders independently of other pipelines. If both a Player build and an Addressables build reference a shader, Unity compiles two separate copies of the shader, one for each pipeline.

This process doesn’t account for any runtime information such as keywords, textures, or changes due to code execution. If you want to include specific shaders in your build, either include a ShaderVariantCollection with the desired shaders, or include the shader in your build manually by adding it to the Always Included Shaders list in your graphics settings.
```

从文档总结就是，假设我们有一个Shader定义了多个关键字，如果我们只想打包某几个关键字相关的变体时，我们可以采用以下几种方式：
- 创建与变体相对应的材质，然后将这些材质一起打包到这个Shader的AssetBundle中
- 创建变体集，如ShaderVar.shadervariants，以shadervariants为后缀的资源，在这个变体集中定义变体，然后将变体集一起打包到这个Shader的AssetBundle中
- 使用Graphics Settings,（暂时不知道怎么使用）。

有一个很重要的点是，一个Shader可以打包进多个AssetBundle，各个AssetBundle中所生成的变体是相互独立的，最终运行时也是相互独立看待。

## Shader变体分析

既然谈到Shader变体，这里就再多说几句，在Unity中Shader变体主要是受Shader关键字控制，当然根据官网描述，一个shader里面拥有多个Pass，也是形成变体的因素。

这里主要是谈一下关键字，首选关键字的增加，对于变体的数量影响很大，最极端的情况是变体的数量是关键字数量的指数级。所以我们需要控制关键字的数量，尽量减少关键字的使用。

假设当前关键字的数量已经是最优的了，我们还可以控制关键字的搭配来控制变体的数量。上一节提到了变体与关键字的关系，就是通过材质和shadervariants等形式，来限定哪些变体生成。

但是目前Unity在关键字上还有一个限制，就是关键字数量不超过256，其中Unity内部已经占用了一部分，所以留给我们用来定义关键字的数量根本够不到256。所以在定义关键字时是需要精打细算的。

在Unity2018版本以及之前的版本，关键字只有global全局关键字，也就是说，所有Shader定义过的关键字数量之和不得超过256，这个限制就非常大，如果使用一些第三方资源，有可能会突破这个限制，最终导致项目崩溃等问题。

Unity2018之后的版本，关键字分为了local和global两种，其中global还是延续了全局关键字的逻辑，也就是所有shader共享的，但是local关键字是从属于当前定义的shader。关键字256的数量限制还在，只不过是local和global两种关键字的总和。怎么理解呢？

举个例子，现在只有两个Shader，Shader1和Shader2，Shader1中定义了100个global关键字，50个local关键字，Shader2中定义了90个global关键字，49个local关键字，这些关键字没有重复的。那么在Shader1看来，gloabl关键字有190个，local关键字有50个，两者之和没有超过256。那么在Shader2看来，gloabl关键字有190个，local关键字有49个，两者之和没有超过256。但是将所有关键字都加起来其实是超过256的。但是这两个Shader的变体组合是有效的。也就是说新的关键字规则将一部分全局关键字的份额分配给了局部关键字，具体来说就是192个全局关键字，和64个局部关键字。

关于关键字的上限，以及因为这个上限导致的问题在[这里](https://forum.unity.com/threads/shader-keyword-limit.545491/page-2)有很多讨论，其中网友发现问题最多的是使用HDRP时，很容易超出限制，以及使用AssetStore里面的资源也会很容易超出限制。所以网友希望能够直接提高上限，或者给个上限可选项，但是Unity客服表示不太愿意这样做。这个问题提出了好几年了，关键字分为了local和global两种也是基于这个问题给出的一个解决方案。

从程序优化的角度来讲，控制关键字的数量是非常有必要的，因此我们首先需要知道有哪些关键字被使用的了，Unity本身并没有提供这个功能（我目前没看到，并且从上面的讨论链接里面，Unity客服也表示这个很难实现）。但是广大的开发网友开发了一些工具可以用来辅助分析关键字。这里我知道的有两个：
- [Shader Control](https://assetstore.unity.com/packages/vfx/shaders/shader-control-74817) ：Shader Control 是一款功能强大的编辑器插件，让你可以完全控制着色器的编译和关键字的使用以及它们在游戏中的影响(广告是这么说的，我穷没用过)
- [Shader Keywords Tool UNITY3D](https://bitbucket.org/ArtIsDarkGames/shader-keywords-tool-unity3d/src/master/) ：免费开源的，用了一下下，可用。

另外[这里](https://blog.unity.com/technology/stripping-scriptable-shader-variants)以及[这里](https://docs.unity3d.com/Manual/shader-variant-stripping.html)详细的介绍了变体相关的东西，包括变体数量的计算，以及如何控制变体的数量。而且Unity还提供`scriptable shader variants stripping`自定义变体剔除的方法，只要是实现[IPreprocessShaders](https://docs.unity3d.com/ScriptReference/Build.IPreprocessShaders.OnProcessShader.html)，这个回调接口在打包项目和打包Assetbundle资源时都会调用。