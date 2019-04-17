---
title: MySQL-08-事务到底是隔离的还是不隔离的？
date: 2019/04/08 06:59:11
categories: 
  - [MySQL-in-Action]
tags: 
  - [geektime]
  - [MySQL]
---

事务的启动时机：`begin/start transaction` 命令并不是一个事务的起点，在执行到他们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果想要马上启动一个事务，可以使用 `start transaction with consistent snapshot` 这个命令。

- 第一种启动方式，一致性视图是在执行第一个快照读语句时创建的；
- 第二种启动方式，一致性视图实在执行 `start transaction with consistent snapshot` 时创建的。

在 MySQL 里，有两个『视图』的概念：

1. 一个是 view，他是一个用查询语句定义的虚拟表，在调用的时候执行查询语句并生成结果。创建视图的语法是 `CREATE VIEW ...`，而它的查询方法与表一样；
2. 另一个是 InnoDB 在实现 MVCC 时用到的一致性读视图，即 consistent read view，用于支持 RC（Read Conmitted，读提交）和 RR（Repeatable Read，可重复读）隔离级别的实现。

<!-- more -->

## 『快照』在 MVCC 里是怎么工作的

InnoDB 里面每个事务有一个唯一的事务 ID，叫做 transaction id。它是在事务开始的时候向 InnoDB 的事务系统申请的，是按申请顺序严格递增的。

而每行数据也都是有多个版本的，每次事务更新数据的时候，都会生成一个新的数据版本，并且把 transaction id 赋值给这个数据版本的事务 ID，记为 `row trx_id`。同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息可以直接拿到它。

也就是说，数据表中的一行记录，其实可能有多个版本（row），每个八本有自己的 `row trx_id`。

InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在『活跃』的所有事务 ID。『活跃』值得就是，启动了但还没提交。

数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位。

这个视图数组和高水位，就组成了当前事务的一致性视图（read-view）。

而数据版本的可见性规则，就是基于数据的 `row trx_id` 和这个一致性视图的对比结果得到的。

![数据版本一致性规则](https://raw.githubusercontent.com/N0nb0at/mysql-in-action-geektime/dev/resource/data-version-visibility-rules.png)

这个视图数组把所有的 `row trx_id` 分成了几种不同的情况：

1. 如果落在绿色部分，表示这个版本是已提交的事务或者当前事务自己生成的，这个数据是可见的；
2. 如果落在红色部分，表示这个版本是由将来启动的事务生成的，肯定不可见；
3. 如果落在黄色部分，那就包括两种情况；
    a. 若 `row trx_id` 在数组中，表示这个版本是由还没提交的事务生成的，不可见；
    b. 若 `row trx_id` 不在数组中，表示这个版本是已经提交了的事务生成的，可见。

InnoDB 利用了『所有数据都有多个版本』的这个特性，实现了『秒级创建快照』的能力。

## 更新逻辑

更新数据都是先读后写的，而这个读，只能读当前的值，称为『当前读』（current read）。

除了 update 语句外，select 语句如果加锁，也是当前读。

下面这两个语句，就是分别加了读锁（S 锁，共享锁）和写锁（X 锁，排它锁）：

``` sql
SELECT k FROM tbl WHERE id=1 LOCK IN SHARE MODE;
SELECT k FROM tbl WHERE id=1 FOR UPDATE;
```

事务的可重复读能力是如何实现的：可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前记录的行锁被其他事务占用的话，就需要进入锁等待。

而读提交的逻辑和可重复读的逻辑类似，他们最主要的区别是：

- 在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；
- 在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图。
