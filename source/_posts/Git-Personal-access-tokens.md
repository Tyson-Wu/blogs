---
title: Git Personal Access Tokens
date: 2022-04-4 15:01:00
categories:
- [Git]
tags:
- Git
---


最近使用GitHub提交项目时，发现原先的账号密码的方式已经失效，GitHub官方表示不再支持使用账号密码的方式，而是改为使用`Personal Access Tokens`访问令牌的方式。
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-error.webp)

`Personal Access Tokens`是通过在GitHub上创建令牌，并且设置令牌的使用时限，来限制访问者的访问权限。也可以直接删除令牌来清除访问者的访问权限。而令牌的作用域时针对整个GitHub账号下的所有项目。所以创建方式也是在GitHub下的总Setting下设置。创造令牌的步骤如下：
1：登录GitHub账号；
2：选择右上角的头像图标 `-> Profile Pic -> Setting`；
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-settings.webp)
3：然后在Setting页面左侧菜单栏里面选择`Developer Settings`；
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-developer-settings.webp)
4：然后在`Developer Settings`页面上选择`Personal Access Token `选项；
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-generate.webp)
5：然后选择`Generate New Token`按钮开始创建令牌;
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-generate-new-token.webp)
6：给令牌选择一个名字，令牌的使用时限，以及令牌的访问权限范围；
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-new-scopes.webp)
7：然后点击`Generate Token`生成令牌；
8：生成的令牌是一串字符串，只有在令牌生成的时候才能看到令牌明码，我们需要复制这个字符串，后续再打开这个令牌界面的时候，就看不到令牌字符串了；
![](https://jhooq.com/wp-content/uploads/ssl/git-personal-access-token/git-personal-access-token-copy-token.webp)
拿到令牌字符串后，我们就可以使用这个令牌来clone或者push该GitHub账号下的项目了。使用令牌时，我们需要使用令牌字符串来拼接git资源路径。
下面以[https://github.com/Tyson-Wu/Unity_Jenkins](https://github.com/Tyson-Wu/Unity_Jenkins)这个项目为例。通常我们clone时直接使用一下命令:
```sh
git clone https://github.com/Tyson-Wu/Unity_Jenkins.git
```
但是这个命令只能拷贝资源，拷贝后并不能上传修改后的资源。想要拥有上传权限，那就要用令牌来修改访问权限了。而拼接的规则如下：
```sh
https://<username>:<personal_access_token>@github.com/<username>/<project_name>.git
```
假设此时我创建的令牌字符串为`ghp_7qb6ZektsWIFkHzJiaL4gH2GKzNUqD3P8ZoA`,对于上面子资源，拼接后的路径应该是：
```sh
git clone https://ghp_7qb6ZektsWIFkHzJiaL4gH2GKzNUqD3P8ZoA@github.com/Tyson-Wu/Unity_Jenkins.git
```
当拷贝完后，我们进入到Unity_Jenkins的本地拷贝根目录下，打开git命令行，然后修改它的orgion。
```sh
git remote set-url origin https://ghp_7qb6ZektsWIFkHzJiaL4gH2GKzNUqD3P8ZoA@github.com/Tyson-Wu/Unity_Jenkins.git
```
这时候就设置好了访问权限，假设我们本地文件修改后想提交，直接使用`Push`命令就行了。
```sh
git push
```