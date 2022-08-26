---
title: 在Win10下面配置基于Bonobo Git Server和Jenkins的Unity项目部署
date: 2022-8-27 19:01:34
categories:
- [Jenkins]
tags:
- Jenkins
- Git
- Bonobo Git Server
---

## 记录
这里假设我们已经在Win10系统上安装安装好了Jenkins和Bonobo Git Server，并且有一个Unity工程JenkinsTest，同时安装了Unity，我这里是Unity 2020.3.33f1c2 (64-bit)，另外还装了Git，并且在Bonobo Git Server服务上搭建了JenkinsTest的版本仓库。

在这些都准备好之后，下面是需要操作的步骤：
- 在Jenkins里面新建一个Item，也就是一个新的部署项目。随便给它取个名字，这里使用JenkinsTest，然后选FreeStyle Project,最后点确定。
- 之后会自动跳转到一个设置界面，这里有很多设置，但是本教程只需要关注Git这个选项的配置，我们需要加一个Git仓库配置，勾选Git，然后点击Add Repository，会出现Repository URL，这里填JenkinsTest项目在Bonobo Git Server中的URL，我这里是htt://localhost:8888/JenkinsTest.git。然后是Credentials，这里需要填一个Git拉去的访问权限的东西，有几种选择，因为Bonobo Git Server只提供账号密码的访问形式，所以这里我们在下面添加按钮，打开新建一个凭证，这个凭证是采用账号密码的形式，然后在对应框中输入账号和密码，点击确定，然后回到Credentials下，选择新建好的凭证。
- 到这里为止，如果我们保存配置，然后对Jenkins进行构建的话，Jenkins会自动将Jenkins项目拉取到它的工作区。除此之外不会做任何其他事情。
- 为了最终打包工程，我们需要在上面那个设置界面下的构建设置里面，选择构建的方式。在Windows中，可以使用Execute Windows batch command来构建，主要是通过调用一些bat类型的可执行文件，来间接调用Unity项目中的打包脚本。这总方法在网上有很多，比如[这个](http://dingxiaowei.cn/2018/11/17/)教程。
- 但是还有一种方式，是直接使用Unity Editor的方式来打包，这种方式需要在Jenkins中安装Unity3D Plugin插件。然后在Jenkins的Global Tool Configuration的配置里面，找到Unity3d部分，那个有个Unity安装的按钮，点开，然后将电脑上已有的Unity配置上去。举个例子，我这里的Unity的安装路径是C:\Program Files\Unity\Hub\Editor\2020.3.33f1c2，然后别名随便。
- 然后再会到JenkinsTest的配置界面，就会发现构建部分多了Invoke Unity3d Editor的选项，选它，然后Unity3d installation name这里会自动出现前面配置的Unity别名，意思是我们选择这个Unity版本来打包。
- 然后在Editor command line arguments里面配置打包命令，这个是在Unity手册里面是有介绍到。我这里填的打包命令是`-quit -batchmode - - executeMethod BuildTools.BuildApk`。其中BuildTools是JenkinsTest工程下的编辑器脚本，BuildApk是该脚本下的一个打包函数，执行具体的打包操作。
- 了解过Execute Windows batch command打包方式的人应该记得，打包命令里面一般会包含Unity的路径和需要打包的项目路径，但是这里并没有，这是因为前面已经配置了Unity的安装路径，而项目路径是Jenkins自动拉取的git目录，所以这两个目录Jenkins是知道的。我最开始配了这两个反而导致报错。
- 最后，点击构建，开始打包JenkinsTest项目。

到这里整个流程就走完了，可以愉快的一键打包。但是打包的时候我发现打包前要构建Unity工程的Library文件夹，这个是Unity自动生成的，这个文件夹构建一次要花很长时间，所以需要部署资源缓存服务。我又开始找相应的教程，然后发现这个资源缓存服务不亚于Git服务的搭建，要装安多东西。
所以最终我还是决定方式Jenkins和Bonobo Git Server的组合。因为我都打算安装那么多服务了，就想直接使用Docker来作为容器，这样什么服务都可以搭，那就搭一个更牛逼的Git服务吧，比如说GitLab。
不过Jenkins和Bonobo Git Server的组合，整个安装过程都是非常简单的，适合初学者使用，或者需求没有那么复杂的个人使用。

## 参考

[Unity-Jenkins打包部署工具（一）](https://cloud.tencent.com/developer/article/1654086)
[Jenkins!自動建置Unity專案-本地建置](https://unoniceday.github.io/jenkins-autobuild-unity-local/)
[Unity为Git项目部署Jenkins](https://blog.csdn.net/C_yuxuan/article/details/119182706)