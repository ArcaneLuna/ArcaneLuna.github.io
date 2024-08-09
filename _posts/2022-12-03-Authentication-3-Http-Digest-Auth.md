---

layout: post
title: 【认证机制】3-Http Digest Auth
date: 2022-12-03
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Authentication

---

[[为弥补[ BASIC 认证存在的弱点，从[ HTTP/1.1 起就有了[ DIGEST
认证。[ DIGEST 认证同样使用挑战应答机制[[，**但不会像[ BASIC
认证那样直接发送明文密码]{lang="EN-US"}**。]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}]{lang="EN-US"}

## [[1. 基本流程]{lang="EN-US"}]{lang="EN-US"}

[[![图解HTTP](/img/2022-12-03-Authentication-3-Http-Digest-Auth/1.png "图解HTTP"){width="599"
height="415"}]{lang="EN-US"}]{lang="EN-US"}

 

① 客户端请求需认证的资源。

② 服务器发送401(Authorization Required)响应，头部包括 WWW-Authenticate
字段。该字段内包含realm（认证域），nonce（临时**质询码**），algorithm（算法字符串）。

**【nonce 是一种每次随返回的 401
响应生成的任意随机字符串。该字符串通常推荐由Base64
编码的十六进制数的组成形式，但实际内容依赖服务器的具体实现】**

③ 接收到401状态码的客户端，返回的响应头部中包含 Authorization
字段。其中包含 **username**、realm、nonce、**uri**
和**response**等信息。

【username是realm
限定范围内可进行认证的用户名。uri（digest-uri）即Request-URI的值，但考虑到经代理转发后Request-URI的值可能被修改因此事先会复制一份副本保存在
uri内。response 也可叫做 Request-Digest，存放经过 MD5
运算后的密码字符串，形成**响应码】**

④ 接收到包含首部字段 Authorization
请求的服务器，会确认认证信息的正确性。认证通过后则返回包含 Request-URI
资源的响应。并且这时会在首部字段 Authentication-Info
写入一些认证成功的相关信息（比如下一个随机数，用于**预授权**）。

## 2. 摘要算法

[RFC
2069](https://www.rfc-editor.org/rfc/rfc2069) 大致定义了摘要认证框架：摘要算法由
algorithm 指定的一对函数构成：H(data)和KD(secret,
data)。algorithm默认为"MD5"。

> For the \"MD5\" algorithm
> 
> 　　H(data) = MD5(data)
> 
> and
> 
> 　　KD(secret, data) = H(concat(secret, \":\", data))

具体实现上，RFC 2069 还定义了两个数据块A1和A2

::: cnblogs_code
    A1 = username-value : realm-value : password
    A2 = Method : digest-uri-value
:::

response的值request-digest计算方式：

::: cnblogs_code
    request-digest = KD ( H(A1), nonce-value : H(A2)) = MD5(MD5(A1) : nonce-value : MD5(A2))
:::

[RFC
2617](https://www.rfc-editor.org/rfc/rfc2617) 随后引入了一系列安全增强的选项：保护质量（qop）、随机数计数器由客户端增加、客户产生的随机数（cnonce），且保持了对
RFC 2069 的兼容。

根据 algorithm 的取值，A1有两种定义：

- MD5，同 RFC 2069
- MD5-sess，A1 = MD5(username-value : realm-value : password) :
  nonce-value : cnone-value

根据 qop 的取值，A2有两种定义：

- auth 或未定义，同 RFC 2069
- auth-int，A2 = Method : digest-uri-value : H(entity-body)

根据 qop 是否定义，request-digest有两种计算方式：

- 未定义，同 RFC 2069
- auth 或 auth-int，request-digest = KD (
  [H(A]{style="color: #000000;"}1), nonce-value : nc-value :
  cnonce-value : qop-value : H(A2) )

可以看出qop提供了**对称认证**，auth-int表示提供**报文完整性保护**。

## 3. 用PHP模拟HTTP认证

延伸阅读：[PHP: 用 PHP 进行 HTTP 认证 -
Manual](https://www.php.net/manual/zh/features.http-auth.php)
