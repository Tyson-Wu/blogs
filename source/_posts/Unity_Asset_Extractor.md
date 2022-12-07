---
title: Unity资源逆向解析
date: 2022-11-10 10:01:00
categories:
- [Unity]
tags:
- Unity
---

## Asset Ripper

支持打开打包后的资源，对于图片等资源可以直接预览，但是Shader只能查看到名字属性等基本信息，里面的代码变体等信息都是假的。
[下载链接](https://assetripper.github.io/AssetRipper/articles/Downloads.html)

## UABE(AssetsBundleExtractor)

支持解析打包后的资源，但是好像只能罗列出资源信息，不能像`Asset Ripper`一样完美的将资源解包出来。但是支持修改打包后的资源，这是其他工具所不具备的。
[下载链接](https://github.com/SeriousCache/UABE/releases/tag/2.2stablec)
[社区](https://community.7daystodie.com/topic/1871-uabe-asset-bundle-extractor/page/35/)

## AssetStudio
和`Asset Ripper`一样，支持打开打包后的资源，但是Shader信息展示的更多一点，可以看到变体信息，但是代码块部分仍然看不了。
[下载链接](https://github.com/Perfare/AssetStudio/releases)

## 关于Unity Resources
Unity项目运行时可以通过Resources和AssetsBundle方式读取资源。推荐是使用AssetsBundle，但是Resource资源的组织方式影响着包体的大小，所以即便项目中没有使用到Resources资源读取方式，也还是需要有所了解。
- 首先Resources资源读取方式，需要将资源放置在Assets目录及其子目录下的Resources目录。只有在这个文件夹下的资源才可以使用Resources的方式读取。
- 打包项目工程时，Resources下的资源会直接打到项目App，如Apk。
- Resources资源在App中，是存储在`resources.assets`、`resources.assets.resS`、`sharedassets0.assets`、`sharedassets0.assets.resS`文件中。
- 关于`*.assets`和`*.assets.resS`两个后缀文件，同名文件中的*.assets存储的是序列化文件。而`*.assets.resS`里面存储的是二进制数据源，如图片、音效等。
- `*.assets`文件类似于`Prefab`，存储着资源的所有序列化信息，以及对二进制数据源的引用，比如说图片，而`*.assets.resS`里面便是`*.assets`里面所引用的实际的图片、音效等数据。
- Resources目录下的资源是打包到`resources*`中还是打包到`sharedassets*`文件中，是有一定的规律。未被引用的资源是打包到`resources*`中，比如新放入的图片还没有使用。而被引用的资源是被打包到`sharedassets*`，比如被其他Prefab引用的图片。

除了我们自己创建的Resources资源，Unity在打包时还会拷贝一些默认资源，比如内置的着色器，按钮图集等等。这些资源是被统一打包到`unity default resources`文件中(实际操作是将这个文件直接拷贝到包内，这个资源文件是平台预先就打包好的)。`unity default resources`资源实际上是`*.assets`和`*.assets.resS`文件的合体，也就是包含序列化数据和二进制源数据。整个文件有好几兆，但是实际项目中我们可能不会使用这些内置资源，所以无疑会导致包体变大，资源空间浪费。而且在苹果IOS查重时，这个文件还可能被标记。所以可以适当的修改这个文件。

以上这些Resources资源文件，都是可以使用上面的解包工具打开，可以查看到这些文件中具体包含了哪些资源。

## 参考
+ [https://fileinfo.com/extension/ress](https://fileinfo.com/extension/ress)
+ [https://github.com/SeriousCache/UABE/issues/32](https://github.com/SeriousCache/UABE/issues/32)