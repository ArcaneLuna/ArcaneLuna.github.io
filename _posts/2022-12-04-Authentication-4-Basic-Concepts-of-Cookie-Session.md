---

layout: post
title: 【认证机制】4-Cookie-Session基本概念
date: 2022-12-04
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Authentication

---

## **1.  ** **Cookie**

### 1.1 概述

Cookie是一种在远程[**浏览器端**]{style="background-color: #ffff00; color: #ff0000;"}存储数据并以此来**跟踪**和**识别**用户的机制。

Cookie是Web服务器暂时存储在用户硬盘上的一个**文本文件**，当用户再次访问Web网站时，网站通过读取Cookie文件**记录这位访客的特定信息**（如上次访问的信息、花费的事件、用户名和密码等），从而迅速做出响应。（如重新打开浏览器后不需要再输入用户的ID和密码即可直接访问）

在Cookies文件夹下，每个Cookie文件都是一个简单而普通的**文本文件**，而不是程序。Cookies中的内容大多经过了加密处理，因此，表面看来只是一些字母和数字组合，而只有服务器的CGI处理程序才知道它们的真正含义。

文本文件格式：用户名@网站地址.txt

### 1.2 功能

①记录访客的某些信息。如可以利用Cookie记录用户访问网页的次数，或者记录访客曾经输入过的信息，另外，某些网站可以使用Cookie自动记录访客上次登录的用户名。

②在页面之间传递变量。浏览器并不会保存当前页面上任何变量信息，当页面被关闭时页面上的所有变量信息将随之消失。如果用户声明一个变量id=8，要把这个变量传递到另一个页面，可以把变量id以Cookie形式保存下来，然后在下一页通过读取该Cookie来获取变量的值。

③将所查看的Internet页存储在Cookies临时文件夹中，提高以后的浏览速度。

### 1.3 会话Cookie和持久Cookie

会话Cookie将Cookie存在在浏览器内存，浏览器关闭后Cookie即被销毁。

持久Cookie保存在客户端硬盘，下次打开浏览器可以继续使用。

实际上两者的区别在于Cookie的Max-Age或Expires字段不同，默认Max-Age为-1，表示浏览器关闭会清除Cookie（会话Cookie）。

注：设置Max-Age为0会删除Cookie，无论Cookie在浏览器内存还是在硬盘中。

## **2.  ** **Session**

### 2.1 概述

Session，会话，本义是指有始有终的一系列动作/消息，如打电话时从拿起电话拨号到挂断电话这一系列过程可以称为一个Session。在计算机专业术语中，Session是指一个终端用户与交互系统进行通信的时间间隔，通常指从注册进入系统到注销退出系统所经过的时间。

Session在Web技术中非常重要。由于互联网应用层协议大多是基于无状态的HTTP和HTTPS的，因此无法得知用户的**浏览状态**。通过Session可记录用户的有关信息，以供用户再次以此身份对Web服务器提交要求时作确认。例如，在电子商务网站中，通过Session记录用户的登录信息，以及购买的商品，如果没有Session，那么用户每进入一个页面都需要登录一次用户名和密码。

**session一般来说要配合cookie使用**，如果用户浏览器禁用了cookie，那么可以使用**URL重写**来实现session的存储功能。单纯的使用session来存储用户回话信息，那么当用户量较多时，session文件数量会很多，会存在session查询慢的问题。

本质上：[session技术就是一种**基于[后端]{style="color: #ff0000;"}有别于数据库**的**临时存储技术**]{style="background-color: #ffff00;"}

### 2.2 工作流程

当用户访问到一个服务器，如果服务器启用Session，服务器首先检查用户请求中是否包含session_id（即头部Cookie字段或URL），如果否，则为该用户创建一个session对象（session本身是一个map集合），同时生成一个随机且唯一的session_id，然后通过set-cookie返回给客户端；如果是，说明之前用户已经登录并创建过Session，服务器按照session_id从内存中查找Session文件（文件名为session_id，值为当前用户的一些信息）。

### 2.3 Session与Cookie的区别

**（1）数据存放位置不同：**

cookie数据存放在客户的浏览器上，session数据放在服务器上。

**（2）安全程度不同：**

cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗,考虑到安全应当使用session。

**（3）性能使用程度不同：**

session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能,考虑到减轻服务器性能方面，应当使用cookie。

**（4）数据存储大小不同：**

单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie，而session则存储与服务端，浏览器对其没有限制。

**（5）会话机制不同：**

session会话机制：session会话机制是一种服务器端机制，它使用类似于哈希表（可能还有哈希表）的结构来保存信息。

cookies会话机制：cookie是服务器存储在本地计算机上的小块文本，并随每个请求发送到同一服务器。
Web服务器使用HTTP标头将cookie发送到客户端。在客户端终端，浏览器解析cookie并将其保存为本地文件，该文件自动将来自同一服务器的任何请求绑定到这些cookie。

### 2.4 Session的生命周期

【Session何时生效】

Sessinon在用户访问第一次访问服务器时创建，需要注意只有访问JSP、Servlet、PHP等文件时才会创建Session，只访问HTML、IMAGE等静态资源并不会创建Session,可调用request.getSession(true)强制生成Session。

【Session何时失效】

（1）服务器会把长时间没有活动的Session从服务器内存中清除，此时Session便失效。Tomcat中Session的默认失效时间为20分钟。从session不活动的时候（**最后访问时间**）开始计算，**如果session一直活动，session就总不会过期**。从该Session未被访问,开始计时;
一旦Session被访问,计时清0;

（2）设置session的失效时间（setcookie和session_set_cookie_params）；

（3）销毁session

- JAVA: session.invalidate()
- PHP: session.unset(); session.destroy()

### 2.5 误区

关闭浏览器并不会使得会话消失，造成这种错觉是因为使用了会话Cookie保存会话ID，浏览器关闭后会话ID丢失，从而导致再次和服务器连接时无法找到对应会话。

## 3. PHP SESSION

### 3.1 启动对话

（1）session_start()
会判断当前**\$\_COOKIE\[session_name()\]**是否存在。

- 如果不存在，会生成一个session_id，在session.save_path创建名为\'SESS\_\'.session_id()的文件，然后把生成的session_id作为COOKIE的值传递到客户端，相当于执行了：

[setcookie(session_name(), session_id(), session.cookie_lifetime,
session.cookie_path,
session.cookie_domain, [session.cookie_secure[, [[[session.cookie_httponly]{.string}]{.keyword}]{.default}]{.keyword}]{.string})]{style="background-color: #800000; color: #ffffff;"}

#当然，无法直接使用setcookie，因为此时session_id()为空

#在session_start()前，可以使用session_set_cookie_params设置lifetime、path、secure等参数

- 如果存在，则[session_id
  =\$\_COOKIE\[session_name()\]]{style="background-color: #800000; color: #ffffff;"}，然后从session.save_path查找名为\'SESS\_\'.session_id()的文件，读取文件的内容反序列化，放到\$\_SESSION中

可以看看 [这里](https://www.php.net/manual/zh/function.session-start.php#123917){target="_blank"} 的一段代码：

> The function my_session_start() does almost the same thing as
> session_start().

注：首次生成session_id后，\$\_COOKIE\[session_name()\]为空。这个因为浏览器第一次请求还没有在\$\_COOKIE中存储session_id，只是在客户端创建了cookie文件，这个cookie的一个特性，**只有当下一次请求之后，服务器接收到请求才在服务器端设置\$\_COOKIE**，存储session_id。

（2）设置php.ini文件的session.auto_start指令

当有人访问站点时自动启动会话，缺点是无法以对象方式使用会话变量，因为对象的类定义必须在启动会话和创建会话对象之前载入。

### 3.2  注册会话变量/使用会话变量 {#注册会话变量使用会话变量 align="left"}

 首次生成session_id后，可使用超级全局数组\$\_SESSION创建会话变量：

\$\_SESSION\[\'test\'\] = 1;

该变量生命周期持续至会话结束，此前也可用unset销毁该变量。

当session_id存在时，可调用被载入的会话变量。

### 3.3 销毁变量和对话 {#销毁变量和对话 align="left"}

- session_unset

> 释放当前在内存中已经创建的所有\$\_SESSION变量，但不删除session存储文件以及不释放对应的session_id。

效果等同于\$\_SESSION = array();

- session_destroy()

> 删除当前用户对应的session存储文件以及释放session_id，内存中的\$\_SESSION变量依然保存。

在
[php手册](https://www.php.net/manual/zh/function.session-destroy.php){target="_blank"}
中提到：

> 为了彻底销毁会话，必须同时重置会话 ID。 如果是通过 cookie 方式传送会话
> ID 的，那么同时也需要 调用 [setcookie() 函数来 删除客户端的会话
> cookie。]{.function}

[否则会产生如下的问题：]{.function}

[php - session_destroy not unsetting the session_id - Stack
Overflow](https://stackoverflow.com/questions/8641929/session-destroy-not-unsetting-the-session-id)

    <?php 
        session_start(); 
        echo session_id(); //4ef975b277b52 
        session_destroy(); 
        session_start(); 
        echo session_id(); //4ef975b277b52 
    ?> 

究其原因，是因为\$\_COOKIE\[session_name()\]存在，导致赋值给了session_id，所以需要删除客户端的cookie。

### 3.4 通过URL传递session_id

（1）

    <?php
        session_start();
        echo '<a href="test.php?'.SID.'">URL</a>';
    ?> 

SID当用户开启cookie时，输出空

SID当用户关闭cookie时，输出当前用户session信息，具体格式是 
session_name=session_id;

如：http://localhost/php/cookie-session/test.php?PHPSESSID=1574bf7dea71d26d53f3a4f6eada8044

（2）处理页面(test.php)中使用\$\_GET提取session：

    <?php
        if ($_GET[session_name()])
           session_id($_GET[session_name()]);
        session_start();
    ?> 

注：session_id()当含有参数时是指，以参数中的id为参考找到sessoin文件，此时session_id()必须在session_start()前面

## 参考

1. 《PHP从入门到精通》
2. [深入解析Session工作原理及运行流程_java_脚本之家
   (jb51.net)](https://www.jb51.net/article/191825.htm)
3. [session的工作原理 - 简书
   (jianshu.com)](https://www.jianshu.com/p/0e5ac2b64bea/)
4. [Apache使用mysql认证用户 - 走看看
   (zoukankan.com)](http://t.zoukankan.com/nyxd-p-5354396.html)
5. [使用cookie和session实现用户的认证和授权(原生方式，不使用安全框架)\_炳秦的博客-CSDN博客](https://blog.csdn.net/weixin_43266529/article/details/123456820)
6. [PHP 基于 Cookie + Session 实现用户认证功能 -
   腾讯云开发者社区-腾讯云
   (tencent.com)](https://cloud.tencent.com/developer/article/1720714)
7. [php中cookie和session -
   CSDN](https://www.csdn.net/tags/MtTaYgwsNDU3MTEtYmxvZwO0O0OO0O0O.html)
8. [ session详解\--以php为例_云闲不收的博客-CSDN博客_session格式](https://blog.csdn.net/S_ZaiJiangHu/article/details/124770565)
9. [php中的数据库操作和字符串操作session与cookie操作,Session机制详解\-\--Cookie与Session\...\_六堡茶之家的博客-CSDN博客](https://blog.csdn.net/weixin_35252187/article/details/115168808)
10. [php中的session详解,PHP中的session机制详解_aragakikun的博客-CSDN博客](https://blog.csdn.net/weixin_34873655/article/details/116228070)
