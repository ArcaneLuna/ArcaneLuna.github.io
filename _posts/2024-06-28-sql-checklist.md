---
layout: post
title: "SQL checklist"
subtitle: "CRUD自查"
date: 2024-06-28
author: "3thernet"
header-img: "img/bg-touhou-1.jpg"
tags: []
---

SQL和bash一样不经常写的话很容易忘，之前选过数据库的课做了不少笔记，但量太大，因此整理压缩一下自己过去所学，方便查阅。

## 0x01 基础使用

### 1. 准备

```powershell
net start mysql // 启动 mysql 服务器
net stop mysql
mysql -u root -p
exit/quit
```

### 2. 常用CRUD

```sql
select version();
show engines;
alter table t type=innodb;
-- 修改表引擎
-- 设置InnoDB为默认引擎：在配置文件my.ini中的 [mysqld] 下面加入default-storage-engine=INNODB
show databases;
create database if not exists db;
use db;
select database();
drop database if exists db;
show tables;
show table status;
-- 查看所有表的信息及结构
show table status like 'table_name';
-- 查看特定表的信息及结构
desc t;
-- 查看表结构
create table t(
    id INT,
    name VARCHAR(50)
)engine=innodb;
create table t1(col1, col2) as select col1, col2 from t2;
alter table t1 rename to/as t2;
rename table t1 to t2;
-- 重命名表，如果要重命名数据库，则需要先创建新的数据库，然后改名原数据库下的所有表
alter table t add id int;
-- 添加列
alter table t modify [column] id double;
-- 修改列数据类型
alter table t change id n_id INT;
-- 修改列名和数据类型
alter table t drop id;
-- 删除列
alter table t drop primary key;
alter table t add primary key(sno);
-- 删除原主键（保留字段）并设置设置新主键
create index idx_name on t(colname);
alter table t add idx_name (col_name);
-- 为某个列添加索引
alter table t drop index idx_name;
-- 删除索引
show index from t;
-- 查看表中的索引信息
show create table t;
show create procedure p;
select * from t;
-- select子句目标列可以为列名, *, 算数表达式, 聚集函数
select found_rows();
-- 获得select语句影响的行数
insert into t(id, name) values(1, 'wang'),(2, 'li');
replace into t(id, name) values(1, 'zhang');
update t set id = 3 where id = 1;
select row_count();
-- 获得update或delete影响的行数
delete from t where id = 2;
-- 删除数据
truncate table t;
-- 清空表，保留结构
drop table t;
-- 删除表
```

grade between 60 and 80: 等价于`grade >= 60 and grade <= 80`

**distinct**: 去重（SQL缺省使用all关键词保留重复行）

**order by** 列名 [asc | desc]: 输出显示顺序，默认升序

old_name [as] new_name: 关系和属性别名，as可选

`select sno as id from sc;`

子查询别名：from子句必须必须有别名，where子句可选

#### 2.1 随机插入数据

```sql
CREATE TABLE testIndex (
    id INT AUTO_INCREMENT PRIMARY KEY,
    A INT,
    B INT,
    C VARCHAR(255)
);
INSERT INTO testIndex(A, B, C)
SELECT FLOOR(RAND() * 1000), FLOOR(RAND() * 1000),
    CONCAT(CHAR(FLOOR(RAND() * 26) + 65),
        CHAR(FLOOR(RAND() * 26) + 65),
        CHAR()
FROM information_schema.tables t1, information_schema.tables t2
LIMIT 10000;
-- RAND返回[0,1)的浮点数
-- 连接操作防止数据不够
CREATE TABLE SC (
    SNO INT NOT NULL,
    CNO INT NOT NULL,
    PRIMARY KEY (`SNO`, `CNO`)
);
DELIMITER //
CREATE PROCEDURE insertSC()
BEGIN
    DECLARE i INT DEFAULT 1;
    DECLARE j INT DEFAULT 1;
    DECLARE k INT DEFAULT 1;
    WHILE k <= 1000000 DO
        SET i = 1 + FLOOR(RAND() * 10000);
        -- Assuming we have 10000 different students
        SET j = 1 + FLOOR(RAND() * 1000);
        -- Assuming we have 1000 courses
        SET k = k + 1;
    END WHILE;
END//
```

关于information_shcema.tables: [【整理】mysql中information_schema.tables字段说明-CSDN博客](https://blog.csdn.net/boshuzhang/article/details/65632708)

#### 2.2 临时表&内存表

mysql8.0 以前不支持 `WITH AS`定义SQL片段，因此可能会使用大量临时表来保证可读性

临时表仅在当前会话可见，并且在会话结束时自动删除，定义时加入temporary关键字即可

内存表存储在内存中，因此数据的修改会立即生效，并且对所有用户可见。但是，当MySQL服务器关闭时，内存表中的数据将丢失。因此，它适用于临时存储数据或缓存等场景，`engine=memory`

#### 2.3 常用函数

```sql
-- 字符串函数
CONCAT(string1, string2, string3);
LENGTH();
SUBSTRING(string, position, length);
-- 数学函数
ABS(); -- 绝对值 
CEILING(); -- 向上取整
FLOOR(); -- 向下取整
ROUND(); -- 四舍五入
RAND() -- [0,1)随机数
-- 日期函数
CURDATE(), CURRENT_DATE(): -- 当前日期
CURTIME(), CURRENT_TIME(); -- 当前时间
NOW(), CURRENT_TIMESTAMP(); -- 返回当前日期和时间
DATEDIFF('2019-12-31', '2010-01-01'); -- 返回两个日期之间的天数
DATE_ADD(date, INTERVAL expr type), DATE_SUB()
SELECT DATE_ADD(CURDATE(), INTERVAL 10 DAY);
-- 聚合函数
AVG();
COUNT();
SUM();
MAX(), MIN();
-- 条件表达式
IF(expr, value_if_true, value_if_false)
SELECT name, salary, IF(salary > 3000, 'hith', 'low') AS salary_level FROM employees;
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ..
    ELSE resultN
END
COALESCE(value1, value2, ..., valueN) -- 返回第一个非空值
-- 类型转换函数
CAST('123' AS SIGNED) -- 将字符串转化为整数
CONVERT(expr, type)
CONVERT(expression USING charset) -- 可用于字符集转换
```

#### 2.4 导入（导出）数据

当前查看文件格式：

```bash
-- linux
less filename.txt
more filename.txt
head -n 10 filename.txt
tail -n 10 filename.txt
-- windows
Get-Content filename.txt -TotalCount 10
```

（1）直接导入

load data使用方法：[LOAD DATA INFILE使用与详解-CSDN博客](https://blog.csdn.net/longzhoufeng/article/details/112377942)

修改`secure_file_priv =`表示允许所有路径

[Mysql 导入文件提示 --secure-file-priv option 问题 - kaizenly - 博客园(cnblogs.com)](https://www.cnblogs.com/Braveliu/p/10728162.html)

```sql
-- 导入数据
LOAD DATA LOCAL INFILE 'path/to/your/file.txt'
[replace | ignore] -- 表示对主键的处理
-- 注意windows双反斜杠
INTO TABLE twitter
FIELDS TERMINATED BY ' - '
LINES TERMINATED BY '\n'
IGNORE 1 LINES # 忽略CSV首行标题
(Email, Name, ScreenName, Followers, CreateAt);

-- 导出数据
SELECT *
INTO OUTFILE '/path/to/your/output.csv'
CHARACTER SET 'utf8'
FIELDS TERMINATED BY ','
ENCLOSED BY '"' -- 顺序不能错
LINES TERMINATED BY '\n'
FROM your_table_name;
-- ENCLOSED BY '"'：指定所有字段都被双引号包围，这对于包含分隔符或换行符的文本字段很有用。
```

[load data local infile 与 load data infile 的区别与注意事项_ygc2022的博客-CSDN博客](https://blog.csdn.net/youngerchen/article/details/7881678)

如果你没有给出local,则服务器按如下方法对其进行定位:

1)如果你的filename为绝对路径,则服务器从根目录开始查找该文件.

2)如果你的filename为相对路径,则服务器从数据库的数据目录中开始查找该文件.

如果你给出了local,则文件将按以下方式进行定位:

1)如果你的filename为绝对路径,则客户机从根目录开始查找该文件.

2)如果你的filename为相对路径,则客户机从当前目录开始查找该文件.

[【MySQL笔记】ERROR 1148(42000): The used command is not allowed with this MySQL version_AXIMI的博客-CSDN博客](https://blog.csdn.net/AXIMI/article/details/89054799)

需要开启local-infile，但在终端修改后，每次重启又会off，需要修改my.ini

[MySQL文件导入时的若干问题 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/29287995)

字符串含有转义字符，加入`escaped by ''`

导入字符串含有特殊字符时，会遇到这样一条错误：

`ERROR 1300 (HY000): Invalid utf8 character string`

这时因为MySQL中的utf8最多只用三个字节，有一些使用四个字节表示的特殊字符（emoji）无法导入

更改表和数据库字符集：

```sql
alter database dbname character set = utf8mb4;
alter table tbname character set = utf8mb4;
alter table tbname modify colname char(...) character set utf8mb4;
```

然后load时加入语句：SET CHARACTER utf8mb4

#### 2.5 lost connection错误

[解决Lost connection to MySQL server during query错误方法 - 简书 (jianshu.com)](https://www.jianshu.com/p/98c7a63b84c3)

```sql
show global variables like '%allowed_packet';
set global max_allowed_packet=1024*1024*16;
show global variables like 'wait_timeout';
set global wait_timeout = 28800;
```

#### 2.6 动态SQL

甚至可以将列名、表名作为参数传递给存储过程

[Passing column name as a parameter in mysql stored function - Stack Overflow](https://stackoverflow.com/questions/24487371/passing-column-name-as-a-parameter-in-mysql-stored-function)

```sql
SET @sql_stmt = CONCAT('SELECT ', col_name, 'FROM t');
PREPARE stmt FROM @sql_stmt;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
-- 释放资源
PREPARE stmt FROM 'INSERT INTO employees(name, age) VALUES (?, ?)';
SET @name = 'John';
SET @age = 25;
EXECUTE stmt USING @name, @age;
DEALLOCATE PREPARE stmt;
```

### 3. 控制命令

```sql
create user 'test'@'localhost' identified by 'test';
-- 创建用户
alter user 'root'@'localhost' identified by 'newpassword';
set password for 'test'@'localhost' = password('test');
-- 修改密码
set password for 'test'@'localhost' = set password by password('newpassword');
drop user 'test'@'localhost';
grant all privileges on *.* to 'test'@'localhost';
-- 授予test用户对所有数据库的完全访问权限
grant select, insert, update on sql_study.* to 'test'@'localhost';
-- 部分权限
grant all privileges on *.* to 'test'@'localhost' with grant option;
-- 该用户可以进一步分配权限 
revoke all privileges on *.* from 'test'@'localhost';
-- 撤销test用户对所有数据库的访问权限
show grants for 'test'@'localhost';
-- 查看用户权限
flush privileges;
-- 刷新权限
```

### 4. NULL

where null 或者 where not null 都不执行

逻辑计算：

- false=0, unknown=1, true=2

- and=min, or=max

is [not] null: 用来测试指定列的值是否为空值，唯一的空值查找条件

MySQL空值处理函数：

- isnull(expr)

- ifnull(check_expr, replace_value): 如果check_expr值为空，返回replace_value，否则返回check_expr

- nullif(expr1, expr2): 如果两个表达式相等则返回空值，否则返回第一个表达式

- coalesce(expr1, expr2, ...): 返回第一个不为null的expr

注：指定order by时，asc首先输出空值，desc最后输出空值

### 5. 连接运算

连接条件： on <谓词>

- (inner) join: 只返回两个表中满足连接的行

- straight join: 结果与inner join相同，但优化器不能改变连接顺序

- left (outer) join: 返回左表所有行以及右表中满足连接条件的行

- fight(outer) join: 返回右表所有行以及左表中满足连接条件的行

- full outer join: A和B的并集

- left/right/full join excluding inner join

- cross join: 笛卡尔积，A cross join B相当于A, B

### 6. 集合运算

- 集合并：union (all)

- 集合交：intersect (all)

- 集合差：except (all)

### 7. 分组运算

`group by 列名 [having 条件表达式]`

可以理解为某种程度的去重。

select子句中选择的列要么出现在group by子句中，要么是**聚集函数参数**。

having用于对分组进行选择，只将聚集函数作用到满足条件的分组上。

例：查找某种类型产品均价和最高价相等的ID号：
`select ProductModelID from product group by ProductModelID having avg(ListPrice) = max(ListPrice);`

例：查找销量大于5的产品：

`select productID, sum(OrderQty) AS count from salesorderdetail group by ProductID having count > 5 order by count DESC;`

**常用聚集函数：avg, min, max, sum, count**

#### 7.1 group_concat

`group_concat(列名 order by 排序列 separator 分隔符)`

例：`SELECT department, GROUP_CONCAT(DISTINCT employee_name ORDER BY employee_name ASC SEPARATOR ' | ') AS employees FROM employees GROUP BY department;`

#### 7.2 cube & rollup

注：MySQL5.7并不支持cube操作

cube相当于按所有的可能进行聚合，比如对于cube(col1, col2)：

- 对 column1 和 column2 的所有值进行聚合。(group by col1, col2)

- 对 column1 的每个值与 column2 的所有值进行聚合。(group by col1)

- 对 column2 的每个值与 column1 的所有值进行聚合。(gropu by col2)

- 对整个数据集进行聚合（没有分组列, group by {}，col1和col2都为NULL）

语法：`select from Model, Year, Color, sum(Sales)
car_sales group by Model, Year, Color with cube`

rollup会生成某一层次的组合，例如：`group by A, B, C with rollup`

= group by {} + group by A + group by A, B + group by A, B, C

#### 7.3 分组属性集

如果分组比较随意，笨拙的方法是通过union来把多个group by结果合并，更好的方法是使用`grouping sets`

例：`select model, car_year, color, sum(sales) from car_sales group by grouping sets((model, theyear), (model, color), (theyear, color), ())`

#### 7.4 grouping

`grouping()`函数可以标示每一行到底和哪个group by相关联

对于`grouping(A, B, C)`

- group by A, B, C的标识为为4\*0+2\*0+1\*0=0

- group by A, B的标识为为4\*0+2\*0+1*1=1

- group by A的标识为为4\*0+2\*1+1\*1=3

- group by ()的标识为为4\*1+2\*1+1\*1=7

### 8. 子查询

#### 8.1 [not] in (子查询)

例：列出选修了c1课程的学生姓名

```sql
select sname
from S, SC
where S.sno = SC.sno
and cno = c1;

select sname
from S
where sno in (
    select sno
    from SC
    where cno = c1
);
```

#### 8.2 some/all 子查询

相当于 $\exists$ 和 $\forall$ 

例：

```sql
select value1 from Comp1
where value1 > some (select value2 from Comp2);

select value1 from Comp1
where value1 > all (select value2 from Comp2);
```

#### 8.3 exists 子查询

in 后的子查询与外层查询无关，每个子查询执行一次，称为无关子查询

exists 后的子查询与外层查询有关，需要执行多次，称为相关子查询

子查询返回{null}也是true，{}才是false

例：列出选修了c1课程的学生姓名

```sql
select sname
from S
where exists (
    select *
    from SC
    where cno = c1
    and sno = S.sno
)
```

实际数据库实现中会转化为表连接执行

#### 8.4 简单性能比较

例：列出选修了c1和c2课程学生学号（956232条数据）

```sql
-- exists子查询，非常慢
select sno
from SC SC1
where SC1.cno = 1
and exists (
    select sno
    from SC
    where cno = 2
    and sno = SC1.sno
);
-- 连接运算，0.1s
select SC1.sno
from SC SC1 join SC SC2
on SC1.sno = SC2.sno
where SC1.cno = 1 and SC2.cno = 2;
-- in子查询，0.09s
select sno
from SC SC1
where SC1.cno = 1
and SC1.sno in (
    select sno
    from SC SC2
    where SC2.cno = 2
);
-- 集合运算，MySQL5.7不支持
select sno from SC where cno = 1
intersect
select sno from SC where cno = 2;
```

### 9. 存储过程/函数

存储过程将程序在服务器中预先编译好并存储起来，然后应用程序只需简单地向服务器发出调用该存储过程的请求即可

函数必须返回一个值，存储过程可以返回零个、一个或多个值，而且存储过程可以多次SELECT打印中间表

函数可以在SQL语句中直接调用，而存储过程一般通过CALL调用

**存储过程一般用于对数据库修改或设置，用户定义函数则适于提取数据**

```sql
-- 查看当前数据库下的存储过程
SELECT ROUTINE_NAME, CREATED, LAST_ALTERED
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_TYPE = 'PROCEDURE' -- 可换成FUNCTION
AND ROUTINE_SCHEMA = 'DATABASE()';
```

[MySql中 delimiter 详解_mysql delimiter$$ 用法-CSDN博客](https://blog.csdn.net/yuxin6866/article/details/52722913)

[MYSQL Function函数创建和调用-CSDN博客](https://blog.csdn.net/WHYbeHERE/article/details/109222263)

[【MySQL8.0】解决函数体function不能使用declare什么变量_mysql 8.0 declare-CSDN博客](https://blog.csdn.net/weixin_42251246/article/details/103302907)

```sql
CREATE PROCEDURE AddEmployee(IN empName VARCHAR(100), IN empDepartment VARCHAR(50))
BEGIN
    INSERT INTO employees (name, department)
    VALUES (empName, empDepartment);
END
CALL AddEmployee('test', 'dep1');

CREATE FUNCTION IsOlderThan30(empID INT)
RETURNS BOOLEAN
BEGIN
    DECLARE age INT;
    SELECT age INTO age FROM employees WHERE id = empID;
    IF age > 30 THEN
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END

SELECT id, name FROM employees WHERE IsOlderThan30(id) = TRUE;
```

sqlserver还支持表值函数、apply关键字等操作

#### 9.1 DECLARE变量

存储过程和函数中通过declare定义变量在BEGIN...END中，且在语句之前。

```sql
BEGIN
    DECLARE crs INT DEFAULT 0;
END
```

#### 9.2 SET定义变量

```sql
SET @t1=0, @t2=1;
```

declare用于定义局部变量，@用于定义会话变量（可以理解为一个连接中的全局变量）

### 10. pymysql

[python操作mysql之只看这篇就够了 - 简书 (jianshu.com)](https://www.jianshu.com/p/4e72faebd27f)

[Python中pymysql模块详解：安装、连接、执行SQL语句等常见操作-CSDN博客](https://blog.csdn.net/qq_43341612/article/details/132113053)

## 0x02 特殊操作

### 1. 游标

[SQL -- 游标（详细）_sql 游标-CSDN博客](https://blog.csdn.net/M1234uy/article/details/107460439)

用于对记录逐行读取和操作，使用步骤如下：

#### 1.1 声明游标

```sql
-- 简单使用
DECLARE cursor_name CURSOR FOR select_statement;
-- 详细格式，sqlserver
DECLARE 游标名称 CURSOR       
[ LOCAL | GLOBAL ]                                   --游标的作用域
[ FORWORD_ONLY | SCROLL ]                            --游标的移动方向
[ STATIC | KEYSET | DYNAMIC | FAST_FORWARD ]         --游标的类型
[ READ_ONLY | SCROLL_LOCKS | OPTIMISTIC ]            --游标的访问类型
[ TYPE_WARNING]                                      --类型转换警告语句
FOR SELECT 语句                                      --SELECT查询语句
[ FOR { READ ONLY | UPDATE [OF 列名称]}][,...n]      --可修改的列
```

#### 1.2 打开游标

```sql
OPEN cursor_name;
```

#### 1.3 读取游标

```sql
FETCH cursor_name INTO variable_list;
```

#### 1.4 关闭游标

```sql
CLOSE cursor_name;
```

进一步还可以free或者deallocate，无法再次open（sqlserver）

#### 1.5 案例

还需要加入控制变量`done`，声明为整型并初始化为`FALSE`

再设置继处理器`DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;`

它是MySQL中的一个错误处理机制。`NOT FOUND` 是一个特殊的条件，当游标尝试读取超出结果集末尾的数据时触发。在这种情况下，处理器将 `done` 的值设置为 `TRUE`（或1）。

```sql
CREATE PROCEDURE ListEmployees()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE employee_name VARCHAR(100);
    DECLARE cur_employee CURSOR FOR SELECT name FROM employees;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur_employee;  -- 打开游标

    employee_loop: LOOP
        FETCH cur_employee INTO employee_name;
        IF done THEN
            LEAVE employee_loop;
        END IF;
        -- 此处可以处理每一个employee_name
    END LOOP;

    CLOSE cur_employee;  -- 关闭游标
END
```

错误情况：declare continue handler for not found set done = true 是对全局的select有效的，只要有一条select语句返回空，那么就是触发该语句。

[mysql游标提前退出（详细说明，完整用例，结果）_declare continue handler for not found set done = -CSDN博客](https://blog.csdn.net/pure_dreams/article/details/121338426)

### 2. 触发器

```sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT|UPDATE|DELETE}
ON table_name FOR EACH ROW
trigger_body;
```

"NEW . column_name"或者"OLD . column_name".这样在技术上处理（NEW | OLD . column_name）新和旧

对于INSERT语句，只有NEW是合法的，对于DELETE语句，只有OLD是合法的，对于UPDATE语句，可以同时使用NEW以及OLD

```sql
-- 自定义自增列
-- 这里把BEFORE改成AFTER会无法创建
DELIMITER //
CREATE TRIGGER autoID
BEFORE INSERT ON triggertest
FOR EACH ROW
BEGIN
    SET NEW.ID = (SELECT COUNT(*) FROM triggertest) + 1;
END//

-- 更新行后会加入标识
DELIMITER //
CREATE TRIGGER autoChange
BEFORE UPDATE ON triggertest
FOR EACH ROW
BEGIN
    SET NEW.NAME = CONCAT(OLD.NAME,'(CHANGED)');
END//
```

MySQL不支持语句级触发器、递归触发器、替代触发器。

错误处理：

MySQL 在触发器执行过程中处理错误的方式如下：

- 如果`BEFORE`触发器失败，则不会执行对相应行的操作。

- *尝试*`BEFORE`插入或修改行时会激活触发器 ，而不管尝试随后是否成功 。

- `AFTER`仅当任何 `BEFORE`触发器和行操作成功 执行时，才会 执行触发器。

- `BEFORE`a或 触发器 期间的错误`AFTER`会导致导致触发器调用的整个语句失败。

- 对于事务表，一条语句的失败应该导致该语句执行的所有更改的回滚。触发器失败会导致语句失败，因此触发器失败也会导致回滚。对于非事务表，无法执行此类回滚，因此尽管语句失败，但在错误点之前执行的任何更改仍然有效。

[25.3.1 触发器语法和示例_MySQL 8.0 参考手册](https://mysql.net.cn/doc/refman/8.0/en/trigger-syntax.html)

### 3. 约束

MySQL5.7不支持约束，但允许约束语法。

[SQL 用户自定义约束check的三种添加方式_添加check约束-CSDN博客](https://blog.csdn.net/z_y_z_g/article/details/124660339)

[SQL CHECK 约束 | 菜鸟教程 (runoob.com)](https://www.runoob.com/sql/sql-check.html)

[MySQL 8.0 新特性之检查约束（CHECK）_mysql8.0 check-CSDN博客](https://blog.csdn.net/horses/article/details/106529341)

### 4. 事务

1. 显式事务：用begin transaction（或start transaction）开始，commit/rollback结束

2. 隐含事务：在sqlserver中用set implicit_transaction on启用，第一次运行alert/insert/create…语句时会自动开启一个事务，然后需要利用commit/rollback结束；结束后下次再第一次运行上述语句时又开启一个事务

3. 自动事务：set (session/global) autocommit={1,0}，每条语句都作为一个事务

```sql
start transaction;
-- sql expression
commit;
```

Myisam引擎不支持事务，而Innodb引擎经过测试，事务执行到一半发生错误时会保留已经成功执行的语句。如果加入 DECLARE EXIT HANDLER FOR SQLEXCEPTION ROLLBACK; 则会回滚。

sqlserver中可以打开XACT_ABORT开关使得回滚整个事务。

```sql
DELIMITER //
CREATE PROCEDURE transTest()
BEGIN
    --DECLARE EXIT HANDLER FOR SQLEXCEPTION ROLLBACK;
    START TRANSACTION;
        UPDATE `shiwu_innodb` SET id = 1 WHERE id = 3;
        UPDATE `shiwu_innodb` SET id = 'a' WHERE id = 2;
    COMMIT;
END//

CALL `transTest()`;
```

### 4. 直方图

[MySQL8.0新特性学习笔记（三）：直方图_等宽直方图 mysql-CSDN博客](https://blog.csdn.net/lkforce/article/details/102939338)

[一文读懂MySQL 8.0直方图-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1628479)

## 0x03 性能

### 1. Partition by

MySQL中，partition by常用于分区表，以及在窗口函数中用于划分数据集（MySQL8支持），进行复杂的分析计算，比如计算每组内的行号、求和、平均值等。

窗口函数例：

```sql
SELECT department, employee_id, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank
FROM employees;
-- 为每个部门的员工按薪资排序
-- row_number为每个分区内的行分配一个唯一的序号
```

#### 1.1 表分区

表分区是一种数据库设计技术，用于把逻辑上统一的数据分割成较小的、可以独立
管理的物理单元（分片）进行存储。分区可以提高查询性能、优化数据加载和备份操作。

**插入、更新数据时，不需要指定分区，MySQL会自动帮我们处理**

数据分区优点：

- 增强可用性：如果表的某个分区出现故障，表在其他分区的数据仍然可用

- 维护方便：如果表的某个分区出现故障，需要修复数据，只修复该分区即可

- 均衡I/O：可以把不同的分区映射到磁盘以平衡I/O，改善整个系统性能

- 改善查询性能：对分区对象的查询可以仅搜索自己关心的分区，提高检索速度

一般分区方式：

- 范围分区：根据某个属性值的范围，决定将该数据存储在哪个分区上
  
  - range
  
  - list
  
  - range columns
  
  - list columns

- 散列分区：通过分区编号将数据均匀散列到I/O设备上，使得这些分区大小一致

- 复合分区：先使用范围分区，然后在每个分区内再使用散列分区

```sql
-- range分区
CREATE TABLE sales (
    sale_date DATE NOT NULL,
    amount DECIMAL(10,2) NOT NULL
)
PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN (1992),
    PARTITION p2 VALUES LESS THAN (1993),
    PARTITION p3 VALUES LESS THAN (1994)
);

-- list分区
CREATE TABLE testlist (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(10)
)
PARTITION BY LIST (id) (
    PARTITION p0 VALUES IN (1,2,3),
    PARTITION p1 VALUES IN (4,6,7)  
);
INSERT INTO testlist values(5, '李三');
-- ERROR 1526 (HY000): Table has no partition for value 5

-- hash分区
CREATE TABLE testhash1 (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(10)
)    
PARTITION BY HASH(id) PARTITIONS 4 
);
-- 取模运算。假设分区数为4，则有0, 1, 2, 3四个值，对应分区为四个
-- 83模4取余得3，数据插入到编号3的分区
```

range columns、list columns能够按照多个列进行分区，这里不赘述

根据分区表将数据分布在不同的物理磁盘上：

```sql
CREATE TABLE sales (
    sale_date DATE NOT NULL,
    amount DECIMAL(10, 2) NOT NULL
)
PARTITION BY RANGE(YEAR(sale_date)) (
    PARTITION p2020 VALUES LESS THAN (2021) 
        DATA DIRECTORY = '/mnt/disk1/data2020' 
        INDEX DIRECTORY = '/mnt/disk1/index2020',
    PARTITION p2021 VALUES LESS THAN (2022) 
        DATA DIRECTORY = '/mnt/disk2/data2021' 
        INDEX DIRECTORY = '/mnt/disk2/index2021',
    PARTITION p2022 VALUES LESS THAN (2023) 
        DATA DIRECTORY = '/mnt/disk3/data2022' 
        INDEX DIRECTORY = '/mnt/disk3/index2022'
);
-- 可能要先设置innodb_file_per_table为1
```

[超详细的mysql分库分表方案_mysql大表分表方案-CSDN博客](https://blog.csdn.net/agonie201218/article/details/110823552)

### 2. BENCHMARK

```sql
BENCHMARK(count, expr)
-- 可以测试单条SQL语句的执行速度
```

[MYSQL BENCHMARK()函数 - duanxz - 博客园 (cnblogs.com)](https://www.cnblogs.com/duanxz/p/3936759.html)

### 
