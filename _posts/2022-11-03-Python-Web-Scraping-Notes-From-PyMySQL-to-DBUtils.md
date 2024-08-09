---
layout: post
title: 【Python】爬虫笔记-从PyMySQL到DBUtils
date: 2022-11-03
author: "3thernet"
header-img: "img/bg-touhou-7.jpg"
tags: 

- Python

---

## 1. PyMySQL

### 1.1 基本使用

PyMySQL 是在 Python3.x 版本中用于连接 MySQL
服务器的一个库，Python2中则使用mysqldb。

PyMySQL 遵循 Python 数据库 API v2.0 规范，并包含了 pure-Python MySQL
客户端库。

PyMySQL 中有两个对象 **connection** 和 **cursor**。

```{.highlighter-hljs
import pymysql
conn = pymysql.connect(host, user, password, database, port)
cusor = conn.cursor()
```

connection 对象常用方法：

- cursor(Cursor=None)：创建一个执行查询的游标
- commit()：将更改提交到稳定存储（不能再回滚）
- rollback()：回滚当前事务
- select_db(db)：设置当前数据库
- open()：判断连接是否打开
- close()：发送退出消息并关闭套接字

cursor 对象常用方法：

- execute(query,
  args=None)：参数为查询字符串和查询参数（元组、列表或字典），执行查询
- executemany(query, args)
- fetchone()：提取**下一行**
- fetchmany(size=None)：根据参数提取若干行
- fetchall()：提取所有行
- mogrify(query, args=None)：接收和 execute
  一样的参数，返回将要发送到数据库的确切字符串
- callproc(procname, args=())：调用存储过程
- close()：关闭游标

使用方法很简单：

1. 打开数据库连接
2. 创建 cursor
3. 构建 sql 语句和字典参数
4. 执行 sql 语句
5. 提交修改/错误回滚
6. 关闭游标和连接

```{.highlighter-hljs
data = {
'id': '20120001',
'name': 'Bob',
'age': 20
}
table = 'students'
keys = ','.join(data.keys())
values = ','.join(['%s'] * len(data))
#使用占位符
sql = f"INSERT INTO {table}({keys}) VALUES({values})"
try:
    if cursor.execute(sql, tuple(data.values())):
        print('Successful')
        conn.commit()
except:
    print('Failed')
    conn.rollback()
```

在Python数据库编程中，当**游标建立**之时，就自动开始了一个**隐形的数据库事务**。

commit() 方法将更改提交到稳定存储，rollback()
方法回滚当前游标的所有操作。**每一个方法都开始了一个新的事务。**

我们注意到对事务的操作 ( commit, rollback ) 是在 connection
对象上，这意味着没有办法创建多个游标来管理多个事务，一个 connection
只能有一个 cursor。

更严谨的说，尽管可以创建多个 cursor 实例，但[**一个时间点只能有一个
cursor 实例存活**]{style="color: #3366ff;"}：

[Merge pull request #201 from stsci-sienkiew/master ·
pymssql/pymssql@af54cb3 ·
GitHub](https://github.com/pymssql/pymssql/commit/af54cb3ab1e149eea8407de979d79843119c3ca8)

### 1.2 线程安全

MySQL 事务本身具备
ACID。具体到并发读写时，需要考察其隔离性级别，MySQL默认是
REPEATABLE-READ。

*查看 MySQL 事务隔离级别：*[【MySQL】ERROR 1193 (HY000): Unknown system
variable \'tx_isolation\' - 江南笑书生 - 博客园
(cnblogs.com)](https://www.cnblogs.com/ME-WE/p/13431148.html#:~:text=mysql%3E%20select%20%40%40tx_isolation%3B%20ERROR%201193%20%28HY000%29%3A%20Unknown%20system,%E5%9C%A8%20MySQL%208%20%E5%8F%8A%E4%B9%8B%E5%90%8E%E7%9A%84%E7%89%88%E6%9C%AC%E4%B8%AD%EF%BC%8C%E5%8F%AA%E9%9C%80%E5%B0%86%E8%AF%AD%E5%8F%A5%E4%B8%AD%E7%9A%84%20tx_isolation%20%E6%9B%BF%E6%8D%A2%E4%B8%BA%20transaction_isolation%20%E5%8D%B3%E5%8F%AF%E3%80%82)

不过当我们使用 PyMySQL 时不用考虑这些，只需要知道 PyMySQL
线程不安全，无法通过多线程共享 cursor 或 connection。

[python - Is pymysql connection thread safe? Is pymysql cursor thread
safe? - Stack
Overflow](https://stackoverflow.com/questions/47163438/is-pymysql-connection-thread-safe-is-pymysql-cursor-thread-safe)

DB API 线程安全定义：（PyMySQL 为1）

> - 0: 不支持线程安全, 多个线程不能共享此模块
> - 1: 初级线程安全支持: 线程可以共享模块, 但不能共享连接
> - 2: 中级线程安全支持 线程可以共享模块和连接, 但不能共享游标
> - 3: 完全线程安全支持 线程可以共享模块, 连接及游标.

## 2. 数据库连接池

### 2.1 为什么需要数据库连接池？

使用数据库直接连接的情况：

1. 建立连接
2. 发送请求（CRUD）
3. 关闭连接

业务量流量不大，并发量也不大的情况下，连接临时建立完全可以。但并发量起来，达到百级、千级，其中建立连接、关闭连接的操作会造成性能瓶颈，所以得考虑连接池来优化上述
1 和 3 操作：

1. 取出连接（业务服务启动时，初始化若干个连接，放在连接存储中）
2. 发送请求（当有请求，从连接存储中中取出）
3. 放回连接（执行完毕，连接放回连接存储中）

直接连接的网络交互情况：

1. TCP建立连接的三次握手（客户端与MySQL服务器的连接基于TCP协议）
2. [MySQL认证的三次握手]{style="color: #ff6600;"}
3. [执行SQL]{style="color: #ff0000;"}
4. [MySQL的关闭]{style="color: #ff6600;"}
5. TCP的四次握手关闭

可见每次执行SQL都要与MySQL进行握手和关闭。

使用连接池时，只需要第一次访问建立MySQL连接，之后的访问均会**复用**之前创建的连接。

![](/img/2022-11-03-Python-Web-Scraping-Notes-From-PyMySQL-to-DBUtils/1.png)

 

数据库连接池技术的好处：

- **[资源复用]{style="color: #3366ff;"}**：由于数据库连接得到重用，避免了频繁创建、释放连接引起的大量性能开销。在减少系统消耗的基础上，另一方面也增进了系统运行环境的平稳性（减少内存碎片以及数据库临时进程/线程的数量）。
- **[更快的系统响应速度]{style="color: #339966;"}**：数据库连接池在初始化过程中，往往已经创建了若干数据库连接置于池中备用。此时连接的初始化工作均已完成。对于业务请求处理而言，直接利用了现有可用连接，避免了数据库连接初始化和释放过程的时间开销，从而缩减了系统整体响应时间。
- [**统一的连接管理，避免数据库连接泄露**]{style="color: #ff9900;"}：在较为完整的数据库连接池中，可根据预先的连接中超时设定，强制收回被占用连接。从而避免了常规数据库连接操作中可能出现的资源泄露。

数据库连接池还可以控制连接数量。使用一个等待连接的队列加不太大的连接池是比较好的方案。

其他：

- [数据库连接池终于搞对了，这次直接从100ms优化到3ms！ - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/181644853)
- [为什么数据库和数据库连接池不采用类似java
  nio的IO多路复用技术使用一个连接来维护和数据库的数据交换？ - 知乎
  (zhihu.com)](https://www.zhihu.com/question/23084473/answer/334920663)

### 2.2 DBUtils

> DBUtils是一套Python模块，允许在线程Python应用程序和数据库之间以安全有效的方式连接。
> 
> DBUtils最初是专门为Webware编写的，用于Python作为应用程序和PyGreSQL作为PostgreSQL数据库的适配器，但它同时可用于任何其他Python应用程序和DB-API
> 2兼容数据库适配器。

下载安装这里不再赘述，如果安装后 import
依旧报错，则可能是版本错误。（Python3.7只需要安装1.2版本的DBUtils即可）

官方文档：[DBUtils User\'s Guide
(webwareforpython.github.io)](https://webwareforpython.github.io/DBUtils/main.html)

DBUtils常用的两种外部接口： 

- PersistentDB ：提供线程**专用**的数据库连接，并自动管理连接。 
- PooledDB ：提供线程间**可共享**的数据库连接，并自动管理连接。 

**（1）PersistentDB**

对于PersistentDB，每当线程首次打开数据库连接时，都会打开一个新的数据库连接，此连接将从现在开始用于该特定线程。当线程关闭数据库连接时，它仍将保持打开状态，以便下次**[同一线程]{style="color: #ff6600;"}**请求连接时，可以使用该已打开的连接。线程死亡后，连接将自动关闭。

简而言之：PersistentDB会尝试回收数据库连接以提高数据库访问性能，且它能确保**连接永远不会在线程之间共享**（即使你的基础数据库驱动不是线程安全的，也比如pymysql，能确保不共享）。

![](/img/2022-11-03-Python-Web-Scraping-Notes-From-PyMySQL-to-DBUtils/2.png)

 

（2）PooledDB

如果将 maxshared 设为正值，且基础 DB-API 2
在连接级别是线程安全的，则默认会[**共享**]{style="color: #ff6600;"}打开的数据库连接。**PooledDB
最大的特点是能够设置 maxconnections 用于控制连接数量**，以及 mincached
和 maxcached
作为空闲连接的最少/初始数量和最大数量（维持空闲连接能使线程获取连接更快）。

![](/img/2022-11-03-Python-Web-Scraping-Notes-From-PyMySQL-to-DBUtils/3.png)

对于 PyMySQL 来讲，使用 PersistentDB 和 PooledDB
其实没什么区别。可以依靠线程池与 PersistentDB
配合从而重用专有连接。同时，尽管 PyMySQL 不是线程安全的，PooledDB
也会通过线程锁来保证线程安全。不过官方还是建议当更改数据库会话或执行分布在多个
SQL 命令上的事务时，应该小心地使用专用连接。

 简单使用：

```{.highlighter-hljs
import pymysql
from DBUtils.PooledDB import PooledDB
pool = PooledDB(creator=pymysql, maxconnections=10, host='127.0.0.1', user='xxx', passwd='xxx', port=3308, db='xxx')
conn = pool.connection()
cursor = conn.cursor()
sql = "select * from `user_info` limit 100"
cursor.execute(sql)
result = cursor.fetchall()
print(result) #三元组元组
cursor.close()
conn.close()
```

获取连接后使用方法与 PyMySQL 一致，实际上得到的是 DB-API 2 连接的强化
s[teady_db 版本。]{.docutils .literal}

[如果设置了非零 [maxshared 参数并且 DB-API 2
模块允许这样做，则默认情况下可能会与其他线程共享连接。指定获得专用连接：]{.docutils
.literal}]{.docutils .literal}

- [[db = pool.connection(shareable=False)]{.docutils
  .literal}]{.docutils .literal}
- [[db = pool.dedicated_connection()]{.docutils .literal}]{.docutils
  .literal}

另外不推荐这么写：pool.connection().cursor().execute(\...)，可能会过早释放连接。

## 3. 参考

- [Connection Object --- PyMySQL 0.7.2
  documentation](https://pymysql.readthedocs.io/en/latest/modules/connections.html)
- [PEP 249 -- Python Database API Specification v2.0 \|
  peps.python.org](https://peps.python.org/pep-0249/#rollback)
- [MySQL事务隔离级别详解（附带实例）
  (biancheng.net)](http://c.biancheng.net/view/7265.html)
- [为什么需要数据库连接池_攻城狮百里的博客-CSDN博客](https://blog.csdn.net/weixin_52622200/article/details/119334415)
- [Python操作Mysql及创建连接池 - 知乎
  (zhihu.com)](https://zhuanlan.zhihu.com/p/344642744)
- [Python3 MySQL 数据库连接 -- PyMySQL 驱动 \| 菜鸟教程
  (runoob.com)](https://www.runoob.com/python3/python3-mysql.html)
- [Python3中多线程操作MySQL插入数据的方法 - 开发技术 - 亿速云
  (yisu.com)](https://www.yisu.com/zixun/457255.html#:~:text=%E5%A4%9A%E7%BA%BF%E7%A8%8B%20%28%E8%BF%9E%E6%8E%A5%E6%B1%A0%29%E6%93%8D%E4%BD%9CMySQL%E6%8F%92%E5%85%A5%E6%95%B0%E6%8D%AE%201%20%E9%A6%96%E5%85%88%E6%98%AF%E5%8F%AF%E4%BB%A5%E6%9E%84%E5%BB%BA%E8%BF%9E%E6%8E%A5%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9A%84%) 可以用本案例进行测试

 
