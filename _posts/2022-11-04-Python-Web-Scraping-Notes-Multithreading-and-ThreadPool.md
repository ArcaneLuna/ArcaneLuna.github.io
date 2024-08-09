---
layout: post
title: 【Python】爬虫笔记-多线程&线程池
date: 2022-11-04
author: "qtunneling"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

## 1. 基本概念

### 1.1 并发和并行

并发和并行的概念并不是对立的，并发（concurrent）对应的是顺序（sequential），并行（parallel）对应的是串行（serial）。

- 顺序：上一个开始执行的任务完成后，当前任务才能开始执行
- 并发：无论上一个开始执行的任务是否完成，当前任务都可以开始执行
- 串行：有一个任务执行单元，从物理上就只能一个任务、一个任务地执行
- 并行：有多个任务执行单元，从物理上就可以多个任务一起执行

顺序/并发关注的是任务的调度，而串行/并行更关注执行单元（单核/多核CPU）。

单核多线程就属于串行（只有一个实际执行任务的 CPU
核）并发（不必等上一个任务完成才开始下一个任务）。

**也可以将并行理解为并发的[子集]{style="color: #3366ff;"}**：如果某个系统支持两个或者多个动作（Action）**同时存在**，那么这个系统就是一个并发系统。如果某个系统支持两个或者多个动作**同时执行**，那么这个系统就是一个并行系统。并发系统与并行系统这两个定义之间的关键差异在于"存在"这个词。

Erlang 之父 Joe Armstrong 的图很好地展示了并发和并行的区别：

![](/img/2022-11-04-Python-Web-Scraping-Notes-Multithreading-and-ThreadPool/1.jpg)

### 1.2 进程和线程

**进程**：进程是资源分配的最小单位，拥有独立的地址空间。

**线程**：线程是CPU调度的最小单位，共享同一进程的地址空间。每个进程中可以执行多个任务，每个任务就是线程，线程可以说是程序用CPU的一个基本单元，如果程序中是单一的执行路径，那么就是单线程的，如果有多个执行路径，那么就是多线程的。对于python来说，存在GIL
全局解释器锁，它的存在使得一个CPU同一个时刻只能执行一个线程，python的多线程就更像是伪多线程，一个线程运行时，其他，多线程代码不是同时执行，而是交替执行。而不同线程之间的切换需要耗费资源的，因为需要存储线程的上下文，不断的切换就会持续耗费资源。同时，多线程也不能够有效的利用CPU的多核性能，提升程序的运行速度。

**协程**：独立的栈空间，共享堆空间，调度由用户自己控制，本质上有点类似于用户级线程，这些用户级线程的调度也是自己实现的。一个线程上可以跑多个协程，协程是轻量级的线程。

## 2. 多线程

python常用多线程模块：[\_thread]{style="text-decoration: line-through;"},
threading, Queue

threading支持_thread中所有功能，Queue用于提供线程安全的队列类

### 2.1 threading.Thread类

两种使用方法：

（1）使用Thread实例化一个线程对象，传入想要执行的函数

thread1 = Thread(target=print, args=(1,))

（2）继承Thread类，重写run方法

初始化：

```{.highlighter-hljs
Thread(
    group=None,      
    target=None,      #run方法使用的可调用对象（函数）
    name=None,        #线程的名称
    args=(),          #传入的函数的参数组成的列表或元组
    kwargs=None,      #传入的函数的关键字参数
    *,
    daemon=None)      #守护线程
```

常用方法：

- [start()：]{style="color: #ff0000;"}启动线程
- [run()：]{style="color: #ff0000;"}线程执行的代码，一般重写此方法
- join(timeout=None)：[阻塞]{style="color: #ff0000;"}调用线程（一般是主进程），等到被调用线程终止（正常或者未处理的异常）或直到超时。
- getName()/setName()：获取/设置线程名字，也可直接使用name属性
- isDaemon()/setDaemon()：判断和设置线程为守护线程，也可直接使用daemon属性

### 2.2  守护进程

Daemon
Thread，也叫后台进程，它的目的是为其他线程提供服务，**其存在依赖于其他线程，如果其他线程消亡，它会随之消亡。**

```{.highlighter-hljs
import time
import threading

class MyThread(threading.Thread):
    def __init__(self, n):
        super().__init__()
        self.n = n

    def run(self) -> None:
        while True:
            _count = threading.active_count()
            print(self.n, f"当前活跃的线程个数：{_count}")
            time.sleep(self.n)

for i in range(1, 4):
    t = MyThread(i)
    t.setDaemon(False)
    t.start()
```

本例中创建了3个线程，如果setDaemon为False，当主线程终止后他们依然会在while循环中运行，如果为True，当主线程终止后它们也会终止。

### 2.3 阻塞进程

- [Python Thread join()用法详解
  (biancheng.net)](http://c.biancheng.net/view/2609.html)

join方法会使调用进程被阻塞，等待被调用join方法的子进程运行结束

### 2.4 线程安全

为保证数据安全，需要使用线程锁来保护数据。

当两个线程同时对一全局变量进行累加，不加锁就会导致资源抢夺，从而输出不确定的值。每次上锁后都要记得及时释放（release），也可以使用**with**模块。

**（1）Lock()：原始锁**

在Python中，它是能用的**最低级的同步基元组件**，由 \_thread
扩展模块直接实现。一个线程获得一个锁后，会阻塞随后尝试获得锁的线程，直到它被释放；[**任何线程都可以释放它**]{style="color: #ff6600;"}。

threading.Lock() 返回的原始锁有 acquire() 和 release()
两个方法，对象本身有"锁定"和"非锁定"两种状态。当状态为非锁定时调用
acquire() 方法会将状态改为锁定，并立即返回，当状态为锁定时调用 acquire()
方法会阻塞线程，直到其他线程调用了 release()
方法将锁状态改为非锁定，然后 acquire() 重置其为锁定状态并返回。release()
只在锁定状态下调用；它将状态改为非锁定并立即返回。如果尝试释放一个非锁定的锁，则会引发
RuntimeError  异常。

示例：

```{.highlighter-hljs
import time
import threading

number = 0
lock = threading.Lock() #实例化一个锁

class MyThread(threading.Thread):
    def __init__(self, n, k):
        super().__init__()
        self.n = n
        self.k = k

    def run(self) -> None:
        global number
        for i in range(self.k):
            #lock.acquire()
            number += 1
            #lock.release()

t1 = MyThread(1, 1000000)
t2 = MyThread(2, 1000000)
t1.start()
t2.start()
t1.join()
t2.join()

print("main", number)
```

acquire() 方法有两个缺省参数，一般来讲我们不传入任何参数就相当于
acquire(blocking=True,
timeout=-1)，当无法获得锁时，这会无限期地阻塞线程，直到获得锁，然后返回
True。设置 blocking 为 False
表示以不阻塞线程的方式尝试获得锁，并立即返回 True 或者 False。当
blocking 为 True时还可设置 timeout 表示最多阻塞线程的时间。

**（2）RLock()：重入锁/递归锁。**

用法与原始锁基本相同，相比原始锁加入了"所属线程"和"递归等级"的概念。特点：①**[重入锁必须由获取它的线程释放]{style="color: #ff6600;"}**；②一旦线程获得了重入锁，同一个线程再次获取它将不阻塞（**原始锁会阻塞自己**）；③线程必须在每次获取它时释放一次。

### 2.5 条件对象

当线程在系统中运行时，线程的调度具有一定的透明性，通常程序无法准确控制线程的**轮换执行**，如果有需要，Python
可通过**线程通信**来保证线程协调运行。使用 threading.Condition
可以让那些己经得到 Lock 对象却无法继续执行的线程释放 Lock
对象，Condition 对象也可以**唤醒**其他处于等待状态的线程。

例子：两个线程模拟的智能机器人互相对话

> 天猫精灵：小爱同学
> 
> 小爱：在
> 
> 天猫精灵：我们来对古诗吧
> 
> 小爱：好啊
> 
> 天猫精灵：我住长江头
> 
> 小爱：不聊了，再见

这是一个同步问题（线程同步：**[多个线程协按照一定的顺序协同完成某一任务]{style="color: #3366ff;"}**），用原始锁和信号量就能实现，大致如下：

```{.highlighter-hljs
class TianMao(threading.Thread):
    --snip--
    def run(self):
        self.lock_A.acquire()
        self.lock_B.acquire()
        print("tianmao:小爱同学")
        self.lock_A.release()
        self.lock_B.acquire()
        print("tianmao:我们来对古诗吧")
        self.lock_A.release()
        self.lock_B.acquire()
        print("tianmao:我住长江头")
        self.lock_A.release()
        self.lock_B.release()

class XiaoAi(threading.Thread):
    --snip--
    def run(self):
        self.lock_A.acquire()
        print("xiaoai:在")
        self.lock_B.release()
        self.lock_A.acquire()
        print("xiaoai:好啊")
        self.lock_B.release()
        self.lock_A.acquire()
        print("xiaoai:不聊了，再见")
        self.lock_A.release()
```

Condition 对象的逻辑与之相似，同样也用到了两个锁对象：（[\[python\]
线程间同步之条件变量Condition - 简书
(jianshu.com)](https://www.jianshu.com/p/5d2579938517)）

> Codition有两层锁，一把底层锁会在进入wait方法的时候释放，离开wait方法的时候再次获取，上层锁会在每次调用wait时分配一个新的锁，并放入condition的等待队列中，而notify负责释放这个锁。

acquire() 和 release() 都是 RLock
的方法，用来对底层锁的获得和释放，wait()
方法会先**释放底层锁**，然后阻塞调用进程，等待notify**唤醒**，再尝试**获得底层锁**；notify(n=1)由获得锁的线程调用，会唤醒等待队列中的n个线程，但不会释放底层锁，这之后必须由线程主动释放锁。

虽然都能实现线程轮换的效果，但 Condition 对象的代码显然更容易理解：

    class TianMao(threading.Thread):
        --snip--
        def run(self):
            self.con_lock.acquire()
            print("tianmao:小爱同学")
            self.con_lock.wait()
            print("tianmao:我们来对古诗吧")
            self.con_lock.notify()
            self.con_lock.wait()
            print("tianmao:我住长江头")
            self.con_lock.notify()
            self.con_lock.release()
    
    class XiaoAi(threading.Thread):
        --snip--
        def run(self):
            self.con_lock.acquire()
            print("xiaoai:在")
            self.con_lock.notify()
            self.con_lock.wait()
            print("xiaoai:好啊")
            self.con_lock.notify()
            self.con_lock.wait()
            print("xiaoai:不聊了，再见")
            self.con_lock.release()

条件对象时常配合循环的条件检查使用，参考 [Python
文档](https://docs.python.org/zh-cn/3/library/threading.html#condition-objects) 提供的生产消费者案例：

```{.highlighter-hljs
# Consume one item
with cv:
    while not an_item_is_available():
        cv.wait()
    get_an_available_item()

# Produce one item
with cv:
    make_an_item_available()
    cv.notify()
```

条件检查也可使用 wait_for(predicate, timeout=None)
方法自动化实现，predicate 是一个返回布尔值的可调用对象。

### 2.6 信号量

Python 通过 threading.Semaphore(value=1)
来创建信号量，value赋予内部计数器初始值，小于零会引发 ValueError
异常。acquire()方法相当于P操作，如果内部计数器大于零，则减一并立即返回，如果内部计数器等于零，阻塞线程，直到其他线程调用release()方法使内部计数器大于零，然后再减一返回（也可以像锁变量那样设置blocking和timeout）；release(n=1)方法相当于V操作，将内部计数器加n。为了防止信号量被过多释放，可使用
**threading.BoundedSemaphore(value=1) **创建有界信号量，使得其内部计数器值不超过初始值。

信号量通常用于保护数量有限的资源，例如数据库服务器。（连接池）

### 2.7 事件对象

> 这是线程之间通信的最简单机制之一：一个线程发出事件信号，而其他线程等待该信号。

（个人理解）尽管 Condition
对象能在一定程度上实现线程通信，但主要目的是为了同步调度。这中间总是涉及到对锁的争抢，如果单纯想让一个线程控制其他线程的执行，可以使用
Event 类，它的概念十分简单。

- threading.Event 创建一个事件管理flag，其取值为False（默认）或True；
- set() 和 clear() 方法用于将 flag 变为 True 或 False；
- 当 flag 为 False 时，wait(timeout=None) 方法会阻塞线程直到 flag 变为
  True；
- is_set() 方法可获取当前flag标志值（从而让线程执行不同的操作）。

### 2.8 定时器

- [python---threading.Timer【threading模块介绍03】\_cxc_17的博客-CSDN博客](https://blog.csdn.net/a349458532/article/details/51586971)

Timer是Thread的子类，实例化一个Timer相当于创建了一个线程。

> class threading.Timer(interval, function, args=\[\], kwargs={})

创建后也需要 start 方法启动，然后会在interval秒过去后，以参数 args
和关键字参数 kwargs 运行function。

如果把 function 设置为另一个线程的 start
方法，则会延迟启动该线程；在执行function前可以调用cancel方法取消线程执行

### 2.9 栅栏对象

（略）

threading.Barrier
栅栏类提供一个简单的同步原语，用于应对固定数量的线程需要彼此相互等待的情况。线程调用
wait() 方法后将阻塞，直到所有线程都调用了 wait()
方法。此时所有线程将被同时释放。

## 3 线程池

### 3.1 ThreadPoolExecutor

线程池可以重复利用线程，不会造成过多的浪费。从Python3.2开始，标准库为我们提供了
concurrent.futures 模块，它提供了 [ThreadPoolExecutor
(线程池)]{style="color: #ff6600;"}和ProcessPoolExecutor
(进程池)两个类。（两者都是 Executor 的子类）

> **ThreadPoolExecutor**(*max_workers=None*, *thread_name_prefix=\'\'*, *initializer=None*, *initargs=()*)

ThreadPoolExecutor类使用最多 [max_workers]{style="color: #3366ff;"}
个线程的[线程池]{style="color: #3366ff;"}来异步执行调用。Executor类向上提供了三个方法：

[[①
**submit**[(*[[fn]{.pre}]{.n}*, *[[/]{.pre}]{.o}*, *[[\*[[args]{.pre}]{.n}]{.pre}]{.o}*, *[[\*\*[[kwargs]{.pre}]{.n}]{.pre}]{.o}*[)]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}

[[[[通过 submit
方法提交可调用对象，然后得到一个 Future 对象，通过它可以获悉线程的状态。]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}

[[[②
**map**[(*[[func]{.pre}]{.n}*, *[[\*[[iterables]{.pre}]{.n}]{.pre}]{.o}*, *[[timeout[[=[[None]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*, *[[chunksize[[=[[1]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*[)]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.sig-name .descname}

[[[[[map 方法类似 Python
内置的map函数，适合对同一函数多次异步调用。]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.sig-name .descname}

[[[[[[[③
**shutdown**[(*[[wait[[=[[True]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*, *[[\*]{.pre}]{.o}*, *[[cancel_futures[[=[[False]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*[)]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.sig-name .descname}

**关闭线程池（不影响正在执行的线程）**。关闭后再调用submit和map会引发RuntimeError。

wait为True，则本方法会等待（阻塞）直到所有待执行的future执行且释放资源才返回；wait
为
False，方法立即返回。（不论wait值是什么，**整个python程序**将等到所有待执行的
future 对象完成执行后才退出）

（3.9版本增加）如果 *cancel_futures* 为 True，此方法将**取消**所有执行器还**未开始运行**的挂起的
Future。 任何已完成或正在运行的 Future
将不会被取消，无论 *cancel_futures* 的值是什么。

[使用 with
语句创建ThreadPoolExecutor类能够确保及时关闭线程池。(效果等同于
Executor.shutdown(wait=True))]{style="color: #ff6600;"}

### 3.2 Future对象

- cancel()：尝试取消调用。如果调用正在执行或已结束运行不能被取消则该方法将返回
  False，否则调用会被取消并且该方法将返回 True
- cancelled()：如果调用成功取消返回True
- [done()]{style="color: #ff6600;"}：判断任务是否被取消或正常结束，是则返回True
- [result(timeout=None)]{style="color: #ff6600;"}：**阻塞**线程直到获取任务返回值，类似join
- exception(timeout=None)：阻塞线程直到返回调用产生的异常，设置timeout会引发TimeoutError，如果future在完成前被取消则返回CancelledError，如果调用正常完成，则返回None
- add_done_callback(fn)：附加可调用fn到future对象。**当 future
  对象被取消或完成运行时，将会调用 fn，而这个 future
  对象将作为它唯一的参数**。加入的可调用对象总被属于添加它们的进程中的线程按加入的顺序调用。如果可调用对象引发一个
  Exception 子类，它会被记录下来并被忽略掉。如果可调用对象引发一个
  BaseException 子类，这个行为没有定义。如果 future
  对象已经完成或已取消，fn 会被立即调用。

**※如果程序不希望直接调用 result() 方法阻塞线程，则可通过 Future 的
add_done_callback() 方法来添加回调函数**

```{.highlighter-hljs
def get_result(future):
    print(future.result())
future1.add_done_callback(get_result)
future2.add_done_callback(get_result)
```

### 3.3 模块函数

concurrent.futures 模块有两个有用的函数：

① [[concurrent.futures.[[**wait**[(*[[fs]{.pre}]{.n}*, *[[timeout[[=[[None]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*, *[[return_when[[=[[ALL_COMPLETED]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*[)]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename .descclassname}

[[[[[[**[等待（阻塞）]{style="color: #ff6600;"}**由 fs 指定的 Future
实例（可能由不同的 Executor 实例创建）完成。[【一般可以传入一个由 Future
实例组成的列表或者元组】]{style="color: #0000ff;"}]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename .descclassname}

[[[[[[重复传给 fs 的 future 会被移除并将只返回一次。
返回一个由集合组成的具名 2 元组。 第一个集合的名称为
done，包含在等待完成之前已完成的 future（包括正常结束或被取消的
future）。 第二个集合的名称为 not_done，包含未完成的
future（包括挂起的或正在运行的
future）。]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename .descclassname}

> [[[[[[(done=set(),
> not_done=set())]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
> .descname}]{.pre}]{.sig-prename .descclassname}

[[[[[[如果不设置
timeout，则默认会等待到全部任务完成，not_done为空集。]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename .descclassname}

return_when 指定此函数应在何时返回。可以取值：

- FITST_COMPLETED   #在任意可等待对象结束或取消时返回。
- FIRST_EXCEPTION   
  #在任意可等待对象因引发异常而结束时返回。当没有引发任何异常时它就相当于
  ALL_COMPLETED。
- ALL_COMPLETED      #在所有可等待对象结束或取消时返回。

[[[[[[② [[concurrent.futures.[[**as_completed**[(*[[fs]{.pre}]{.n}*, *[[timeout[[=[[None]{.pre}]{.default_value}]{.pre}]{.o}]{.pre}]{.n}*[)]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename
.descclassname}]{.sig-paren}]{.sig-paren}]{.pre}]{.sig-name
.descname}]{.pre}]{.sig-prename .descclassname}

在fs给定的
Future实例（可能由不同的Executor实例创建）上返回一个[**迭代器**]{style="color: #ff6600;"}，该迭代器在
Future
完成时（结束或取消）生成。fs提供的任何重复Future都将只返回一次。在调用as_completed()
之前完成的 Future 会首先生成。如果调用\_\_next\_\_()，并且在
as_completed() 调用后的的 timeout
秒后结果不可用，则返回的迭代器将引发**TimeoutError**。timeout
可以是int或float。如果未指定超时或无，则等待时间没有限制。

用法和迭代器一样，Python 官方给的例子：

```{.highlighter-hljs
import concurrent.futures
import urllib.request

URLS = ['http://www.foxnews.com/',
        'http://www.cnn.com/',
        'http://europe.wsj.com/',
        'http://www.bbc.co.uk/',
        'http://some-made-up-domain.com/']

# Retrieve a single page and report the URL and contents
def load_url(url, timeout):
    with urllib.request.urlopen(url, timeout=timeout) as conn:
        return conn.read()

# We can use a with statement to ensure threads are cleaned up promptly
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    # Start the load operations and mark each future with its URL
    future_to_url = {executor.submit(load_url, url, 60): url for url in URLS}
    for future in concurrent.futures.as_completed(future_to_url):
        url = future_to_url[future]
        try:
            data = future.result()
        except Exception as exc:
            print('%r generated an exception: %s' % (url, exc))
        else:
            print('%r page is %d bytes' % (url, len(data)))
```

future_to_url是一个future对象到url字符串的字典，其中有五对键值，其中作为键的future的状态是动态变化的。可以看出 as_completed(fs)作为一个生成器，没有任务完成时会一直**[阻塞]{style="color: #ff6600;"}**（除非设置timeout）；有任务完成时，会
**[yield]{style="color: #ff6600;"}**
这个任务，就能执行for循环下面的语句，然后继续阻塞住，**循环直到所有任务结束**。先完成的任务会先返回给主线程。

【future对象有几个状态：**Pending、Running、Finished、Cancelled**。创建future的时候，状态为pending，执行的时候是running，执行完毕就是finished】

## 4 Queue

Python 的 queue
模块从设计目的上就是为了提供安全的**多线程之间的信息交换**，其内部使用锁来临时阻塞竞争线程。

模块实现了 FIFO、LIFO、优先队列三种类型的队列，以及一个"简单的"无界 FIFO
队列。

在典型的"生产者消费者"问题中，使用队列能起到[**缓冲区**]{style="color: #3366ff;"}的作用。

![](/img/2022-11-04-Python-Web-Scraping-Notes-Multithreading-and-ThreadPool/2.jpg){loading="lazy"}

 

 

 

当使用多线程爬取网站时，可以通过 queue.Queue
模块维护一个存放URL的FIFO队列，相较于使用列表，有两个优点：

- 无需维护一个 tag 用于标记提供给线程的URL元素，并且队列的 get
  方法能自动移除队列头部的元素，使得不占据过多空间
- 当使用多线程一边添加一边取出队列元素时，是线程安全的，否则需要考虑同步问题

常用方法：

- qsize()：返回队列大小
- empty()/full()：判断队列是否为空/满
- **put(*item, block=True,
  timeout=None*)**：将项目插入队列，如果队列已满，根据block参数决定是否阻塞。默认阻塞至有空闲插槽
- put_nowait(*item*)：相当于put(*item, block=False*)
- **get(*block=True, timeout=None*)**：从队列中移除并返回一个项目\...
- get_nowait()：相当于get(*False*)
- task_done()：每调用一个get()，后续调用的 task_done()
  就告诉队列一个任务已完成。当 task_done() 调用次数等于 put()
  次数时，将解除阻塞，如果调用的 task_done() 次数超过 put()
  次数，会引发 ValueError
- join()：阻塞至队列中**[所有的元素]{style="color: #ff0000;"}**都被接收和处理完毕。**[当条目添加
  (*put*)
  到队列的时候，未完成任务的计数就会增加。]{style="color: #ff6600;"}**每当消费者线程调用
  **[task_done()]{style="color: #3366ff;"}**
  表示这个条目已经被回收，该条目所有工作已经完成，**[未完成计数就会减少]{style="color: #3366ff;"}**。当未完成计数降到零的时候，
  join() 阻塞被解除。

```{.highlighter-hljs
import threading
import queue

q = queue.Queue()

def worker():
    while True:
        item = q.get()
        print(f'Working on {item}')
        print(f'Finished {item}')
        q.task_done()

# Turn-on the worker thread.
threading.Thread(target=worker, daemon=True).start()

# Send thirty task requests to the worker.
for item in range(30):
    q.put(item)

# Block until all tasks are done.
q.join()
print('All work completed')
```

## 5 参考

- [并发执行 --- Python 3.11.1
  文档](https://docs.python.org/zh-cn/3/library/concurrency.html)
- [并发与并行的区别是什么？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/33515481)
- [Python Thread join()用法详解
  (biancheng.net)](http://c.biancheng.net/view/2609.html)
- [\[python\] 线程间同步之条件变量Condition - 简书
  (jianshu.com)](https://www.jianshu.com/p/5d2579938517)
- [python---threading.Timer【threading模块介绍03】\_cxc_17的博客-CSDN博客](https://blog.csdn.net/a349458532/article/details/51586971)
- [Python爬虫小白教程（五）------
  多线程爬虫_YonminMa的博客-CSDN博客_多线程爬虫](https://blog.csdn.net/weixin_44547562/article/details/103955734)
- [二十三、Python队列实现多线程（下篇） - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/268548189)
- [【python】多线程：锁 、全局锁、Queue队列以及线程池 - 简书
  (jianshu.com)](https://www.jianshu.com/p/04af68ab2c9b)
- [Python线程池及其原理和使用（超级详细）
  (biancheng.net)](http://c.biancheng.net/view/2627.html#:~:text=%E5%9C%A8%E7%94%A8%E5%AE%8C%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%90%8E%EF%BC%8C%E5%BA%94%E8%AF%A5%E8%B0%83%E7%94%A8%E8%AF%A5%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%20shutdown%20%28%29%20%E6%96%B9%E6%B3%95%EF%BC%8C%E8%AF%A5%E6%96%B9%E6%B3%9)
- [python线程池 ThreadPoolExecutor 的用法及实战 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/65638744)
- [python线程池 ThreadPoolExecutor 的用法及实战 - 简书
  (jianshu.com)](https://www.jianshu.com/p/6d6e4f745c27)
- [【python】协程之asyncio：调用步骤、阻塞和await、task任务、future对象 -
  简书 (jianshu.com)](https://www.jianshu.com/p/6aae8aa2ebcd)

 

 
