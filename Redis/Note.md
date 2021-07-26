### Redis是什么

remote dictionary server

dictionary
	kv，键值对，与map类似
server
	作为第三方提供服务，支持分布式
简单的set get 命令介绍

redis还支持哨兵模式，集群，发布、订阅，高可用，高扩展，数据可存磁盘

redis的性能  11W/s读 8W/s写，机器性能

开源 BSD协议

内存型数据库 NoSql

5种基本数据结构 string、list、hash、set、zset
string基本命令 append strlen setrange setnx

### 常用操作

### SDS

#### 什么是SDS？

SDS全称为简单动态字符串（**simple dynamic string**），是Redis封装出的一种数据结构类型，主要用于存储Redis的默认字符串表示。

#### SDS有什么优点？

Redis为什么不直接使用C的字符串？

#### SDS的实现

3.2之前，一种

```c
struct sdshdr {
    // buf 中已占用空间的长度
    int len;
    // buf 中剩余可用空间的长度
    int free;
    // 数据空间
    char buf[];
};
```

3.2之后，区分了5种sds的类型：

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

```c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

```c
typedef char *sds;
```

不同长度的字符串可以使用不同大小的header，



Header部分主要包含以下几个部分：

- len：表示字符串真正的长度，不包括空终止字符
- alloc：表示字符串的最大容量，不包含Header和最后的空终止字符
- flags：表示Header的类型



节省内存，```__attribute__ ((__packed__))```告诉编译器取消字节对齐，方便的获取flags

由于sds的header共有五种，要想得到sds的header属性，就必须先知道header的类型，flags字段存储了header的类型。假如我们定义了sds* s，那么获取flags字段仅仅需要将s向前移动一个字节，即unsigned char flags = s[-1]。



set key value

key，value分别是怎么存储



#### Redis SDS与C语言比较

**1）获取字符串长度的时间复杂度**

SDS获取字符串长度：O(1)。

C字符串获取字符串长度时间复杂度为O(N),需要遍历字符串，以空字符为结尾。

使用SDS可以确保获取字符串长度的操作不会成为Redis的性能瓶颈。

**2）杜绝缓冲区溢出**

C字符串不记录自身长度和空闲空间，容易造成缓冲区溢出，使用SDS则不会，SDS拼接字符串之前会先检测剩余空间能否满足需求，不能满足需求的就会扩容。

**3）减少修改字符串时带来的内存重分配次数**

使用C字符串的话：

每次对一个C字符串进行增长或缩短操作，长度都需要对这个C字符串数组进行一次内存重分配，比如C字符串的拼接，程序要先进行内存重分配来扩展字符串数组的大小，避免缓冲区溢出，又比如C字符串的缩短操作，程序需要通过内存重分配来释放不再使用的那部分空间，避免内存泄漏，所以C语言中每次修改字符串都会造成内存重分配。

使用SDS的话：

通过SDS的len属性和free属性可以实现两种内存分配的优化策略：空间预分配和惰性空间释放。

1.针对内存分配的策略：空间预分配（SDS字符串扩容操作）

在对SDS的空间进行扩展的时候，程序不仅会为SDS分配修改所必须的空间，还会为SDS分配额外的未使用的空间

这样可以减少连续执行字符串增长操作所需的内存重分配次数，通过这种预分配的策略，SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次，这是个很大的性能提升。

额外分配未使用空间的大小由以下策略决定：
在扩展sds空间之前，sds api会检查未使用的空间是否够用，如果够用则直接使用未使用的空间，无须执行内存重分配。

如果空间不够用则执行内存重分配：

![img](https://img2020.cnblogs.com/blog/1477786/202006/1477786-20200606160640123-1895539316.jpg)

2.针对内存释放的策略：惰性空间释放（SDS字符串缩短操作）

在对SDS的字符串进行缩短操作的时候，程序并不会立刻使用内存重分配来回收缩短之后多出来的字节，而是使用free属性将这些字节的数量记录下来等待将来使用。

通过惰性空间释放策略，SDS避免了缩短字符串时所需的内存重分配次数，并且为将来可能有的增长操作提供了优化！

当然如果我们在有需要的时候，也可以通过sds api来释放未使用的空间，不用担心惰性空间释放策略会造成内存浪费。

**4）二进制安全**

为了确保数据库可以二进制数据（图片，视频等），SDS的API都是二进制安全的，所有的API都会以处理二进制的方式来处理存放在SDS的buf数组里面的数据，程序不会对其中的数据做任何的限制，过滤，数据存进去是什么样子，读出来就是什么样子，这也是buf数组叫做字节数组而不是叫字符数组的原因，以为它是用来保存一系列二进制数据的。

通过二进制安全的SDS，Redis不仅可以保存文本数据，还可以保存任意格式是二进制数。

而C语言字符串的字符必须符号某种编码(比如ascii)，并且除了末尾的空字符，字符串其他位置不能包含空字符，所以C语言字符串只能保存文本数据，不能保存二进制数据。

**5）兼容部分c语言函数**



sds优缺点