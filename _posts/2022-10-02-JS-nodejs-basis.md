---

layout: post
title: 【JS】node.js初探
date: 2022-10-02
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Encoding

---

原本只是为了方便在VSCode中调试JS代码安装了node.js，但最近看了许多关于
node.js 的文章，心血来潮打算用JS写写后端。

## 1. 理解node.js

> 懂一些 JavaScript 和开发 Node.js 应用是两回事儿

node.js 本质上是基于 Chrome V8 引擎的的 JavaScript
**运行环境**，而非编程语言。node.js 能做的事情很多，但最常用的还是作为
Web 服务器。

### 1.1 与常见的 Web 服务器区别

<div>

> **Apache**就是**静态网页服务器**，就是将本地页面文件做一个网络映射，可以添加mod来扩展功能，例如php模块就扩展了基于php的CGI动态页面页面能力，代理模块就是成了代理服务器。
> 
> **nginx**同，不过更多主职于代理服务器。
> 
> **tomcat**就是一个Java
> Servlet**容器**，换个说法就是基于java的CGI动态页面服务器，静态页面只是一个附属功能。
> 
> **node.js**同样一个**容器**，换个说法就是基于JavaScript的CGI动态页面服务器，看上去静态页面不算是直接功能。

### 1.2 与 PHP 对比 {#与-php-对比 pid="FPmoBKT7"}

理论上 node.js 是可以像 Apache 一样在上面运行 PHP
的，两者并不是同一个层次的概念。但为了便于理解，可以拿最简单的 hello
world 程序来进行比较。

用PHP写一段 Hello World：

1\. 安装 web 服务器程序（如：Apache）

2\. 安装 PHP 引擎，并在 Apache 中进行相应配置

3\. 在 web 目录（DocumentRoot）下新建文件 index.php，输入以下代码：

    <?php
        echo 'Hello World!';
    ?>

 4. 启动 Apache，访问 localhost/index.php

而 node.js 的流程相对简单一些（但代码更长一些），安装 node.js
后，在任意位置新建文件 server.js，输入代码：

    var http = require('http');
    http.createServer(function (req, res) {
        res.writeHead(200, {'Content-Type': 'text/plain'});
        res.end('Hello World!');
    }).listen(3000, '127.0.0.1');

使用 node 命令执行该文件，访问 127.0.0.1:3000 即可

注意到尽管上面的 server.js
能够响应浏览器的请求返回一个（纯文本）文档，但连**路由**的功能都没有，也无法对
HTTP 动词（GET、POST）做出反应。

手工造一堆轮子明显不现实，还是从一个好的 Web 框架开始吧。

### 参考

- [apache、node.js、nginx、tomcat谁能帮我捋一捋关系？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/57688289)
- [PHP vs Node.js
  深入讨论](https://www.runoob.com/w3cnote/php-vs-node-js.html)
- [手撸一个Web Server【基于Node.js原生API】 - 掘金
  (juejin.cn)](https://juejin.cn/post/6996827368576778247)
- [GitHub - liuxing/node-abc:
  《Node.js入门教程》](https://github.com/liuxing/node-abc)
- [Node.js 的 Web 框架的 3 个层次，理清了就不迷茫 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/416742435)

## 2. Express框架

Express.js 是最早出现的 node.js
框架，到现在依然很流行。它以类似洋葱的顺序调用中间件，没有模块和 MVC
的划分，适合小型的服务。

安装和学习可以参考：

- [Express 应用程序生成器 - Express 中文文档 \| Express 中文网
  (expressjs.com.cn)](https://www.expressjs.com.cn/starter/generator.html)
- [Express Web Framework (Node.js/JavaScript) - 学习 Web 开发 \| MDN
  (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Learn/Server-side/Express_Nodejs) 这里提供了一个搭建本地图书馆的案例，适合初学者
- [Node.js Express 框架 \| 菜鸟教程
  (runoob.com)](https://www.runoob.com/nodejs/nodejs-express-framework.html)

在挂着代理的时候，npm 无需像 pip
那样进行额外配置。但如果代理不稳定也会遇到
ECONNRESET，需要重复执行安装。或者也可以使用淘宝的镜像 cnpm。

首次安装 express 时遇到了警告：

npm WARN deprecated mkdirp@0.5.1: Legacy versions of mkdirp are no
longer supported. Please update mkdirp 1.x. (Note that the API surface
has changed to use Promises in 1.x.)

这可能是由于 express 的版本过低而 npm 的版本过高导致（两者依赖不同版本的
mkdirp），解决方法是手动安装更高或者最新版本的 express：

- [node.js - How to update mkdir in order to install express-generator
  without warnings? - Stack
  Overflow](https://stackoverflow.com/questions/64439465/how-to-update-mkdir-in-order-to-install-express-generator-without-warnings)

当然也可以降低 npm 的版本：

- [npm install -g
  koa-generator出错，安装不上的问题_koa安装失败_puyaguo的博客-CSDN博客](https://blog.csdn.net/puyaguo/article/details/123662856)

或者，在高版本的 node.js(\>=8.2.0) 中使用以下命令安装
express-generator：

> npx express-generator

这相当于旧版本的：

> npm install express-generator -g
> 
> express

除此以外，npm audit fix \--force 能够较好的修复包的版本问题。

</div>
