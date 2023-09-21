---
layout: post
title: 【认证机制】1-HTTP Basic Auth
date: 2022-12-01
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Authentication

---

## 1. 简介

参考：[HTTP basic authentication - IBM
Documentation](https://www.ibm.com/docs/en/cics-ts/5.4?topic=concepts-http-basic-authentication)

HTTP Basic
Auth是HTTP协议提供的一种简单的挑战应答机制，服务器可以通过该机制从客户机请求认证信息（用户标识和密码）。客户端在授权标头中将身份验证信息凭证（采用
base-64 编码）传递给服务器，从而可以访问受保护的资源。

服务端开启Basic Auth后，通过认证的方式有两种：

\(1\)
在访问URL的时候主动代码账号和密码信息，如http://user:password@www.baidu.com，其中user是账户名，password是密码。

\(2\) 在请求头部加上Authorization头部，并将值设置为Basic
xxxx，xxxx是账号和密码的base64编码结果。

一般我们使用的是第二种方式：如果网站需要认证，浏览器会自动会弹出登录框，手动输入账号密码后浏览器在头部带上认证信息访问。

## 2. 基本流程

![](/img/2022-12-01-Authentication-1-Http-Basic-Auth/1.jpg){width="515"
height="591"}

1\. 浏览器GET请求受保护的网络资源

2\.
服务器返回401响应（Unauthorized），表示该资源需要授权访问，头部带有WWW-Authenticate字段：

[WWW-Authenticate: Basic realm="Authorization Required",
charset="UTF-8"]{style="background-color: #000000; color: #ffffff;"}

【Basic表示使用basic认证算法；realm：认证域。明文信息，**用于提示客户端使用哪些用户名和密码**】

3\.
浏览器弹出用户名和密码输入框，让用户提供凭证。用户输入信息后，将凭证信息进行**Base64编码**后，再次发送请求到服务器，头部带有Authorization字段：

[Authorization: Basic
YWRtaW46cGFzcw==]{style="background-color: #000000; color: #ffffff;"}

Authorization分为两部分，第一部分是Basic，第二部分是用户凭证信息的编码：

- 用户名与密码通过":"组合，注意用户名不能包含":"
- 组合字符串默认使用US-ASCII编码，当然服务器也可以要求UTF-8编码
- 组合后的字符串进行Base64编码
- Admin:pass的Base64编码为：YWRtaW46cGFzcw==

4\.
服务器收到用户的请求，提取出用户凭证，校验匹配后，返回包含该资源信息的200成功响应。

注：从上述流程中可以看出authentication（认证）和authorization（授权）的区别，后者相当于凭证

## 3. Basic Auth优缺点

### **3.1 Basic Auth优点** {#basic-auth优点 align="left"}

- 简单 -
  认证信息使用标准的http头部字段进行传输，不需要使用cookies/session等，也没有复杂的客户端和服务端协商过程。

### **3.2 Basic Auth缺点** {#basic-auth缺点 align="left"}

- 安全 -
  认证信息仅仅使用Base64进行编码，并未使用加密算法，基本上等于明文传输，需要使用https进行加密传输。
- 即使密码被强加密，第三方仍可通过加密后的用户名和密码进行**重放攻击**
- 效率低，服务端需要对每条请求资源验证身份
- 如果想再进行一次 BASIC
  认证时，一般的浏览器却**无法实现认证注销**操作。所以，BASIC
  认证使用上不够灵活，且达不到多数 Web
  网站期望的安全性等级，因此并不常用。
  - 　　[http - How to log out user from web site using BASIC
    authentication? - Stack
    Overflow](https://stackoverflow.com/questions/233507/how-to-log-out-user-from-web-site-using-basic-authentication)

### **3.3 Basic Auth应用场景** {#basic-auth应用场景 align="left"}

- Basic
  Auth比较适合简单的认证场景，尤其是在内网中使用。通常我们会配合https进行信息加密，来提高安全性。
