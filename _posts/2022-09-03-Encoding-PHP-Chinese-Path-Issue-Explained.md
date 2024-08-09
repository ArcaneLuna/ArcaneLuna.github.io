---

layout: post
title: 【编码】PHP中文路径问题详解
date: 2022-09-03
author: "qtunneling"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Encoding

---

## 1. 问题

低版本的PHP可能会遇到不支持中文路径的情况：

　(1) require(\'http://localhost/中文路径/test.php\');

　(2) require(\'\\中文路径\\test.php\');

　(3) \$file = fopen(\'http://localhost/中文路径/test.php\');

　(4) \$file = fopen(\'\\中文路径\\test.php\');

　(5) 通过浏览器访问: http://127.0.0.1/中文路径/test.php

在Windows10+Apache2.4.41+PHP5.6.40环境下，测试文件编码为UTF-8，会发现除了用fopen打开URL外(3)，其他情况（包括用fopen打开物理路径）PHP都会报错：No
such file or directory

注：使用fopen(\'url\')或require(\'url\')需要在php.ini中进行配置

      ; Whether to allow the treatment of URLs (like http:// or ftp://) as files.
      ; http://php.net/allow-url-fopen
      allow_url_fopen = On
      ; Whether to allow include/require to open URLs (like http:// or ftp://) as files.
      ; http://php.net/allow-url-include
      allow_url_include = On

查看浏览器请求头，(5)的"中文路径"也被编码为[UTF-8，]{lang="EN-US"}如果对路径字符串进行转码：

　(1) require(iconv(\'utf-8\', \'gbk\',
\'http://localhost/中文路径/test.php\'));

　(2) require(iconv(\'utf-8\', \'gbk\',  \'\\中文路径\\test.php\');

　(3) \$file = fopen(iconv(\'utf-8\', \'gbk\',
\'http://localhost/中文路径/test.php\'));

　(4) \$file = fopen(iconv(\'utf-8\', \'gbk\', 
\'\\中文路径\\test.php\');

　(5) 通过浏览器访问:
[http://127.0.0.1/php/%D6%D0%CE%C4%C2%B7%BE%B6/form.php]{lang="EN-US"}

发现：(2)和(4)成功运行，(1)，(3)，(5)报错，但这次不是PHP报错，而是Apache报错：HTTP/1.1
403 Forbidden

+-------------+-------------+-------------+-------------+-------------+
| ** **       | **物理路    | **物理      | **U         | *           |
|             | 径(utf-8)** | 路径(gbk)** | RL(utf-8)** | *URL(gbk)** |
+-------------+-------------+-------------+-------------+-------------+
| **fopen**   | Invalid     | 成功        | 成功        | HTTP/1.1    |
|             | argument/   |             |             | 403         |
|             |             |             |             | Forbidden   |
|             | No such     |             |             |             |
|             | file or     |             |             |             |
|             | directory   |             |             |             |
+-------------+-------------+-------------+-------------+-------------+
| **require** | Invalid     | 成功        | Invalid     | HTTP/1.1    |
|             | argument/   |             | argument/   | 403         |
|             |             |             |             | Forbidden   |
|             | No such     |             | No such     |             |
|             | file or     |             | file or     |             |
|             | directory   |             | directory   |             |
+-------------+-------------+-------------+-------------+-------------+

据此得出结论：

- 用require和fopen打开URL，会向Apache服务器请求资源，但**Apache只支持解析UTF-8编码的URL**
- 用require和fopen打开物理路径，直接与Windows文件系统交互，即ANSI编码

注：require会调用compile_filename，而在compile_filename会进行路径解析。

但是，为什么用浏览器或者require打开UTF-8或者GBK编码的URL都会失败呢？

## 2. PHP与Apache主要通信方式

### 2.1 CGI方式

CGI英文叫做公共网关接口，就是Apache在遇到PHP脚本的时候会将PHP程序提交给**CGI应用程序（php-cgi.exe）**解释，解释之后的结果返回给Apache，然后再返回给相应的请求用户。这种模式速度慢(php解释器不是常驻内存的)、占用空间(请求一次开启一次cgi)。已经被fast-cgi代替。

在Apache中配置：

Action application/x-httpd-php "/php/php-cgi.exe"

### 2.2 模块化方式（默认）

PHP作为Apache的一个模块（mod_php）运行，随apache一起启动，**属于一个进程，之间的通信是内部通信。**mod_php
这种嵌入的方式最大的弊端就是内存占用大，不论是否用到 PHP
解释器都会将其加载到内存中，典型的就是处理CSS、JS之类的静态文件是完全没有必要加载解释器。

![](https://img2022.cnblogs.com/blog/1632897/202211/1632897-20221126084542394-711810570.webp)

 

### 2.3 FastCGI方式

这种形式是CGI的加强版本，CGI是单进程，多线程的运行方式，程序执行完成之后就会销毁，所以每次都需要加载配置和环境变量fork-and-execute（创建-执行）。而FastCGI则不同，FastCGI
像是一个常驻 (long-live) 型的
CGI，它可以一直执行着，只要激活后，不会每次都要花费时间去 fork
一次。FastCGI进程管理器自身初始化，启动多个CGI解释器进程
(在任务管理器中可见多个php-cgi.exe)并等待来自Web
Server的连接**。FastCGI与Apache是两个独立的进程，通过端口通信。**

它的具体实现到php中就是php的php-fpm模块，但是在apache中是用的专门的fastcgi模块，需要下载.so文件，php-fpm在php5.3以后不再作为第三方的模块而是集成到了php中，它会提前的开启多个cgi程序，管理这些进程，并提供方式合理有效的调度，保证了并发性。

## [3. Apache与PHP交互过程]{lang="EN-US"}

### 3.1 Apache生命周期

![](https://img2022.cnblogs.com/blog/1632897/202211/1632897-20221126084823492-1747456133.webp){style="display: block; margin-left: auto; margin-right: auto;"}

### 3.2 Apache处理流程

![](https://img2022.cnblogs.com/blog/1632897/202211/1632897-20221126084846390-1904258155.webp){style="display: block; margin-left: auto; margin-right: auto;"}

 

1. Post-Read-Request阶段:
   在正常请求处理流程中，这是模块可以插入钩子的第一个阶段。对于那些想很早进入处理请求的模块来说，这个阶段可以被利用。
2. URI Translation阶段 : 
   Apache在本阶段的主要工作：**将请求的URL映射到本地文件系统**。模块可以在这阶段插入钩子，执行自己的映射逻辑。mod_alias就是利用这个阶段工作的。
3. Header Parsing阶段 : 
   Apache在本阶段的主要工作：检查请求的头部。由于模块可以在请求处理流程的任何一个点上执行检查请求头部的任务，因此这个钩子很少被使用。mod_setenvif就是利用这个阶段工作的。
4. Access Control阶段 : 
   Apache在本阶段的主要工作：**根据配置文件检查是否允许访问请求的资源**。Apache的标准逻辑实现了允许和拒绝指令。mod_authz_host就是利用这个阶段工作的。
5. Authentication阶段 : 
   Apache在本阶段的主要工作：按照配置文件设定的策略对用户进行认证，并设定用户名区域。模块可以在这阶段插入钩子，实现一个认证方法。
6. Authorization阶段 : 
   Apache在本阶段的主要工作：根据配置文件检查是否允许认证过的用户执行请求的操作。模块可以在这阶段插入钩子，实现一个用户权限管理的方法。
7. MIME Type Checking阶段 : 
   Apache在本阶段的主要工作：根据请求资源的MIME类型的相关规则，判定将要使用的内容处理函数。标准模块mod_negotiation和mod_mime实现了这个钩子。
8. FixUp阶段 : 
   这是一个通用的阶段，允许模块在内容生成器之前，运行任何必要的处理流程。和Post_Read_Request类似，这是一个能够捕获任何信息的钩子，也是最常使用的钩子。
9. Response阶段 :
   Apache在本阶段的主要工作：生成返回客户端的内容，负责给客户端发送一个恰当的回复。这个阶段是整个处理流程的核心部分。
10. Logging阶段 : 
    Apache在本阶段的主要工作：在回复已经发送给客户端之后记录事务。模块可能修改或者替换Apache的标准日志记录。
11. CleanUp阶段 :
    Apache在本阶段的主要工作：清理本次请求事务处理完成之后遗留的环境，比如文件、目录的处理或者Socket的关闭等等，这是Apache一次请求处理的最后一个阶段。

### 3.3 mod_php工作周期

1. Apache接受请求
2. Apache传递请求给mod_php
3. **mod_php定位磁盘文件，并加载到内存中**
4. mod_php编译源代码成为opcode树（APC）
5. mod_php执行opcode树

## 4. 结论

### 4.1 两次资源检查

根据上述工作流程可大致推断：

1. 通过浏览器访问url产生一个request，Apache服务器会检查是否允许访问请求的资源，要求url是UTF-8编码；
2. mod_php会通过Apache传递的请求定位磁盘文件，这里可能要求是GBK编码，因此PHP报错（仅猜测，具体Apache传递给PHP的请求是什么格式暂时不清楚）；

### 4.2 远程文件

对于require(url)，这个过程可能更加复杂：

参照：[PHP: include -
Manual](https://www.php.net/manual/zh/function.include.php)

> **安全警告**
> 
> 远程文件可能会经远程服务器处理（根据文件后缀以及远程服务器是否在运行
> PHP 而定），但必须产生出一个合法的 PHP
> 脚本，因为其将被本地服务器处理。

根据这条警告的意思，如果远程服务器在运行PHP，通过include/require请求的PHP文件可能会被处理，然后收到一个包含了静态的HTML的PHP脚本，如果远程服务器没有运行PHP，那么将收到原始的PHP脚本在本地进行执行。由于测试请求的是本地的PHP文件，远程服务器就相当于本地环境，远程服务器同样进行了两次资源检查导致出错。

### 4.3 替代解决方案

设置-\>语言设置-\>管理语言设置-\>更改系统区域设置-\>勾选Beta版使所有程序都用Unicode
UTF-8提供语言支持，但在运行其他文件时可能出现乱码。

### 4.4 使用PHP7

如果使用PHP7，以上问题全都不会出现。

## 参考

1. [Apache运行机制 - 简书
   (jianshu.com)](https://www.jianshu.com/p/5187a3506bd3)
2. [POST-READ-REQUEST阶段 - 豆丁网
   (docin.com)](https://www.docin.com/p-222908664.html)
3. [php底层深度探索(4) \-\--Apache运行阶段分析 王泽宾 - 编程宝库 -
   博客园
   (cnblogs.com)](https://www.cnblogs.com/wanghao72214/archive/2009/02/21/1395566.html)
4. [apache是怎么运行php的_PHP与WEB服务器是如何交互的_weixin_39761481的博客-CSDN博客](https://blog.csdn.net/weixin_39761481/article/details/110516476)
5. [Windows下Apache服务器运行PHP的三种运行方式（php_mod、cgi、fastcgi）
   \| 夜风博客 (homedt.net)](https://www.homedt.net/12281.html)
6. [(5条消息) 深入理解php原理之include include_once require
   require_once_冰心丹的博客-CSDN博客](https://blog.csdn.net/zm_bingxindan/article/details/45654565)

 
