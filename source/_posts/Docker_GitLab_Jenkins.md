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
- 首先在Jenkins的插件安装界面找到`Publish Over SSH`并安装。安装完成后进入Jenkins的系统配置，拉到最下方应该可以看到`Publish over SSH`的配置，

## 参考

[docker+gitlab+jenkins从零搭建自动化部署](https://blog.csdn.net/weboof/article/details/104491998)