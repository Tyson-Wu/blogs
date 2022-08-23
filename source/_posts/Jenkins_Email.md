---
title: Config Jenkins Email
date: 2022-8-23 19:01:34
categories:
- [Jenkins]
tags:
- Jenkins
- Email
---

## 记录

自动化部署是项目开发中经常用到的，当部署完成后最好能够通知到相关人员部署情况，这里记录基于Jenkins的邮件通知配置。
这里假设已经安装部署好了Jenkins，并且创建了一个叫`hello world`的部署项目。
![wwwwwww](/blogs/images/src/Snipaste_2022-08-24_00-31-11.png)
关于邮件通知配置，只需要配两个地方，一个是`Manage Jenkins->System Configuration->Configure System`,这个是总的配置。另一个是`hello world->配置`，这个主要是针对特定项目的邮件设置。
在`Manage Jenkins->System Configuration->Configure System`这里的设置，主要是针对`Extended E-mail Notification`和`邮件通知`两部分的设置。具体设置如下：
![wwwwwww](/blogs/images/src/Snipaste_2022-08-24_00-47-04.png)
上面主要是设置了用于发送系统消息的邮箱账号，邮箱的类型，授权码等信息。这里我使用的是QQ邮箱，QQ邮箱的授权码获取方式可以参考这里[开启邮箱的SMTP服务获取授权码(QQ邮箱、163邮箱)](https://blog.csdn.net/xiaochenXIHUA/article/details/123358350)
为了让邮件看起来比较正式，相应的邮件格式可以配置一下，这里采用HTML的例子，直接配到`Default Content`里面。具体内容如下：

```HTML
<!DOCTYPE html>    
<html>    
<head>    
<meta charset="UTF-8">    
<title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>    
</head>    
<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"    
    offset="0">    
    <table width="95%" cellpadding="0" cellspacing="0"  style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">    
        <tr>    
            本邮件由系统自动发出，无需回复！<br/>            
            各位同事，大家好，以下为${PROJECT_NAME }项目构建信息</br> 
            <td><font color="#CC0000">构建结果 - ${BUILD_STATUS}</font></td>   
        </tr>    
        <tr>    
            <td><br />    
            <b><font color="#0B610B">构建信息</font></b>    
            <hr size="2" width="100%" align="center" /></td>    
        </tr>    
        <tr>    
            <td>    
                <ul>    
                    <li>项目名称 ： ${PROJECT_NAME}</li>    
                    <li>构建编号 ： 第${BUILD_NUMBER}次构建</li>    
                    <li>触发原因： ${CAUSE}</li>    
                    <li>构建状态： ${BUILD_STATUS}</li>    
                    <li>构建日志： <a href="${BUILD_URL}console">${BUILD_URL}console</a></li>    
                    <li>构建  Url ： <a href="${BUILD_URL}">${BUILD_URL}</a></li>    
                    <li>工作目录 ： <a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>    
                    <li>项目  Url ： <a href="${PROJECT_URL}">${PROJECT_URL}</a></li>    
                </ul>    

<h4><font color="#0B610B">失败用例</font></h4>
<hr size="2" width="100%" />
$FAILED_TESTS<br/>

<h4><font color="#0B610B">最近提交(#$SVN_REVISION)</font></h4>
<hr size="2" width="100%" />
<ul>
${CHANGES_SINCE_LAST_SUCCESS, reverse=true, format="%c", changesFormat="<li>%d [%a] %m</li>"}
</ul>
详细提交: <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a><br/>

            </td>    
        </tr>    
    </table>    
</body>    
</html>
```

在`hello world->配置`这里的设置，主要是针对`构建后操作`的设置。选择`Editable Email Notification`选项。配置可以保持默认，但是如果希望不同的项目发送邮件的对象、内容、发送触发条件不同的话，就需要在这里修改。

## 参考

[jenkins邮件配置和邮件发送](https://blog.csdn.net/weixin_45878889/article/details/123925316)
[开启邮箱的SMTP服务获取授权码(QQ邮箱、163邮箱)](https://blog.csdn.net/xiaochenXIHUA/article/details/123358350)
[Unity和Jenkins真是绝配，将打包彻底一键化！](https://www.cnblogs.com/wuzhang/p/20190512wuzhang.html)
[如何搭建Jenkins导出Unity安卓环境](http://dingxiaowei.cn/2018/11/17/)