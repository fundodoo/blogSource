---
title: Mysql 索引失效分析
categories:
  - Mysql
top: false
top_img:
  - /images/top4.jpeg
date: 2020-04-23 09:39:49
tags:
- Mysql
- Mysql索引
keywords: Mysql索引
description:
comments:
cover: /images/mysql_index_loss.png
---

## 案例环境
```Sql
CREATE TABLE user(
	id int PRIMARY KEY,
	name varchar(100),
	age int,
	sex char(1),
	address varchar(100)
);

alter table user add index idx_name_age_sex(name(100),age,sex(1));

INSERT INTO user(id,name,age,sex,address) values(1,'zhangsan',20,'0','Fundodoo大厦');
```

![mysql_index_loss](/images/mysql_index_loss.png)

### 全值匹配我最爱
> 条件与索引一一对应

- 正确示例

  ```Sql
  explain SELECT * FROM user where name='zhangsan';
  explain SELECT * FROM user where name='zhangsan' and age=1;
  explain SELECT * FROM user where name='zhangsan' and age=1 and sex='1';
  ```

### 最左前缀法则
> **组合索引**。如果索引了多个列，要遵守最佳左前缀法则，就是查询从索引的最左前列开始并且不能跳过索引中的列。

- 错误示例
  ```Sql
  -- 带头索引死
  explain select * from user where age=23;

  -- 中间索引断（带头索引生效，其他索引列失效），通过key_len可以看出使用的索引长度
  explain select * from user where name='zhangsan' and sex='1';
  ```

### 不要在索引上做计算
> 不要在索引列上做这些操作：计算、使用函数、自动/手动类型转化，这些任一操作都会导致索引失效而转向全表扫描

- 示例
  ```Sql
  -- 比较查看下面两种写法的执行计划
  -- 正确写法
  explain select * from user where name='fundodoo'
  -- 错误写法
  explain select * from user where left(name,2)='fu'
  explain select * from user where trim(name)='fundodoo'

  -- age有做计算->age列索引失效，通过查看key_len的长度判断是否使用了age列的索引
  -- 正确写法
  explain select * from user where name='fundodoo' and age = 1
  -- 错误写法
  explain select * from user where name='fundodoo' and age+1 = 1

  -- sex值自动转换了->sex列索引失效，通过key_len长度区分
  -- 正确写法
  explain SELECT * FROM user where name='zhangsan' and age=1 and sex='1';
  -- 错误写法
  explain SELECT * FROM user where name='zhangsan' and age=1 and sex=1;
  ```

### 范围条件右边的列索引失效
> 不能继续使用索引中**范围条件(bettween、<、>、in等)右边**的*列*

- 示例
  ```Sql
  explain SELECT * FROM user where name='zhangsan' and age=1 and sex='1';
  -- 失效 (通过key_len可以判断)
  explain SELECT * FROM user where name='zhangsan' and age>1 and sex='1';
  ```

### 尽量使用覆盖索引
> 尽量使用**覆盖索引**(只查询索引的列)，也就是索引列和查询列一致，减少使用`select *`

- 示例
  ```Sql
  -- 没有使用覆盖索引
  explain select * from user;
  -- 下面sql语句会使用覆盖索引,Extra列的值为 Using index
  explain SELECT name FROM user;
  explain SELECT name,age FROM user;
  explain SELECT name,age,sex FROM user;
  ```

### 索引字段上不要使用不等
> 索引字段上使用(!= 或者 < >) 判断时，会导致索引失效而转向全表扫描。注：主键索引会使用范围索引，辅助索引会失效

- 示例
  ```Sql
  -- 索引正常使用
  explain SELECT * FROM user where name='zhangsan';
  -- 索引失效写法
  explain SELECT * FROM user where name !='zhangsan';
  ```

### 索引字段上不要判断null
> **主键**字段上不可以判断null，**索引字段**上使用is null判断时，可以使用索引

- 示例
  ```Sql
  -- 索引列判断is null 可以使用索引
  explain select * from user where name is null; 
  -- 主键判断null，不能使用索引
  explain SELECT * FROM user where id is not null;
  explain SELECT * FROM user where id is null;
  ```

### 索引字段使用like不以通配符开头
> 索引字段使用`like`以通配符开头('%xxx')时，会导致索引失效而转向全表扫描
> `like`以通配符结束相当于范围查找，但不会导致右边的索引列失效。（与bettween、<、>、in等）不同
> 使用**覆盖索引**可以解决`like '%xxx%'`时索引失效问题

- 示例
  ```Sql
  -- 索引失效
  explain SELECT * FROM user where name like '%f%';
  -- 索引有效
  explain SELECT * FROM user where name like 'f%';
  -- 覆盖索引解决 like ‘%xxx’ 通配符开头问题
  explain SELECT name,age,sex FROM user where name like '%f%';
  ```

### 索引字段字符串要加单引号
> 索引字段是字符串，但查询时不加单引号，会导致索引失效而转向全表扫描

- 示例
  ```Sql
  -- 索引有效
  explain SELECT * FROM user where name='zhangsan';
  -- 索引失效
  explain SELECT * FROM user where name=123;
  ```

### 索引字段不要使用or
> **主键**索引字段使用`or`时，会使用`range`；**非主键**索引字段使用`or`时，会导致索引失效而转向全表扫描

- 示例
  ```Sql
  -- 非主键，索引失效
  explain SELECT * FROM user where name='zhangsan' or name='fundodoo';
  -- 主键，type=range
  explain SELECT * FROM user where id=1 or id = 2;
  -- 主键，type=const
  explain SELECT * FROM user where id=1;
  ```

## 总结
> 假设index(a,b,c)

|where 语句|是否使用索引|
|----------|---------------|
|where a=3 |Y，使用到a       |
|where a=3 and b = 5 |Y，使用到a,b       |
|where a=3 and b=5 and c=6 |Y，使用到a,b,c       |
|where b=3 或者 where b=3 and c=2 或者 where c=4|**N**     |
|where a=3 and c=5|Y，使用到a;但是c不可以，因为b中间断了     |
|where a=3 and b>5 and c=2 |Y，使用到a,b; 但c没有使用，因为b使用了范围     |
|where a=3 and b like 'xx%' and c=1 |Y，使用到a,b,c       |
|where a=3 and b like '%xx' and c=1 |Y，只使用到a      |
|where a=3 and b like '%xx%' and c=1 |Y，只使用到a      |
|where a=3 and b like 'x%xx%' and c=1 |Y，使用到a,b,c     |

### 优化口诀
> 全值匹配我最爱，最左前缀要遵守；
> 带头大哥不能死，中间兄弟不能断；
> 索引列上少计算，范围之后全失效；
> LIKE百分写最右，覆盖索引不写星；
> 不等空值还有or，索引失效要少用；