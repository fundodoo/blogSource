---
title: Mysql 索引基础
categories:
  - Mysql
top: false
top_img:
  - /images/top3.jpeg
date: 2020-04-11 14:11:45
tags:
- Mysql
- Mysql索引
keywords: Mysql索引
description: 
comments: 
cover: /images/mysql_index.jpeg
---

## 索引介绍
> 索引是帮助Mysql高效获取数据的数据结构，就像一本书前面的目录，能加快数据库的查询速度。

### 优势
- 可以提高数据**检索效率**，降低数据库的**IO成本**   ---检索
- 通过索引列对数据排序，**降低**数据**排序的成本**，**降低**CPU的消耗    ---排序
  - 被索引的列会自动进行排序，包括【单列索引】、【组合索引】。组合索引的排序要复杂一些
  - 如果是按照**索引列的顺序**进行排序，对应order by语句，效率就会提升很多
  - where 索引列，在**存储引擎层处理**，这里使用到了**索引下推**（**ICP**）
  - **覆盖索引**  `select 字段` 字段是索引列，则**不**需要**回表查询**。

### 劣势
- 索引会**占据磁盘空间**
- 索引提高了查询效率，但是会**降低更新表**的效率。比如每次对表进行增删改操作，Mysql不仅要保存数据，还要保存或者更新对应的索引文件

## 索引分类

### 单列索引
- **普通索引**：Mysql中的基本索引类型，允许在定义索引的列中插入重复值和空值，纯粹为了查询速度更快一些
- **唯一索引**：索引列中的**值**必须是**唯一**的，但是**允许**空值
- **主键索引**：特殊的唯一索引，**不允许**空值

### 组合索引
- 在表中**多个列**上创建的索引
- 组合索引的使用，需要遵循**最左前缀原则**
- 建议使用组合索引替代单列索引

### 全文索引
> 只有在MyIsam引擎上使用，而且只能在CHAR、VARCHAR、TEXT类型字段才能使用全文索引

### 空间索引
> 

## 索引使用

### 创建索引
> 创建索引时，可以指定索引列使用**多大的长度**作为索引，但是**数值类型不要指定**。

- 单列索引--普通索引
  ```Sql
  CREATE INDEX index_name ON table(column(length));  //方法1
  ALTER TABLE table_name ADD INDEX index_name (column(length));    //方法2   
  ```

- 单列索引--唯一索引
  ```Sql
  CREATE UNIQUE INDEX index_name ON table(column(length));  //方法1
  ALTER TABLE table_name ADD UNIQUE INDEX index_name (column(length));    //方法2   
  ```

- 单列索引--全文索引
  ```Sql
  CREATE FULLTEXT INDEX index_name ON table(column(length));  //方法1
  ALTER TABLE table_name ADD FULLTEXT INDEX index_name (column);    //方法2   
  ```

- 组合索引
  ```Sql
  ALTER TABLE table_name ADD INDEX index_name (column1(length),column2(length)); 
  ```

### 删除索引
- 
  ```Sql
  DROP INDEX index_name ON table
  ```

### 查看索引
- 
  ```Sql
  SHOW INDEX FROM table  \G
  ```

## 索引原理分析

### 页、块、扇区
> **内存**以`页`这个单位去进行IO读取，一般大小为`4k`,在Mysql中可以通过`innodb_page_size`设置大小，一般设置为`16k`
> **操作系统**以`块`这个逻辑单位去操作磁盘，常见为`4k`
> **磁盘**以`扇区`这个物理最小磁盘单位去存储数据，常见为`512Byte`

> `页`大小查看：`getconf PAGE_SIZE`,常见为4k
> `块`大小查看：`stat /boot/|grep "IO Block"`,常见为4k
> `扇区`大小查看：`fdisk -l`,常见为512Byte

> **指针**默认长度为`6bit`，如果**key**为bigint则是`8bit`，那么一个**索引**为8+6=`14bit`

### 索引存储结构
- 索引是在**存储引擎中实现**的，也就是说不同的存储引擎使用不同的索引
- MyISAM和**InnoDB**存储引擎：**只**支持**B+ TREE**索引，也就是说默认使用BTREE，**不能更换**
- Memory存储引擎：支持HASH、BTREE索引

### B树和B+树
> 数据结构演示网站：https://www.cs.usfca.edu/~galles/visualization/Algorithms.html

- B树
  - B树是为了磁盘或其他存储设备而设计的一种多叉平衡查找树
  - B树的高度一般都在2-4这个高度，树的高度直接影响IO读写次数
  - 如果是三层树结构---可以支撑20G的数据，如果是四层结构---可以支撑几十 T
    ![B TREE](/images/btree.png)

- B+ 树
    ![B TREE](/images/b+tree.png)

- <span style="color:red;">**区别**</span>
  -  B树和B+树的最大区别在于**非叶子节点是否存储数据**
      1. B树是非叶子节点和叶子节点都会存储数据
      2. B+树**只**有**叶子节点**存储数据，而且存储的数据都是在一行上，这些数据都是有指针指向的，也就是有顺序的。

### 非聚集索引(MyISAM)
> B+树叶子节点只会存储**数据行(数据文件)的指针**，也即是**数据和索引**存储**在不同的文件**就是非聚集索引
> 非聚集索引包含**主键索引**和**辅助索引**都会存**储指针的值**

- 主键索引
  ![非聚集主键索引](/images/mysql_feijj.jpg)

- 辅助索引(次要索引)
  ![非聚集辅助索引](/images/mysql_feijjfz.jpg)

### 聚集索引(InnoDB)
> 完整的记录，存储在主键索引中，通过**主键**索引就可以**获取**记录**所有的列**，也就是说**数据和索引**是在**一起**
> **主键索引**<span style="color:red;">必须要有</span>，表如果没有设置主键，则系统会默认生成一个隐藏的唯一列，由该列去创建key作为主键
> 主键最好是int
> **次要索引**只*存储*主键索引的**主键key**，不存储表数据

- 主键索引
  ![聚集主键索引](/images/mysql_jj.jpg)

- 辅助索引(次要索引)
  ![聚集辅助索引](/images/mysql_jjfz2.jpg)

  <span style="color:red;">**注意**</span>：**聚集索引**如果是**非主键查询**，则需要**搜索两次**索引树(一次是搜索次要索引树，一次是搜索主键索引树)才能查出数据

  ![聚集辅助索引](/images/mysql_jjfz.jpg)