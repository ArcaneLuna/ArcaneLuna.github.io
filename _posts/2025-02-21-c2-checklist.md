---
layout: post
title: "CobaltStrike checklist"
date: 2025-02-21
author: "3thernet"
header-img: "img/city-3.jpg"
tags: []
---

## 一、安装

[Cobalt Strike 4.9.1 Cracked 破解版下载 - 🔰雨苁ℒ🔰](https://www.ddosi.org/cobalt-strike-4-9-1-cracked/)

听说有些知识星球版本的Cobalt Strike4.9.1在cmd中植入后门，因此一定要在虚拟机中使用

### 安装nginx

```
yum update -y
yum install -y epel-release
yum install -y nginx
systemctl start nginx
systemctl enable nginx #开机自启动
```

### 客户端安装

客户端安装java环境后直接点击cmd结尾的文件

[Cobalt Strike 安装与配置 - XSTARK 公司](https://red.lintstar.top/RAT/CobaltStrike/deploy)

[JDK安装教程，Win11环境_win11如何看本机jdk-CSDN博客](https://blog.csdn.net/weixin_43974616/article/details/124362974)

### 服务端安装

```
yum install java-11-openjdk-devel
java -version
update-java-alternatives -s
java-1.11.0-openjdk-amd64
#安装wget和unrar：
yum install wget
wget https://www.rarlab.com/rar/rarlinux-x64-6.0.2.tar.gz
tar xf rarlinux-x64-6.0.2.tar.gz -C /usr/local/
ln -s /usr/local/rar/rar /usr/local/bin/rar
ln -s /usr/local/rar/unrar
/usr/local/bin/unrar
unrar -v
#把cs压缩包传上去
unrar x -pwww.ddosi.org
CobaltSrike_4.9.1.rar
mv CobaltSrike_4.9.1_Cracked_www.ddosi.org CS
cd CS/Server
chmod +x teamserver
chmod +x TeamServerImage
```

## 二、隐藏特征

**在启动前必须隐藏特征并修改端口，否则长时间启动后会被威胁情报标记，然后listener会连接大量虚假sessions**

### 修改端口号

vi teamserver

修改最底下的50050为指定端口（比如12345）和密码

```
./TeamServerImage -Dcobaltstrike.server_port=12345 -Dcobaltstrike.server_bindto=0.0.0.0 -Djavax.net.ssl.keyStore=./cobaltstrike.store -Djavax.net.ssl.keyStorePassword=test123456 teamserver $*
```

密码是cobaltstrike.store解密密码，如果没有该文件，teamserver启动时会生成一个，默认密码0123456

[教你修改cobalt strike的50050端口 - 3HACK](https://www.3hack.com/note/96.html)

### 更换CS证书

不要使用默认的cobaltstrike.store，删除原本的cobaltstrike.store文件

```
keytool -keystore cobaltstrike.store -storepass test123456 -keypass test123456 -genkey -keyalg RSA -alias blackcat -dname "CN=Chen,OU=MOPR,O=Microsoft Corporation, L=Redmond, ST=Washington, C=US"

keytool -importkeystore -srckeystore cobaltstrike.store -destkeystore cobaltstrike.store -deststoretype pkcs12

keytool -list -v -keystore cobaltstrike.store -storepass test123456
```

### 域前置

购买域名并更改ns为cloudflare后，设置cloudflare ssl为完全（严格），添加A记录，然后源站点生成证书，创建服务器证书server.pem和私钥server.key

cloudflare还要配置”缓存规则“，这里设置的是对”所有传入请求“”绕过缓存“。

以下假设我购买的C2域名为：example.com

```
openssl pkcs12 -export -in server.pem -inkey server.key -out example.p12 -name example.com -passout pass:test123456

keytool -importkeystore -deststorepass test123456 -destkeypass test123456 -destkeystore example.store -srckeystore example.p12 -srcstoretype PKCS12 -srcstorepass test123456 -alias example.com
```

上述命令导入密钥到了`example.store`

确保c2.profile中包含：

```
https-certificate {
    set keystore "example.store";
    set password "test123456";
}
```

[CobaltStrike特征修改-安全客 - 安全资讯平台](https://www.anquanke.com/post/id/278690)

https://xz.aliyun.com/t/9616

[c2隐藏&流量加密 - 小新07 - 博客园](https://www.cnblogs.com/xiaoxin07/p/18203577#%E5%9F%9F%E5%89%8D%E7%BD%AE%E9%85%8D%E7%BD%AE)

[Cobalt Strike去特征：配置Nginx反向代理、CDN与Cloudflare Worker - MYZXCG](https://myzxcg.com/2020/12/Cobalt-Strike%E5%8E%BB%E7%89%B9%E5%BE%81%E9%85%8D%E7%BD%AENginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86CDN%E4%B8%8ECloudflare-Worker/)

配置完cdn后，会发现机器能上线，但是执行命令无回显。这个问题困扰了好半天，一直以为是cloudlflare缓存配置问题，然而禁用缓存后问题依旧存在。最后逐步定位到是cs配置文件的问题。

这里用的是[jquery-c2.4.0.profile](https://github.com/threatexpress/malleable-c2/blob/master/jquery-c2.4.0.profile)配置文件。它设置`header "Content-Type" "application/javascript; charset=utf-8";`为响应头。修改所有响应头为：`header "Content-Type" "application/*; charset=utf-8";`即可正常执行命令回显。

这里的mime-type如果为application/javascript、text/html等，机器执行命令就无法回显。猜测是cdn会检测响应头content-type的值，如果是一些静态文件的mime-type可能就导致这个问题。

[Cobalt Strike特征隐藏与流量分析 - ol4three](https://www.ol4three.com/2021/10/28/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F/CobaltStrike/Cobalt-Strike%E7%89%B9%E5%BE%81%E9%9A%90%E8%97%8F%E4%B8%8E%E6%B5%81%E9%87%8F%E5%88%86%E6%9E%90/)

### nginx

网上有些域前置攻略没有配置nginx转发请求，我经过测试发现这样往往无法回连

先将pem和key复制到另一个目录

/etc/nginx/nginx.conf包含以下内容

```
    server {
        listen       80;
        listen       [::]:80;
        server_name  example.com;
        return 301 https://$host$request_uri;
    }
    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/opt/zs/server.pem";
        ssl_certificate_key "/opt/zs/server.key";

        location / {
            proxy_pass          https://127.0.0.1:8777;
        }
    }
```

这里proxy_pass转发的端口就是我们设置监听器的bindto端口（C2端口为443）

配置防火墙，防止被监听端口被nmap扫描

注意iptables需要先删除规则：

`5    REJECT     all  --  anywhere             anywhere             reject-with icmp-host-prohibited`

```
iptables -A INPUT -s 127.0.0.1 -p tcp --dport 8777 -j ACCEPT
iptables -A INPUT -p tcp --dport 8777 -j DROP
```

nginx设置指定useragent：

```
location / {
        if ($http_user_agent != "Mozilla/5.0 (Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko") {
        return 302 $REDIRECT_DOMAIN$request_uri;
        }
        proxy_pass          https://127.0.0.1:8777;
}
```

## 三、Malleable C2 Profile

Malleable C2 Profile是一个可以控制CobaltStrike流量特征的文件，可以通过修改配置文件来改变流量特征，从而躲避各种流量检测设备及防病毒系统

通过修改框架内的各种默认值，操作者可以修改Beacon的内存占用，更改其检入的频率，甚至可以修改 Beacon的网络流量。所有这些功能都是由Malleable C2配置文件控制，该配置文件在启动团队服务器时选择。

[Cobalt Strike特征消除第二篇:配置C2 profile规避流量检测](https://mp.weixin.qq.com/s/CNJ-HS-z0SV0W8DKgRDMgw)

[Cobalt Strike特征隐藏与流量分析 - ol4three](https://www.ol4three.com/2021/10/28/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F/CobaltStrike/Cobalt-Strike%E7%89%B9%E5%BE%81%E9%9A%90%E8%97%8F%E4%B8%8E%E6%B5%81%E9%87%8F%E5%88%86%E6%9E%90/)

[threatexpress/malleable-c2: Cobalt Strike Malleable C2 Design and Reference Guide](https://github.com/threatexpress/malleable-c2)

[malleable-c2/MalleableExplained.md 在 master ·威胁快递/可塑性 C2](https://github.com/threatexpress/malleable-c2/blob/master/MalleableExplained.md)

[深入研究cobalt strike malleable C2配置文件 - 先知社区](https://xz.aliyun.com/t/2796?time__1311=n4%2Bxni0%3DG%3DGQeDK0QG8Dlx63oxmqbRib3orTD)

[rsmudge/Malleable-C2-Profiles: Malleable C2 is a domain specific language to redefine indicators in Beacon's communication. This repository is a collection of Malleable C2 profiles that you may use. These profiles work with Cobalt Strike 3.x.](https://github.com/rsmudge/Malleable-C2-Profiles/tree/master)

我的这里c2.profile是从上面github复制的jquery-c2.4.9.profile进行修改

检测语法是否有问题：

`./c2lint c2.profile`

### 关闭stager payload

set host_stage "false";

[大海捞“帧”：Cobalt Strike服务器识别与staging beacon扫描_cobalt strike服务器扫描-CSDN博客](https://blog.csdn.net/qq_53058639/article/details/132080913)

- 服务器去特征：修改默认配置

- 服务器去特征：域前置

- 流量去特征：Malleable C2 Profile

- Beacon staging server去特征：修改stager异或密钥

[Bypass cobaltstrike beacon config scan-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1764340)

尽量使用stageless payload

## 四、使用

### 基础

启动关闭：

nohup ./teamserver [ip地址] [密码] c2.profile &

pkill -f teamserver

乱码解决：

客户端打开cmd复制java启动内容添加选项-Dfile.encoding=utf-8

[界面功能介绍 - Cobalt Strike](https://wbglil.gitbook.io/cobalt-strike/cobalt-strikeji-ben-shi-yong/jie-mian-gong-neng-jie-shao)

[Cobalt_Strike_wiki/第一节[环境搭建与基本功能].md at master ·阿伦兹/Cobalt_Strike_wiki](https://github.com/aleenzz/Cobalt_Strike_wiki/blob/master/%E7%AC%AC%E4%B8%80%E8%8A%82%5B%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E4%B8%8E%E5%9F%BA%E6%9C%AC%E5%8A%9F%E8%83%BD%5D.md)

### 监听器和后门

[cobaltstrike之创建监听器与生成后门_cobaltstrike windows executable windows
executable-CSDN博客](https://blog.csdn.net/qq_44159028/article/details/118157559)

[CobaltStrike之后门生成_生成hta(payloads -> html application)与exe (payloads -CSDN博客](https://blog.csdn.net/bring_coco/article/details/116425165)

### 经验教训

[奇安信攻防社区-反制Cobaltstrike的那些手段](https://forum.butian.net/share/1975)

[记对cobalt strike的反制思路研究 - 先知社区](https://xz.aliyun.com/t/14464?u_atoken=f42b4b1d9501164dd188d0afb6e937a6&u_asig=ac11000117314195430843597e0038&time__1311=muDtAIqjrx8WDsD7GG7DyDIEzK6krF2awpD)

[【攻防演练】记一次攻防演练被某部委安全团队拷打全过程 - CN-SEC 中文网](https://cn-sec.com/archives/3205275.html)

## 五、插件

尽量从客户端加载cna文件，服务器端也可以加载，但我这里失败了

[lintstar/LSTAR: LSTAR - CobaltStrike 综合后渗透插件](https://github.com/lintstar/LSTAR) 略旧，2022年的

[lintstar/CS-AutoPostChain: 基于 OPSEC 的 CobaltStrike 后渗透自动化链](https://github.com/lintstar/CS-AutoPostChain?tab=readme-ov-file)

[k8gege/Aggressor: Ladon 911 for Cobalt Strike](https://github.com/k8gege/Aggressor)

[OLa 一款cobalt strike后渗透模块插件 - 🔰雨苁ℒ🔰](https://www.ddosi.org/cobalt-strike-ola/)

[Cobalt Strike Aggressor Script - f_carey - 博客园](https://www.cnblogs.com/f-carey/p/17536733.html)
