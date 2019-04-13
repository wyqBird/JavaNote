# 一、概述

Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。

键的类型只能为字符串，值支持五种数据类型：字符串、列表、集合、散列表、有序集合。

Redis 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。

# 二、数据类型简介

string：

- 可存储的值：字符串、整数或者浮点数
- 操作：对整个字符串或者字符串的其中一部分进行操作；对整数和浮点数执行自增或者自减操作

list：

- 可存储的值：列表
- 操作：从两端压入或者弹出元素；根据偏移量进行修剪，保留某范围内元素；读取单个或者多个元素；按值查找或删除元素

set：

- 可存储的值：无序集合
- 操作：添加、获取、移除单个元素；检查一个元素是否存在于集合中；从集合里面随机获取元素；计算交集、并集、差集

hash：

- 可存储的值：包含键值对的无序散列表
- 操作：添加、获取、移除单个键值对；获取所有键值对；检查某个键是否存在

zset：

- 可存储的值：有序集合
- 操作：添加、获取、删除元素；根据分值范围或者成员来获取元素；计算一个键的排名

> [What Redis data structures look like](https://redislabs.com/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/1-2-what-redis-data-structures-look-like/)

## 1.STRING
结构：
<div align="center"> <img src="../pics//RIA_fig1-01.png" width="400"/> </div><br>

命令：

- get：获取指定 key 中的数据
- set：设置指定 key 中的值
- del：删除指定 key 中的值（适用于所有类型）

例子：
```html
> set hello world
OK
> get hello
"world"
> del hello
(integer) 1
> get hello
(nil)
```

## 2.LIST
结构：
<div align="center"> <img src="../pics//RIA_fig1-02.png" width="400"/> </div><br>

命令：

- rpush：将值插入到列表的右端
- lrange：从列表中获取所有值
- lindex：获取列表中给定位置的值
- lpop：弹出列表左端的值并返回它
 
例子：
```html
> rpush list-key item
(integer) 1
> rpush list-key item2
(integer) 2
> rpush list-key item
(integer) 3

> lrange list-key 0 -1
1) "item"
2) "item2"
3) "item"

> lindex list-key 1
"item2"

> lpop list-key
"item"

> lrange list-key 0 -1
1) "item2"
2) "item"
```

## 3.SET
结构：
<div align="center"> <img src="../pics//RIA_fig1-03.png" width="400"/> </div><br>

命令：

- sadd：把元素添加到集合中
- smembers：返回集合中所有元素
- sismember：检查元素是否在集合中
- srem：如果元素在集合中，则移除它

例子：
```html
> sadd set-key item
(integer) 1
> sadd set-key item2
(integer) 1
> sadd set-key item3
(integer) 1
> sadd set-key item
(integer) 0

> smembers set-key
1) "item"
2) "item2"
3) "item3"

> sismember set-key item4
(integer) 0
> sismember set-key item
(integer) 1

> srem set-key item2
(integer) 1
> srem set-key item2
(integer) 0

> smembers set-key
1) "item"
2) "item3"
```

## 4.HASH
结构：
<div align="center"> <img src="../pics//RIA_fig1-04.png" width="400"/> </div><br>

命令：

- hset：存储键值对
- hget：获取指定 key 的值
- hgetall：获取哈希表中所有键值对
- hdel：删除指定键值对（如果 key 存在）

例子：
```html
> hset hash-key sub-key1 value1
(integer) 1
> hset hash-key sub-key2 value2
(integer) 1
> hset hash-key sub-key1 value1
(integer) 0

> hgetall hash-key
1) "sub-key1"
2) "value1"
3) "sub-key2"
4) "value2"

> hdel hash-key sub-key2
(integer) 1
> hdel hash-key sub-key2
(integer) 0

> hget hash-key sub-key1
"value1"

> hgetall hash-key
1) "sub-key1"
2) "value1"
```

## 5.ZSET
结构：
<div align="center"> <img src="../pics//RIA_fig1-05.png" width="400"/> </div><br>

命令：

- zadd：添加元素
- zrange：按存储顺序获取所有元素
- zrangebyscore：按分值顺序获取所有元素
- zrem：移除元素（如果它存在）

例子：
```html
> zadd zset-key 728 member1
(integer) 1
> zadd zset-key 982 member0
(integer) 1
> zadd zset-key 982 member0
(integer) 0

> zrange zset-key 0 -1 withscores
1) "member1"
2) "728"
3) "member0"
4) "982"

> zrangebyscore zset-key 0 800 withscores
1) "member1"
2) "728"

> zrem zset-key member1
(integer) 1
> zrem zset-key member1
(integer) 0

> zrange zset-key 0 -1 withscores
1) "member0"
2) "982"
```

# 三、数据结构深入

## 1.简单动态字符串
### 1.1 SDS定义

```c
struct sdshdr {

    // 字符串长度
    int len;

    // buf 中未使用字节的数量
    int free;

    // 字节数组，为什么是字节数组？为保存二进制数据
    char buf[];
};
```

- SDS遵循C字符串以空字符结尾的惯例，但是那1个字节不计算在len中。
- 为空字符分配额外的 1 字节空间，以及添加空字符到字符串末尾等操作，都是有SDS函数自动完成。
- 遵循C字符串以空字符结尾的惯例：SDS可直接重用一部分C字符串函数库中的函数。

### 1.2 SDS和C字符串的区别
C语言使用的这种简单的字符串表示方式，并不能满足 Redis 对字符串在安全性、效率以及功能方面的要求。
#### 常数复杂度获取字符串长度
问题：

- C语言如果要获取字符串的长度，需要从第一个字符开始，遍历整个字符串，直到遍历到\0符号，时间复杂度是O(N)，即字符串的长度。

解决：

- SDS在len属性中记录了SDS本身长度，所以获取SDS长度的复杂度仅为O(1).

分析：

- 空间换时间：用一个字段存储长度，将获取长度的时间复杂度由 O（N）降低到 O（1）.

好处：

- 即使我们对一个非常长的字符串键反复执行 strlen 命令，也不会对系统性能造成任何影响，因为该命令的复杂度仅为 O（1）.

#### 杜绝缓冲区溢出
问题：

- C字符串不记录自身长度，容易造成缓冲区溢出。
- eg：使用字符串拼接等方式时，就很容易出现此问题。而如果每次拼接之前都要计算每个字符串的长度，时间上又要耗费很久。

解决：

- redis在执行操作之前，其会先检查空间是否足够。如果free的值不够，会再申请内存空间，避免溢出。
#### 减少内存分配次数
问题：

- C语言的字符串长度和底层数组之间存在关联（包含N个字符的字符串的空间是N+1个字符长的数组），因此总要进行内存重分配操作：字符串长度增加时，需要再分配存储空间，避免溢出；字符串长度减少时，需要释放存储空间，避免内存泄漏。

解决：

- redis的SDS，主要是通过free字段，来进行判断。
- 通过未使用空间大小，实现了空间预分配和惰性空间释放。

##### 空间预分配:

- 当需要增长字符串时，sds不仅会分配足够的空间用于增长，还会预分配未使用空间。
- 若SDS修改后，SDS的长度len < 1MB，则分配和 len 一样大小的未使用空间，即len=free.
- 若SDS修改后，len >= 1MB，则分配 1MB 的未使用空间，即free = 1MB。
- 通过该策略，redis 可减少连续执行字符串增长操作时所需的内存重分配次数。
- 每次字符串增长之前，sds会先检查空间是否足够，如果足够则直接使用预分配的空间，否则按照上述机制申请使用空间。

好处：

- 通过空间预分配策略，sds 将连续增长 N 次字符串所需的内存重分配次数从必定 N 次降低为 最多 N 次。 

##### 惰性空间释放
- 懒惰空间释放用于优化sds字符串缩短的操作
- 当需要缩短sds的长度时，并不立即释放空间，而是使用free来保存剩余可用长度，并等待将来使用。
- 当有剩余空间，而又有增长字符串操作时，则又会调用空间预分配机制。
- 当redis内存空间不足时，会自动释放sds中未使用的空间，因此也不需要担心内存泄漏问题。
#### 二进制安全
- SDS 的 API 都是二进制安全的
- 所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 buf 数组里的数据
- 程序不会对其中的数据做任何限制、过滤、或者假设
- 数据在写入时是什么样的， 它被读取时就是什么样。
- sds判断字符串结束，是通过len属性，而不是通过\0来判断。
- sds的buf属性设置为字节数组的原因：redis 不是用这个数组来保存字符，而是用来保存一系列二进制数据。
#### 兼容部分C语言字符串函数

- redis兼容c语言对于字符串末尾采用\0进行处理，这样使得其可以复用部分c语言字符串函数的代码，实现代码的精简性。
### 1.3 SDS API
## 2.链表
## 3.字典
## 4.跳跃表
## 5.整数集合
## 6.压缩列表
## 7.对象

# 参考资料

- 黄健宏. Redis 设计与实现 [M]. 机械工业出版社, 2014.
