---
layout: post
title: 【Python】爬虫笔记-ConnectionResetError(10054)
date: 2022-11-01
author: "qtunneling"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

## 0x01

在对网站图片进行批量爬取的过程中遇到了一个典型问题：

> requests.exceptions.ConnectionError: (\'Connection aborted.\',
> ConnectionResetError(10054, \'An existing connection was forcibly
> closed by the remote host\', None, 10054, None))

最开始以为是网站的反爬措施，于是加上了 User-Agent 字段。

【参考：[python requests 报错 Connection aborted ConnectionResetError
RemoteDisconnected
解决方法_whatday的博客-CSDN博客](https://blog.csdn.net/whatday/article/details/109257348)】

```{.highlighter-hljs
def download_image(i, url):
    user_agent_list = [
        "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36",
        "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36",
        "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.186 Safari/537.36",
        "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.62 Safari/537.36",
        "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/45.0.2454.101 Safari/537.36",
        "Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0)",
        "Mozilla/5.0 (Macintosh; U; PPC Mac OS X 10.5; en-US; rv:1.9.2.15) Gecko/20110303 Firefox/3.6.15",
    ]

    headers = {}
    headers['User-Agent'] = random.choice(user_agent_list)
    --snip--
```

<div>

::: {style="margin-left: 30px;"}
同时在每次下载图片后加上睡眠：time.sleep(3)，但问题未解决，每爬几个链接就抛出异常。
:::

::: {style="margin-left: 30px;"}
猜测是不是因为每次 response = requests.get(url, headers=headers)
后没有释放连接，又添加 response.close()，但不起作用。
:::

::: {style="margin-left: 30px;"}
【注：[python中requests.get后需要close吗？ - 知乎
(zhihu.com)](https://www.zhihu.com/question/507193607) 建议如果没有指定同一session的情况下，调用
requests.get() 不需要 close 释放资源】
:::

## 0x02

::: {style="margin-left: 30px;"}
尝试在代码中忽略掉异常：
:::

```{.highlighter-hljs
try:
    r = requests.get(url, headers=headers)
    with open(myPath, 'wb') as f:
        f.write(r.content)
except requests.exceptions.RequestException as e:
    pass
```

::: {style="margin-left: 30px;"}
【注：ConnectionError是RequestException的子类】
:::

::: {style="margin-left: 30px;"}
程序没有按照预想地执行下去，反而会直接卡住。
:::

::: {style="margin-left: 30px;"}
有文章指出是因为 requests
默认设置请求超时，但没有设置读取超时，导致程序不报错也不继续执行。设置
timeout 参数即可。
:::

::: {style="margin-left: 30px;"}
【参考：[python requests 超时。卡住没反应的处理方法。 - 简书
(jianshu.com)](https://www.jianshu.com/p/fdfb5591536c)】
:::

::: {style="margin-left: 30px;"}
解决读取超时问题后，发现后续地爬取的页面几乎全部失败，但浏览器直接访问图片链接却不受影响。
:::

## 0x03

<div>

于是上 StackOverflow 上逛了一圈，发现有三种说法：

</div>

### ① 服务器-客户端超时分歧

> ::: {style="margin-left: 30px;"}
> This can be caused by the two sides of the connection disagreeing over
> whether the connection timed out or not during a keepalive. (Your code
> tries to reused the connection just as the server is closing it
> because it has been idle for too long.) **You should basically just
> retry the operation over a new connection.** (I\'m surprised your
> library doesn\'t do this automatically.)
> :::

::: {style="margin-left: 30px;"}
【参考：[twitter - python: \[Errno 10054\] An existing connection was
forcibly closed by the remote host - Stack
Overflow](https://stackoverflow.com/questions/8814802/python-errno-10054-an-existing-connection-was-forcibly-closed-by-the-remote-h)】
:::

这里提到"代码尝试重用连接"和"服务器被闲置太久"。

根据requests库作者在close()的注释，requests.get()可能会复用连接，防止多次握手挥手。

那么应该是服务器设置的keepalive时间相较于客户端这边较短，于是客户端会尝试复用，然而服务器那边已经关闭了连接。

有些文章使用心跳来维持TCP长连接保活。

根据测试，使用session也能有效减少此问题的出现。

### ② 请求频率问题，服务器识别恶意访问

> ::: {style="margin-left: 30px;"}
> The web server actively rejected your connection. That\'s usually
> because it is congested, has rate limiting or thinks that you are
> launching a denial of service attack. If you get this from a server,
> you should sleep a bit before trying again. In fact, if you don\'t
> sleep before retry, you are a denial of service attack. **The polite
> thing to do is implement a progressive sleep of, say, (1,2,4,8,16,32)
> seconds.**
> :::

::: {style="margin-left: 30px;"}
【参考：[python - How to solve the 10054 error - Stack
Overflow](https://stackoverflow.com/questions/27333671/how-to-solve-the-10054-error)】
:::

### ③ 标头问题

::: {style="margin-left: 30px;"}
除了设置 User-Agent 外，可能还需要设置 Accept, Accept-Encoding,
Content-Type, Connection 等属性。
:::

## 0x04

<div>

由于已经尝试设置过 sleep 和
User-Agent，且在抛出异常时不影响浏览器访问链接，故判断是服务器-客户端超时分歧。重复执行请求有多种方案：

</div>

### ① 递归调用

<div>

```{.highlighter-hljs
try:
    r = requests.get(url, headers=headers, timeout=10)
    with open(mypath, 'wb') as f:
        f.write(r.content)
except requests.exceptions.RequestException as e:
    download_image(i, url)
```

### ② while循环

用while包裹住try-except即可

### ③ Session

试着换用 session.get()
后，较好解决了问题，而且不会像函数回调那样频繁地请求。但要注意 timeout
设置太短的话可能抛出 read time
out，继而在接下来的连接中抛出ConnectionReset（并不是100%，而是有概率）。所以还需要设置maxtries用于重复操作。

```{.highlighter-hljs
my_session.mount('http://', HTTPAdapter(max_retries=10))
my_session.mount('https://', HTTPAdapter(max_retries=10))
```

### ④ retry模块

::: {style="margin-left: 30px;"}
[Python爬虫实例（三）：错误重试，超时处理的解决方法 - 知乎
(zhihu.com)](https://zhuanlan.zhihu.com/p/386108242)
:::

## 0x05

<div>

time.sleep 进阶用法：

</div>

<div>

- [Python 爬虫经常需要睡眠防止被封IP time
  sleep_DN_XIAOXIAO的博客-CSDN博客_爬虫time.
  sleep的优点](https://blog.csdn.net/DN_XIAOXIAO/article/details/115717354)

</div>

<div>

关于keep-alive：

</div>

<div>

- [Http------Keep-Alive机制 - 曹伟雄 - 博客园
  (cnblogs.com)](https://www.cnblogs.com/caoweixiong/p/14720254.html)
- [HTTP keep-alive和TCP keepalive的区别，你了解吗？ - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/224595048)
- [python-requests
  必需如下使用才能保持keep-alive_黄传通的博客-CSDN博客](https://blog.csdn.net/toontong/article/details/25730829)
- [【Python】socket编程报错： ConnectionResetError: \[WinError 10054\]
  远程主机强迫关闭了一个现有的连接_忘尘\~的博客-CSDN博客_connectionreseterror](https://blog.csdn.net/BobYuan888/article/details/115009233)
- [记一次网络请求异常：ConnectionResetError: \[WinError 10054\]
  远程主机强迫关闭了一个现有的连接。 - 代码先锋网
  (codeleading.com)](https://codeleading.com/article/48502586053/)
- [Python+socket完美实现TCP长连接保持存活 - 腾讯云开发者社区-腾讯云
  (tencent.com)](https://cloud.tencent.com/developer/article/1614536)
- [Session会话恢复：两种简短的握手总结SessionID&SessionTicket_大力海棠的博客-CSDN博客_session
  ticket](https://blog.csdn.net/justinzengTM/article/details/105491809)
- [给面试官上一课：HTTPS是先进行TCP三次握手，再进行TLS四次握手 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/399105434)
- [4个实验，彻底搞懂TCP连接的断开 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/425788027)

</div>

<div>

反爬&反反爬：

</div>

- [Python常见的反爬手段和反反爬虫方法-云社区-华为云
  (huaweicloud.com)](https://bbs.huaweicloud.com/blogs/262215)
- [Python反爬研究总结 - 腾讯云开发者社区-腾讯云
  (tencent.com)](https://cloud.tencent.com/developer/article/1886418)
- [Python处理超强反爬(TSec防火墙+CSS图片背景偏移定位)-技术圈
  (proginn.com)](https://jishuin.proginn.com/p/763bfbd6d60c)
- [6种Python反反爬虫技术，看完后我的爬虫技术提升了_Python_sn的博客-CSDN博客_反反爬虫技术](https://blog.csdn.net/Python_sn/article/details/109258447)

<div>

关于异常：

</div>

- [Python爬虫常用库之requests详解 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/366457854)
- [requests请求超时处理与异常总结-老董笔记
  (python66.com)](http://www.python66.com/pythonpachong/116.html)

<div>

 

</div>

<div>

 

</div>

<div>

 

</div>

<div>

 

</div>

</div>

</div>
