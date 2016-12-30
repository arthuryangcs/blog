---
title: Redis 源码剖析（二） - HyperLogLog小议
date: 2016-12-26 17:57:47
tags:
  - Redis
  - 源码
  - HyperLogLog
categories: Redis
---

最近研究了下 Redis 中的 HyperLogLog。之前做 ACM 题主要都是确定性算法，这次接触到随机算法，顿时觉得惊为天人。在允许结果有一定误差的情况，通过随机算法往往比确定性算法需要小的多的时间复杂度或空间复杂度。HyperLogLog 算法就是其中的典型例子，也是我最喜欢的算法之一。

HyperLogLog 算法主要解决的是基数问题，就是计算一个列表中不同元素的个数。例如列表 \[1, 2, 1, 3, 5, 5, 3\] 的基数为4，其中有四个不相同的元素 \[1, 2, 3, 5\]。最典型的应用场景就是：对于一个网站，查询一段时间内共有多少不同的 IP 访问了网站。Redis 中的 HyperLogLog 可以做到仅用 12KB 的数据，就可以统计不超过 \\( 2^{64} \\) 的基数，且标准差为 0.81%。

<!-- more -->

## 算法原理

### Linear Counting

有一个长度为 n 的 01 串，每次操作随即将其中一位设为1，问经过 m 次后，有多少位为 1？反过来，当我们知道了有多少个1，能否大概估计出操作的次数？这就是 Linear Counting 算法的思想。

LC的基本思路是：设有一哈希函数H，其哈希结果空间有m个值（最小值0，最大值m-1），并且哈希结果服从均匀分布。使用一个长度为m的bitmap，每个bit为一个桶，均初始化为0，设一个集合的基数为n，此集合所有元素通过H哈希到bitmap中，如果某一个元素被哈希到第k个比特并且第k个比特为0，则将其置为1。当集合所有元素哈希完成后，设bitmap中还有u个bit为0。则：

$$\hat{n}=-mlog\frac{u}{m}$$

为n的一个估计，且为最大似然估计（MLE）。

我们简单想一下可知，对于前几次操作，由于 n 较大，碰撞的概率很小，随着 1 的个数的增多，碰撞的数量也在增加，当全部位都为 1 时，算法也失去了统计的意义。

### LogLog Counting

我们每次操作就是投若干次硬币，若硬币为正就继续投下去，直到硬币为反，投出正面的次数就是这次操作的数值，若干次操作后，我们统计正面次数最多的那次操作一共投出几次正面。不难发现，我操作的次数越多，出现最大连续正面的长度也就越长。当然，也有可能只投了一次就有很多正面，所以我们会首先分出若干个桶，将每次操作先随机到其中一个桶，每个桶保存在这个桶中最长的连续正面，最后所有的桶再计算出一个平均值。

这个算法随着操作次数的增多，偶然性造成的影响就会越小，而当操作次数小时，随机性造成的误差就会很大。

### Adaptive Counting

Linear Counting 算法在基数小时误差小，基数大时误差大；LogLog Counting 算法在基数小时误差大，基数大时误差小。那么将他们结合起来不就好了吗？这就是 Adaptive Counting 算法。

如果分析一下 LC 和 LLC 的存储结构，可以发现两者是兼容的，区别仅仅在于LLC关心每个桶的最大值，而LC仅关心此桶是否为空。因此只要简单认为最大值不为0的桶为非空，0为空，就可以使用LLC的数据结构做LC估计了。

当空桶率低于 0.051 时，就会采用 Linear Counting 算法，否则就会采用 LogLog Counting 算法。

### HyperLogLog Counting

接下来就是 Redis 中使用的 HyperLogLog Counting 算法了，它也是 LC 和 LLC的组合，但相比 AC 算法要复杂一些。

首先，在之前的 LLC 算法中，对各个桶取的是几何平均数，由于几何平均数对于离群值（例如这里的0）特别敏感，因此当存在离群值时，LLC的偏差就会很大，这也从另一个角度解释了为什么n不太大时LLC的效果不太好。这是因为n较小时，可能存在较多空桶，而这些特殊的离群值强烈干扰了几何平均数的稳定性。

因此，HYLL使用调和平均数来代替几何平均数，调和平均数的定义如下：

$$H = \frac{n}{\frac{1}{x\_1} + \frac{1}{x\_2} + ... + \frac{1}{x\_n}} = \frac{n}{\sum\_{i=1}^n \frac{1}{x\_i}}$$

调和平均数可以有效抵抗离群值的扰动。

另外同 AC 算法，HYLL 还给出了不同段的修正方案，用不同的公式进行计算。

如果想要了解算法的更多细节，可参考[《解读Cardinality Estimation算法（第一部分：基本概念）》](http://blog.codinglabs.org/articles/algorithms-for-cardinality-estimation-part-i.html)这个系列，讲解更加详细。如果想了解算法的具体推导过程，可查阅相关论文。

## HYLL 算法在 Redis 中的具体实现

### 数据结构

在 Redis 中的 HYLL 分为头部和数据两个部分。

``` c
+------+---+-----+----------+
| HYLL | E | N/U | Cardin.  |
+------+---+-----+----------+
```

头部占 16 字节，首先 4 字节的"HYLL"字符，然后1个字节确定数据部分为稀疏存储还是稠密存储。接着 3 个无用的占位字节，均设为 0。最后 4 个字节存储没有修改过的最近一次计算得到的基数，因为很多请求之间其实并不改变基数。

数据部分分为两种，一种是稠密储存，一种是稀疏储存，根据头部确定。这类似于矩阵的存储。当基数很小时，使用稀疏存储，只特殊标记非 0 位。当基数达到一定程度时转为稠密存储。

Redis 中的 HYLL 采用 MurmurHash2 哈希算法，64 位结果，其中最后 14 位作为索引桶标记，共分为 \\( 2^{14} = 16384 \\) 个桶，后 50 位作为随机生成的01串，以从一端开始连续 1 的个数当做 LLC 算法中抛硬币的结果，范围为 1~50。

当数据部分为稠密存储时，每个桶需要 6 位来存储最长结果，共 16384 个桶，加上头部一共为 $(16384 * 6) / 8 + 16 = 12304$ 字节，约为 12KB，可表示最大 \\( 2^{64} \\) 的基数。

### 算法优化

在实际使用中，由于分段计算和哈希函数的原因，最终结果并不是完全无偏的，会在某些情况下有波动，因此经过多次试验后，当 HYLL 采用 LLC 算法时，会通过一个四次函数回归结果，使结果更加准确。

在[《Redis new data structure: the HyperLogLog》](http://antirez.com/news/75) 中有算法的测试结果。

实现细节可看我的 Redis 中的 [hyperloglog.c](https://github.com/arthuryangcs/redis-3.0-annotated/blob/unstable/src/hyperloglog.c) 注释。

## 小结

由于 HYLL 算法不存储每条结果，因此无法得到基数对应的每条结果究竟是什么，这也是为了节省空间的必然结果。

**P.S** 在[《HyperLogLog的核心思想原理》](http://www.feellin.com/hyperloglogde-he-xin-si-xiang-yuan-li/)这篇博客中有一个小错误。作者举例当计算网站 PV 时不能用 HYLL，而需要将 url 拼接数字进行统计。其实大可不必，我们只需要维护一个整数变量记录访问次数就可，根本用不到基数统计算法，这就陷入了惯性思维中。由于此文章在 HYLL 算法中 Google 排名比较靠前，且不支持留言，特地在此修正一下，以防误导读者。

## 参考文档

1. [解读Cardinality Estimation算法（第一部分：基本概念）](http://blog.codinglabs.org/articles/algorithms-for-cardinality-estimation-part-i.html)
2. [五种常用基数估计算法效果实验及实践建议](http://blog.codinglabs.org/articles/cardinality-estimate-exper.html)
3. [Redis new data structure: the HyperLogLog](http://antirez.com/news/75)
4. [HyperLogLog的核心思想原理](http://www.feellin.com/hyperloglogde-he-xin-si-xiang-yuan-li/)

