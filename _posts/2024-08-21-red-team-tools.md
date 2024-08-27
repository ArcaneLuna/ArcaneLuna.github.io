---
layout: post
title: "【网络安全】红队工具自测"
subtitle: "长期更新"
date: 2024-08-21
author: "3thernet"
header-img: "img/bg-touhou-1.jpg"
tags: []
---

## 0x01 信息收集

### 爱企查

同类产品不一一枚举，信息收集的第一步：APP、企业信息、ICP备案、微信公众号等

备案拿到主域名后就可以开展进一步信息收集

抓包APP、微信公众号的服务也能发现一些网址，~~防备通常比主域名低~~

### TscanPlus

https://github.com/TideSec/TscanPlus

集成化GUI工具，好用，但是闭源

### ShuiZe

https://github.com/0x727/ShuiZe_0x727

较全面的信息收集，最好用docker部署，好用

### Oneforall

https://github.com/shmilylty/OneForAll

用的时候有未知bug，总爆内存

### 洞鉴X-RAY

https://xray.chaitin.cn/

企业级漏扫工具，试用了两天。智能信息收集和标注脆弱资产

### dddd

https://github.com/SleepingBag945/dddd

支持hunter、fofa api，适合DIY配置指纹和POC

### ARL灯塔

[Aabyss-Team/ARL: ARL官方仓库备份项目](https://github.com/Aabyss-Team/ARL)

广受好评

### 阿波罗

[b0bac/ApolloScanner: 自动化巡航扫描框架](https://github.com/b0bac/ApolloScanner)

提供虚拟机版本，还未试用

### GitHack

[lijiejie/GitHack](https://github.com/lijiejie/GitHack)

源码泄露重建

### 各种空间测绘GUI

```
https://github.com/HHa1ey/TKHunter  # Hunter
https://github.com/G3et/Search_Viewer  # fofa,hunter,shodan,quake,zoomeye,censys
https://github.com/ZuoJunhao/QuakeViewer  # quake
https://github.com/qiwentaidi/Slack  # 和tscanplus比较像
https://github.com/fasnow/fine  # fofa,hunter,quake,零零信安
```

### Google Hacking

在线：[在线Google Hacking小工具 (se7ensec.cn)](https://ght.se7ensec.cn/#)

语法：

```
管理后台地址
site:target.com intext:管理 | 后台 | 后台管理 | 登陆 | 登录 | 用户名 | 密码 | 系统 | 账号 | login | system
site:target.com inurl:login | inurl:admin | inurl:manage | inurl:manager | inurl:admin_login | inurl:system | inurl:backend
site:target.com intitle:管理 | 后台 | 后台管理 | 登陆 | 登录

上传类漏洞
site:target.com inurl:file
site:target.com inurl:upload

注入漏洞
site:target.com inurl:php?id=

编辑器
site:target.com inurl:ewebeditor

目录遍历漏洞
site:target.com intitle:index.of

SQL错误信息
site:target.com intext:"sql syntax near" | intext:"syntax error has occurred" | intext:"incorrect syntax near" | intext:"unexpected end of SQL command" | intext:"Warning: mysql_connect()" | intext:”Warning: mysql_query()" | intext:”Warning: pg_connect()"
phpinfo()
site:target.com ext:php intitle:phpinfo "published by the PHP Group"

配置文件泄露
site:target.com ext:.xml | .conf | .cnf | .reg | .inf | .rdp | .cfg | .txt | .ora | .ini

数据库文件泄露
site:target.com ext:.sql | .dbf | .mdb | .db

日志文件泄露
site:target.com ext:.log

备份和历史文件泄露
site:target.com ext:.bkf | .bkp | .old | .backup | .bak | .swp | .rar | .txt | .zip | .7z | .sql | .tar.gz | .tgz | .tar

公开文件泄露
site:target.com filetype:.doc | .docx | .xls | .xlsx | .ppt | .pptx | .odt | .pdf | .rtf | .sxw | .psw | .csv

邮箱信息
site:target.com intext:@target.com
site:target.com 邮件
site:target.com email

社工信息
site:target.com intitle:账号 | 密码 | 工号 | 学号 | 身份证
```

### 漏洞情报

[微步在线X情报社区-威胁情报查询_威胁分析平台_开放社区 (threatbook.com)](https://x.threatbook.com/)

以及各种公众号

## 0x02 指纹识别

### Wappalyzer

浏览器插件，识别技术栈比较全面。

### kscan

https://github.com/lcvvvv/kscan

比较精巧，用来主动识别应用层指纹尚可。指纹库很久没更新，不一定有空间测绘引擎识别的精准。

### Ehole

https://github.com/EdgeSecurityTeam/EHole

同上，指纹库去年还有更新。

### 0x03 登录爆破

[各大OA产品试用地址&初始账户密码 - komomon - 博客园 (cnblogs.com)](https://www.cnblogs.com/forforever/p/13611321.html) 不是很全

### 字典

[gh0stkey/Web-Fuzzing-Box: Web Fuzzing Box - Web 模糊测试字典与一些Payloads (github.com)](https://github.com/gh0stkey/Web-Fuzzing-Box/tree/main) 综合字典，2k+stars且近期还有更新

其他字典集合：[wsecz/dict-hub: 红队字典：默认凭证、弱用户名、弱口令、弱Web路径 (github.com)](https://github.com/wsecz/dict-hub)

## 0x04 漏洞利用

**POC:** [wy876/POC: 收集整理漏洞EXP/POC,大部分漏洞来源网络，目前收集整理了900多个poc/exp，长期更新。 (github.com)](https://github.com/wy876/POC)

**Webshell**: 菜刀，冰蝎，哥斯拉，metasploit

### 4.1 重点OA

[【工具更新】红蓝对抗重点OA系统漏洞利用工具增强版V2.3 发布！ (qq.com)](https://mp.weixin.qq.com/s/1MEkYv5A47DJLCKqXKT0zQ)

[R4gd0ll/I-Wanna-Get-All: OA漏洞利用工具 (github.com)](https://github.com/R4gd0ll/I-Wanna-Get-All)

泛微OA：[zhaoyumi/WeaverExploit_All: 泛微最近的漏洞利用工具（PS：2023） (github.com)](https://github.com/zhaoyumi/WeaverExploit_All)

### 4.2 Ruoyi

#### （1）SQL注入+定时任务RCE

2024较新，v4.7.8，需要登录后台

https://github.com/charonlight/RuoYiExploitGUI

[若依4.7.8版本计划任务rce复现_若依计划任务rce-CSDN博客](https://blog.csdn.net/qq_45813980/article/details/139775272)

### （2）ruoyi综合利用工具

[若依漏洞综合利用工具 (qq.com)](https://mp.weixin.qq.com/s/HU1OqZ7YFGjfc4HE3M8tPA)

### 4.3 jboss

[Jboss综合利用工具 (qq.com)](https://mp.weixin.qq.com/s/O0sMwNHRZKn4G4uJC55YlQ)

[fupinglee/JavaTools: 一些Java编写的小工具。 (github.com)](https://github.com/fupinglee/JavaTools)

很老的jboss, shiro利用工具

### 4.4 Shiro

[SummerSec/ShiroAttack2: shiro反序列化漏洞综合利用](https://github.com/SummerSec/ShiroAttack2)

### 4.5 tomcat

[einzbernnn/Tomcatscan: Tomcat漏洞批量检测工具 (github.com)](https://github.com/einzbernnn/Tomcatscan)

批量检测 CVE-2020-1938

### 4.6 log4j

[fullhunt/log4j-scan: A fully automated, accurate, and extensive scanner for finding log4j RCE CVE-2021-44228 (github.com)](https://github.com/fullhunt/log4j-scan)

经典CVE-2021-4104，建议使用`--dns-callback-provider dnslog.cn`选项避免回调错误

### 4.7 FineReport

[Drac0nids/FineReportExploit: 基于go语言的帆软报表漏洞检测工具 (github.com)](https://github.com/Drac0nids/FineReportExploit)

### 4.8 Citrix

[securekomodo/citrixInspector: Accurately fingerprint and detect vulnerable (and patched!) versions of Netscaler / Citrix ADC to CVE-2023-3519 (github.com)](https://github.com/securekomodo/citrixInspector)

### 0x06 参考

- [红队实战手册 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/283768.html) 写的比较详细

- [r0eXpeR/RedTeamAttack: 关于红队方面的一些工具\资料\Checklist (github.com)](https://github.com/r0eXpeR/RedTeamAttack)

- [r0eXpeR/redteam_vul: 红队作战中比较常遇到的一些重点系统漏洞整理。 (github.com)](https://github.com/r0eXpeR/redteam_vul)
