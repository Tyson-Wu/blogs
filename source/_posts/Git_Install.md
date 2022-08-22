---
title: 4.1 Git Install
date: 2022-8-22 19:01:34
categories:
- [Git, Tortoisegit]
tags:
- Git
- Tortoisegit
---

Git安装其实没什么好讲的，但是单纯只装Git基本上只能使用命令行，当然Git也有客户端，但是功能有限。
所以通常我们会一同安装Tortoisegit软件，作为Git的图形交互界面。下面地址是一位大佬写的安装流程，涉及到如何配置等操作。
[下载安装Git及Tortoisegit](https://www.cnblogs.com/anayigeren/p/10177027.html)
因为Git提供HTTPS、SSH两种方式来提交拉取Git资源，而文中提到`通过使用 SSH URL 来提交代码便可以一劳永逸了`。所以这里打算采用SSH的方式，下面的方法就是SSH的方式。

>当然如果想使用HTTPS的方式，可以看这里:
>[Git Personal Access Tokens](https://tyson-wu.github.io/blogs/2022/04/04/Git-Personal-access-tokens/)

但是使用上面SSH的方式，在从git服务拉取项目时，会报`No supported authentication methods available`的错误。这里有一篇文章讲述了如何解决这个问题：
[TortoiseGit提示No supported authentication methods available错误](https://blog.csdn.net/Jeffxu_lib/article/details/112259246)
