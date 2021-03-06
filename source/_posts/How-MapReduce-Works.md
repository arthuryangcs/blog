---
title: Hadoop MapReduce 详解
date: 2017-02-28 15:26:26
tags: 
 - Hadoop
 - MapReduce
categories: 大数据
---

这里我们详细介绍一下 Hadoop 中 MapReduce 的执行流程，这可以帮助我们写出更加高性能的 MapReduce 程序。

<!-- more -->

## MapReduce 作业（job）剖析

下图展示了一个 MapReduce 作业的整体流程：

![MapReduce 作业整体流程](How-MapReduce-Works/mapreduce-job.png)

由五个独立的部分组成：

* 客户端，提交 MapReduce 任务
* YARN 资源管理器，负责协调集群的计算资源
* YARN 节点管理器，负责运行和监控集群机器上的计算容器
* MapReduce 应用主节点，负责协调 MapReduce 作业中的任务
* 分布式文件系统，这里通常是 HDFS，用来在实体间分享作业文件

### 作业提交

由客户端来提交一个新的作业（图中步骤 1），提交过程主要由以下几部分组成：

* 向资源管理器申请一个新的应用 ID，作为 MapReduce 作业 ID（步骤 2）。
* 检查作业的输出规范。若输出文件夹没有指定或者已经存在则返回错误，不提交作业。
* 计算作业输入文件划分，如果无法计算（例如输入路径不存在）则返回错误，不提交作业。
* 复制作业需要的资源到分布式文件系统中，以作业 ID 作为文件夹名称（步骤 3）。其中包括作业的 JAR 包、配置文件、输入文件划分。
* 向资源管理器提交任务（步骤 4）。

### 作业初始化

当资源管理器接收到客户端提交的任务，它将请求发往 YARN 调度者。调度者给作业分配一个容器，资源管理器在节点管理器的管理下运行这个应用（步骤 5a 和步骤 5b）。

应用主节点初始化作业，建立一定数量的记录对象来追踪任务的进度，来接收任务进度和结束报告（步骤 6）。接着，它从分布室文件系统中获取输入文件划分（步骤 7），为每个划分建立 Map 任务，和一定数量的 Reduce任务（由 *mapreduce.job.reduces* 属性设定）。

应用主节点必须确定如何执行任务。如果作业比较小的话，应用主节点会选择在同一个 JVM 中串行运行任务，这被称作 *uberized*，这个任务被称作 *uber* 任务 ；否则则会在多个节点上并行执行任务。 uber 任务可降低任务的延迟。

### 任务分配

如果作业没有按照 uber 模式执行，应用主节点会向资源管理器请求 Map 任务和 Reduce 任务所需要的容器（步骤 8）。Map 任务相比 Reduce 任务先被创造，并且有更高的优先级。所有 Map 任务完成后才会开始排序阶段，5% 以上的 Map 任务执行完后才会开始 Reduce 任务。

Reduce 任务可执行在集群中的任何机器上，但 Map 任务会尽量在数据所在机器上执行，或在相同机器机架上的机器执行，这是为了减少网络带宽的使用。

请求同时也会为任务分配内存和 CPU，默认情况狂下，每一个 Map 任务或者 Reduce 任务分配 1024 MB 的内存和一个虚拟节点。也可通过以下属性进行配置：mapreduce.map.memory.mb、mapreduce.reduce.memory.mb、mapreduce.map.cpu.vcores、mapreduce.reduce.cpu.vcore。

### 任务执行

一旦任务通过资源管理器分配到某一节点的容器及资源，任务管理器就会联系节点管理器开始执行容器（步骤 9a 和步骤 9b）。在任务开始之前，节点会从分布式缓存中获取需要的配置文件和 JAR 包（步骤 10）。最后，节点执行任务（步骤 11）。

## Shuffle 和 Sort 详解

MapReduce 会保证每个 Reduce 任务的输入都是按照主键排序的，这是 Sort 阶段执行的。MapReduce 会将 Map 任务的输出转化成 Reduce 任务的输入，这个过程称作 Shuffle。接下来我们会详细讲解一下 Shuffle 阶段是如何工作的。

### Map 端

当 Map 任务产生输出后，它不是简单地写到磁盘上，而是会将数据缓存到内存并做预排序来提高整体性能。下图展示了这个操作过程：

{% asset_img shuffle-and-sort.png MapReduce 中 Shuffle 和 Sort 流程 %}

每个 Map 任务会有维护一个循环内存缓冲区，默认情况下是 100 MB（可通过 mapreduce.task.io.sort.mb 配置）。当缓冲区内容达到一定阈值（mapreduce.map.sort.spill.percent 配置，默认情况下为 0.80，80%），就会开启一个后台线程来将缓冲区的内容写到磁盘上，这个过程成为 spill，在这过程中 Map 任务仍将输出内容写入缓冲区，如果这时缓冲区满了，Map 任务会阻塞直到后台线程完毕。

在写入磁盘前，后台线程首先会将数据按照 Reduce 任务的分配来分割数据，然后对每个分块内部按照主键进行排序。这些都可由用户自定义方法定义，如果用户制定了合并函数，还会在排序结束后将数据尝试进行合并，这样能减小写入磁盘和传送到 Reduce 端的数据量。

每有一个 spill 后台线程运行，都会产生一个 spill 文件，当spill 文件个数到达一定数量或者 Map 任务完成后，就会将 spill 文件合并到一个分段并排序的文件中。这个最大的 spill 文件数量默认为 10，可由 mapreduce.task.io.sort.factor 配置设定。

如果超过三个 spill 文件进行合并，在合并过程中也会调用用户定义的合并函数对数据进行合并。如果 spill 文件少于三个，那么就没有额外进行合并的必要了。

一般来说，对写入磁盘的数据进行压缩是有帮助的，这同样可以减小写入磁盘和传送到 Reduce 端的数据量。默认情况下并不会对数据进行压缩。我们可以将 mapreduce.map.output.compress 设置为 true 来打开压缩，通过  mapreduce.map.output.compress.codec 来选择压缩方法。

最后，Map 端的输出文件会通过 HTTP 协议传输到 Reduce 端。

### Reduce 端

Reduce 端会收到每个 Map 任务输出文件中划分到自己的那部分数据。由于 Map 任务不是同时完成的，所以每有一个 Map 任务完成，Reduce 任务就会复制自己那个划分到本地，这个阶段叫做 Copy 阶段。每个 Map 任务完成后都会在 Reduce 端启动一个线程复制数据，所以复制数据是并行的。默认情况下最大允许 5 个复制线程，可以通过 mapreduce.reduce.shuffle.parallelcopies 配置。

如果数据比较小的话 Map 任务输出会保存到 JVM 的内存中，这个缓冲区大小由 mapreduce.reduce.shuffle.input.buffer.percent 配置，表示这个部分占总堆空间的百分比。否则，会将数据存入磁盘。当缓冲区到达一定阈值（由 mapreduce.reduce.shuffle.merge.percent 配置）或 Map 输出到达一定大小（由 mapreduce.reduce.merge.inmem.threshold 配置），就会将数据合并到硬盘。如果用户制定了合并函数，就会在数据合并过程同时执行用户指定的合并函数。如果数据是压缩的，会先将数据进行解压缩。

当所有 Map 任务完成后，Reduce 任务就会进入 Sort 阶段。会将所有接收到的 Map 端数据进行归并排序，来保持主键顺序。这个过程会按照回合来进行。比如，如果有 50 个 Map 输出，而合并因子设定为 10 的话， 将会有 5 个回合，每个回合合并 10 个文件到 1 个文件，最后产生 5 个中间文件。这个合并因子可通过 mapreduce.task.io.sort.factor 设置。

相比最后将 5 个文件合并成一个文件，这里会有一个优化。比如下图这种情况：

{% asset_img efficiently-merge.png 更高效地以合并因子 10 合并 40 个文件 %}

将 40 个文件以合并因子 10 合并。第一回合合并 4 个文件，接下来 3 个回合每回合合并 10 个文件， 最后将 4 个 合并文件和剩下的 6 个原始文件合并为一个最终文件。这么做不会减少回合次数，但能够减少写磁盘的总数据量。

合并成一个最终文件后，就会进入 reducde 阶段。这个阶段会对每个主键执行 reduce 函数，输出直接写到输出文件系统中去，通常为 HDFS。因为节点管理器通常也是数据节点，所以第一个副本将会写到本地磁盘。

## 参考

1. 《hadoop: The Definitive Guide (Fourth Edition)》

