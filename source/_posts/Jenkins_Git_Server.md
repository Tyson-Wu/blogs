---
title: Configure Git Hooks on Bonobo Git Server in Windows
date: 2022-8-25 19:01:34
categories:
- [Jenkins]
tags:
- Jenkins
- Git
- Bonobo Git Server
---

## 记录

因为最近想自己搭建个人自动部署服务，所以考虑采用Jenkins和Git结合的方式。其中Jenkins负责项目自动化部署，Git负责版本控制，两者相结合需要一个Git服务。目前我所了解到的Git服务有[GitHub](https://github.com/)、[GitLab](https://about.gitlab.com/)、[Gitee](https://gitee.com/)、[gitblit](http://gitblit.github.io/gitblit/)、[Bonobo Git Server](https://bonobogitserver.com/)。
这里前面几个是比较知名的Git服务了，前三者有在线运营的服务，我们可以直接注册，功能相对比较完善，用的人也多，但是免费版的存储、流量限制比较大。后面两者可能很少人知道，但是可以作为小团队Git服务进行部署私服，当然[GitLab](https://about.gitlab.com/)作为开源项目也可以部署私服，但是部署方式会比较繁琐，相比而言，最后这两种方式部署更为简单。
因为我是个人使用，同时经济限制，只有一台笔记本，Windows系统，所以我选择[Bonobo Git Server](https://bonobogitserver.com/)，第一次部署完成后，真的被她的简陋的功能给震惊了，一度怀疑被收了智商税，因为我发现除了上传、下载、commit、用户管理、仓库管理，就没了，我无法想象她是如何和Jenkins联动的。好在懒惰拯救了我，当我想放弃[Bonobo Git Server](https://bonobogitserver.com/)的时候，我再转头研究[gitblit](http://gitblit.github.io/gitblit/)，发现要下载一堆相关联的软件，所以我再一次抱着试一试的心态看有没有教程将如何将[Bonobo Git Server](https://bonobogitserver.com/)与Jenkins关联起来，还真被我找到了，最终结果你们应该也猜到了，确实实现了两者的完美结合（ps：毕竟不完美的完美才是最大的完美）。好了废话有点多，进入正题吧。

首先安装[Bonobo Git Server](https://bonobogitserver.com/)本地服务，可以看[这篇](https://blog.csdn.net/u013851294/article/details/117811760)文章，我找了很多篇教程，这篇无疑是最简单而且真实有效的。

当然我们也要提前安装号Jenkins，找到一篇真正意义上的保姆级教程，在这里谢谢这位无私的大佬。教程路径[【游戏开发进阶】教你Unity通过Jenkins实现自动化打包，打包这种事情就交给策划了（保姆级教程 | 命令行打包 | 自动构建）](https://blog.csdn.net/linxinfa/article/details/118816132)

> 这里提一嘴，Jenkins安装完成后，默认工作路径一般是在C盘，也就是所有的需要部署的文件副本都会放在这个工作路径下，很显然C盘不适合当作工作路径，所以需要更改其工作路径。更改方式其实也不难，主要是有很多老旧不适用的方式在网络上混淆视听，给我们错误的引导。这里简单说一下，我们需要找到Jenkins的安装目录，这里我是`D:\Program Files\Jenkins`，在安装目录下找到`jenkins.xml`文件，然后修改其中的工作路径配置。注意这里使用的`Jenkins`版本是`Jenkins 2.346.3`。按下面更改，改前要关闭Jenkins,改后再重启。对了，我是参考的这里[Jenkins 修改主目录 JENKINS_HOME](https://blog.csdn.net/devalone/article/details/119299044)
> ```HTML
>  <!-- <env name="JENKINS_HOME" value="%ProgramData%\Jenkins\.jenkins"/> --> <!-- 这行是默认设置 -->
>  <env name="JENKINS_HOME" value="D:\ServicesData\Jenkins\.jenkins"/> <!-- 这行是我更换的路径，路径你也可以自己选 -->
> ```

重点来了，[这篇文章](https://www.cnblogs.com/wyzhhhh/p/15762137.html)不算很详细的介绍了如何配置的过程，但是相比于网络上稀缺的资料，蚊子再小也是肉。按照这篇文章大概能了解其中的基本步骤，但是其中还有有一些谬误，具体说就是`post-receive`这个文件，在该文章中写成了`post_receive`，导致我搞了半天也没成功。最后是在[这里](https://blog.dangl.me/archive/configure-git-hooks-on-bonobo-git-server-in-windows/)找到了解决方案。当然前面这篇中文教程也是有很大的参考意义的。前面说了[Bonobo Git Server](https://bonobogitserver.com/)非常简陋，没有钩子之类的工具(ps:我其实不太懂什么是钩子，我猜应该是网站之间相互订阅、通知之类的事件传输)。因此需要借助[curl](https://curl.se/download.html)来实现，搜了一下，`curl`应该是用来发送`http`这类网络协议的，所以可以实现两者通信。

目前为止，确实实现了Jenkins部署时，会自动拉取Git服务中的项目，但是教程里面提到的Git上传修改会自动触发Jenkins进行部署的方法，我还没有试验成功。在文章中提到的方式就是借用`curl`，然后还需要配置一个`post-receive`文件，在提交Git时，确实能看到控制台里面多打印了一些消息，但是Jenkins并没有触发自动部署。所以这个问题暂时先搁置，等我找到方法再补充。

这个博客有点简陋，没有评论交流的功能，所以有兴趣可以跳到知乎给我点个赞（哈哈哈，算了，知乎点赞好像也没什么用，等我有精力好好升级升级我这个破站）。

## 参考

[Configure Git Hooks on Bonobo Git Server in Windows](https://blog.dangl.me/archive/configure-git-hooks-on-bonobo-git-server-in-windows/)
[curl](https://curl.se/download.html)
[Jenkins+Bonobo.Git.Server自动部署.net core项目](https://www.cnblogs.com/wyzhhhh/p/15762137.html)
[Windows环境搭建 Gitlab 服务器](https://blog.csdn.net/u013851294/article/details/117811760)
[【游戏开发进阶】教你Unity通过Jenkins实现自动化打包，打包这种事情就交给策划了（保姆级教程 | 命令行打包 | 自动构建）](https://blog.csdn.net/linxinfa/article/details/118816132)