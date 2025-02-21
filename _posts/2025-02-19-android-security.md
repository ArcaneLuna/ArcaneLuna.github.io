---
layout: post
title: "【系统安全】Android Security"
date: 2025-02-19
author: "3thernet"
header-img: "img/city-1.jpg"
tags: []
---

## 一、Android系统架构

![](/img/2025-02-19-android-security/1.png)

**Linux内核层**：最底层

**硬件抽象层（HAL）**：硬件接口标准化实现，屏蔽硬件实现的差异

**Android Runtime（ART）**：包含核心库和运行时环境。每个应用都在其自己的进程中运行，都有自己的虚拟机实例。ART通过执行DEX文件可在设备运行多个虚拟机，**DEX文件是一种专为Android设计的字节码格式文件**，经过优化，使用内存很少。

**Application Framework**：

通过提供开放的开发平台，android使开发者能够编制极其丰富和新颖的应用程序。开发者可以自由地利用设备硬件优势、访问位置信息、运行后台服务、向状态栏添加通知。

开发者完全可以利用核心应用程序所使用的框架APIs。应用程序的体系结构旨在简化组件的重用，任何应用程序都能发布它的功能且任何其它应用程序可以使用这些功能（需要从框架执行的安全限制）。这一机制允许用户替换组件。所有的应用程序其实是一组服务和系统：

A）View：丰富的、可扩展的视图集合，可用于构建一个应用程序。包括列表，网格、文本框、按钮，甚至内嵌的网络浏览器。

B）Content Provider：是应用程序能访问其他应用程序（如通讯录）的数据，或共享自己的数据。

C）Resource Manager：提供访问非代码资源，如本地化字符串、图形和布局文件。

D）Notification Manager：使所有的应用程序能够在状态栏显示自定义警告。

E）Activity Manager：管理应用程序生命周期，提供通用的导航回退功能。

**App层**：

所有的应用程序都是使用java语言编写的，每一个应用程序由一个或多个activity组成，activity类似于操作系统上的进程，但是活动比操作系统的进程要更为灵活，与进程类似的是，活动在多种状态之间切换。

利用java的跨平台性质，基于android框架开发的应用程序可以不用编译而运行于任何一台安装有Android系统的平台。

## 二、DVM、DEX文件和ART

JVM 是 JAVA 虚拟机，用来运行 JAVA 字节码程序。Dalvik 是 Google 设计的用于 Android平台的运行时环境，适合移动环境下内存和处理器速度有限的系统。ART 即 Android Runtime，是 Google 为了替换 Dalvik 设计的新 Android 运行时环境，在Android 4.4推出。ART 比 Dalvik 的性能更好。Android 程序一般使用 Java 语言开发，但是 Dalvik 虚拟机并不支持直接执行 JAVA 字节码，所以会对编译生成的 .class 文件进行翻译、重构、解释、压缩等处理，这个处理过程是由 dx 进行处理，处理完成后生成的产物会以 .dex 结尾，称为 Dex 文件。Dex 文件格式是专为 Dalvik 设计的一种压缩格式。所以可以简单的理解为：Dex 文件是很多 .class 文件处理后的产物，最终可以在 Android 运行时环境执行。（.java->.class->.dex）

DVM是Dalvik Virtual Machine的缩写，是安卓虚拟机的缩写（为什么不叫AVM-Android Virtual Machine呢？原因是其作者以其祖上居住过的名为Dalvik的村子命名）。DVM是针对JVM（Java Virtual Machine）而言的，因为JVM是Oracle公司（原SUN公司）的产品，担心版权的问题，既然Java是开源的，索性就研究了JVM，写出了DVM

## 三、Android四大组件

Android 四大组件包括 **Activity、Service、Content Provider 和 Broadcast Receiver**，它们是 Android 应用程序的基本构成单元，主要用于实现应用的功能模块和用户交互。它们属于 Android 系统架构中的 **Application Framework（应用框架层）**。

### 1、Activity

用户界面屏幕，每个界面通常对应一个Activity，例如登陆页面、设置页面等

**特点**：

- 通过生命周期方法（如 onCreate、onResume）管理界面。
- 可以通过 Intent 启动其他 Activity。

**典型场景**：
 显示用户界面并处理用户操作，比如登录按钮的点击事件。

### 2、Services

后台执行长时间运行任务的组件，不直接与用户交互。

**特点**：

- 可以运行在前台（Foreground Service）或后台（Background     Service）。
- 常用于处理音乐播放、文件下载等需要持续运行的任务。

**典型场景**：
 实现音乐播放应用的后台播放功能，即使用户切换到其他应用，音乐仍然继续播放。

### 3、Content Provider

Content Provider 用于在不同应用之间共享数据，提供统一的数据访问接口。

**特点**：

- 使用 URI（统一资源标识符）定位数据。
- 支持数据的增删改查操作。
- 可访问设备上的联系人、媒体等共享资源。

**典型场景**：
 一个日历应用通过 Content Provider 将事件信息提供给其他应用。

### 4、Broadcast Receiver：

用于接收和处理系统广播或应用间的广播消息。

**特点**：

- 无用户界面，常用于响应事件（如设备启动、电池电量变化等）。
- 广播分为两类：标准广播和有序广播。

**典型场景**：
 接收系统广播，例如监听设备启动完成（BOOT_COMPLETED）以执行特定操作。

## 四、Android安全挑战

1、Activity安全

Activity Launch Mode

绕过本地认证，通过软件key验证，免费激活

隐式启动Intent包含敏感数据：https://wenku.csdn.net/column/cr5grdpeqd

Fragment注入

webview RCE

2、Services安全

切换到另一个应用也会继续运行

组件可以绑定到一个service来交互

权限提升

jp.aplix.midp.tools应用宝

system运行并提供ApkInstaller服务，可导致安装删除任意package

Service劫持

3、Content Provider安全

Android四种存储方式：SQLite, SharedPreference, File, ContentProvider

![](/img/2025-02-19-android-security/2.png)

暴露ContentProvider, SQL注入，OpenFile文件遍历

4、Broadcast安全

很多广播源自系统代码，应用程序也可以进行广播。应用程序也可以进行广播。应用程序可以拥有任意数量的BR对所有感兴趣的进行响应。

（1）伪造消息代码执行

盗取隐私文件和执行任意代码

（2）拒绝服务：不完整intent

（3）广播账号密码

## 五、Android安全机制

### 1、沙箱隔离机制

（1）UID机制：每个 Android 应用安装时，系统会为其分配一个唯一的 Linux UID（通常从 10000 开始）。该应用的所有进程都以该 UID 运行，其他应用进程无法访问其资源。【**注意不是PID**】这种机制确保了即使一个应用被恶意软件感染，其他应用和系统文件仍然是安全的。

如果两个应用由同一个开发者签名，并在 AndroidManifest.xml 中声明相同的 sharedUserId，它们可以共享相同的 UID：

<manifest xmlns:android="http://schemas.android.com/apk/res/android"

  package="com.example.app"

**android:sharedUserId**="com.example.shared">

但 Google 从 Android 10 开始不再支持 sharedUserId

（2）标准的进程隔离机制

（3）文件系统隔离

每个应用的数据存储在 /data/data/<package_name>/ 目录下：

/data/data/com.example.appA/

├── shared_prefs/  # 存储应用的 SharedPreferences

├── databases/   # SQLite 数据库

├── cache/     # 缓存数据

├── files/     # 应用存储的普通文件

**默认情况下，只有该应用的 UID 进程可以访问这个目录**，其他应用无法访问。

$ ls -l /data/data/com.example.appA/

drwx------ 2 u0_a1 u0_a1 4096 Jun 20 10:00 shared_prefs

drwx------ 2 u0_a1 u0_a1 4096 Jun 20 10:00 databases

**drwx------（700 权限）**：只有 u0_a1（应用 A 的 UID）可以访问，其他应用无法访问。

应用可以通过 **Content Provider** 公开部分数据供其他应用访问：

Uri uri = Uri.parse("content://com.example.provider/data");

Cursor cursor = getContentResolver().query(uri, null, null, null, null);

漏洞：写到SDcard的文件会被共享

### 2、访问控制机制(SELinux)

Android 4.3 引入 **SELinux（Security-Enhanced Linux）**，Android 5.0 以后默认强制启用：

- **SELinux 采用强制访问控制（MAC, Mandatory Access     Control）**，进一步限制应用进程的行为。
- 例如，即使一个应用获取了 root 权限，SELinux 仍然可以阻止其访问某些敏感数据。

**SELinux 策略示例**

$ adb shell dmesg | grep "avc: denied"

avc: denied { read } for pid=2345 comm="app_process" name="passwd" dev="mmcblk0p3"

- 这表明某个应用试图读取 /etc/passwd 文件，但被 SELinux 阻止。

### 3、应用程序签名机制

Android 的签名方案，发展到现在，不是一蹴而就的。Android 现在已经支持三种应用签名方案：

- v1 方案：基于 JAR 签名。
- v2 方案：APK 签名方案     v2，在 Android 7.0 引入。
- v3 方案：APK 签名方案     v3，在 Android 9.0 引入。

v1 到 v2 是颠覆性的，为了解决 JAR 签名方案的安全性问题，而到了 v3 方案，其实结构上并没有太大的调整，可以理解为 v2 签名方案的升级版，有一些资料也把它称之为 v2+ 方案。

因为这种签名方案的升级，就是向下兼容的，所以只要使用得当，这个过程对开发者是透明的。

v1 到 v2 方案的升级，对开发者影响最大的，就是渠道签署的问题。在当下这个大环境下，我们想让不同渠道、市场的安装包有所区别，携带渠道的唯一标识，这就是我们俗称的渠道包。好在各大厂都开源了自己的签渠道方案，例如：Walle（美团）、VasDolly（腾讯）都是非常优秀的方案。

#### （1）APK签名方案v1

Android 应用的签名工具有两种：jarsigner 和 apksigner。它们的签名算法没什么区别，主要是签名使用的文件不同。

- jarsigner：jdk 自带的签名工具，可以对 jar 进行签名。使用 keystore 文件进行签名。生成的签名文件默认使用 keystore 的别名命名。
- apksigner：Android sdk 提供的专门用于 Android 应用的签名工具。使用 pk8、x509.pem 文件进行签名。其中 pk8 是私钥文件，x509.pem 是含有公钥的文件。生成的签名文件统一使用“CERT”命名。

既然这两个工具都是给 APK 签名的，那么 keystore 文件和 pk8，x509.pem 他们之间是不是有什么联系呢？答案是肯定的，他们之间是可以转化的，这里就不再分析如何进行转化，网上的例子很多。

还有一个需要注意的知识点，如果我们查看一个keystore 文件的内容，会发现里面包含有一个 MD5 和 SHA1 摘要，这个就是 keystore 文件中私钥的数据摘要，这个信息也是我们在申请很多开发平台账号时需要填入的信息。

在 META-INF 文件夹下有三个文件：MANIFEST.MF、CERT.SF、CERT.RSA。它们就是签名过程中生成的文件

![](/img/2025-02-19-android-security/3.png)

**v1 签名机制的劣势**

从 Android 7.0 开始，Android 支持了一套全新的 V2 签名机制，为什么要推出新的签名机制呢？通过前面的分析，可以发现 v1 签名有两个地方可以改进：

- 签名校验速度慢

校验过程中需要对apk中所有文件进行摘要计算，在 APK 资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢。

- 完整性保障不够

META-INF 目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件。

为了解决这两个问题，在 Android 7.0 Nougat 中引入了全新的 APK Signature Scheme v2。

**v2 带来了什么变化？**

由于在 v1 仅针对单个 ZIP 条目进行验证，因此，在 APK 签署后可进行许多修改 — 可以移动甚至重新压缩文件。事实上，编译过程中要用到的 ZIPalign 工具就是这么做的，它用于根据正确的字节限制调整 ZIP 条目，以改进运行时性能。而且我们也可以利用这个东西，在打包之后修改 META-INF 目录下面的内容，或者修改 ZIP 的注释来实现多渠道的打包，在 v1 签名中都可以校验通过。

v2 签名将验证归档中的所有字节，而不是单个 ZIP 条目，因此，在签署后无法再运行 ZIPalign（必须在签名之前执行）。正因如此，现在，在编译过程中，Google 将压缩、调整和签署合并成一步完成。

#### （2）APK签名方案v2

简单来说，v2 签名模式在原先 APK 块中增加了一个新的块（签名块），新的块存储了签名，摘要，签名算法，证书链，额外属性等信息

![](/img/2025-02-19-android-security/4.png)

签名过程：

首先，说一下 APK 摘要计算规则，对于每个摘要算法，计算结果如下:

- 将 APK 中文件 ZIP 条目的内容、ZIP 中央目录、ZIP 中央目录结尾按照 1MB 大小分割成一些小块。
- 计算每个小块的数据摘要，数据内容是 0xa5 + 块字节长度 + 块内容。
- 计算整体的数据摘要，数据内容是 0x5a + 数据块的数量 + 每个数据块的摘要内容

总之，就是把 APK 按照 1M 大小分割，分别计算这些分段的摘要，最后把这些分段的摘要在进行计算得到最终的摘要也就是 APK 的摘要。然后将 APK 的摘要 + 数字证书 + 其他属性生成签名数据写入到 APK Signing Block 区块。

![](/img/2025-02-19-android-security/5.png)

具体参考：[Android-ReadTheFuckingSourceCode/article/android/basic/09_signature.md at master · jeanboydev/Android-ReadTheFuckingSourceCode · GitHub](https://github.com/jeanboydev/Android-ReadTheFuckingSourceCode/blob/master/article/android/basic/09_signature.md#manifestmf)

### 4、权限声明机制

如果您的应用要请求应用权限，您必须在应用的**清单文件**中声明这些权限。这些声明可帮助应用商店和用户了解您的应用可能会请求的权限组合。

请求权限的过程取决于权限类型：

- 如果是[安装时权限](https://developer.android.com/guide/topics/permissions/overview?hl=zh-cn#install-time)（例如一般权限或签名权限），系统会在安装您的应用时自动为其授予相应权限。
- 如果是[运行时权限](https://developer.android.com/guide/topics/permissions/overview?hl=zh-cn#runtime)或[特殊权限](https://developer.android.com/guide/topics/permissions/overview?hl=zh-cn#special)，并且您的应用安装在搭载 Android 6.0（API 级别 23）或更高版本的设备上，则您必须自己请求[运行时权限](https://developer.android.com/training/permissions/requesting?hl=zh-cn)或[特殊权限](https://developer.android.com/training/permissions/requesting-special?hl=zh-cn)。

向应用清单添加声明

如需声明应用可能请求的权限，请在应用的清单文件中添加相应的 <uses-permission> 元素。例如，如果应用需要访问相机，则应在 AndroidManifest.xml 中添加以下代码行：

<manifest ...>
   **<uses-permission android:name="android.permission.CAMERA"/>**
   <application ...>
     ...
   </application>
 </manifest>

### 5、进程通信机制(Inter-Process Communication)

Android 提供了多种 IPC 方式，主要包括：

- **Binder（核心 IPC 机制）**
- Messenger（基于 Binder 的轻量级 IPC）
- **AIDL（Android Interface Definition     Language）**
- ContentProvider（基于 SQL 的数据共享方式）
- BroadcastReceiver（进程间广播）
- Socket（基于网络的 IPC 方式）
- Shared Memory（共享内存，提高数据传输效率）

#### 5.1 Binder（Android 核心 IPC 机制）

5.1.1 什么是 Binder？

Binder 是 Android 系统的核心 IPC 机制，底层基于 Linux 的 Binder 驱动（/dev/binder） 实现。它的作用是：

- 跨进程传输数据
- 支持同步/异步调用
- 提高 IPC 效率，避免传统 Linux IPC（如 Socket、管道、共享内存）的复杂性和低效

Binder 采用 C/S（客户端-服务器）架构：

- Server（服务端）：提供服务，并注册到 Service Manager。
- Client（客户端）：请求服务端执行操作，并获取返回结果。
- Service Manager（Binder 驱动的管理者）：负责维护 Binder 对象的映射，管理进程间的服务查找。

5.1.2. Binder 的工作流程

1. 服务端创建 Binder：应用进程 A 通过 IBinder 创建一个 Binder 代理，并注册到系统 ServiceManager 中。
2. 客户端请求 Binder：应用进程 B 通过 ServiceManager 获取 Binder 对象的引用。
3. 客户端调用远程方法：应用进程 B 通过 Binder 调用进程 A 提供的远程方法。
4. Binder 驱动中转数据：Binder 内核驱动负责跨进程数据的传输，并将请求转发到进程 A 进行处理。
5. 进程 A 处理请求并返回数据：Binder 机制将数据返回给进程 B。

5.1.3 Binder 的优点

- 高效：相比传统 IPC 方式（如 Socket、管道），Binder 采用**内存映射（mmap）**提高了数据传输效率，减少了数据拷贝次数。
- 安全：Binder 进程间通信会自动携带进程 UID/PID，避免了未经授权的 IPC 通信。
- 易用：Android 提供了 AIDL、Messenger 这些上层封装，方便开发者使用。

#### 5.2 Messenger（基于 Binder 的轻量级 IPC）

Messenger 是 基于 Handler 的 IPC 方式，适用于简单的进程间消息传递（如发送字符串、整数等小数据）。其底层仍然基于 Binder。

5.2.1 Messenger 适用场景

- 适用于 一对一、轻量级进程通信
- 主要用于跨进程的异步消息传递
- 适合 Service 和 Activity 之间通信

5.2.2 Messenger 的特点

- 基于 Handler，适合消息传递
- 一次只能处理一个请求
- 轻量级，不适用于复杂的进程通信

#### 5.3 AIDL（Android Interface Definition Language）

AIDL 是 Android 提供的高级 IPC 方式，用于多个应用进程之间的远程方法调用。AIDL 是基于 Binder 的封装，适用于复杂的进程间通信（如对象传递）。

5.3.1 AIDL 适用场景

- 远程服务（Remote Service）
- 进程间传输复杂数据对象
- 适用于多个客户端访问同一服务

5.3.3 AIDL 的特点

- 支持复杂对象传输（Parcelable）
- 适合跨进程远程方法调用
- 底层基于 Binder，性能较高

![](/img/2025-02-19-android-security/6.png)

参考：

[Android 中的权限  | Android Developers](https://developer.android.com/guide/topics/permissions/overview?hl=zh-cn)

[声明应用权限  | Android Developers](https://developer.android.com/training/permissions/declaring?hl=zh-cn)

## 六、APP执行机理和链条

**Dalvik**：Dalvik 是 Android 早期的虚拟机（VM），它使用的是 **解释执行** 和 **JIT 编译** 方式。每当应用运行时，Dalvik 会将字节码解释为中间代码，并将其逐条执行。为了提高性能，Dalvik 会对热点代码进行即时编译（JIT）。

**ART**（Android Runtime）：ART 是 Android 5.0（Lollipop）开始引入的替代 Dalvik 的运行时，它主要采用 **AOT（Ahead-Of-Time）编译** 和 **JIT 编译**。ART 将应用的字节码预先编译为本地机器码，这大大提高了启动速度和运行效率，并且减少了运行时的开销。

### 1、Dalvik执行链

**Java/Kotlin** **代码**（源代码）

- 通过 javac 编译器编译成 .class 文件。

**dx/d8** **工具**：

- 将 .class 文件转换为 .dex 文件（Dalvik 字节码）。

**APK** **打包**：

- 将 .dex 文件打包到 APK 中。

**Dalvik VM** **解释执行**：

- **dexopt**工具对.dex文件进行优化，生成**odex**文件，从而提高应用的启动和执行速度
- **libdvm.so** 负责解释执行 .dex 字节码。
- 每条字节码在执行时都被逐步解析为中间代码，并由 Dalvik 虚拟机执行。

**JIT** **编译**：

- 对频繁执行的热点代码进行     **即时编译（JIT）**，将其转换为本地机器码，以提高执行效率。

**优点**：

- 初期 Android 系统适配设备资源有限，Dalvik 的解释执行模式在内存和 CPU 使用上相对较节省。

**缺点**：

- 启动速度慢，因为每次运行时都需要解释字节码。
- JIT 编译过程有延迟，应用的性能较差，特别是对于首次运行的代码。

### 2、ART执行链

**ART**（Android Runtime）自 **Android 5.0（Lollipop）** 开始成为默认的运行时。它的主要改进是 **AOT 编译**，通过提前编译（安装时）将 dex 文件编译为机器码，提升了应用启动速度和整体性能。

**ART 执行链：**

1. **Java/Kotlin 代码**（源代码）
   - 通过 javac 编译成 .class 文件。
2. **dx/d8 工具**：
   - 将 .class 文件转换为 .dex 文件。
3. **APK 打包**：
   - 将 .dex 文件打包到 APK 中。
4. **dex2oat 工具**（AOT 编译）：
   - 安装应用时，dex2oat 工具将 .dex 文件转换为 **OAT 文件（Optimized Android Runtime）**，这是 ART 运行时的优化字节码格式。
   - OAT 文件包含预编译的机器码，减少了运行时的开销。
5. **ART 执行**：
   - **libart.so** 负责执行 OAT 文件中的机器码。
   - ART 不需要每次解释执行字节码，而是直接运行本地机器码，大大提高了性能。
6. **JIT 编译（补充优化）**：
   - 如果有代码是首次运行或没有被预编译为机器码，ART 会启用 **JIT 编译**（即时编译），将这些代码编译为机器码。（兼容DVM）

**ART 优缺点：**

- **优点**：
  - 提升了      **应用启动速度** 和 **运行时性能**，因为大部分字节码都已经在安装时预编译成本地机器码。
  - 减少了运行时的延迟。
- **缺点**：
  - 安装时需要更多的时间进行      **AOT 编译**，因此首次安装应用时比 Dalvik 慢。
  - 对存储空间要求更高，因为需要保存预编译后的 OAT 文件。

![](/img/2025-02-19-android-security/7.png)

### 3、APP代码执行

DexFile->DexFile Parsing->Class Loading->Method Resolution->Method Executioin

![](/img/2025-02-19-android-security/8.png)

[PackerGrind: An Adaptive Unpacking System for Android Apps](https://faculty.ist.psu.edu/wu/papers/PackerGrind-TSE.pdf)

[【Android 逆向】整体加固脱壳 ( 脱壳点简介 | 修改系统源码进行脱壳 )_android 脱壳点-CSDN博客](https://blog.csdn.net/shulianghan/article/details/121921993)

## 七、逆向分析

逆向目的：还原程序功能，第三方平台重打包

Java层逆向：.APK->.DEX->.CLASS->.JAVA

Native层逆向

特点：使用C/C++编写，编译为ELF文件，与硬件或系统接口直接交互

目标：定位关键逻辑（如加密算法），分析JNI函数，排查内存操作漏洞

### 1、基础逆向

（1）使用APKTool查看Manifest.xml，初步判断权限是否超出需求

（2）解压APK，提取classes.dex, dex2jar反编译

（3）jd-gui（jadx）阅读Java代码

工具介绍：

#### （1）APKTool

APKTool 是一个开源工具，用于反编译和重编译 Android APK 文件。它将 APK 反编译成易于理解的资源文件和 Smali 代码（Android 的汇编语言），非常适合进行 APK 文件的反向工程。

**特点：**

- **反编译 APK：** 将 APK 中的资源文件（如布局文件、图片、manifest 文件等）解压出来，并将 DEX（Dalvik 字节码）转化为 Smali 代码。
- **重编译 APK：** 修改文件后，可以用 APKTool 重新打包生成新的 APK。
- **易于分析资源：**     可以轻松查看和修改 Android 项目的 XML 配置文件、图形资源等。
- **支持解密与解压缩**：可以解压 APK 文件并对加密的 APK 文件进行处理。

**优点：**

- 易于使用，界面友好。
- 可以修改并重新打包     APK，方便后续的实验与调试。
- 适合分析 APK 中的资源文件和 Smali 代码。

**缺点：**

- 无法直接获取 Java 源代码，只能获取 Smali 代码，分析难度较高。
- 不支持高级的控制流分析和 Java 反编译。

#### （2）soot

Soot 是一个 Java 静态分析框架，支持将 Java 字节码（包括 Android DEX 文件）转化为易于分析的中间表示，并进行控制流分析和数据流分析。Soot 可以与其他分析工具结合，提供对应用程序的全面分析。

**特点：**

- **支持多种输入格式：**     包括 Java 类文件、DEX 文件等。
- **高级分析：**     提供详细的控制流分析、数据流分析和调用图生成。
- **中间表示：**     使用 Jimple 或 Baf 作为中间表示，可以更方便地进行各种分析。
- **Java 反编译：** 可以将 DEX 反编译为 Java 代码，适合深入分析程序的结构。

**优点：**

- 强大的静态分析功能，可以进行详细的程序控制流分析。
- 支持的输入格式多，可以进行跨平台的分析。
- 能够处理 Android 应用中的复杂代码，适合安全研究人员和开发人员使用。

**缺点：**

- 学习曲线较陡峭，需要对静态分析和 Java 字节码有一定了解。
- 输出结果的可读性较差，需要一定的后处理。

**适用场景：**

- **漏洞挖掘：**     对 Android 应用进行深度分析，寻找潜在的漏洞。
- **控制流分析：**     用于复杂代码的控制流分析和数据流分析。

#### （3）JADX

JADX 是一个用于将 Android DEX 文件反编译为 Java 源代码的工具。它支持 Android APK 文件的反编译，能够将字节码转换为相对易懂的 Java 源代码。

**特点：**

- **Java 反编译：** 直接将 DEX 文件转换为 Java 源代码，易于分析。
- **支持 APK 文件：** 直接对 APK 文件进行反编译，无需先进行其他工具的处理。
- **集成 GUI：** 除了命令行工具，JADX 还提供了图形化界面，方便查看反编译后的代码和资源。

**优点：**

- 反编译后的 Java 代码相对较易理解，适合快速分析。
- 集成 GUI 和命令行版本，操作灵活。
- 支持直接查看 APK 中的 Java 类和资源文件。

**缺点：**

- 反编译的代码有时较难理解，特别是混淆后的代码。
- 无法提供像 Soot 那样深入的静态分析。

**适用场景：**

- **快速反编译：**     快速查看和分析 Android 应用中的 Java 代码。
- **混淆代码分析：**     对混淆代码进行一定的反混淆，帮助理解应用逻辑。

#### （4）AndroGuard

AndroGuard 是一个 Python 编写的静态分析框架，专门用于分析 Android APK 和 DEX 文件。它可以进行 APK 反编译、字节码分析、以及高级的控制流和数据流分析。

**特点：**

- **反编译：**     支持将 APK 文件反编译为 Smali 代码，并将 DEX 文件转化为 Java 代码。
- **支持多种分析功能：**     包括控制流分析、数据流分析、调用图生成等。
- **集成静态分析功能：**     提供了一些高级分析功能，如 API 调用图、网络通信分析等。
- **插件支持：**     可以扩展自定义插件来增加分析功能，灵活性较高。

**优点：**

- 丰富的静态分析功能，可以生成 API 调用图、代码结构等。
- 适合大规模的 APK 分析和自动化处理。
- Python 编写，易于集成到其他分析工具中。

**缺点：**

- 由于是 Python 实现，性能可能相对较差，分析大型 APK 时可能会比较慢。
- 需要较高的编程技能进行定制和扩展。

**适用场景：**

- **深度静态分析：**     用于 API 调用分析、控制流分析、漏洞挖掘等。
- **自动化分析：**     适合对大量 APK 文件进行批量分析和报告生成。

![](/img/2025-02-19-android-security/9.png)

### 2、动态调试

Frida：动态插桩

Xposed：

Android Native分析：

* 提取native文件
* 工具分析：分析ELF文件的结构，反汇编和分析函数逻辑
* 常见挑战：
  * 符号表丢失
  * 代码混淆和加密

IDA Pro，Ghidra，Radare2

Android Native动态分析：

动态调试工具，Hook关键函数，监控运行时的行为，数据捕获

行为分析：内存操作，缓冲区溢出，网络请求

Native代码由JNI反射调用Java API（混淆的一种）
