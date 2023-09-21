---
layout: post
title: 【Python】爬虫实战-基于代理池的高并发爬虫
subtitle: 手动切换策略
date: 2022-11-06
author: "H4kur31"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

最近在写一个基于代理池的高并发爬虫，目标是用单机从某网站 API
爬取十亿级别的JSON数据（均为公开非敏感信息）。

## 代理池

有两种方式能够实现爬虫对代理池的充分利用：

1. 搭建一个 Tunnel Proxy 服务器维护代理池
2. 在爬虫项目内部自动切换代理

所谓 Tunnel Proxy
实际上是将切换代理的操作交给了代理服务器，很多市面上的代理软件都有此类功能。

如果要自行搭建可参考以下项目：

- [GitHub -
  urdr-gungnir/TunnelProxy](https://github.com/urdr-gungnir/TunnelProxy)

- [GitHub -
  pingc0y/go_proxy_pool](https://github.com/pingc0y/go_proxy_pool)

考虑到高并发，在爬虫项目内部切换代理更加灵活一些。代理池选一个能用的就行：[GitHub -
jhao104/proxy_pool](https://github.com/jhao104/proxy_pool)

记得加上匿名校验：[能否设置代理池只获取高匿IP · Issue #169 ·
jhao104/proxy_pool ·
GitHub](https://github.com/jhao104/proxy_pool/issues/169)

### 代理切换策略

如果简单的在**多线程**中对每个 requests.get()
使用不同的代理，那么一定会遇到内存泄露的问题：

- [内存泄露问题 · Issue #522 · jhao104/proxy_pool ·
  GitHub](https://github.com/jhao104/proxy_pool/issues/522)
- [Requests memory leak · Issue #4601 · psf/requests ·
  GitHub](https://github.com/psf/requests/issues/4601#issuecomment-603326738)
- [Memory Leak in Python requests -
  GeeksforGeeks](https://www.geeksforgeeks.org/memory-leak-in-python-requests/)

即便写成：

    session = requests.session()
    response = session.get(url, headers=headers, proxies=proxies)
    response.close()
    session.close()

甚至在加上 gc.collect() 也无济于事。

因此需要控制创建 session 对象的数量，只在请求失败后切换代理和创建新的
session。

## 工作流程

![](/img/2022-11-06-Python-Web-Scraping-in-Action-High-Concurrency-Crawling-with-a-Proxy-Pool/1.png){width="566"
height="344"}

① 主线程根据 URL 数量动态创建子进程，虚线框内为子进程任务

② crawler_task 为线程任务，执行发送请求和解析JSON

### 插入策略

每个子进程维护一个 url_queue 和 insert_queue。

线程会从 url_queue
取出URL执行爬取任务，由于JSON数据占用的空间不大，所以线程会先将每个
response 经过简单解析后存到**列表**中。

等到 url_queue 为空时（不要使用不安全的 queue.empty() 判断），get
方法会触发 Timeout 异常，然后线程会将列表插入到 insert_queue 中。

所有线程任务结束后，子进程再执行 executemany 将数据批量插入到 MySQL。

## 其他

爬取JSON数据产生的流量不大，但需要考虑 PPS（packet per
second），如果网络设施不到位的话可能严重影响爬取效率。

网络上获取的免费代理大多是透明代理，如果使用开源项目 Proxy_Pool
作为代理池并加入匿名校验，可能会间歇性导致代理池没有可用代理。（所以最好还是从一些网络空间测绘引擎上通过特征抓取）

### 代理蜜罐

代理池虽好，但也会遇到不讲武德的蜜罐：

- [GitHub - netxfly/x-proxy: honeypot
  proxy](https://github.com/netxfly/x-proxy)

（这项目发布于2019年5月，提前预测了微博的数据泄露？）
