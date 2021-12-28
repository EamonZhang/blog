---
title: "ClickHouse"
date: 2021-12-21T09:10:29+08:00
draft: false
toc: false
categories: []
tags: []
---

## 调研总结

- 不适用于多个大表的join操作
- 没有事务保障机制
- merage最终一致性

不适合财务相关系统

对数据精度要求不高，实时性有要求的场景比较适合。
