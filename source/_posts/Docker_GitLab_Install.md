---
title: GitLab Install
date: 2022-06-27 15:03:00
categories:
- [Docker]
tags:
- Docker
- GitLab
---

在Win10上安装完Docker后，就可以在Docker上安装GitLab了。
参考[TaylorShi](https://www.cnblogs.com/taylorshi/p/13585821.html)和[windows下使用docker安装gitlab](https://www.pudn.com/news/627b28dbebb030486da7d323.html)两篇教程，以后买这篇为主。单独根据其中某一篇我都没有安装成功。

这里记录一下最终采用的步骤：
- 首先打开命令行窗口，然后输入`docker pull gitlab/gitlab-ce:latest`回车，会开始拉取GitLab镜像，拉去后我们可以在Docker的Image里面找到。
- 然后执行`docker run -d --publish 9980:80 --publish 9443:443 --publish 9922:22 --name gitlab --restart always --volume D:/ServicesData/GitLab/config:/etc/gitlab --volume D:/ServicesData/GitLab/logs:/var/log/gitlab --volume D:/ServicesData/GitLab/data:/var/opt/gitlab gitlab/gitlab-ce`这行命令。会在Docker里面创建容器，这里有一个`-d`的参数可以加也可以不加，但是不加的话，创建后的容器就是跟这个控制台窗口绑定了，关闭控制台窗口也就等于关闭GitLab容器。这里我是加了，表示容器在后台运行。之后就可以在Docker的容器目录里面找到。另外就是`--volume *:/var/*`，冒号前面的表示实际的路径，冒号后面的表示虚拟的映射路径，这我将他们映射到D盘下。
- 然后点击[http://127.0.0.1:9980/](http://127.0.0.1:9980/)跳转到我们搭建的GitLab的登录界面了(第一次可能会跳转失败，要多等一下，多试几次)。但是我们不知道账号密码，我们可以再次进入命令行窗口，输入 `docker exec -it gitlab bash`，进入linux的bash命令行，然后在输入`gitlab-rails console -e production`，然后是`user=User.where(id:1).first`查看账号是多少，然后`user.password = 123456789`，以及`user.password_confirmation = 123456789`修改密码，这里`123456789`是新密码，你也可以设置为其他。再使用`user.save!`命令保存设置，然后输入`exit`退出。刚刚修改密码的操作好像是Linux环境下的命令（ps:我不知道，我猜的），除了在这里修改，还可以在Docker界面上，里面每个容器右侧都有对应的terminal，可以直接输入。
下面是修改密码的命令行输入输出：
```bash
(c) Microsoft Corporation。保留所有权利。
C:\Users\10524>docker ps
              PORTS                                                               NAMES
f70589c630be   gitlab/gitlab-ce   "/assets/wrapper"        23 minutes ago   Up 23 minutes (healthy)   0.0.0.0:9922->22/tcp, 0.0.0.0:9980->80/tcp, 0.0.0.0:9443->443/tcp   gitlab
9af9c4c71518   jenkins/jenkins    "/usr/bin/tini -- /u…"   5 hours ago      Up 5 hours                50000/tcp, 0.0.0.0:8000->8080/tcp                                   jenkins

C:\Users\10524>docker exec -it gitlab bash
root@f70589c630be:/# gitlab-rails console -e production


--------------------------------------------------------------------------------
 Ruby:         ruby 2.7.5p203 (2021-11-24 revision f69aeb8314) [x86_64-linux]
 GitLab:       15.3.1 (16f34bd10fc) FOSS
 GitLab Shell: 14.10.0
 PostgreSQL:   13.6
------------------------------------------------------------[ booted in 26.85s ]
Loading production environment (Rails 6.1.6.1)
irb(main):001:0>
irb(main):002:0>
irb(main):003:0> user=User.where(id:1).first
=> #<User id:1 @root>
irb(main):004:0> user.password = 123456789
=> 123456789
irb(main):005:0> user.password_confirmation = 123456789
=> 123456789
irb(main):006:0> user.save
=> true
irb(main):007:0> user.save!
=> true
irb(main):008:0>
```

> 补充，我发现[Docker 搭建 Gitlab 服务器 (完整详细版)](https://blog.csdn.net/BThinker/article/details/124097795)教程好像更完整。但是关于修改IP的那些操作我都没有成功，可能是应该我在本地搭的服务吧。我这里说的未成功是只我想改成任意Ip没有成功，因为最终GitLab网站都是通过`http://127.0.0.1:9980/`或者`http://localhost:9980/`访问的。但是这一步还是能少，只不过我们将修改的IP选择为`127.0.0.1`而已，因为如果不修改的话，GetLab上的仓库的路径URL会是一个随机数，所以没办法正确索引到正确的git路径。

## 参考

[Vim命令合集](https://www.jianshu.com/p/117253829581)