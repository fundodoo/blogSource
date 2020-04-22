---
title: Mysql 执行计划
categories:
  - Mysql
top: false
top_img:
  - /images/top3.jpeg
date: 2020-04-11 14:11:45
tags:
- Mysql
- Mysql Explain
keywords: Mysql执行计划
description: 
comments: 
cover: /images/mysql_explain.png
---

## Mysql执行计划

> Mysql提供了一个`Explain`命令,它可以对 **`SELECT语句`** 的执行计划进行**分析**,并输出SELECT执行的详细信息**供开发人员**有针对性的**优化**

> 使用`Explain`这个命令来查看一个SQL语句的**执行计划**,查看该SQL语句有没有使用上索引,有没有做权标扫描

> 可以通过`Explain`命令深入了解Mysql的**基于开销的优化器**,还可以获得很多可能被优化器考虑到的访问策略细节,以及当运行SQL语句时优化器采用的策略

> `Explain`命令用法简单,在`SELECT`语句**前**加上`Explain`就可以.

## 建表语句

```Sql
CREATE TABLE user(
    id int primary key,
    name varchar(100),
    age int,
    sex char(1),
    address varchar(100)
);

ALTER TABLE user ADD INDEX idx_name_age(name(100),age);
ALTER TABLE user ADD INDEX idx_sex(sex(1));

INSERT INTO user(id,name,age,sex,address) VALUES (1,'zhansan',20,'0','Fundodoo大厦');
```

![Mysql Explain](/images/mysql_explain.png)


## Explain参数说明

- id: SELECT查询的标识符,每个SELECT都会自动分配一个唯一的标识符
- **select_type**: SELECT查询的类型
- table: 查询的是哪个表
- partitions: 匹配的分区
- **type**: join类型
- possible_keys: 此次查询中可能选用的索引
- key: 此次查询中确切使用到的索引
- ref: 哪个字段或常数与key一起被使用
- rows: 显示此查询一共扫描多少行,这个是一个估计值
- filtered: 此查询条件所过滤的数据百分比
- **extra**: 额外的信息

### id
> 每个单位查询的SELECT语句都会自动分配的一个唯一标识符,表示查询中操作表的顺序
    
- id相同: 执行书序由上到下
- id不同: 如果是子查询,id号会自增,**id越大-->优先级越高**
- id相同的不同的同时存在
- id列为null的就表示这个一个结果集,不需要使用它来进行查询

### select_type(重要)
> 单位查询的查询类型,eg: 普通查询、联合查询(union、union all)、子查询等复杂查询.

- simple: 表示**不**需要`union`操作或者**不包括***子查询*的简单`SELECT`查询.有连接查询时,外层的查询为simple,且只有一个.
    ```Sql
    explain select * from user ;
    ```

- primary: 一个需要`union`操作*或*者含有**子查询**的`select`,位于最外层的单位查询的select即为primary.且只有一个
    ```Sql
    explain select (select name from user) from user;
    ```

- union: **连接**的两个`select`查询,第一个查询时`dervied`派生表,除了第一个表外,第二个以后的表*select_type*都是`union`
    ```Sql
    explain select * from user a union select * from user b;
    explain select * from (select * from user a union select * from user b) c;
    ```

- dependent union: 与union一样,出现在union或union all 语句中,但是这个查询要**受到外部查询的影响**
    ```Sql
    explain select * from user u where u.id in (select id from user a union select id from user b);
    ```

- union result: 包含union的结果集,在union和union all语句中,因为它不需要参与查询,所以id字段为null
    ```Sql
    explain select * from user a union select * from user b;
    ```

- subquery: 除了from子句中包含的子查询外,其他地方出现的子查询都可能是subquery
    ```Sql
    explain select (select name from user) from user;
    ```

- dependent subquery: 与dependent union类似,表示这个subquery的查询要受到外部表查询的影响
    ```Sql
    explain select (select name from user a where a.id = b.id) from user b;
    ```

- derived: from子句中出现的子查询,也叫派生表
    ```Sql
    explain select * from (select * from user) t;
    ```

### table
> 显示的单位查询的表名,有以下几种情况:

- 查询使用了别名,那么这里显示的是别名
- 不涉及对数据表的操作,那么这里显示为null
- 显示为尖括号括起来的就表示这个是临时表,后边的数字(N)就是执行计划中的id,表示结果来自于这个查询产生
- 是尖括号括起来<union M,N>,也是一个临时表,表示这个结果来自于union查询的id为M,N的结果集

### type(重要)
> 显示的是单位查询的连接类型或者理解为访问类型,访问性能依次**从好到差**:`system`、`const`、`eq_ref`、`ref`、`fulltext`、`ref_or_null`、`unique_subquery`、`index_subquery`、`range`、`index_range`、`index`、`All`

#### 注意事项
> 除了All之外,其他的type都可以使用到索引
> 除了index_merge之外,其他的type只可以用到一个索引
> 最少要使用到range级别

- system: 表中只有一行数据或者是空表
    ```Sql
    explain select * from (select * from user where id = 1) t;
    ```

- **const**(重要): where条件使用了唯一索引或者主键,返回记录一定是**1**行的等值where条件时,通常type是const
    ```Sql
    explain select * from user where id = 1;
    ```

- **eq_ref**(重要): 此类型通常出现在多表的join查询,表示对于前表的每一个结果,**都只能匹配到后表的一行记录**,并且查询的比较操作同事是`=`(等值连接的两个表的列是唯一索引或者主见列)
    ```Sql
    explain select * from user a left join user b on a.id = b.id;
    ```

- **ref**(重要): 针对非唯一索引使用等值(=)查询,或者使用了最左前缀规则索引的查询
    - 组合索引
        ```Sql
        explain select * from user a left join user b on a.name = b.name;
        explain select * from user where name = 'zhangsan';
        ```
    - 非唯一索引
        ```Sql
        explain select * from user where sex = '1';
        ```

- fulltext: 全文索引检索,要注意,全文索引的优先级很高,若全文索引和普通索引同事存在,Mysql不管代价,优先选择使用全文索引.

- ref_or_null: 与ref方法类似,只是增加了null值得比较

- unique_subquery: 用于where中的in形式子查询,子查询返回不重复唯一值

- index_subquery: 用于in形式子查询使用到了辅助索引或者in常数列表,子查询可能返回重复值,可以使用索引将子查询去重

- **range**(重要): **索引范围扫描**,常见于使用`>`、`<`、`is null`、 `between`、`in`、`like`等运算符的查询
    ```Sql
    explain select * from user where name like 'a%';
    explain select * from user where id > 1;
    ```

- index_range: 表示查询使用了两个以上的索引,最后取交集或者并集,常见and、or的条件使用了不同的索引,官方排序这个在ref_or_null之后,但是实际上由于要读取??索引,性能可能大部分时间都不如range

- **index**(重要): select结果列中使用了索引,type会显示为index.*全部索引扫描*,常见于使用索引列就可以处理,不需要读取数据文件的查询、可以使用索引排序或者分组的查询
    ```Sql
    explain select name from user;
    explain select age from user;
    ```

- **all**(重要): 全表扫描数据文件,然后在**server层进行过滤**返回符合要求的记录
    ```Sql
    explain select * from user;
    explain select * from user where address = 'fundodoo大厦';
    ```

### possible_keys
> 此次查询中可能选用的索引,一个或多个

### key
> 查询真正使用到的索引,select_type为index_merge时,这里可能出现两个以上的索引,其他的select_type这里只会出现一个

### key_len
> 用于处理查询的索引长度,如果是单列索引,那就是整个索引长度,如果是多列索引,那么查询不一定都能使用到所有的列,具体使用到了多少个列的索引,这里就会计算进去,没有使用到的列,这里不会计算进去.

> 留意这个列的值,算一下多列索引总长度就能知道有没有使用索引的所有列

> key_len只计算where条件用到的索引长度,而排序和分组就算用到了索引,也不会计算到key_len中

### ref
> 如果where条件使用的常数等值查询,这里会显示const
> 如果是连接查询,被驱动表的执行计划这里会显示驱动表的关联字段
> 如果是条件使用了表达式或者函数、或者条件列发生了内部隐式转换,这里可能显示为func

### rows
> 这里是执行计划中估算的扫描行数,不是精确值
> InnoDB不是精确值,因为InnoDB使用了MvCC并发机制
> MyISAM是精确的值

### extra(重要)

- using filesort: 排序时无法使用到索引,就会出现这个.常见于`order by` 和 `group by`语句中
    ```Sql
    explain select * from user order by address;
    ```

    #### 排序优化与索引使用
    > 为了优化SQL语句的排序性能,最好的情况是避免排序,合理利用索引是一个不错的方法.因为索引本身也是有序的,如果在需要排序的字段上面建立了合适的索引,那么久可以跳过排序的过程,提高SQL的查询速度

    > 假设 *t1* 表存在索引 *key1(key_part1,key_part2)*、*key2(key2)*

    - 可以利用索引避免排序的SQL
        ```Sql
        SELECT * FROM t1 ORDER BY key_part1,key_part2;
        SELECT * FROM t1 WHERE key_part1 = constant ORDER BY key_part2;
        SELECT * FROM t1 WHERE key_part1 > constant ORDER BY key_part1 ASC;
        SELECT * FROM t1 WHERE key_part1=constant1 AND key_part2>constant2 ORDER BY key_part2;
        ```
    
    - 不能利用索引避免排序的SQL
        ```Sql
        -- 排序字段在多个索引中，无法使用索引排序
        SELECT * FROM t1 ORDER BY key_part1,key_part2, key2;
        -- 排序键顺序与索引中列顺序不一致，无法使用索引排序
        SELECT * FROM t1 ORDER BY key_part2, key_part1;
        -- 升降序不一致，无法使用索引排序
        SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC;
        -- key_part1是范围查询，key_part2无法使用索引排序
        SELECT * FROM t1 WHERE key_part1> constant ORDER BY key_part2;
        ```

    #### 排序实现的算法 
    > 略(待补充)

- using index: 查询时**不需要回表查询**,直接通过索引就可以获取查询的数据
    - 表示相应的SELECT查询中使用到了**覆盖索引(Covering index)**,避免访问表的数据行,效率高
    - 如果同时出现**Using where**,说明索引被用来**执行查找索引键值**
    - 如果没有同时出现**Using where**,表明索引用来**读取数据而非执行查找动作**

    ```Sql
    explain select name,age from user;
    ```

- using index condition: 先条件过滤索引,过滤索引后找到所有符合索引条件的数据行,随后用WHERE子句中的其他条件对数据行进行过滤.
    #### 索引下推(ICP)
    - 略(待补充)
    #### WHERE 条件分类
    - index key: 用于确定SQL查询在*索引*中的**连续范围**(起始范围+结束范围)的查询条件,被称之为`index key`.由于一个范围,至少包含一个起始和一个终止,因此`index key`也被拆分为`index first key`和`index last key`分别用于定位索引查找的起始和终止条件.也就是说**根据索引来确定扫描范围**
    - index filter: 在使用`index key`确定了起始范围和结束范围之后,在此范围之内,还有一些记录不符合where条件,如果这些条件可以使用索引进行过滤,那么就是`index filter`.也就是说**用索引来进行where条件过滤**
    - table filter: where中的条件不能使用索引进行处理的,只能访问table进行条件过滤

- using temporary: 使用了**临时表**存储中间结果.Mysql在对查询结果order by和group by时使用了临时表.临时表可以使内存临时表和磁盘临时表,在执行计划中无法区分,需要查看status变量,used_tmp_table,used_tmp_disk_table才能看出来.

- distinct: 在select部分使用了distinct关键字(索引字段)

- using where: 表示存储引擎返回的记录并不是所有的都满足查询条件,需要在server层进行过滤.
    - 查询条件分为**限制条件**和**检查条件**,在*5.6之前*,存储引擎只能根据**限制条件**扫描数据并返回,然后*server层*根据**检查条件**进行过滤再返回真正符合查询的数据.5.6.x之后支持ICP特性,可以把检查条件也下推到存储引擎层,不符合**检查条件**和**限制条件**的数据,直接不读取,这样就大大减少了存储引擎扫描的记录数量
    - extra列显示`using index condition`





