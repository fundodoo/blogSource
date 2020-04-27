---
title: Redis 内存模型
categories:
  - Redis
top: false
top_img:
  - /images/top3.jpeg
date: 2020-04-26 09:26:45
tags:
- Redis
keywords: redis,redis内存模型
description:
comments:
cover: /images/redis_jemalloc_memory.jpg
---

## Redis内存统计
> info命令可以显示redis服务器的许多信息，包括服务器的基本信息、CPU、内存、持久化、客户端连接信息等

> 查看Redis的内存使用情况，使用redis-cli客户端连接服务器，输入 `info memory` ，如下

```Code
127.0.0.1:6379> info memory
# Memory
# Redis 分配的内存总量，包括虚拟内存(单位：字节)
used_memory:853464
# 占操作系统的内存(包括进分配器分配的内存、程运行本身需要的内存、内存碎片等)，不包括虚拟内存(单位：字节)
used_memory_rss:12247040
# 内存碎片比例 如果小于0说明使用了虚拟内存
mem_fragmentation_ratio:15.07
# Redis使用的内存分配器
mem_allocator:jemalloc-5.1.0
```

- `used_memory`和`used_memory_rss`，`used_memory`是从**Redis角度**衡量的，`used_memory_rss`是从**操作系统角度**衡量的。两者不同，一方面是因为**内存碎片**和**Redis进程**运行需要占用内存，使得used_memory比较小；另一方面**虚拟内存**的存在，有可能使used_memory比used_memory_rss大

- 实际使用中，Redis的数据量会比较大，此时进程运行占用的内存与Redis数据量和内存碎片相比，都会小很多；因此used_memory_rss和used_memory的比例，便成了衡量Redis内存碎片率的参数`mem_fragmentation_ratio`(**used_memory_rss/used_memory**)
  - mem_fragmentation_ratio**一般大于** 1，且该**值越大**，说明**内存碎片比例越大**
  - mem_fragmentation_ratio **小于** 1，说明Redis使用了 **虚拟内存(swap)**，由于虚拟内存的媒介是磁盘，比内存速度要慢很多，当这种情况出现时，应该几十排查，如果内存不足应该及时处理，如增加Redis节点、增加Redis服务器内存、优化应用等
  - `jemalloc`的内存分配器下`mem_fragmentation_ratio`健康值一般在**1.03**左右，上面演示mem_fragmentation_ratio值比较大，是因为还没有向Redis中存入数据，Redis进程本身运行的内存使得used_memory_rss比used_memory大很多

- Redis的内存分配器是在编译时指定的；可以是`libc`、`jemalloc`或者`tcmalloc`。`jemalloc`是**默认值**


## Redis内存划分
> Redis作为内存数据库，在内存中存储的内容主要是数据(键值对)
> 除了数据以外，Redis的其他部分也会占用内存

### 数据
> 作为数据库，数据是最主要的部分；这部分占用的内存会统计在used_memory中
> Redis使用键值对存储数据，其中的值(对象)包括5种类型：字符串、哈希、列表、集合、有序集合。这5种类型是Redis对外提供的，实际上，在Redis内部，每种类型可能有2种或者更多的内部编码实现；此外，Redis在存储对象时，并不是直接将数据扔进内存，而是会对对象进行各种包装。如redisObject、SDS等； 

### 进程内存
> Redis主进程本身运行肯定是需要占用操作系统内存的，如代码、常量池等；
> 进程内存大约几兆，在大多数生产环境中与Redis数据占用的内存相比可以忽略不计
> 这部分内存不是由jemalloc(或其他内存分配器)分配，因此不会统计在used_memory中

- 除了主进程外，Redis创建的子进程运行也会占用内存，如Redis执行AOF、RDB重写时创建的子进程。但是这部分内存不属于Redis进程，不会统计在used_memory和used_memory_rss中

### 缓冲内存
> 缓冲内存包括客户端缓冲区、复制积压缓冲区、AOF缓冲区等；这部分内存有jemalloc分配，因此会统计在used_memory中
- 客户端缓冲：存储客户端连接的输入输出缓冲；
- 复制积压缓冲：用于部分复制功能
- AOF缓冲区：在进行AOF重写时，保存最近的写入命令


### 内存碎片
> 内存碎片是Redis在分配、回收物理内存过程中产生的。例如，对数据的频繁更改且数据之间的大小相差很大，可能导致redis释放的空间在物理内存中并没有释放，但redis又无法有效利用，这就形成了内存碎片。内存碎片不会统计在used_memory中

> 内存碎片的产生与对数据进行的操作、数据的特点等都有关；此外，与使用的内存分配器也有关系：如果内存分配器设计合理，可以尽可能的减少内存碎片的产生

> 如果Redis服务器的内存碎片已经很大，可以通过安全重启的方式减少内存碎片：因为重启之后，Redis重新从备份文件中读取数据，在内存中进行重排，为每个数据重新选择合适的内存单元，减少内存碎片

## Redis数据存储
> Redis 是一个k-V NOSQL非关系型数据库
> Redis 有5种数据类型：String、hash、list、set、zset

- dictEntry：Redis是Key-Value数据库，因此对每个键值对都会有一个dictEntry，里面存储了指向Key和Value的指针；next指向下一个dictEntry
- Key：Key并不是直接以字符串存储，而是存储在SDS结构中
- RedisObject：Value既不是直接以字符串存储，也不是像Key一样直接存储在SDS中，而是存储在redisObject中。实际上，不论Value是5种类型的哪一种，都是通过redisObject来存储的；而redisObject中的type字段指明了Value对象的类型，ptr字段则指向对象所在的地址。实际上，redisObject除了type和ptr字段之外，还有其他字段，如用于指定对象内部编码的字段
- jemalloc：无论是DictEntry对象、还是redisObject、SDS对象，都需要内存分配器分配内存进行存储。以DictEntry对象为例，有3个指针组成，在64位机器下占24个字节，jemalloc会为它分配32个字节大小的内存单元

### jemalloc
> jemalloc作为Redis的默认内存分配器，在减少内存碎片方面做的相对比较好。jemalloc在64位系统中，将内存空间划分为小、大、巨大三个范围；每个范围内又划分许多小的内存块单元；当Redis存储数据时，会选择大小最合适的内存块进行存储

> jemalloc划分的内存单元如下图

![redis_jemalloc_memory.jpg](/images/redis_jemalloc_memory.jpg)

> 例如，如果需要存储大小为300字节的对象，jemalloc会将其放入320字节的内存单元中

### redisObject  <span style="color:red;">16字节</span>
> redis的5种类型，无论是哪种类型，Redis都不会直接存储，而是通过redisObject对象进行存储
> Redis对象的类型、内部编码、内存回收、共享对象等功能，都需要redisObject支持

> redisObject数据结构如下
```C
typedef struct redisObject {
  //类型
  unsigned type:4;
  //编码
  unsigned encoding:4;
  //
  unsigned lru:LRU_BITS;
  //引用技术
  int refcount;
  //指向底层实现的指针
  void *ptr;
} robj;
```

#### type 
> 表示对象的类型，占<span style="color:red;">4个比特</span>；目前包括`REDIS_STRING`(字符串)、`REDIS_LIST`(列表)、`REDIS_HASH`(哈希)、`REDIS_SET`(集合)、`REDIS_ZSET`(有序集合)

> type值：
```C
#define OBJ_STRING 0 /* String object */
#define OBJ_LIST 1   /* List object */
#define OBJ_SET 2    /* Set object */
#define OBJ_ZSET 3   /* Sorted set object */
#define OBJ_HASH 4   /* Hash object */
```

> 在客户端(如redis-cli)可使用type命令，读取redisObject的type字段获取对象的类型
> 命令： `type key值`

#### encoding
> 表示对象的内部编码，占<span style="color:red;">4个比特</span>

> encoding值：
```C
#define OBJ_ENCODING_RAW 0 /* Raw representation */
#define OBJ_ENCODING_INT 1 /* Encoded as integer */
#define OBJ_ENCODING_HT 2 /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3 /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6 /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7 /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8 /* Encoded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

> 对于Redis支持的每种类型，都有至少两种内部编码，例如对于**字符串**，有`int`、`embstr`、`raw`三种编码。通过encoding属性，Redis可以根据不同的使用场景来为对象设置不同的编码，大大提高了Redis的灵活性和效率。以**列表**对象为例，有**压缩列表**(元素个数小于<span style="color:red;">512个</span>)、双端链表两种编码方式；如果列表的元素较少，Redis倾向于使用压缩列表进行存储，因为压缩列表占用内存更少，而且比双端链表可以更快载入；当列表对象元素较多时，压缩列表就会转化为更适合存储大量元素的双端链表
> 命令： `object encoding key值`，可以读取`redisObject`的编码

#### lru
> lru记录的是对象最后一次被命令程序访问的时间，占据的比特数不同版本有所不同(如4.0版本占<span style="color:red;">24比特</span>，2.6版本占<span style="color:red;">22比特</span>)

> 通过对比`lru`时间与当前时间，可以**计算**某个对象的**闲置时间**
> 命令：`object idletime key值`可以显示某个redisObject的闲置时间(单位：**秒**)；`object idletime`命令不会改变lru的值

- `lru`值除了通过`object idletime`命令打印之外，还与Redis的内存回收有关系；如果Redis打开了`maxmemory`选项，且内存回收算法选择的是`volatile-lru`或`allkeys-lru`，那么当Redis内存占用超过`maxmemory`指定的值，Redis会优先选择空转时间最长的对象进行释放

#### refcount
> `refcount`记录的是该对象被引用的次数，类型为**整形**。`refcount`的**作用**，主要在于对象的**引用计数**和**内存回收**。当创建新对象时，`refcount`初始化为**1**；当有新程序使用该对象时，refcount加1；当对象不再被一个新程序使用时，refcount减1；当refcount变为0时，对象占用的内存会被释放

> Redis中被多次使用的对象(<span style="color:blue;">refcount>1</span>)，称为**共享对象**。Redis为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的对象。这个被重复使用的对象，就是共享对象。目前共享对象**仅支持整数值的字符串对象**.
  - 之所以只支持整数值的字符串，实际上是对**内存和CPU(时间)的平衡**：共享对抗虽然降低内存消耗，但是判断两个对象是否相等却需要消耗额外的时间。对于整数值，判断操作复杂度为O(1);对于普通字符串，判断复杂度为O(n);而对于哈希、列表、集合、有序集判断的复杂度为O(n^2).
  - 虽然共享对象只能是整数值的字符串对象，但是5种类型都可能使用共享对象(如哈希、列表等的元素都可以使用)
  - 就目前的实现，Redis服务器在初始化时，会创建**10000**个字符串对象，值分别是0～9999的整数值；当Redis需要使用值为0～9999的字符串对象时，可以直接使用这些共享对象。10000这个数字可以通过调整参数REDIS_SHARED_INTEGERS(4.0版本是OBJ_SHARED_INTEGERS)的值进行改变

> 命令：`object refcount key值`可以查看共享对象的**引用次数**


#### ptr
> ptr指针指向具体的数据

#### 总结
> redisObject的结构与对象类型、编码、内存回收、共享对象都有关系；一个`redisObject`对象的**大小为<span style="color:red;">16字节</span>**：*4bit+4bit+24bit+4Byte+8Byte=16Byte*

### SDS
> Redis没有直接使用C字符串作为默认的字符串表示，而是使用了SDS(**Simple Dynamic String**)

#### SDS数据结构
```C
struct sdshdr{
  // 记录buf数组中已使用的字节数量，等于sds保存字符串的长度
  int len;

  // 记录buf中未使用的字节数量
  int free;

  //字节数据
  char buf[];
}
```
> 通过SDS的结构可以看出，buf数组的长度 = free + len + 1(其中1表示字符串结构的空字符)；所以，一个**SDS结构占据的空间**为：free所占长度+len所占长度+buf数组的长度= 4 + 4 + free + len + 1 = <span style="color:red;">free + len + 9</span>

#### SDS与C字符串的比较
> SDS 在C字符串的基础上加入了free和len字段，带来了很多好处

- 获取字符串长度： SDS是O(1),C字符串是O(n)
- 缓冲区溢出：使用C字符串的API时，如果字符串长度增加(如strcat操作)而忘记重新分配内存，很容易造成缓冲区溢出；而SDS由于记录了长度，相应的API在可能造成缓冲区溢出时会自动重新分配内存，杜绝了缓冲区溢出
- 修改字符串时的内存重分配：对于C字符串，如果要修改字符串，必须要重新分配内存(先释放再申请)，因为如果没有重新分配，字符串长度增加时会造成内存缓冲区溢出，字符串长度减少时会造成内存泄露。而对于SDS，由于可以记录len和free，因此接触了字符串长度和空间数组长度之间的关联，可以在此基础上进行优化：空间预分配策略(即分配内存时比实际需要的多)使得字符串长度增大时重新分配内存的概率大大减少；惰性空间释放策略使得字符串长度减少时重新分配内存的概率大大减小。
- 存取二进制数据：SDS可以，C字符串不可以。因为C字符串以空字符串作为字符串结束的标识，而对于一些二进制文件(如图片)，内容可能包含空字符串，因此C字符串无法正确获取；而SDS以字符串长度len来作为字符串结束标识，因此没有这个问题。

> 由于SDS中的buf仍然使用了C字符串(即以‘\0’结尾)，因此SDS可以使用C字符串库中的部分函数；但是需要注意的是，只有当SDS用来存储文本数据时才可以这样使用，在存储二进制数据时则不行

#### SDS与C字符串的应用
> Redis在存储对象时，一律使用SDS代替C字符串

> eg: set hello world命令，hello和world都是以SDS的形式存储的
> eg: sadd myset m1 m2 m3命令，无论是键myset，还是集合中的元素(m1,m2,m3)，都是以SDS的形式存储
> 除了存储对象，SDS还用于存储各种缓冲区

> 只有在字符串不会改变的情况下，如打印日志时，才会使用C字符串

## Redis的对象类型与内部编码
> 前面已经说过，Redis支持5种对象类型，而每种结构都有至少两种编码
> 这样做的好处
  - 接口和实现分离，当需要增加或改变内部编码时，用户使用不受影响
  - 可以根据不同的应用场景切换内部编码，提高效率

> Redis各种对象类型支持的内部编码如下表(只列出重点)：

|类型|编码|Object encoding 命令输出|对象|
|----|----|-----------|------------|
|REDIS_STRING|REDIS_ENCODING_INT|"int"|使用整数值实现的字符串对象|
|REDIS_STRING|REDIS_ENCODING_EMBSTR|"embstr"|使用embstr编码的简单动态字符串实现的字符串对象|
|REDIS_STRING|REDIS_ENCODING_RAW|"raw"|使用简单动态字符串实现的字符串对象|
|REDIS_LIST|REDIS_ENCODING_ZIPLIST|"ziplist"|使用压缩列表实现的列表对象|
|REDIS_LIST|REDIS_ENCODING_LINKEDLIST|"linkedlist"|使用双端链表实现的列表对象|
|REDIS_HASH|REDIS_ENCODING_ZIPLIST|"ziplist"|使用压缩列表实现的哈希对象|
|REDIS_HASH|REDIS_ENCODING_HT|"hashtable"|使用字典实现的哈希对象|
|REDIS_SET|REDIS_ENCODING_INTSET|"intset"|使用整数集合实现的集合对象|
|REDIS_SET|REDIS_ENCODING_HT|"hashtable"|使用字典实现的集合对象|
|REDIS_ZSET|REDIS_ENCODING_ZIPLIST|"ziplist"|使用压缩列表实现的有序集合对象|
|REDIS_ZSET|REDIS_ENCODING_SKIPLIST|"skiplist"|使用跳跃列表和字典实现的有序集合对象|

> Redis内部编码的转换，符合如下规律：<span style="color:red">编码转换在Redis写入数据时完成，且转换过程不可逆，只能从小内存编码向大内存编码转换</span>

### 字符串
> 字符串是最基础的类型，因为所有的键都是字符串类型，且字符串之外的其他复杂类型的元素也是字符串
> 字符串长度*不能超过* **512MB**

#### 内部编码
> 字符串类型的内部编码有3中，它们的应用场景如下：
- **int**：8个字节的长整形。字符串值是整形时，这个值使用long整形表示
- **embstr**：<=44字节的字符串。`embstr`与`raw`都使用**redisObject和SDS保存数据**，区别在于`embstr`的使用只**分配一次**内存空间(因此redisObject和SDS是连续的)，而`raw`需要**分配两次**内存空间(分别为redisObject和SDS分配空间)。因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而`embstr`的**不足**也很明显，如果字符串的长度需要**重新分配内存**时，整个`redisObject`和`SDS`都需要重新分配空间，因此redis中的**embstr实现为只读**
- **raw**：大于44个字节的字符串

> `embstr`和`raw`进行**区分**的长度是<span style="color:red;">39</span>；因为`redisObject`的长度是**16**，`SDS`的长度是**9+字符串长度**，因此当字符串长度是**39**时，embstr的长度正好是**16+9+39=64**，`jemolloc`正好可以**分配64字节**的内存单元

#### 编码转换
> 当`int`数据不再是整数，或大小超过了long的范围时，自动转化为`raw`
> 而对于`embstr`，由于其实现是**只读**的，因此在对`embstr`对象进行**修改**时，都会**转化为raw**再进行修改，因此，只要是修改embstr对象，修改后的对象一定是raw，无论是否达到了39个字节

### 列表(list)
> 列表用来存储多个有序的字符串，每个字符串称为元素
> 一个列表可以存储**2^32-1**个元素
> Redis中的列表支持两端插入和弹出，并可以获得指定位置(或范围)的元素，可以充当数组、队列、栈等

#### 内部编码
> 列表的内部编码可以是**压缩列表**(ziplist)或**双端链表**(linkedlist)

##### 双端链表：
- 同时保存了表头指针、表尾指针，并且每个节点都具有指向前和指向后的指针
- 链表中保存了列表的长度
- dup、free和match为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值
- 链表中每个节点指向的是type为字符串的redisObject

##### 压缩列表
- 压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块(而不是像双端链表一样每个节点是指针)组成的顺序型数据结构

#### 编码转化
> 只有**同时满足**下面两个条件时，才会使用**压缩列表**

- 列表中的*元素数量* **小于**<span style="color:red;">512</span>个
- 列表中**所有**字符串对象都**不足**<span style="color:red;">64</span>字节

### 哈希
> 哈希(一种数据结构)，不仅是Redis对外提供的5种对象类型的一种(字符串、列表、集合、有序集合、哈希)，也是Redis作为Key-Value数据库所使用的数据结构。为了说明的方便，后面当使用"**内层的哈希**"时，代表的是Redis对外提供的5种对象类型的一种；使用"**外层的哈希**"代指Redis作为Key-Value数据库所使用的数据结构

#### 内部编码
> **内层的哈希**使用的内部编码可以是压缩列表(ziplist)和哈希表(hashtable)两种；Redis**外层的哈希**则只使用了hashtable。

##### hashtable
> 一个hashtable由1个`dict`结构、2个`dictht`结构、1个`dictEntry`指针数组(称为bucket)和多个`dictEntry`结构组成.如下图(hashtable没有进行refresh)

![redis_hashtable](/images/redis_hashtable.jpeg)

> `dictEntry`结构 （在64位系统中，一个`dictEntry`对象占<span style="color:red;">24</span>字节(key/val/next各占8字节)）

```C
typedef struct dictEntry{
  void *key;
  union{
    void *val;
    uint64_tu64;
    int64_ts64;
  }v;
  struct dictEntry *next;
}dictEntry;
```

- **key**：键值对中的键
- **val**：键值对中的值，使用union(即共用体)实现，存储的内容即可能是一个指向值的指针，也可能是64位整形，或无符号64位整形
- **next**：指向下一个dictEntry，用于解决哈希冲突问题

> `bucket` 是一个数组，数组的每个元素都指向`dictentry`结构的指针。Redis中的bucket数组的大小计算规则如下：**大于** *dictEntry的数量*的**最小**的<span style="color:red;">2^n</span>;
> eg：如果有1000个dictEntry，那么bucket大小为1024；如果有1500个dictEntry，则bucket大小为2048

> dictht结构

```C
typedef struct dictht{
  dictEntry **table;
  unsigned long size;
  unsigned long sizemask;
  unsigned long used;
}dictht;
```

- **table**：指针，指向bucket
- **size**：记录了哈希表的大小，即bucket的大小
- **used**：记录了已使用的dictEntry的数量
- **sizemask**：值总是为**size-1**，这个属性和哈希值一起决定一个键在table中的存储位置

> `dict`。一般情况下通过使用dictht和dictEntry结构，便可以实现普通哈希表的功能，但是Redis的实现中，在dictht结构的上层，还有一个dict结构

```C
typedef struct dict{
  dictType *type;
  void *privdata;
  dictht ht[2];
  int trehashidx;
} dict;
```

- **type、privdata**：为了适应不同类型的键值对，用于创建多态字典
- **ht、trehashidx**：用于rehash。即当哈希表需要扩展或者收缩时使用。ht是一个包含两个项的数组，每项都指向一个dictht结构，这也是Redis的哈希会有1个dict、2个dictht结构的原因。通常情况下，所有的数据都是存放在dict的ht[0]中，ht[1]只有在rehash的时候使用。dict进行**rehash**操作的时候，将ht[0]中的素有数据rehash到ht[1]中。然后将ht[1]赋值给ht[0]，并情况ht[1].

#### 编码转换
> 内层的哈希只有在同时满足以下2个条件时，才会使用压缩列表

- 哈希中的元素数量**小于**<span style="color:red;">512</span>个
- 哈希中所有键值对的**键和值**字符串长度都**小于**<span style="color:red;">64</span>字节 

### 集合
> 集合(set)与列表类似，都是用来保存多个字符串，但是集合与列表有两点不同：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能重复。
> 一个集合中最多可以存储**2^32-1**个元素；除了支持常规的增删改查，Redis还支持多个集合取**交集、并集、差集**。

> 下面是整数集合结构

```C
typedef struct intset{
  uint32_t encoding;
  uint32_t length;
  int8_t contents[];
} intset;
```

- **encoding**：contents中存储内容的类型，虽然contents(存储集合中的元素)是int8_t类型，但实际上存储的值是int16_t、int32_t或者int64_t,具体的类型由encoding决定
- **length**：元素个数

#### 编码转换
> 只有同时满足下面两个条件，集合才会使用整数集合

- 集合中元素数量小于<span style="color:red;">512</span>个
- 集合中所有元素都是整数值

### 有序集合
> 有序集合与集合一样，元素都不能重复；但与集合不同的是，有序集合中的元素是有顺序的。与列表使用索引下标作为排序依据不同，有序集合为每个元素设置一个分数(score)作为排序依据。

#### 内部编码
> 有序集合的内部编码可以是**压缩列表**(ziplist)或**跳跃表**(skiplist)
> 跳跃表是一种有序数据结构，通过在每个节点维持多个指向其他节点的指针，从而达到快速访问节点的目的

#### 编码转换
> 只有同时满足以下2个条件，才会使用压缩列表

- 元素数量小于<span style="color:red;">128</span>个
- 所有成员长度不足<span style="color:red;">64</span>字节