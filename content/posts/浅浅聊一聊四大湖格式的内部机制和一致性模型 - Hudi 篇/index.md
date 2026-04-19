---
title: "浅浅聊一聊四大湖格式的内部机制和一致性模型 - Hudi 篇"
date: 2025-09-13T20:07:52+08:00
draft: false
tags: ["数据湖"]
categories: ["数据湖"]
summary: "文章深入解析了Hudi的内部机制与一致性模型，重点阐述其基于Timeline和FileGroup的读写流程、乐观并发控制及对写入端的时间戳单调性等严格要求，揭示了其复杂性与潜在数据一致性风险。"
ShowToc: true
TocOpen: false
---

前言：本来并不想聊 Hudi 的，因为我发现 Hudi 这玩意过于复杂和晦涩。但是有始有终吧，把数据湖系列都更新完吧。

## 内部机制

### 读写流程

写流程：

Writer 首先写入数据文件，然后写Timeline（后面的部分会介绍）文件来引用这些数据文件以提交这次写入。如下图所示：

![](images/img_01.png)

其中，数据文件会被组织成 File groups（后面的部分也会介绍），目前可以简单理解为分桶的概念，为了快速定位到某一条 key。

读流程：

读的时候会扫描 Timeline 来得到当前表的所有数据文件，然后读取这些数据文件即可。

### TimeLine

Hudi 的所有提交操作都会写一个 instant 文件来记录这次提交。 instant 文件的格式为：\[操作时间戳（以毫秒为单位）.\[操作类型\].\[操作状态\]。instant 文件按照递增的操作时间戳组织成 Timeline。

操作状态分为如下三种：

- Requested

- Inflight

- Completed

Writer 初始化的时候会先写入一个 .requested 文件，然后等到 Writer 开始写入数据前会写一个 .infight 文件，数据写完之后就会将写一个 .commit 文件，这个时候这次写入正式对外可见了。

可以看到这个 instant 文件的操作时间戳需要是单调的， 如果两个 Writer 拿到了相同的时间戳，那就会存在相互覆盖的情况（如果底层 filesystem 不支持 put if absent 的话）。 然而 Hudi 让 Writer 端自己去保证操作时间戳的唯一性。Hudi V5 Spec 表示需要你自己去做这个保证，如果违背了这个约束，就会带来非预期的行为。

PS：我觉得让 Writer 端自己去保证操作时间戳的单调对 Writer 端来说并不简单。

### FileGroup

Hudi 表的数据被组织为分区，分区下面还有一层 FIleGroup，一个 FIleGroup 就是若干文件的集合，任何给定的主键都映射到一个 FileGroup。所以我感觉 FIleGroup 有点像是 Bucket 的概念。

每一个 FileGroup 对应一个 file id，FileGroup 的所有文件都以这个 file id 为前缀，具体而言则是：\[file\_id\]\_\[write\_token\]\_\[timestamp\].\[file\_extension\]。

FileGroup 里面文件的具体组织取决于表是 Copy-On-Write 表还是 Merge-on-Read 表。

#### Copy-On-Write 表

任何数据的修改都需要重写整个数据文件。下面是一个 Copy-On-Write 表的 FileGroup 里面文件的示例：

![](images/img_02.png)

#### Merge-On-Read 表

数据的修改都是在一个 base 文件上写一个 log 文件来记录数据的修改。读的时候就需要将这个 base 文件和 log 文件记录的修改进行 merge 以得到最终的数据。下面是一个 Merge-On-Read 表的 FileGroup 里面文件的示例：

![](images/img_03.png)

### TimeLine & FileGroup

理解了 Timeline 和 FileGroup，我们可以看一下 Timeline 和 FileGroup 是如何组织在一起的，下图是一个具体的示例：

![](images/img_04.png)

## 一致性模型

### 写入流程详解

在理解 Hudi 的一致性模型之前，我们需要深入理解一下 Hudi 的写入流程，以 COW 为例写入流程如下图所示：

![](images/img_05.png)

1. 写入端获取一个时间戳

2. 写入端写一个 .request 的 instant 文件

3. 查找 Key

   - 通过 index 查看键是否存在（用于将 upsert 标记为插入或更新）。

   - 如果这个 key 存在，则找到其对应的 FileGroup。如果不存在，则会为这个 key 分配一个 FileGroup。写入端会 在 FileGroup pool 中选择一个，选择哪个是不确定的，取决于具体的实现。

4. 读 File Slice

   - 加载 timeline，找到当前最大的一个 complete (.commit 文件) 的 instant 的时间戳，作为 targe timestamp

   - 找出所有 touch 到这个 file group 的 已 complete(.commit) 的 instant 文件，并且这些 instant 文件时间戳 <= targe timestamp

   - 读出这些 instant 文件对应的，在这个 file group 下的所有的file slice，如果发现有任何 file slice 的 timestamp 大于当前 writer 的时间戳，就直接 abort

5. 写 File Slice

   - merge 上一步读出来的 file slice，进行重写，在该 file group 写入新的 file slice。

6. 获得表锁

7. 更新 index

   - 如果是插入操作，需要更新 index 来记录这个key 到 file group 的映射关系

8. 乐观并发控制检查

   - 加载 timeline

   - scan 出所有complete的 instant 文件，如果发现这些 instant 文件引用了任何一个大于 target timestamp 的 file slice，并且 file slice 属于当前 writer 要写入如的 file group，说明这期间有其他writer 进行了写入，并且写入出现了冲突，touch 到了相同的 file group，当前 writer 直接 abort

9. 写入完成

   - 写一个 complete 的 instant 文件

   - 释放表锁

PS：感觉 Hudi 和其他湖格式还不太一样，Hudi 强依赖表锁，但是其实其他湖格式可以通过文件系统的 PutIfAbsent 能力来避免元数据的冲突。

我们来详细解释一下乐观并发控制检查的机制，假设在某一时刻，两个 writer W1 和 W2 都准备进行提交，timeline 如下所示：

![](images/img_06.png)

然后：

1. W2 获得了表锁，并成功提交了 file slice <file\_id = 1, ts = 101>，释放表锁

2. W1 获得表锁，但是发现存在一个已提交的 file slice， <file\_id = 1, ts = 101>，并且其 ts 比 W1 读 File Slice 时对应的 ts = 50 还要大，于是就 abort 自己，提交失败。

### 时间戳冲突的影响

#### 写丢失：

如果两个 writer 用了相同的时间戳，且底层 filesystem 不支持 put if absent 的话，会存在互相覆盖的情况，如下图所示：

![](images/img_07.png)

Operation 2 会覆盖掉 Operation 1 的操作，导致 Operation 1 的操作丢失了。

#### FIle slice 的覆盖

之前我们介绍过，Hudi File group 里面的 file slice 文件也是以 timestmap 为文件名的一部分的，\[file\_id\]\_\[write\_token\]\_\[timestamp\].\[file\_extension\]。如果 timestmap 一样，也会存在冲突的情况，导致 Hudi 引用一个从未提交的事务写的文件，如下图所示：

![](images/img_08.png)

- Operation1 已经写了一个 file\_id = 1, ts = 100 的文件，假设文件名1\_100.parquet

- Operation2 也有相同的 timestamp，然后也写一个 file\_id = 1, ts = 100 的文件，文件名也为 1\_100.parquet。但是这个时候失败了。这样 Operation1 写的文件 1\_100.parquet 就会被 Operation2 写的文件覆盖了，虽然 Operation2 是一次失败的写入

不过如果底层 filesystem 支持 put if absent 或者 file slice 的文件名能加个随机值就能解决这个问题。

另外，博客的作者还做了一个实验，结论就是如果直接使用 current timestamp 的话，冲突的概率还是挺大，如下图所示：

![](images/img_09.png)

### Hudi 一致性模型 对 Writer 的要求

为了满足数据的一致性，Hudi 对 Wrier 有如下要求：

1. Timestmap 必须是单调的

2. 开启并发控制检查，即检测这期间是否有其他 Writer 写入

3. 开启 key 冲突的检测

4. 底层文件存储支持 put if absent（如果 1 满足的话，4 也可以不满足）

满足了上述条件后， Writer 写 Hudi 就不会破坏数据的一致性了。接下来我们看几种不满足上面条件的 case 来帮助理解。

- Case 1: 不开启并发控制检查

会出现写入丢失的问题，如下图所示：

![](images/img_10.png)

1. W1 写 k1，给 k1 分配一个 file group = 1，写入一个文件 f1，包含数据 k1=A

2. W2 写 k2，给 k2 分配一个相同的 file group = 1，写入一个文件 f2，包含数据 k2 = B

3. W1 提交成功，file group 只包含 f1

4. W2 也提交成功，file group 只包含 W2 写的 文件 f2，只有数据 k2 = B，导致 W1 的写入丢失了

核心原因是 W2 写入的时候，没有 merge W1 写入的内容。

如果开启了并发控制检查的话，则会在第4步 W2 提交的时候，发现 W1 也写了相同的 file group，并且 W1 写的 ts 为1，比自己读数据用的 ts 0 要大，W2 读数据的时候没有 merge W1 的写入，于是检测到冲突，直接 abort。

- Case 2: 不开启 key 冲突的检测

会出现重复 key 的情况，如下图所示：

![](images/img_11.png)

1. W1 写 k1，发现 k1 不存在，给 k1 分配一个 file group = 1，写入 file group = 1

2. W2 写 k1，也发现 k1 不存在，但是给 k1 分配一个不同的 file group = 2，写入 file group = 2

3. W1 提交，更新 index，记录 k1 映射到 file group = 1，提交成功

4. W2 提交，更新 index，记录 k1 映射到 file group = 2，提交成功。这个时候虽然 Hudi 的 index 认为 k1 对应的 file group = 2，但是其实有一条 key 为 k1 的数据映射到 file group = 1。并且这条数据依然存在于数据文件中

如果开启 key 冲突的检测的话， 则会在第4步 W2 提交的时候，W2 发现 index 中记录的 k1 映射到 file group = 1 和自己的不一致，就会直接 abort 自己。

- Case 3: Timestmap 不单调，并且底层文件存储不支持 put if absent

同样会导致写入丢失：

![](images/img_12.png)

1. W1 获得 ts = 1，写 k1，对应 file group1 的数据文件 1\_1.parquet

2. W2 也获得 ts = 1，写 k1，同样也写了 file group1 的数据文件 1\_1.parquet，覆盖了W1 写的 1\_1.parquet

3. W1 提交，提交成功

4. W2 提交，进行冲突检测，检测到这期间 W1 写入了一个冲突的文件，于是 abort 自己。但是这个时候 W2 写的这个数据文件已经把 W1 写的数据文件覆盖了，造成不一致的情况。

如果底层文件存储支持 put if absent 的话，则会在第2步 W1 写数据文件 1\_1.parquet 的时候，发现这个文件已经存在，就直接 abort 。

## 总结

Hudi 搞的这一套机制还是挺复杂的，比较容易出错，据我所知，不少公司在使用 Hudi 的时候出现数据正确性问题的时候，排查起来还是非常痛苦的。依然记得，前同事，某 Hudi PMC 玉兆老师排查一个 Hudi 的数据正确性问题排查了一周，那是我见过的，他每天早上来的最早的一周。
