---
layout: post
title: 【Python】爬虫笔记-开源代理池haipproxy使用
date: 2022-11-05
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

## 0. 简介

大规模的数据采集需要用到代理池来突破IP封锁。

一般代理池的构建是先爬取网上的免费代理，校验后存到数据库中，再提供给其他程序api。

github上有很多现成的代理池能拿来用，在知乎看到 [如何设计一个优秀的代理IP池? -
知乎
(zhihu.com)](https://www.zhihu.com/question/40473529/answer/333774864) 这个答案后打算试用一下。

项目地址：[GitHub - SpiderClub/haipproxy: High available distributed ip
proxy pool, powerd by Scrapy and
Redis](https://github.com/SpiderClub/haipproxy)

5200+stars，文档中的校验和调度策略描述清晰可用于学习，但 code
很久没有维护，部分提供免费代理ip的网址已失效，需要根据实际进行调整。

## 1. 安装

### 1.1 docker & scrapy-splash

运行 haipproxy 需要安装 redis 和 scrapy-splash。scrapy-splash
则需要先安装 docker，这里记录一下安装 docker 遇到的坑：

windows 安装 docker 的先决条件分两种：①安装 wsl 2；②开启 hyper
-v（[Install on Windows \| Docker
Documentation](https://docs.docker.com/desktop/install/windows-install/)）

![](/img/2022-11-05-Python-Web-Scraping-Notes-Using-the-Open-Source-Proxy-Pool-haipproxy/1.png){width="751"
height="385" loading="lazy"}

除此之外还对**系统版本**、**内存**和 **BIOS virtualization**
有要求。在"任务管理器-性能"可以查看是否开启虚拟化，如果未开启，请自行搜索对应电脑版本开启
virtualization 的教程。

值得注意的是 Windows 家庭版默认没有 hyper-v 选项。hyper
-v还和VMware冲突：[解决docker和vmware使用冲突_Hero_s_的博客-CSDN博客_docker和vmware冲突](https://blog.csdn.net/Hero_s_/article/details/115555778#:~:text=%E4%B8%BA%E4%BD%95%E5%86%B2%E7%AA%81%3F,VMware%E8%87%AA%E5%B8%A6%E8%99%9A%E6%8B%9F%E5%8C%96%E5%86%85%E6%A0%B8%EF%BC%8C%E4%BD%86%E6%98%AF%E5%9C%A8win10%E4%B8%ADDocker%E7%9A%84%E5%B7%A5%E4%BD%9C%E9%9C%80%E8%A6%81%E4%BE%9D%E8%B5%96Hyper-V%EF%BC%8C%E6)，wsl
2没有此问题。尽管网上有教程强行开启，但一般还是建议家庭版安装 wsl
2。（不过 VM 后来做了让步，VM 16 以后先装docker再装 VM 即可）

关于 hyper-v 和 wsl 2 这里多提几句，两者其实并没有本质不同。

- [Windows下想使用Linux环境，WSL、Docker、VM应该怎么选择？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/339939686)
- [从wsl到wsl2明显是退步，为什么还有人鼓吹wsl2？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/424191615)
- [wsl 2 是否需要启用 Hyper-V？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/439585675)

> Hyper-V其实分两个部分：底层的虚拟机平台，以及上层的虚拟机管理软件。
> 
> 以前的Windows版本，这两个是同一个选项，现在的新版本则是分成Hyper-V和虚拟机平台两个选项。
> 
> wsl2、沙盒本质上是基于Hyper-V的虚拟机，所以虚拟机平台要打开才能用。但作为Windows的两项特殊功能，无需额外使用管理软件对虚拟机进行管理。但也因此wsl2缺失了一些虚拟机常见功能，例如网络只能配置为NAT，不能指定IP/网段，虚拟磁盘管理等。

wsl 1 和 wsl 2 的区别如下：

![](/img/2022-11-05-Python-Web-Scraping-Notes-Using-the-Open-Source-Proxy-Pool-haipproxy/2.png)

 

wsl 2 安装按照官方文档来就行：[安装 WSL \| Microsoft
Learn](https://learn.microsoft.com/zh-cn/windows/wsl/install)

我自己安装的过程：

1\. 首先命令行执行：[wsl
\--status]{style="background-color: #888888; color: #ffffff;"}，查看是否安装了
wsl 2，如果未安装会提示没有 wsl 2 内核。

2\. 根据提示运行 [wsl
\--update]{style="background-color: #888888; color: #ffffff;"}，手动将
wsl 升级到 wsl 2。这里用管理员模式打开 cmd，可每次更新到 90%
时就会莫名其妙提示需要提升权限。然而已经是管理员了如何提升？这里把 UAC
调至从不通知，无效；尝试寻找安全策略，家庭版没有安全策略......没办法，只能先硬着头皮往下走。

3\. 安装 linux 发行版：wsl
\--install，这里没有直接安装，而是弹出帮助文本，根据官方文档，这时可以手动指定发行版安装：

[wsl \--list
\--online]{style="background-color: #888888; color: #ffffff;"}

[[[这里如果报错 0x80072ee7，一般是 raw.githubusercontent.com
这个域名无法访问，挂上代理或者修改 dns
即可。]{style="color: #000000; background-color: #ffffff;"}\
]{style="color: #ffffff;"}]{style="background-color: #000000;"}

[wsl \--install -d
Ubuntu-20.04]{style="background-color: #888888; color: #ffffff;"}

![](/img/2022-11-05-Python-Web-Scraping-Notes-Using-the-Open-Source-Proxy-Pool-haipproxy/3.png){width="714"
height="157" loading="lazy"}

4\. 这个时候 docker 自然是无法正常启动，会卡在 starting。

[wsl -l -v]{style="background-color: #888888; color: #ffffff;"} 查看
VERSION，果然 Ubuntu-20.04 版本为1（指wsl）

5\. 折腾了半天没办法，忽然想到既然 wsl \--update
手动更新失败，那么索性[打开自动更新]{style="background-color: #888888; color: #ffffff;"}算了，于是打开"windows更新设置-高级选项-更新Windows时接收其他
Microsoft 产品的更新"，顺手卸载了刚安的Ubuntu-20.04：

[wsl unregister
Ubuntu-20.04]{style="background-color: #888888; color: #ffffff;"}

6\. 更新重启。

7\. [wsl \--status]{style="background-color: #888888; color: #ffffff;"}
查看还是缺少 wsl 2 内核，然后再尝试安装 Ubuntu-20.04：[wsl \--install -d
Ubuntu-20.04]{style="background-color: #888888; color: #ffffff;"}

结束后查看VERSION，居然是2，总算解决。（不知道是不是自动更新起了作用）

8\. docker 不会再卡在 starting 了，命令行执行：[docker pull
scrapinghub/splash]{style="background-color: #888888; color: #ffffff;"}（需要先打开守护进程或者
docker desktop），拉取 scrapy-splash 的镜像

9\. 运行：[docker run -p 8050:8050
scrapinghub/splash]{style="background-color: #888888; color: #ffffff;"}

10\. 打开
[127.0.0.1:8050]{style="background-color: #888888; color: #ffffff;"}，出现splash页面，成功

说起来我是先装的 docker 再装的 wsl 2......中间只报了一次错：installation
Component CommunityInstaller.ServiceAction
failed，发现是被杀软拦截了，关闭后重新安装即可。

### 1.2 依赖问题 {#依赖问题 align="left"}

（1）首先是自己环境问题（与项目无关）

selenium 4.4.3 requires urllib3\[socks\]\~=1.26, but you have urllib3
1.22 which is incompatible.

mitmproxy 5.3.0 requires click\<8,\>=7.0, but you have click 6.7 which
is incompatible.

 mitmproxy 不是很需要，直接卸了，然后把 urllib3 升级到 1.26，然而：

requests 2.18.4 requires urllib3\<1.23,\>=1.21.1, but you have urllib3
1.26.14 which is incompatible

好家伙，pip install requests 默认版本是 2.18.4，和 selenium
起了冲突，手动指定一个：

pip install requests==2.25.1

（2）运行\>python crawler_booter.py \--usage crawler，报错：

① 0: UserWarning: You do not have a working installation of the
service_identity module: \'cannot import name \'verify_ip_address\' from
\'service_identity.pyopenssl\'
(C:\\Users\\xxx\\AppData\\Roaming\\Python\\Python37\\site-packages\\service_identity\\pyopenssl.py)\'. 
Please install it from \<https://pypi.python.org/pypi/service_identity\>
and make sure all of its dependencies are satisfied.  Without the
service_identity module, Twisted can perform only rudimentary TLS client
hostname verification.  Many valid certificate/hostname mappings may be
rejected.

参考：[You do not have a working installation of the service_identity
module: \'cannot import name
\'opentype_凯旋的皇阿玛的博客-CSDN博客](https://blog.csdn.net/qq1195365047/article/details/79578131)

解决：[pip install --I --U
service_identity]{style="background-color: #888888; color: #ffffff;"}

② AttributeError: module \'OpenSSL.SSL\' has no attribute \'TLS_METHOD\'

参考：

- [python - AttributeError: module \'OpenSSL.SSL\' has no attribute
  \'SSLv3_METHOD\' - Stack
  Overflow](https://stackoverflow.com/questions/73859249/attributeerror-module-openssl-ssl-has-no-attribute-sslv3-method)
- [安装scrapy时出现"AttributeError: module 'OpenSSL.SSL' has no
  attribute
  'TLS_METHOD'"报错的解决方法_闲石观江的博客-CSDN博客](https://blog.csdn.net/weixin_41059258/article/details/127143020)

可知是因为SSL v2和v3被弃用，需要安装低版本的 pyopenssl（22.0.0左右）和
cryptography

安装了22.0.0 的 pyopenssl 和 38.0.0 的 cryptography，问题解决

③ builtins.ImportError: cannot import name \'HTTPClientFactory\' from
\'twisted.web.client\' (unknown location)

[ImportError: cannot import name \'HTTPClientFactory\' from
\'twisted.web.client\' (unknown location) - 安迪9468 - 博客园
(cnblogs.com)](https://www.cnblogs.com/andy9468/p/16423466.html)
认为是twisted版本过高或过低

此时 twisted 版本 22.10.0，安装 20.3.0 即可

### 1.3 redis {#redis align="left"}

redis 安装没什么问题，需要注意的是：

命令行执行 redis-server.exe 后默认不加载配置文件，需要进入 redis
文件夹执行：

[redis-server.exe
redis.windows.conf]{style="background-color: #888888; color: #ffffff;"}

很烦的一点是即便添加了环境变量，对于 conf 文件还是需要输入完整的路径。

这里有些替代解决方法：[Invalid argument during startup: Failed to open
the .conf file: redis.windows.conf
CWD=C:\\Users\\xxx_程序猿中的一股清流的博客-CSDN博客](https://blog.csdn.net/A227311/article/details/86551721)

（如果把 redis 添加到系统服务，加载的配置文件是
redis.windows-server.conf）

haipproxy 在 config/setting.py 中设置 redis 密码为 123456，需要更改 conf
文件中 requirepass 值为 123456（并取消注释）

或者再开一个客户端：redis-cli.exe -h 127.0.0.1 -p 6379

修改密码：config set requirepass 123456（服务端关闭后就无了）

## 2. 测试 {#测试 align="left"}

 分别打开两个终端运行以下两个命令：

python crawler_booter.py \--usage crawler

python crawler_booter.py \--usage validator

期间报错一次

> requests.exceptions.ProxyError:
> HTTPSConnectionPool(host=\'httpbin.org\', port=443)

问题出在 crawler/validator/httpbin.py line 41

> <div>
> 
> self.origin_ip = requests.get(self.urls\[1\]).json().get(\'origin\')
> 
> </div>

这行代码请求的是 https://httpbin.org/ip，应该是哪里代码的代理设置有问题，改成
self.urls\[0\] 就行。

不知道为什么没有自动终止，而是持续在爬，爬了半小时就比半分钟多了几十条，估计在重复请求，感觉差不多了就可以关闭。

或者直接运行 scheduler 也行，可以一直挂着：

[python scheduler_booter.py \--usage
crawler]{style="background-color: #888888; color: #ffffff;"}

[python scheduler_booter.py \--usage
validator]{style="background-color: #888888; color: #ffffff;"}

再开一个终端连接 redis 查看爬取的情况

haipproxy:all 是 set，存放着未校验的所有 proxies，爬了 1000+

![](/img/2022-11-05-Python-Web-Scraping-Notes-Using-the-Open-Source-Proxy-Pool-haipproxy/4.png){width="326"
height="342" loading="lazy"}

 haipproxy:validated:http 和 haipproxy:validated:https
还有其他特定规则的校验规则是 zset，存放着校验排序过的
proxies，只有几十条。

作者的测试代码：

```{.highlighter-hljs
from client.py_cli import ProxyFetcher
args = dict(host='127.0.0.1', port=6379, password='123456', db=0)
＃　这里`zhihu`的意思是，去和`zhihu`相关的代理ip校验队列中获取ip
＃　这么做的原因是同一个代理IP对不同网站代理效果不同
fetcher = ProxyFetcher('zhihu', strategy='greedy', redis_args=args)
# 获取一个可用代理
print(fetcher.get_proxy())
# 获取可用代理列表
print(fetcher.get_proxies()) # or print(fetcher.pool)
```

测试结果：

　![](/img/2022-11-05-Python-Web-Scraping-Notes-Using-the-Open-Source-Proxy-Pool-haipproxy/5.png){loading="lazy"}

 如果想满足大规模数据采集的话，最好手动更新 rules.py 中的代理网站。

## 3. 总结

相比于ProxyPool，更大的收获在于把 docker, redis, scrapy-splash
过了一遍（之前总是找理由拖着）；

Python 的学习不能浅尝辄止，项目源代码还是有一些看不懂需要学习的地方；

下步打算换个活跃的开源代理池对比一下。
