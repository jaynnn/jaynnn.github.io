---
layout: post
title: redis sorted set 原理
date: 2024-07-10 20:33:00
description: The implement of redis sorted set.
tags: redis computer-science algorithm
categories: tech
---

## 目录
1. [前言](#前言)
2. [ziplist && listpack](#ziplist&listpack)
    1. [ziplist](#ziplist)
    2. [listpack](#listpack)
3. [skiplist+hash](#skiplist&hash)
4. [参考](#参考)

## 前言
* 在数据元素较少的情况下，redis采用了ziplist(版本7.0之前)和listpack(版本7.0之前)的方式，这种方式相比普通的双向链表，能够保证内存更加有序(内存碎片减少)且节省(每个元素少存了指向前后节点的指针)。在数据元素较多的情况下，采用跳跃表加哈希表的方式。


## ziplist&listpack
#### ziplist
1. ziplist 就是一种空间紧凑的双向队列。  

优势：  
1. 内存利用率极高，在双端插入有O(1)的性能，在中间插入和查找有O(n)的性能，别看仅是O(n)，由于内存的连续，充分利用了CPU缓存，在小规模数据中有非常好的性能优势。 

缺点：
1. 在于每次插入、删除操作都需要对内存进行调整，性能与内存用量相关。
2. 由于每个元素都会保存上一个元素的长度，且这个长度本身占用的内存是会发生变化的，当一个元素的长度发生变化，可能会导致后续的节点长度发生变化，从而引发级联更新(就是对元素A的操作引发了其他元素的变化)。  

源码中对ziplist的解释：  
```
 The ziplist is a specially encoded dually linked list that is designed to
  be very memory efficient. It stores both strings and integer values, 
  where integers are encoded as actual integers instead of a series of 
  characters. It allows push and pop operations on either side of the list
   in O(1) time. However, because every operation requires a reallocation 
   of the memory used by the ziplist, the actual complexity is related to 
   the amount of memory used by the ziplist.

 ** 翻译 **：
 ziplist是一种特殊编码的双链表，它是为了高内存效率而设计的。它存储字符串和整数
 值，其中整数被编码为实际整数而不是一系列字符。它允许在列表的两侧进行在 O(1) 时间
 复杂度的推入和弹出操作。然而，由于每次操作都需要重新分配ziplist使用的内存，实际
 复杂度与ziplist使用的内存量相关。
```

2. 内存布局
```
ziplist
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

entry
<prevlen> <encoding> <entry-data>
``` 

zlbytes 存储了整个表的内存占用，包含了其本身占用的四个字节。这个字段可以用来避免遍历从而重新调整整个结构的大小。  

zltail 存储了最后一个元素的偏移量，用来弹出最后一个元素而避免遍历。  

zllen 存储了元素的数量，注意这个字段只有两个字节大小，也就是超过2^16-2个元素个数时，该值会设置成2^16-1并且从此获取该值需要遍历所有元素。（这也是ziplist不适合数据元素较多的情形的原因之一。）  

zlend 是特殊的元素，代表着元素列表的末尾。  

entry 由三部分组成。 prevlen 代表着前一个元素的长度，用来从后向前遍历。encoding 存储着元素的类型。entry-data 存储着具体的元素数据（对于小整数可能会没有这个部分）。
* 对于 prevlen 还要分两种情况，当 prevlen < 254, 则用一个字节来表示；如果 prevlen >= 254 则用 0xFE+四个字节 来表示。
* 对于 encoding 源码中的图示很清晰，这里借用一下：

```
|00pppppp| - 1 byte
    字符串类型，长度 <= 63 字节（6位）。
|01pppppp|qqqqqqqq| - 2 bytes
    字符串类型，长度 <= 16383 字节（14位）。
    IMPORTANT: The 14 bit number is stored in big endian.
|10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
    字符串类型，长度 >= 16384 字节（32位，第一个字节的多余位数不再使用）。
    IMPORTANT: The 32 bit number is stored in big endian.
|11000000| - 3 bytes
    整型，存储 int16_t (2 bytes)
|11010000| - 5 bytes
    整型，存储 int32_t (4 bytes)
|11100000| - 9 bytes
    整型，存储 int64_t (8 bytes)
|11110000| - 4 bytes
    整型，存储 24位无符号数 (3 bytes)
|11111110| - 2 bytes
    整型，存储 8位无符号数(1 byte)
|1111xxxx| - (xxxx 从 0001 到 1101) 四位整数，只能存 1~13，因为0000和1111已经被占用了。所以具体的数值需要忽略1111的情况下再减去1，比如十六进制 F1, 则代表的数位 1-1=0
|11111111| - 上文提到的 zlend.
注意：所有的整数都是以小端序存储。
```

具体的例子（不得不赞叹一下redis的文档实在详尽）：

```
以下是存储了两个数字 2 和 5 的例子，总共占用了15个字节：
[0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
      |             |          |       |       |     |
   zlbytes        zltail     zllen    "2"     "5"   end
可以看出，小端序下，[0f 00 00 00] 代表整个结构占用15个字节；[0c 00 00 00]代表第12个字节是最后一个元素，即"5"，；[02 00]代表元素数量为2；[00 f3]中，00代表前一个元素的长度，由于没有则为0，f3表示为|1111xxxx|，则3-1=2，存储的是整数2；[02 f6]的02代表前一个元素占用为2个字节，存储数位6-1=5；[ff]总是代表着结构的末尾。

另外一个entry例子：
[02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]
可以看出，前一个元素长度为2个字节，匹配的是|00pppppp|模式，即长度为11的字符串"Hello World"。
```


#### listpack
1. listpack 主要解决了 ziplist 中的级联更新的问题。
2. 内存布局
与 ziplist 差不多，主要区别在于 entry 的结构不同，不在存储上一个元素的大小，而是存储本元素的大小（注意entry-length一定是在空间布局的最后）。

```
<encoding> <entry-data> <entry-length>
对于 encoding
|0ppppppp| - 1 byte
    整型，0~127 （7位）。
|10pppppp| - 1 byte
    字符串类型，长度 0~63， entry-data存储字符串内容
|110ppppp|qqqqqqqq| - 2byte
    整型，-4096~4095
|1110pppp|qqqqqqqq| - 2byte
    字符串，长度 0~4095，entry-data存储字符串内容
1111|0000 <4 bytes len> <large string>，即32位长度字符串，后4字节为字符串长度，再之后为字符串内容
1111|0001 <16 bits signed integer>，即16位整型，后2字节为数据
1111|0010 <24 bits signed integer>，即24位整型，后3字节为数据
1111|0011 <32 bits signed integer>，即32位整型，后4字节为数据
1111|0100 <64 bits signed integer>，即64位整型，后8字节为数据
1111|0101 to 1111|1110 are currently not used.   当前不用编码
1111|1111 End of listpack，结尾标识
```

3. 既然移除了 prevlen 那么如何做到从后往前遍历呢？  
这是因为 listpack 特殊设计了 entry-length 结构，它的每个字节的第一位代表了是否为 entry-length 所占用空间的头部，当它需要从后往前遍历时，逐个字节往前找，如果第一位为0，说明已经到头了，当前搜寻的字节拼起来就是前一个元素的大小。  

以下是源码：
```
/* Decode the backlen and returns it. If the encoding looks invalid (more than
 * 5 bytes are used), UINT64_MAX is returned to report the problem. */
static inline uint64_t lpDecodeBacklen(unsigned char *p) {
    uint64_t val = 0;
    uint64_t shift = 0;
    do {
        val |= (uint64_t)(p[0] & 127) << shift;
        if (!(p[0] & 128)) break;
        shift += 7;
        p--;
        if (shift > 28) return UINT64_MAX;
    } while(1);
    return val;
}
```


## skiplist&hash
1. 当元素数量超过设定的阈值（max-listpack-entries/max-ziplist-entries）时，则使用跳表加哈希表的方式实现。跳表用来实现快速的插入、删除操作，而哈希表用来是先O(1)级别的查询操作。
    1. 跳表的实现可以用做地铁来表示，假设有地铁站A-B-C-D-E，那么假如我要去C，普通的做法是A-B-C，就是遍历找到。那么架设线路是多层设计的：
    A      ---      E
    A  ---  C  ---  E
    A - B - C - D - E
    那么就可以从A-E，发现坐过头了，再回到A-C，马上就找到了。在数据规模越大的情况下，这个查询优化越明显。另外，如果需要固定多少个节点就设置一层快捷车道，意味着如果新增元素就需要对原先的层级进行重新排布，这显然不够高效。所以redis的做法是让层数的搭建随机化，最高层数为32层，越高的层数随机到的概率越低，以对数递减的方式（第一层为100%，第二层位50%）。

    ![示例](https://p2-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/12/28/16f4ce73b1bf06e4~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)
    
    2. 哈希表是对跳表的查询效率进行优化，使之能达到O(1)的查询效率。

## 参考
[1] https://github.com/redis/redis  
[2] https://juejin.cn/post/7093530299866284045  
[3] https://juejin.cn/post/6844904033589657607  
