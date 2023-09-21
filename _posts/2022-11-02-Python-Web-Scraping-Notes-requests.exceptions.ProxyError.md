---
layout: post
title: 【Python】爬虫笔记-requests.exceptions.ProxyError
subtitle: 代理服务器是否支持HTTPS代理
date: 2022-11-02
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

## 0x01

爬虫使用HTTP/HTTPS代理时报故：

```python
proxy = '127.0.0.1:9743'
proxies = {
    'http': 'http://' + proxy,
    'https': 'https://' + proxy,
}
response = requests.get(url, proxies=proxies)
```

### Ⅰ 完整信息

> requests.exceptions.ProxyError:
> HTTPSConnectionPool(host=\'www.xxx.com\', port=443): Max retries
> exceeded with url: / (Caused by\
> ProxyError(\'Cannot connect to proxy.\', OSError(0, \'Error\')))

### Ⅱ 错误原因

urllilb3 在1.26.0版本后增加了对HTTPS代理的支持，此前 urllib3 不支持HTTPS代理（所以以前设置了HTTPS代理时依然是默默使用 HTTP连接到代理服务器）。

更新后 urllib3 启用HTTPS代理，但代理服务器不支持HTTPS代理导致错误。

### Ⅲ 解决方案

① 配置 HTTP 代理：

```python
proxies = {
    'http': 'http://' + proxy,
    'https': 'http://' + proxy,
}
```

当然，也可以直接将传入的 url 的协议改为 http。

② 降低 urllib3 版本：

pip install urllib3==1.25.11

注意 pip 在20.3版本后内置了 urllib3 包。

## 0x02

实际上这个问题与代理服务器有关，许多代理服务器都支持的是HTTP代理，而非HTTPS代理。

HTTP代理与HTTPS代理的区别主要在于浏览器到代理服务器之间的通信（inbound），前者是明文而后者是经过tls加密的。HTTP代理同样可以访问HTTPS网站（基于HTTP 
Tunnel）。

回到开头，原本的写法本身更像是一个逻辑错误，一般在固定了ip和port的情况下，应该明确使用的是HTTP还是HTTPS代理，除非使用的是不同的端口号同时提供两种代理模式。

### 参考

- [Python 遭遇 ProxyError 问题记录 - DavyCloud - 博客园
  (cnblogs.com)](https://www.cnblogs.com/davyyy/p/14388623.html) 这篇文章给出了更详细的说明与解决方案
- [Advanced Usage --- Requests 2.28.1 documentation
  (python-requests.org)](https://docs.python-requests.org/en/latest/user/advanced/#proxies)
- [Pip 20.3+ break proxy connection · Issue #9216 · pypa/pip ·
  GitHub](https://github.com/pypa/pip/issues/9216)
- [关于 http/https proxy 的问题 · Issue #15 · Dreamacro/clash ·
  GitHub](https://github.com/Dreamacro/clash/issues/15)
- [HTTP隧道 - 维基百科，自由的百科全书
  (wikipedia.org)](https://zh.wikipedia.org/wiki/HTTP%E9%9A%A7%E9%81%93)
- [HTTP/HTTPS/SOCKS5协议的区别 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/355332701)
- [HTTP、HTTPS、SOCKS代理的概念（到底是什么意思？）\_啊大1号的博客-CSDN博客_ss_privoxy.exe](https://blog.csdn.net/a3192048/article/details/104034219)
- [http代理协议和https代理协议有什么区别？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/486794566/answer/2123259705)
