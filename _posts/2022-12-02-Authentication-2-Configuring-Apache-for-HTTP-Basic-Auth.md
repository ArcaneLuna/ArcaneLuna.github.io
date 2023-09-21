---

layout: post
title: 【认证机制】2-Apache配置HTTP Basic Auth
date: 2022-12-02
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Authentication

---

## 1. 基本流程

### 1.1 使用htpasswd命令创建用户文件

进入 apache 安装目录，使用 htpasswd.exe 创建用户
Admin（密码：password），保存在
apache_auth.htpasswd中（生成文件可以是任意名称和格式，比如user.txt）

- htpasswd用法：[htpasswd - Manage user files for basic
  authentication - Apache HTTP Server Version
  2.4](https://httpd.apache.org/docs/current/en/programs/htpasswd.html)
- 密码格式为加盐哈希：[Password Formats - Apache HTTP Server Version
  2.4](https://httpd.apache.org/docs/current/en/misc/password_encryptions.html)

![](/img/2022-12-02-Authentication-2-Configuring-Apache-for-HTTP-Basic-Auth/1.png){width="654"
height="96"}

### 1.2 修改主配置（httpd.conf）或者.htaccess文件

加入如下内容：

- AuthType Basic
- AuthName \"Access to the staging site\"
- AuthUserFile /path/to/.htpasswd
- Require valid-user

AuthType 表示认证方式，这里是 basic 认证

AuthName 表示弹出框给出的提示文字，自己定义即可

AuthUserFile 表示认证用户文件的路径

AuthGroupFile
定义包含用户组和组成员的文本文件。组成员用空格分开，如：group1:user1
user2

Require 定义哪些用户或组才能被授权访问，valid-user
表示在AuthUserFile指定的文件中的所有用户都可以访问

如果是在httpd.conf中：

    <Directory "D:\wamp64\www\basic_auth">
           AuthType Basic
           AuthName "Login"
           AuthUserFile     
           "D:\wamp64\www\basic_auth\apache_auth.htpasswd"
           Require valid-user
    </Directory>

　　修改完成后，重启服务器即可启用Basic Auth.

## 2. 关于.htaccess文件

### 2.1 简介

.htaccess文件(或者\"分布式配置文件\"）,全称是Hypertext
Access(超文本入口)。提供了针对目录改变配置的方法，
即，在一个特定的文档目录中放置一个包含一个或多个指令的文件，
以作用于此目录及其所有子目录。作为用户，所能使用的命令受到限制。管理员可以通过Apache的AllowOverride指令来设置。

参考：[htaccess_百度百科
(baidu.com)](https://baike.baidu.com/item/htaccess/1645473?fr=aladdin)

### 2.2 开启.htaccess

- [apache下htaccess的功能与开启htaccess方法 \| 感触life-博客
  (wymlw.cn)](http://blog.wymlw.cn/blog/?p=53)
- [apache .htaccess文件详解和配置技巧总结 - 走看看
  (zoukankan.com)](http://t.zoukankan.com/wumingcong-p-5044713.html)
- [apache2开启/使能.htaccess文件的方法 - 思念殇千寻 - 博客园
  (cnblogs.com)](https://www.cnblogs.com/chester-cs/p/13851808.html)
- [.htaccess文件首行options +followsymlinks作用 - 走看看
  (zoukankan.com)](http://t.zoukankan.com/cnsec-p-11515931.html)
- [\[翻译\]FollowSymLinks - Fogwind - 博客园
  (cnblogs.com)](https://www.cnblogs.com/fogwind/p/15261776.html)
- [(4条消息) Apache的Order Allow,Deny
  规则_断魂蓝桥的博客-CSDN博客](https://blog.csdn.net/sinat_22319877/article/details/50968103)
- [(4条消息)
  AllowOverride_北方的刀郎的博客-CSDN博客_allowoverride](https://blog.csdn.net/forest_fire/article/details/50943338)

.htaccess文件十分方便，但为什么默认是不开启的？百科写的很详细了：

### 2.2 采用.htaccess文件的优缺点

- 通常网络管理员采用.htaccess文件来进行用户组的目录权限访问控制。没有必要将所有的HTTPd服务器、配置文件以及目录访问权限全部授权给管理员。**利用当前目录的.htaccess文件可以允许管理员灵活的随时按需改变目录访问策略**。
- 采用.htaccess的缺点在于：当系统有成百上千个目录，每个目录下都有对应的.htaccess文件时，网络管理员将会对**如何配置全局访问策略无从下手**。同时，由于**.htaccess文件十分被容易覆盖**，很容易造成用户上一时段能访问目录，而下一时段又访问不了的情况发生。最后，**.htaccess文件也很容易被非授权用户得到，安全性不高**。

### 2.3 不推荐使用.htaccess的理由

- 在/www/htdocs/example目录下的.htaccess文件中放置指令，与在主配置文件中\<Directory
  /www/htdocs/example\>段中放置相同指令，效果上是完全等效的。但在性能上，把配置放在主配置文件中更加高效，因为只需要在Apache启动时读取一次但启用，而.htaccess文件不仅在每次文件请求时都读取，而且需要逐个目录查找.htaccess文件（从根目录一直到文件目录,
  **子目录中的指令会覆盖父目录或者主配置文件中的指令**），造成严重的性能下降。
