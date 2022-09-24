---
title: GitLab Install
date: 2022-08-29 17:03:00
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

> 补充，我发现[Docker 搭建 Gitlab 服务器 (完整详细版)](https://blog.csdn.net/BThinker/article/details/124097795)教程好像更完整。其中修改IP那一步，我一直没有搞成功，后面突然有一次成功了，那一次我是用本机IP。之前没成功的，都是随便选的IP。甚至我还选了`127.0.0.1`，因为本地打开网页就是显示这个IP，修改后，网页正常打开等操作都正常，甚至本地拉去也正常。但是当使用Jenkins去关联GitLab路径时始终，都连接失败，后面IP改成本机实际的IP才成功。如下图：
![wwwwwww](/blogs/images/src/Snipaste_2022-08-29_23-18-45.png)
> gitlab.rb
```bash
external_url 'http://192.**.***'
gitlab_rails['gitlab_ssh_host'] = '192.168.**.***'
gitlab_rails['gitlab_shell_ssh_port'] = 9922

## GitLab configuration settings
```
> gitlab.yml
```bash
  ## GitLab settings
  gitlab:
    ## Web server settings (note: host is the FQDN, do not include http://)
    host: 192.168.'**.***'
    port: 9980
    https: false
```

## 重启后port自动改了

当我第二天重启打开电脑，启动我的Gitlab时，我发现进不去了，因为`gitlab.yml`中的端口号又自动变回80。为了让原本已经配置好的Gitlab正常运行，需要重新修改端口号为9980。
但是直接修改`gitlab.yml`并不行。需要在gitlab运行的时候，然后按照下面的步骤：
- 打开命令行窗口
- 输入`docker exec -it gitlab /bin/bash`，并运行，进入docker命令下。
- 输入`vi /etc/gitlab/gitlab.rb`，在vi里面修改`gitlab.rb`Ip，参考上面的代码。或者直接在本地找到`gitlab.rb`，使用本地文本编辑器修改。
- 输入`gitlab-ctl reconfigure`，这个命令会自动修改`gitlab.yml`的ip，但是端口号仍然是80。
- 输入`vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml`,在vi里面修改`gitlab.yml`端口，参考上面的代码。或者直接在本地找到`gitlab.yml`，使用本地文本编辑器修改。
- 输入`gitlab-ctl restart`,这个命令会让最终修改的端口生效。
- 最后输入`exit`，退出docker命令。

## 参考

[Vim命令合集](https://www.jianshu.com/p/117253829581)