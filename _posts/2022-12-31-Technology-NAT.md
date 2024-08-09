---
layout: post
title: NAT
date: 2022-12-31
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Network

---

## 1. 简介

NAT，全称Network Address Traslation，是一种将 IP
地址从一个地址域映射到另一个地址域的方法，旨在为主机提供透明路由。

通俗地讲，NAT能够让内网中**[已经分配私有IP]{style="color: #ff9900;"}**但**[没有分配公网IP]{style="color: #3366ff;"}**的主机和Internet上的主机通信。

NAT的实质是修改数据包的源地址/目的地址：

- 内部主机访问公共网络，数据包经过NAT路由器时，就将[**源地址**]{style="color: #339966;"}从私有地址转换为公网地址
- 公共网络访问内部主机，数据包经过NAT路由器时，就将[**目的地址**]{style="color: #ff6600;"}从公网地址转换为私有地址

华为的知识百科根据**NAT是对报文中的源地址进行转换还是对目的地址进行转换**，将NAT分为[源NAT]{style="background-color: #ffff00;"}、[目的NAT]{style="background-color: #ffff00;"}和[双向NAT]{style="background-color: #ffff00;"}。

【不过在RFC
2663中并没有目的NAT的说法，只提到了传统NAT/出站NAT和双向NAT。传统NAT相当于源NAT】

RFC 2663 规定了传统NAT的两种方法：**Basic NAT**和**NAPT**(Network
Address Port Traslation)

## 2. Basic NAT

### 2.1 Static Nat

**静态转换**是指将内部网络的私有IP地址**[一对一]{style="color: #ff0000;"}**转换为公有IP地址，而且在整个NAT操作的生存期内都不会释放。

### 2.2 Dynamic Nat/Pooled Nat

**动态转换**通过维护一个外部的IP池，当内部有计算机需要和外部通信时，就从IP池里动态的取出一个IP，并将内部IP到外部IP的映射绑定到NAT表中，通信结束后，这个外网IP将被释放，供其他内部IP地址转换使用。整个过程属于**[多对多]{style="color: #ff0000;"}**，内部计算机获得IP地址是不确定的。当ISP提供的合法IP地址略少于网络内部的计算机数量时。可以采用动态转换的方式。

## 3. NAPT

### 3.1 标准NAPT

网络地址端口转换，对外出数据包的**[传输标识符]{style="color: #ff9900;"}**（例如[**TCP和UDP端口号**]{style="color: #3366ff;"}、ICMP查询标识符）进行转换，允许一组主机共享一个外部地址。NAPT有效解决了IPv4地址不足的问题，且能够对外隐藏内部主机，是最为常见的NAT方法。NAPT常常与Basic
Nat结合，维护一个外部IP池来进行转换。

### 3.2 Easy IP

特殊的NATP。在标准的 NAPT 配置中需要创建公网地址池，也就是必须先知道公网
IP 地址的范围。而在拨号接入的上网方式中，公网 IP
地址是由运营商动态分配的，无法事先确定，标准的 NAPT
无法做地址转换。要解决这个问题，就要使用 **Easy IP** 。

**Easy IP** 又称为基于接口的地址转换。在地址转换时，Easy IP 的工作原理与
NAPT 相同，对数据包的 IP 地址、协议类型、传输层端口号同时进行转换。但
Easy IP 直接使用公网接口的 IP 地址作为转换后的源地址。Easy IP
适用于**拨号接入**互联网，动态获取公网 IP 地址的场合。


## 4. 目的NAT

### 4.1 静态目的NAT & 动态目的NAT

目的NAT可以看作是对源NAT的溯源。根据**转换前后的地址是否存在一种固定的映射关系**，分为静态目的NAT和动态目的NAT，一般来讲内部主机对外提供服务时，公网地址到私有地址的映射是固定的，但也有特殊情况：

在移动终端访问无线网络时，如果其缺省WAP网关地址与所在地运营商的WAP网关地址不一致时，可以在终端与WAP网关中间部署一台设备，并配置目的NAT功能，使设备自动将终端发往错误WAP网关地址的报文自动转发给正确的WAP网关。

### 4.2 NAT Server

特殊的静态目的NAT。**NAT
表项一般由私网主机主动向公网主机发起访问而生成，公网主机无法主动向私网主机发起连接。**因此
NAT
隐藏了内部网络结构，具有屏蔽主机的作用。但是在实际应用中，内网网络可能需要对外提供服务，例如
Web 服务，常规的 NAT
就无法满足需求了。为了满足公网用户访问私网内部服务器的需求，需要使用
**NAT Server**
功能，将私网地址和端口静态映射成公网地址和端口，供公网用户访问。

</div>

## 5. 双向NAT

双向NAT指的是在转换过程中同时转换报文的源信息和目的信息。双向NAT不是一个单独的功能，而是源NAT和目的NAT的组合。双向NAT是针对同一条流，在其经过设备时同时转换报文的源地址和目的地址。双向NAT主要应用在同时有外网用户访问内部服务器和私网用户访问内部服务器的场景。

## 参考

1. [RFC 2663: IP Network Address Translator (NAT) Terminology and
   Considerations
   (rfc-editor.org)](https://www.rfc-editor.org/rfc/rfc2663)
2. [RFC 5382: NAT Behavioral Requirements for TCP
   (rfc-editor.org)](https://www.rfc-editor.org/rfc/rfc5382#section-4.3)
3. [NAT转换是怎么工作的？ - 网工Fox的回答 -
   知乎](https://www.zhihu.com/question/31332694/answer/1917791148){target="_blank"}
4. [什么是NAT？NAT的类型有哪些？ - 华为
   (huawei.com)](https://info.support.huawei.com/info-finder/encyclopedia/zh/NAT.html)
