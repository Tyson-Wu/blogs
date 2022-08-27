---
title: Jenkins Install
date: 2022-06-27 15:02:00
categories:
- [Docker]
tags:
- Docker
- Jenkins
---

在Win10上安装完Docker后，就可以在Docker上安装Jenkins了。
参考官网教程[在Windows上](https://www.jenkins.io/zh/doc/book/installing/)。

- 打开命令提示符窗口。

- 下载 jenkinsci/blueocean 镜像并使用以下 docker run 命令将其作为Docker中的容器运行 ：

```bash
docker run ^
  -u root ^
  --rm ^
  -d ^
  -p 8080:8080 ^
  -p 50000:50000 ^
  -v jenkins-data:/var/jenkins_home ^
  -v /var/run/docker.sock:/var/run/docker.sock ^
  jenkinsci/blueocean
```

上面这段命令是官网上的案例，但是我并没有使用上面的，而是做了修改。后面的 `^`只是表示一行命令多行显示。特别注意这里使用的是`jenkinsci/blueocean`这个是官方推荐的，也有其他版本的Jenkins，但是可能会出现插件安装失败的问题，我之前就看了别人使用其他版本的教程，导致一些奇奇怪怪的问题。
```bash
docker run -u root -d --name jenkins --restart always -p 8000:8080 -p 50000:50000 -v D:/ServicesData/Jenkins/.jenkins:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean
```
这里我加了`--name jenkins`表示我给这个容器取名为jenkins。这样创建的容器的名字就不是随机得了。
另外我删了`--rm`，因为这个表示关闭容器后会自动清理掉容器，也就是说在Docker上找不到了，下次再想开这个容器，还得重新敲命令，所以我就去掉这个。如果我真的想删掉这个容器，也可以直接在Docker容器界面上操作。
最后一个就是我将`jenkins-data`改成了`D:/ServicesData/Jenkins/.jenkins`,首先我们来看这个`-v jenkins-data:/var/jenkins_home`命令是干嘛的，因为Docker就相当与与一个虚拟机，里面有自己的文件管理系统，这里`/var/jenkins_home`就表示当前jenkins容器的工作目录，但是虚拟的毕竟是虚拟的，所以它搞了一个体的概念，就是Volume，专门用来春出数据的，而`jenkins-data`就是其中创建的一个体，是Docker在物理机上构建的一个私有文件目录，我们没办法管理与访问。直接安装Docker是装在C盘的，所以这些体数据，应该也是在C盘（猜的）。为了能够接管这些容器的工作数据，我们可以将虚拟文件目录映射到实际的物理机上，这里我在D盘建立一个目录，也就是说，之后Jenkins容器运行时产生的数据都会放到这个D盘目录下。
通过这个例子，我有点明白了Docker的一个基本运作方式了，打开Docker客户端，出现五个大选项，其中有三个就是容器、镜像、体。另外两个我没有用过，所以也不知道能干啥，但是这三个之间的关系似曾相识：MVC框架，嗯好像又不像。这么说吧，这三者其实是一件事情拆分成三个部分，其中镜像是我们的全局数据中心，然后容器是我们的控制总线，体是控制过程中产生的数据。在需要时，Docker会为我们创建任意个控制总线，相当与是一个壳子，然后向全局数据中心请求生产资料，最终产生的产品堆到各自的仓库里面，也就是体。

继续按照[Post-installation setup wizard](https://www.jenkins.io/zh/doc/book/installing/#setup-wizard)安装。这里我们在浏览器上输入`http://localhost:8080/`，会看到Jenkins的解锁界面，这里需要一个密码，如果安装在物理机下的话，我们能够找到对应的文件，但是现在是装在容器里面，所以要在Docker客户端里面找，我们首先找到Jienkins的容器，然后点击该容器，进入Logs界面，这里就是Jenkins控制台日志输出中，复制自动生成的字母数字密码（在两组星号之间），然后拿到解锁界面解锁。之后的操作就和物理机上安装差不多了。

然后在浏览器里面输入`http://localhost:8000/`就可进入了。
还有一个需要注意的是，如果启动容器后，网页访问不了Jenkins，可能是端口被占用了，换个端口试试。或者如果你知道怎么排查端口占用的情况就更直接了。
