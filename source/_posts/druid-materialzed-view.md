---
title: Druid 物化视图
date: 2019-03-23T12:38:22.536Z
category: 大数据
tags:
  - Druid
  - 物化视图
---

Druid 在 0.13.0 版本更新了物化视图功能，以此来提高查询性能。

<!-- more -->

## 简介

什么是物化视图，假设一个数据源的原始维度有十个列，通过分析查询请求发现，group1 中的三个维度和 group2 中的三个维度分别经常同时出现，剩余的四个维度可能查询频率很低。更加严重的是，没有被查询的维度列里面有一个是高基维，就是 count district 值很大的维度，比如说像 User id 这种。这种情况下会存在很大的查询性能问题，因为高基维度会影响 Druid 的数据预聚合效果，聚合效果差就会导致索引文件 Size 变大，进而导致查询时的读 IO 变大，整体查询性能变差。针对这种 case 的优化，我们会将 group1 和 group2 这种维度分别建一个预聚合索引，然后当收到新的查询请求，系统会先分析请求里要查询维度集合，如果要查询的维度集合是刚才新建的专用的索引维度集合的一个子集，则直接访问刚才新建的索引就可以，不需要去访问原始的聚合索引，查询的性能会有一个比较明显的改善，这就是物化视图的一个设计思路，也是一个典型的用空间换时间的方案。

![](druid%20物化视图/2019-03-23-20-44-56.png)

## 使用方式

官方文档的链接：http://druid.io/docs/latest/development/extensions-contrib/materialized-view.html

物化视图的构建和普通 DataSource 构建类似，不过多了一个 `baseDataSource` 参数，表示这个物化视图的数据源。现阶段物化视图只支持对 `TableDataSource` 类型的数据源， `TopNQuery`、`TimeseriesQuery`、`GroupByQuery` 相关的查询有效。物化视图的 dimensions 和 metrics 必须是原数据源的子集。其实本质上物化视图就是一种 DataSource，只不过它的数据来源于其他 DataSource。

建立好物化视图后，Druid 就会自动启动任务从原数据源加载数据。

## 源码解析

下面着重分析一下通过物化视图查询的优化。这里对代码做了一些简化，去除了数据统计部分。

``` Java
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

package org.apache.druid.query.materializedview;

public class DataSourceOptimizer
{
  private final ReadWriteLock lock = new ReentrantReadWriteLock();
  private final TimelineServerView serverView;

  @Inject
  public DataSourceOptimizer(TimelineServerView serverView) 
  {
    this.serverView = serverView;
  }

  /**
   * Do main work about materialized view selection: transform user query to one or more sub-queries.
   * 
   * In the sub-query, the dataSource is the derivative of dataSource in user query, and sum of all sub-queries' 
   * intervals equals the interval in user query
   * 
   * Derived dataSource with smallest average data size per segment granularity have highest priority to replace the
   * datasource in user query
   * 
   * @param query only TopNQuery/TimeseriesQuery/GroupByQuery can be optimized
   * @return a list of queries with specified derived dataSources and intervals 
   */
  public List<Query> optimize(Query query)
  {
    // only topN/timeseries/groupby query can be optimized
    // only TableDataSource can be optimiezed
    if (!(query instanceof TopNQuery || query instanceof TimeseriesQuery || query instanceof GroupByQuery)
        || !(query.getDataSource() instanceof TableDataSource)) {
      return Collections.singletonList(query);
    }
    String datasourceName = ((TableDataSource) query.getDataSource()).getName();
    // get all derivatives for datasource in query. The derivatives set is sorted by average size of 
    // per segment granularity.
    Set<DerivativeDataSource> derivatives = DerivativeDataSourceManager.getDerivatives(datasourceName);
    
    if (derivatives.isEmpty()) {
      return Collections.singletonList(query);
    }

    lock.readLock().lock();
    try {
      // get all fields which the query required
      Set<String> requiredFields = MaterializedViewUtils.getRequiredFields(query);
      
      Set<DerivativeDataSource> derivativesWithRequiredFields = Sets.newHashSet();
      for (DerivativeDataSource derivativeDataSource : derivatives) {
        derivativesHitCount.putIfAbsent(derivativeDataSource.getName(), new AtomicLong(0));
        if (derivativeDataSource.getColumns().containsAll(requiredFields)) {
          derivativesWithRequiredFields.add(derivativeDataSource);
        }
      }
      // if no derivatives contains all required dimensions, this materialized view selection failed.
      if (derivativesWithRequiredFields.isEmpty()) {
        return Collections.singletonList(query);
      }
      
      List<Query> queries = Lists.newArrayList();
      List<Interval> remainingQueryIntervals = (List<Interval>) query.getIntervals();
      
      for (DerivativeDataSource derivativeDataSource : ImmutableSortedSet.copyOf(derivativesWithRequiredFields)) {
        final List<Interval> derivativeIntervals = remainingQueryIntervals.stream()
            .flatMap(interval -> serverView
                .getTimeline((new TableDataSource(derivativeDataSource.getName())))
                .lookup(interval)
                .stream()
                .map(TimelineObjectHolder::getInterval)
            )
            .collect(Collectors.toList());
        // if the derivative does not contain any parts of intervals in the query, the derivative will
        // not be selected. 
        if (derivativeIntervals.isEmpty()) {
          continue;
        }
        
        remainingQueryIntervals = MaterializedViewUtils.minus(remainingQueryIntervals, derivativeIntervals);
        queries.add(
            query.withDataSource(new TableDataSource(derivativeDataSource.getName()))
                .withQuerySegmentSpec(new MultipleIntervalSegmentSpec(derivativeIntervals))
        );
        derivativesHitCount.get(derivativeDataSource.getName()).incrementAndGet();
        if (remainingQueryIntervals.isEmpty()) {
          break;
        }
      }

      if (queries.isEmpty()) {
        return Collections.singletonList(query);
      }

      //after materialized view selection, the result of the remaining query interval will be computed based on
      // the original datasource. 
      if (!remainingQueryIntervals.isEmpty()) {
        queries.add(query.withQuerySegmentSpec(new MultipleIntervalSegmentSpec(remainingQueryIntervals)));
      }
      return queries;
    } 
    finally {
      lock.readLock().unlock();
    }
  }
}

```

`public List<Query> optimize(Query query)` 将原查询优化成一系列基于物化视图的查询，以此优化整体性能。

``` Java
if (!(query instanceof TopNQuery || query instanceof TimeseriesQuery || query instanceof GroupByQuery)
        || !(query.getDataSource() instanceof TableDataSource)) {
      return Collections.singletonList(query);
}
```

首先检查查询的数据源是否是 TableDataSource 类型，查询是否是 TopNQuery、TimeseriesQuery、GroupByQuery 类型。只有满足这些条件才能够通过物化视图优化。

``` Java
// get all derivatives for datasource in query. The derivatives set is sorted by average size of 
// per segment granularity.
Set<DerivativeDataSource> derivatives = DerivativeDataSourceManager.getDerivatives(datasourceName);

if (derivatives.isEmpty()) {
    return Collections.singletonList(query);
}
```

获取这个数据源下的全部物化视图，如果没有建立任何物化视图，无需优化。**获取到的物化视图按照平均 segement 大小排列。**

``` Java
// get all fields which the query required
Set<String> requiredFields = MaterializedViewUtils.getRequiredFields(query);

Set<DerivativeDataSource> derivativesWithRequiredFields = Sets.newHashSet();
for (DerivativeDataSource derivativeDataSource : derivatives) {
    if (derivativeDataSource.getColumns().containsAll(requiredFields)) {
        derivativesWithRequiredFields.add(derivativeDataSource);
    }
}
// if no derivatives contains all required dimensions, this materialized view selection failed.
if (derivativesWithRequiredFields.isEmpty()) {
    return Collections.singletonList(query);
}
```

获取查询相关的所有字段，筛选出包含全部查询字段的物化视图。如果没有则无法优化。

``` Java
List<Query> queries = Lists.newArrayList();
List<Interval> remainingQueryIntervals = (List<Interval>) query.getIntervals();

for (DerivativeDataSource derivativeDataSource : ImmutableSortedSet.copyOf(derivativesWithRequiredFields)) {
    final List<Interval> derivativeIntervals = remainingQueryIntervals.stream()
        .flatMap(interval -> serverView
            .getTimeline((new TableDataSource(derivativeDataSource.getName())))
            .lookup(interval)
            .stream()
            .map(TimelineObjectHolder::getInterval)
        )
        .collect(Collectors.toList());
    // if the derivative does not contain any parts of intervals in the query, the derivative will
    // not be selected. 
    if (derivativeIntervals.isEmpty()) {
        continue;
    }

    remainingQueryIntervals = MaterializedViewUtils.minus(remainingQueryIntervals, derivativeIntervals);
    queries.add(
        query.withDataSource(new TableDataSource(derivativeDataSource.getName()))
            .withQuerySegmentSpec(new MultipleIntervalSegmentSpec(derivativeIntervals))
    );
    derivativesHitCount.get(derivativeDataSource.getName()).incrementAndGet();
    if (remainingQueryIntervals.isEmpty()) {
        break;
    }
}
```

这里遍历全部物化视图，每个视图尽量覆盖查询的 interval（时间区间）。queries 维护所有的子查询，remainingQueryIntervals 维护每次循环结束后剩余未被覆盖的 interval。如果一个物化视图不包含查询包含的任何一部分 interval，则此次查询将不会使用这个物化视图。如果轮询到某一个视图已经覆盖了查询全部的 interval，遍历提前结束。

``` Java
if (queries.isEmpty()) {
    return Collections.singletonList(query);
}
```

如果没有找到任何可以优化的物化视图，无法优化。

``` Java
//after materialized view selection, the result of the remaining query interval will be computed based on
// the original datasource. 
if (!remainingQueryIntervals.isEmpty()) {
    queries.add(query.withQuerySegmentSpec(new MultipleIntervalSegmentSpec(remainingQueryIntervals)));
}
return queries;
```

最后，如果仍有部分 interval 没有被覆盖，这部分查询直接在原数据源上进行查询。

返回优化后的查询列表 queries。

## 时序物化视图

快手在普通维度物化视图的基础上，又开发了时序物化视图。

![](druid%20物化视图/2019-03-23-20-46-16.png)

在时序物化视图上，建立不同粒度的 segment，对于大跨度时间范围的查询，优先使用大粒度的 segment 进行查询，若不能全部覆盖再逐步递减进行小粒度的查询，最终聚合得到结果。

这部分优化快手并没有开源相关代码。

## 总结

物化视图在大数据处理中是一个比较常用的查询优化手段，将经常查询的数据进行预处理，本质上就是通过空间换时间。