---
title: "浅浅聊一聊四大湖格式的内部机制和一致性模型 - Delta 篇"
date: 2025-09-13T13:16:11+08:00
draft: false
tags: ["数据湖"]
categories: ["数据湖"]
summary: "Delta通过递增版本的DeltaLog记录写入，采用Copy-on-write或Merge-on-read实现数据更新，并利用PutIfAbsent或表锁解决并发写入冲突，其一致性模型基于分区级冲突检测。"
ShowToc: true
TocOpen: false
---

书接上文，接下来聊聊 Delta

## 内部机制

Delta 的写入流程：

1. 写入对应的 Data File

2. 写一个 Delta Log 来记录这次写入，比如写了哪些新文件，逻辑删除了哪些老文件

![](images/img_01.png)

有如下几个点需要注意：

- 这个 Delta Log 的文件名就代表了版本号，是一个严格递增的整数，假设一个 Delta 表提交了三次，则 Delta Log 如下所示：

- ./\_delta\_log/00000000000000000000.json.

- ./\_delta\_log/00000000000000000001.json.

- ./\_delta\_log/00000000000000000002.json.

- 这个 Delta Log 只引用这次写入涉及到的文件，不像 Iceberg & Paimon 一样会引用历史写入的文件，所以对于 Delta 表来说，如果需要读 Delta 表最新的数据，需要依次读取所有的 delta\_log 文件，得到所有的数据文件。不过 Delta 表会定期将 deleta log 进行merge 一个 parquet 文件，这样其实只需要读取一个 parquet 文件和少部分 delta\_log 文件即可。

### Copy-on-write & Merge-on-read

为了支持数据的删除和修改，即主键表模型，Delta 提供了两种模式：

#### Copy-on-write

任何数据的修改都需要重写整个数据文件。写不友好，因为需要重写整个文件。读友好，不需要额外的 Merge 操作，直接读重写后的数据文件即可。如下所示：

![](images/img_02.png)

T1 时刻在数据文件 file1 写了三条数据：<red，1>，<green，1>，<blue，1>。

如果要将 <blue，1> 删掉的话，T2 时候会写一个新的文件 file2，file2 包含两条数据 <red，1>，<green，1>，同时将 file1 标记为删除。

#### Merge-on-read

数据的修改只需要写新的文件来记录数据的修改。写友好，只需要在写的文件中写被修改的数据。读不友好，读的时候需要进行额外的 merge，将新的文件和之前的数据文件 merge 起来，得到最终的数据。

Delta 将数据的 Update 抽象成一次 delete，一次 insert，所以 Delta 需要标识一下哪条数据被删除了，也需要写新的数据。写新的数据比较简单，就是直接写一个 data file 就可以了。

对于标识一下哪条数据被删除了，Delta 采用 Deltion Vector（DV）文件的方式，DV 文件会记录哪个数据文件的哪条数据被删除了，然后读数据文件的时候，将数据文件与 DV 进行 mask，忽略那些被 DV 标记为已删除的数据。如下图所示：

![](images/img_03.png)

T1 时刻在数据文件 file1 写了三条数据：<red，1>，<green，1>，<blue，1>。

如果要将 <blue，1> 删掉的话，T2 时候会写一个新的文件 DV1 文件，DV1只记录要删除的数据所在的文件和其对应在文件的位置。

## 一致性模型

### Delta 如何解决元数据（Delta Log）冲突的问题

对于 Delta 来说，会存在两个不同的 writer 同时写相同文件名的 delta log 的问题，会存在一个 writer 会覆盖掉另一个 writer 的写入，导致数据的丢失，如下图所示：

![](images/img_04.png)

Opeation1 和 Opeation2 都基于表当前最新的版本 V1 开始写数据。

1. Operation1 写了 File2，提交版本 V2，写了名字为 00002.json 的一个 delta log文件

2. Operation2 写了 FIle3，但是它依然认为当前最新的版本是 V1，所以这次提交也还是写一个名字为 00002.json 的 delta log 文件，覆盖了 Operation1 写的 delta log文件，导致 Operation1 的写入丢失了

Delta 可以通过两种方式来解决这个问题：

1. 文件系统的 PutIfAbsent 的能力

借助于文件系统的 PutIfAbsent 的能力，于 是Operation2 写 00002.json 的时候，发现这个文件已经存在了，就会 reload 当前最新的版本号，然后进行数据冲突的检测（这期间有别的 writer 进行了写入，需要进行数据冲突检测，数据冲突检测接下来的内容会介绍到），数据冲突检测通过后就写下一个版本的 delta log 文件，即 00003.json

2. Table lock

但是对于某些不具备 PutIfAbsent 的能力的文件系统来说，比如 S3（注：其实应该S3 现在已经具备 PutIfAbsent 的能力了），就需要借助分布式锁（通常由外部的 catalog 提供，比如 hive catalog）来解决了。任何 Operation 在提交数据到 Delta 前都需要获得这把锁。

- 于是Operation2 在准备提交名字为 00002.json 的 delta log前，需要获得这把锁，避免这期间有并发的提交。reload 当前最新的版本号，现在为 V2，然后进行数据冲突的检测，数据冲突检测通过后就写下一个版本的 delta log 文件，即 00003.json .

### Delta 如何解决数据冲突的问题

其实 Delta 解决数据冲突的方式也很简单，就是：

如果从这个操作 开始时对应的版本 到 当前最新版本 之间的新增的数据文件（data file）和逻辑删除的（data file）对应的分区与这个操作对应的分区有 overlap，就认为冲突了，就直接 abort 这次写入。

可以看到， Delta 解决数据冲突的方式也很简单粗暴，相比于 Iceberg 更为精细化（文件级别检测是否冲突）的冲突检测，Delta更加粗粒度，只是分区级别检测是否冲突。
