---
title: Mysql 锁
categories:
  - Mysql
top: false
top_img:
  - /images/top5.jpeg
date: 2020-04-24 14:59:14
tags:
- Mysql
- Mysql锁
keywords: Mysql锁,Mysql
description: Mysql锁
comments:
cover: /images/mysql_lock.jpg
---

![Mysql_lock](/images/mysql_lock.jpg)

## 锁分类

### 按粒度分
- 全局锁：锁的是整个database。由Mysql的Sql layer层实现
- 表级锁：锁的是某个table。由Mysql的Sql layer层实现
- 行级锁：锁的是某行数据，也可能锁定行之间的间隙。由某些存储引擎实现，如InnoDB

> 表级锁：开销小、加锁快；不会出现死锁；锁定的粒度大，发生锁冲突的概率最高，**并发**度最**低**
> 行级锁：开销大、加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，**并发**度也最**高**

### 按功能分
- 共享读锁
- 排他写锁

### 按实现方式分
- 悲观锁
- 乐观锁

## Mysql 表级锁
> 一种是表锁
> 一种是元数据锁MDL(meta data lock)
> 通过以下命令可以查看表锁状态：`mysql> show status like 'table%'`

![msyql_table_status](/images/mysql_table_status.png)

> Table_locks_immediate：产生表级锁定的次数
> Table_locks_waited：出现表级锁定争用而发生等待的次数

### 表现形式
> 表共享读锁(Table Read Lock)
> 表独占写锁(Table Write Lock)

### 操作

- 手动增加表锁：`lock table 表名称 read(write),表名称2 read(write),...`
- 查看表锁情况：`show open tables;`
- 删除表锁：`unlock tables;`

### 演示

```sql
CREATE TABLE myslock(
	id int(11) NOT NULL auto_increment,
	name varchar(20) DEFAULT NULL,
	primary key (id)
);

INSERT into mylock (id,name) values (1,'a');
INSERT into mylock (id,name) values (2,'b');
```

> 前提： 多个连接Mysql的session

#### 表读锁演示
```sql
-- session1 给mylock表加读锁
lock table mylock read;
-- session1 可以查询锁定的表
select * from mylock;
-- session1 不能访问非锁定的表
select * from tdep;
-- session2 可以查询 没有锁
select * from mylock;
-- session2 修改阻塞，自动加行写锁
update mylock set name='c' where id=2; --等待结果
-- session1 释放表锁
unlock tables;
-- session2 修改执行完成,session2界面可以看到：Rows matched: 1 changed: 1 Warnings: 0
-- session1 可以访问所有表
select * from tdep;
```

#### 表写锁演示
```sql
-- session1 给mylock表加写锁
lock table mylock write;
-- session1 可以查询
select * from mylock;
-- session1 不能访问非锁定表
select * from tdep;
-- session1 可以执行修改
update mylock set name='d' where id=2;
-- session2 查询阻塞
select * from mylock; -- 等待结果
-- session1 释放表锁
unlock tables;
-- session2 阻塞查询获得查询结果： 2 rows in set (18.58 sec)
-- session1 可以查询所有表
select * from tdep;
```

## 元数据锁 (Mysql 5.5版本中引入)
> MDL不需要显示使用，在访问一个表的时候会被**自动加上**
> MDL的作用是：保证读写的正确性。
> MDL读锁：对一个表做**增删改查**操作
> MDL写锁：对**表结构变更**操作

- 读锁之间不互斥，因此可以有多个线程同时对一张表增删改查
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。

## 行锁
> **InnoDB行锁**是通过给**索引上的索引项加锁**来实现的，因此InnoDB这种行锁的特点意味着：只有通过**索引条件检索**(where 索引)的数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁

### 演示

#### 行读锁
```Sql
-- session1 开启事务，手动加name='c'的行读锁，未使用索引
begin;
select * from mylock where name='c' lock in share mode;
-- session2 修改阻塞，由于session1的查询未用索引条件，行锁升级为表锁
update mylock set name='e' where id=2;
-- session1 提交事务 或 rollback 释放读锁
commit;
-- session2 修改成功提示
```

#### 行写锁
```Sql
-- session1 开启事务；手动给id=1加行写锁
begin;
select * from mylock where id=1 for update;
-- session2 
select * from mylock where id=2 -- 可以访问
select * from mylock where id=1 -- 可以读 不加锁
select * from mylock where id=1 lock in share mode; -- 加读锁被阻塞
-- session1
commit; -- session提交事务 或者rollback释放写锁
-- session2 执行成功
```

## 间隙锁
> 防止插入间隙内的数据
> 防止已有数据更新为间隙内的数据

### 略(待补充)


## 死锁
> 两个session互相等待对方的资源释放之后，才能释放自己的资源


