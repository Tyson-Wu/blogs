---
title: GitLab and Jenkins
date: 2022-08-29 18:03:00
categories:
- [Docker]
tags:
- Jenkins
- GitLab
---

这里是介绍如何将GitLab和Jenkins关联到一起。其实主要是安装一些插件、再设置一些关键配置。
首先我们需要安装GitLab，还有Gitlab API、GitLab Authentication、Generic Webhook Trigger,其中GitLab是必须装的，后面几个我也装了，是不是必须的我也不知道。反正油多不坏菜。
装完之后，回到`Configure System`里面，应该可以看到`Gitlab`部分，设置好我们的GitLab服务的URL，如下图所示。
![wwwwwww](/blogs/images/src/Snipaste_2022-08-29_23-18-17.png)
其中我GitLab的Url如下：
![wwwwwww](/blogs/images/src/Snipaste_2022-08-29_23-18-45.png)

然后新建一个Jenkins项目，取名字、选第一个模式、确定。然后进入项目设置界面，这里我们会发现多了一个`GitLab Connection`，它下面有gitlab选项，这个就是前面在`Configure System`里面新建的。然后在`源码管理`的`Git`里面，设置仓库路径，以及凭证。
![wwwwwww](/blogs/images/src/Snipaste_2022-08-29_23-43-02.png)

注意下面还有`指定分支（为空时代表any）`，里面默认是`master`分支，你确认一下有没有这个分支，或者这个分支是否是你想部署的。以及下方的源码库浏览器也可以设置一下。
![wwwwwww](/blogs/images/src/Snipaste_2022-08-29_23-47-34.png)

还有就是，如果我们当前的Git项目有依赖其他Git项目，也就是`git sub modules`，那么我们要确保在构建前，所有的依赖项目都被拉取下来，所以需要额外的设置，如下：
![wwwwwww](/blogs/images/src/Snipaste_2022-09-04_14-01-33.png)

设置好后，保存，点击`立即构建`进行部署。因为我们这里没有设置具体的部署规则，所以目前的操作只是会从GitLab里面拉取相应的项目到Jenkins工作目录下，如果构建成功，说明拉取成功，也就是GitLab和Jenkins关联成功，从而完成了最基本的设置。

## 参考

[docker+gitlab+jenkins从零搭建自动化部署](https://blog.csdn.net/weboof/article/details/104491998)