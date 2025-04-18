---
title: "TinyKV Project1 Standalone KV"
date: 2025-03-24T17:01:44+08:00
lastMod: 2025-03-24T17:01:44+08:00
draft: true # 是否为草稿
author: ["tkk"]

categories: [TinyKV, Go]

tags: [TinyKV, Go, Project1, Standalone]

keywords: [TinyKV, Go, Project1, Standalone]

description: "TinyKV Project1 Standalone KV 实现记录" # 文章描述，与搜索优化相关
summary: "TinyKV Project1 Standalone KV 实现记录" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## 前言

根据实验室新生培训计划安排以及个人时间安排，是时候开始TinyKV了。由于没有学过分布式系统和GO语言，对于[课程内容](https://github.com/thekingking/tinykv/blob/course/doc/project1-StandaloneKV.md)以及代码自然是两眼一抹黑。首先，便是学习GO语言基本语法，这里我是学习的[Go语言圣经](https://golang-china.github.io/gopl-zh/index.html)，基本上是快速过完前面几章了解下Go语言语法基础就开始代码阅读了，我的建议是边看边写边学，学习和实践结合。Go语言学习我就不写了，有上面提到的文档可以看，我写一下我做这部分时遇到的问题，欠缺的知识，以及我的理解。

## CF(ColumnFamily)

在Project1中要求我们实现⼀个⽀持列族的单机键值存储系统，所以首先来了解下什么是CF。

### ColumnFamily简介

ColumnFamily（列族）​ 是一种用于组织数据的逻辑结构，常见于宽列存储（Wide-Column Store）型 NoSQL 数据库。其核心特点是将相关数据列分组存储，同时在物理层面优化存储和查询效率。
> ColumnFamily 的概念最早源于 ​Google Bigtable​（2006 年发表），其设计目标是为海量结构化数据提供高扩展性和高性能存储。Bigtable 中列族需预先定义，且数量有限（通常不超过几百个），而列可无限扩展。

![ColumnFamily-1](/images/TinyKV-Project1/ColumnFamily-1.jpg)

上图便是表中一行数据的CF表示，一行数据对应一个RowKey，将表中列拆分为多个列族，每个列族下面才是实际的多个列。数据写到HBase的时候都会被记录一个时间戳，这个时间戳被我们当做一个版本。比如说，我们修改或者删除某一条的时候，本质上是往里边新增一条数据，记录的版本加一了而已。

CF给我的感觉是以下几点：

1. Key-Value的扩展，进一步利用列值信息，将不同的列进行聚类，拆分为多个列族（实际查询可能不会查询一整行数据，而是查询某几列数据，将其聚合在一块），有点让我想起列式存储，感觉又是一种折中。
2. 空间利用率提高，我看Hbase文章中说，HBase表的每一行中，列的组成都是灵活的，行与行之间的列不需要相同。如图下：
![ColumnFamily-2](/images/TinyKV-Project1/ColumnFamily-2.jpg)
3. 写入更新，通过写入新的数据来更新旧的数据，不清楚咋回收的，后续再看看
![ColumnFamily-3](/images/TinyKV-Project1/ColumnFamily-3.jpg)

### TinyKV 中使用

Project1中要求我们实现单机存储引擎，并保证其实现CF。在TinyKV中我们简单将每一列拆分为一个列族，在Key前面添加CF前缀，如下图：
![ColumnFamily-4](/images/TinyKV-Project1/ColumnFamily-4.png)

## 参考文献

- [Project1 Standalone KV](https://github.com/Smith-Cruise/TinyKV-White-Paper/blob/main/Project1-Standalone-KV.md)
- [Project1](https://github.com/sakura-ysy/TinyKV-2022-doc/blob/main/doc/project1.md)
- [我终于看懂了HBase，太不容易了..](https://zhuanlan.zhihu.com/p/145551967)
- [LSM-Tree](https://www.modb.pro/db/1706109467498729472)
