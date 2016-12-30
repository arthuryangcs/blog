---
title: Redis源码剖析（一） - 深入理解 dict 及 dictScan
date: 2016-12-10 11:06:27
tags:
  - Redis
  - 源码
  - dict
categories: Redis
---

第一篇关于 Redis 源码的文章。此系列并不是对 Redis 源码进行一个全面的讲解，而只是对我在阅读源码的过程中发现的比较有意思的地方做一个总结。

在此推荐一下黄健宏（huangz）的书 [《Redis 设计与实现》](http://redisbook.com/) ，书写的结构清晰，通俗易懂。我就是看完这本书后才萌生了阅读 Redis 源码的打算的。

我阅读源码期间的注释都会记录在 [arthuryangcs/redis-3.0-annotated](https://github.com/arthuryangcs/redis-3.0-annotated) 中。

<!-- more -->

## Redis 字典

Redis 中的字典基于 Hash 实现， 为了让 Hash 表能够自动扩展、缩小，每个字典（dict）中保存着两个哈希表（dictht），用来进行动态扩容。

### 数据结构

哈希表

``` c
   typedef struct dictht {

    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht; 

```

字典

``` c
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;
```

### rehash

Redis 的哈希表通过链地址法解决键冲突。随着对哈希表的操作不断进行，哈希表保存的键值会不断增加或减少，为了让哈希表的负载因子维持在一个合理的范围内，就需要对哈希表进行扩容或收缩。

字典的扩容和收缩操作基于字典中的 ht 元素，其中 ht[0] 保存原有字典中数据，当扩容操作进行时，ht[1] 中保存新的扩容或者收缩的字典。字典扩容操作不是一次性完成的，而是在对字典进行接下来的操作过程中分批完成的，其中rehashidx元素标记为已转移的哈希表元素个数。

哈希表的大小总保持为 \\( 2^n \\) 。

## dictScan

普通的哈希表遍历没有什么困难，但由于有 rehash 的存在，造成遍历的同时字典的结构有可能已经发生了改变，字典中元素的位置也会相应的变化。在这种情况下如何保证在遍历过程中，无论字典如何变化，遍历开始时字典中所有的元素都能够被遍历到就成了一个难题。

### 遍历游标

对此，Redis 字典采用了一套不同的游标遍历方式。一般的遍历都是从 0 到 n 进行遍历，下一个游标为上一个+ 1，而 Redis 字典中下一个游标的计算要相对复杂一下，首先对游标进行二进制翻转，然后 + 1，最后再对游标进行进行二进制翻转。这就相当于对游标最高位 + 1，然后反向进位。

由于哈希表的大小总是 \\( 2^n \\)，这就保证游标不会对首位 + 1 时不会超过表的大小。这个算法也保证了哈希表中所有元素都能获取到。

当哈希表进行放大或缩小时，如何保证所有元素都被遍历一遍呢？

首先，先通过 sizemask 获得当前哈希表的大小。然后再计算下一个游标值。比如，当前游标遍历到了 001，此时 sizemask 为 1111，显然表大小扩大了 2 倍，所以游标扩展到 0001， 则下一个节点为 1001。

当 rehash 中时我们先取小表，再取小表中相对应的所有大表中的节点，比如 001，小表 sizemask 为 111，大表为 1111，我们先取 小表中的 001，再取大表中的 0001、1001、0101、1101，然后返回下一个节点 101。

这样就保证了不管哈希表之前进行了多少次 rehash，或者现在是否在进行 rehash ，都不会对哈希表的扫描造成影响。此外，这个算法还有一个优点就是这个迭代器是完全无状态的，可在不需要任何额外内存的情况下运行。

这个算法有三个主要的缺点：

1. 函数可能会返回重复的元素，不过这个问题可以很容易在应用层解决。

2. 为了不错过任何元素，迭代器需要返回给定桶上的所有键，以及因为扩展哈希表而产生出来的新表，所以迭代器必须在一次迭代中返回多个元素。

3. 二维翻转需要通过循环实现，时间复杂度较高。

### dictScan 源码

``` c
unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,
                       void *privdata)
{
    dictht *t0, *t1;
    const dictEntry *de;
    unsigned long m0, m1;

    // 跳过空字典
    if (dictSize(d) == 0) return 0;

    // 迭代只有一个哈希表的字典
    if (!dictIsRehashing(d)) {

        // 指向哈希表
        t0 = &(d->ht[0]);

        // 记录 mask
        m0 = t0->sizemask;

        /* Emit entries at cursor */
        // 指向哈希桶
        de = t0->table[v & m0];
        // 遍历桶中的所有节点
        while (de) {
            fn(privdata, de);
            de = de->next;
        }

    // 迭代有两个哈希表的字典
    } else {

        // 指向两个哈希表
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        /* Make sure t0 is the smaller and t1 is the bigger table */
        // 确保 t0 比 t1 要小
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        // 记录掩码
        m0 = t0->sizemask;
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        // 指向桶，并迭代桶中的所有节点
        de = t0->table[v & m0];
        while (de) {
            fn(privdata, de);
            de = de->next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */
        // Iterate over indices in larger table             // 迭代大表中的桶
        // that are the expansion of the index pointed to   // 这些桶被索引的 expansion 所指向
        // by the cursor in the smaller table               //
        do {
            /* Emit entries at cursor */
            // 指向桶，并迭代桶中的所有节点
            de = t1->table[v & m1];
            while (de) {
                fn(privdata, de);
                de = de->next;
            }

            /* Increment bits not covered by the smaller mask */
            v = (((v | m0) + 1) & ~m0) | (v & m0);

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1));
    }

    /* Set unmasked bits so incrementing the reversed cursor
     * operates on the masked bits of the smaller table */
    v |= ~m0;

    /* Increment the reverse cursor */
    v = rev(v);
    v++;
    v = rev(v);

    return v;
}
```

## dictScanFunction

dictScanFunction 在 dict.h 中的定义如下：

``` c
typedef void (dictScanFunction)(void *privdata, const dictEntry *de);
```

这句话的意思是对于 参数为 (void \*privdata, const dictEntry \*de) ，返回值为 void 的函数建立别名 dictScanFunction。详细信息可参考 [[typedef]typedef的高级用法](http://blog.sina.com.cn/s/blog_917f16620101fpre.html) 。

第一次见 typedef 这种用法，记录一下。

