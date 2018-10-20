---
title: '"linux上发送邮件"'
categories: linux
tags: linux
date: 2018-10-14 11:26:37
---


####1.  首先需要安装mailx
>yum install mailx

####2.  编辑配置文件

>vim /etc/mail.rc


添加如下内容  
```
//from：对方收到邮件时显示的发件人  
set from=xxxx@qq.com  
//smtp：指定第三方发邮件的smtp服务器地址  
set smtp=smtp.qq.com  
//smtp-auth-user：第三方发邮件的用户名  
set smtp-auth-user=xx@qq.com  
//smtp-auth-password：用户名对应的密码,有些邮箱填的是授权码  
set smtp-auth-password=xxx  
//smtp-auth：SMTP的认证方式，默认是login，也可以改成CRAM-MD5或PLAIN方式  
set smtp-auth=login
```
<!--more-->
####3. 测试  
>mail -s "hello word" xxxx@qq.com < /etc/passwd  
echo "测试邮件" | mail -s "测试" xx@qq.com

* **另外注意**
smtp协议要求服务器25端口是开通的，如果是购买vultr服务器的，需要另外向供应方去申请端口。国外的防止垃圾邮件规则非常麻烦，作者在申请的时候就需要等待14天的考察期，没问题后才被开启。
