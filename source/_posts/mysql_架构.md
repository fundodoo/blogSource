---
title: Mysql 架构
categories:
  - Mysql
top: false
date: 2020-04-09 17:31:21
tags:
- Mysql
keywords:
description: 
top_img: 
  - /images/top3.webp
cover: /images/mysql_jiagou.jpg
---

## 逻辑架构

![Architecture](/images/mysql_jiagou.jpg)

### Connectors
> 连接器：不同语言与SQL的交互

### Management Service & Utilities
> 系统管理和控制工具

### Connection Pool
> 管理缓冲用户连接，线程处理等需要缓存的需求
> 负责监听对Mysql Server的各种请求，接收连接请求，转发所有连接请求到线程管理模块。每一个连接上Myql Server的客户端都会被分配(或创建)一个连接线程为其单独服务。
> 连接线程的主要工作就是负责Mysql Server与客户端的通信，接收客户端的命令请求，传递Server端的结果信息等。线程管理模块则负责管理维护这些线程，包括线程的创建、线程的cache等。

### SQL Interface
> 接收用户的SQL命令(DDL、DML)，并且返回用户需要查询的结果。

### Parser
> 解析器：**验证、解析**SQL命令
  - 将SQL语句进行语义和语法的分析，分解成**数据结构**，然后暗转不同的操作类型进行分类，然后做出针对性的转发到后续步骤。
  - 如果在分解过程中遇到错误，则说明sql语句不合理。

### Optimizer
> 查询优化器：SQL语句在查询之前会使用**查询优化器对查询进行优化**。**explain**语句查看的SQL语句执行计划，就是**查询优化器**生成的。

### Cache & Buffer
> 查询结果缓存
> **Mysql 8.0** 不再使用

### Pluggable Storage Engines
> 存储引擎：存储引擎是**针对表**的，**同一数据库**用户可以根据**不同的需求为数据表**选择**不同的存储引擎**，用户可以根据需要编写自己的存储引擎。
```Sql
create table testxxx() engine=InnoDB   //InnoDB存储引擎
create table testxxx() engine=MyISAM   //MyISAM存储引擎
create table testxxx() engine=Memory   //Memory存储引擎
```
#### 存储引擎类型
|存储引擎   |说明|
|:------------|---------------|
|MyISAM|高速引擎，拥有较高的插入，查询速度，但**不支持事务，不支持行锁**，支持3种不同的存储格式。包括静态型、动态型、压缩型|
|InnoDB|**5.5版本**后Mysql的**默认存储引擎**，**支持事务和行级锁。事务处理、回滚、崩溃修复能力和多版本并发控制的事务安全**，比MyISAM稍慢，支持**外键**|
|Memory|内存存储引擎|
|Falcon|一种新的存储引擎，支持事务处理，传言是InnoDB的替代者|
|Archive|将数据压缩后进行存储，非常适合存储**大量的**、**独立的**作为历史记录的数据，但是**只能**进行**插入和查询**操作|
|xtraDB|`Percona`公司提供的村粗引擎，该公司还对Mysql开源代码二次开发出了`Percona Server`，阿里对于`Percona Server`进行修改，衍生了`alisql`|

#### 查看存储引擎
```mysql
  mysql> show engines;
```

#### InnoDB和MyISAM存储引擎比较
| |InnoDB|MyISAM|
|:------------|---------------|---------------|
|存储文件|`.frm`: 表定义文件 <br> `.ibd`: 数据文件和索引文件|`.frm`: 表定义文件 <br> `.myd`: 数据文件 <br> `.myi`: 索引文件|
|锁|表锁、行锁|表锁|
|事务|支持|不支持|
|CRUD|读、写|读多|
|count|扫表|专门存储的地方(加where后也扫表)|
|索引结构|B+Tree|B+Tree|
|外键|支持|不支持|

### 执行流程

![mysql_process](/images/mysql_process.jpg)

## 物理结构

- Mysql是通过**文件系统**对数据和索引进行存储的
- Mysql从物理结构上可以分为<span style="color:blue;">**日志文件**</span>和<span style="color:green;">数据索引文件</span>
- Mysql在`Linux`中的数据索引文件和日志文件都在 **/var/lib/mysql**目录下
- <span style="color:blue;">**日志文件**</span>采用的是<span style="color:blue;">**顺序IO**</span>方式存储、<span style="color:red;">**数据文件**</span>采用的是<span style="color:red;">**随机IO**</span>方式存储

### 日志文件(顺序IO)
> Mysql 通过日志记录了数据库 ***操作信息***和 ***错误信息***。常用的日志文件包括**错误日志、二进制日志、查询日志、慢查询日志、事务Redo日志、中继日志**等。

可以通过命令查看当前数据库中的日志使用信息
```mysql
  mysql> show variables like 'log_%';  
```

#### 错误日志(errorlog)
> 默认是开启的，而且从v5.5.7后**无法关闭**错误日志，错误日志记录了**运行过程**中遇到的**所有严重的错误信息**，以及Mysql**每次启动和关闭**的详细信息。
> 默认的错误日志名称：hostname.err
> 错误日志所记录的信息是可以通过**log-error**和**log-warnings**来定义，其中log-err是定义是否启用错误日志的功能和错误日志的存储位置；log-warnings是定义是否将警告信息也定义至错误日志中。

#### 二进制日志(bin log)
> bin log记录了数据库**所有DDL语句和DML语句**，但**不包括select**语句内容
> 语句以事件的形式保存，描述了数据的变更顺序，binlog还包括了每个更新语句的**执行时间**信息
> `DDL`语句，则**直接**记录到binlog日志
> `DML`语句，则必须通过**事务提交**才能记录到binlog日志中

> binlog 主要用于实现Mysql**主从复制、数据备份、数据恢复**; 修改`Mysql`配置文件开启
```Properties
# mysql-bin 是binlog日志文件的basename，binlog日志文件的完整名称：mysql-bin-000001.log
log-bin=mysql-bin  
```

#### 通用查询日志(general query log)
> 默认是关闭的，生产环境**不建议开启**

```mysql
  mysql> show global variables like 'general_log%';  
```
> 开启方式，修改`Mysql`配置文件
```Properties
# 启动开关
general_log={ON|OFF}
# 日志文件变量，而general_log_file如果没有指定，默认名是host_name.log
general_log_file=/PATH/TO/file
# 记录类型
log_output={TABLE|FILE|NONE}
```


#### 慢查询日志(show query log)
> 默认是**关闭**的，主要用于**SQL调优**时定位**慢的select**; 需要通过以下配置开启
```Properties
# 开启慢查询日志
slow_query_log=ON
# 慢查询的阈值,查询时间超过long_query_time的则会被记录到慢查询日志文件中。
long_query_time=3
# 日志记录文件如果没有给出file_name值，默认为主机名，后缀为-slow.log;如果给出了文件名，但不是绝对路径名，文件则写入数据目录
slow_query_log_file=file_name
```


#### 重做日志(redo log)
- **作用**
  1. 确保事务的持久性
  2. 防止在发生故障时间点，尚有脏页未写入磁盘，在重启Mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性

- **内容**
  
1. 物理格式的日志，记录的是物理数据页面的修改信息，其redo log是顺序写入redo log file的物理文件中
  
- **什么时候产生**
  
1. 事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo log文件中
  
- **什么时候释放**
  
1. 当对事务的脏页写入到磁盘之后，redo log的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）
  
- **对应的物理文件**
  1. 默认情况下，对应的物理文件位于数据库的data目录下的ib_logfile1 & ib_logfile2

  > innodb_log_group_home_dir : 指定日志文件组所在的路径，默认 **./** 表示在数据库的数据目录下。
  > innodb_log_files_in_group : 指定重做日志文件组中文件的数量，默认**2**
  > innodb_log_file_size : 重做日志文件的大小
  > innodb_mirrored_log_groups : 指定了日志镜像文件组的数量，默认**1**

- <span style="color:red"> **注意** </span>
  > 很重要一点，redo log是什么时候写盘？前面说是在事务开始之后逐步写盘的，之所以所重做日志是在事务开始之后逐步写入重做日志文件，而不一定是事务提交才写入重做日志缓存，原因就是，重做日志有一个**缓冲区**`Innodb_log_buffer`，`Innodb_log_buffer`的默认大小是**8M**,Innodb存储引擎先将重做日志写入`Innodb_log_buffer`中

  > 然后通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘
    1. `Master Thread` **每秒**一次执行刷新Innodb_log_buffer到重做日志文件。
    2. 每个**事务提交**时会将重做日志缓冲刷新到重做日志文件
    3. 当`Innodb_log_buffer` 可用空间 **少于一半** 时，重做日志缓存被刷新到重做日志文件


#### 回滚日志(undo log)
- **作用**
  1. 保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读(MVCC),即非锁定读

- **内容**
  1. 逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的

- **什么时候产生**
  1. 事务开始之前，将当前版本生成undo log，undo log也会产生redo log来保证undo log的可靠性

- **什么时候释放**
  1. 当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否有其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间

#### 中继日志(relay log)
- 在**主从复制**环境中产生的日志
- 主要作用： 为了从机可以从中继日志中获取到主机同步过来的SQL语句，然后执行到从机中

### 数据文件(随机IO)

#### 查看数据文件
```mysql
  mysql> show variables like '%datadir%';
```

#### InnoDB数据文件
- **.frm文件**：主要存放与表相关的数据信息，主要包括**表结构的定义**信息
- **.ibd文件**：使用**独享表空间存储表数据和索引信息**，**一张表**对应**一个ibd**文件
- **ibdata文件**：使用**共享表空间**存储表结构和索引信息，所有表共同使用一个或者多个ibdata文件

#### MyIsam数据文件
- **.frm文件**：主要存放与表相关的数据信息，主要包括**表结构的定义**信息
- **.myd文件**：主要用来存储**表数据**信息
- **.myi文件**：主要用来存储表数据文件中任何**索引的数据**树

