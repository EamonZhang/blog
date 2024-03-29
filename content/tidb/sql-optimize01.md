---
title: "Sql 优化"
date: 2018-11-21T09:18:37+08:00
draft: false
toc: false
categories: ["tidb"]
tags: ["优化"]
---

## 一条sql的执行过程

将 SQL 解析成抽象语法树(AST)，将 AST 变换到内部表示(IR)。然后优化器的输入就是 IR，它将生成最优的查询计划（Plan），然后会变成具体的执行器（Executor），里面有许多的算。

优化的阶段为IR 到生成 Plan 的过程，包括逻辑优化和物理优化

## 逻辑优化

逻辑优化主要是基于规则的优化(RBO)。

## 逻辑算子

- DataSource 这个就是数据源，也就是表。 select * from t 里面的 t
- Selection 选择，就是 select xxx from t where xx = 5 里面的 where 过滤条件条件
- Projection 投影，也就是 select c from t 里面的列 c
- Join 连接， select xx from t1, t2 where t1.c = t2.c 就是把 t1 t2 两个表做 join，这个连接条件一个简单的等值连接。join 有好多种，内关联，左关联，右关联，全关联..

## 列裁剪

只读取需要的列

## 最大最小消除

select min(id) from t

换成

select id from t order by id desc limit 1

## 投影消除

把不必要的 Projection 算子给消除掉

## 讲谓词下推

比如 select * from t1, t2 where t1.a > 3 and t2.b > 5 ，
假设 t1 和 t2 都是 100 条数据。如果把 t1 和 t2 两个表做笛卡尔集了再过滤，我们要处理 10000 条数据，而如果先做过滤条件，那么数据量就会少很多。这就是谓词下推的作用。


## 物理优化

物理优化会具体选择某一个算子具体的实现,比如 Join 用哪一种，读数据用哪一种，这里可能需要用到一些统计信息，决定哪一种方式代价最低，所以是基于代价的优化(CBO)。

物理优化是基于代价的优化，为上一阶段产生的逻辑执行计划制定物理执行计划。这一阶段中，优化器会为逻辑执行计划中的每个算子选择具体的物理实现。  
逻辑算子的不同物理实现有着不同的时间复杂度、资源消耗和物理属性等。在这个过程中，优化器会根据数据的统计信息来确定不同物理实现的代价，并选择整体代价最小的物理执行计划。

