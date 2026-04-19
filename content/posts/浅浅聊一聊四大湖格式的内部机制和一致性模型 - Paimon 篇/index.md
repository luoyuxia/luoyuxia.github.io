---
title: "浅浅聊一聊四大湖格式的内部机制和一致性模型 - Paimon 篇"
date: 2025-09-07T11:10:19+08:00
draft: false
tags: ["数据湖"]
categories: ["数据湖"]
summary: "Paimon通过LSM树和Deletionvector优化主键表读写，多写者不同bucket无一致性问题，但同bucket写入可能导致更新丢失或悬空Deletionvector。"
ShowToc: true
TocOpen: false
---

Paimon 和 Iceberg 在元数据层比较相似，所以接下来聊聊 Paimon，Paimon 支持非主键表和主键表，但是这篇文章我们只考主键表。

## 内部机制

Paimon 分为元数据层和数据层，元数据层记录表的 schema，表有哪些文件等信息，而数据层则是具体的一堆数据文件。

### 元数据层

每次写入的时候，Paimon 都会生成一组 metadata file，记录表当前的数据文件，如下图所示：

![](images/img_01.png)

最上层是 一个 Snapshot 文件，记录表的这个 snapshot 的信息，snapshot 文件会引用 3 个 manifest list 文件：

- Base Mainifest list：引用前一个 snapshot 包含的所有数据文件

- Delta Mainifest list：引用上个 snapshot 到这个snapshot 期间写入的所有数据文件

- Index Mainifest list：引用这个snapshot 的索引文件，比如 deletion vector

单独区分 Base Mainifest list 和 Delta Mainifest list 的好处是方便 list 出来每个 snapshot 增加了哪些数据文件，更好地进行流读。

注：Paimon 的 snapshot 文件的格式为 snapshot-{snapshotid}，其中 snapshotid 是个递增的整数，提交快照2 其实就是写一个名字为 snapshot-2 的文件，Paimon 通过 catalog lock 来保证写一个名字为 snapshot-2 的文件是原子的，即原子的 PutIfAbsent 语义，如果不存在就写入，存在就 abort。避免多writer 场景下的互相覆盖。

下面是一个经过多次写入后，Paimon 元数据的组织：

![](images/img_02.png)

### 数据层

Paimon 会将数据进行分区键（如果有的话）进行分区，然后再根据主键进行分桶。分桶的存在是为了给读取和写入提供更多的并行性。

下面是一个 Paimon 数据组织的示意图：

![](images/img_03.png)

每个 bucket 的数据都组织为一颗独立的 LSM Tree，使用 LSM Tree 的原因是 LSM Tree 支持高效的 Upsert 和点查。LSM Tree 可以理解为 merge-on-read 模型，写入的时候直接写一个新的文件，在新的文件里面记录最新的数据。比如将要将数据 <jack, appple> 更新成数据 <jack, orange>，则会有两个数据文件：

- data file1 包含 <seq=0, jack, appple>

- data file2 包含 <seq=1, jack, orange>

注意：数据文件额外记录了 seq 这个隐藏列，对用户是不可见的，这是用来标记哪一条数据是最新的。

读取的时候，需要将 data file1 和 data file2 合并进行读取才能得到最终的<jack, orange>。这就导致只能用一个并发去读取这个 bucket的所有文件，限制比较大，读取效率较低。

于是 Paimon 引入了 Deletion vector 文件

#### Deletion vector

##### Deletion vector 原理

Deletion vector 文件就是标记某个文件的某条数据被删掉了。如下图所示：

![](images/img_04.png)

就上面的例子而言，Deletion vector 文件会标记 data file1 的第一条数据被删了。于是就可以用两个并发单独去读 data file1 和 data file2。每个并发读取的时候都需要用数据文件和Deletion vector 文件做一次比对，看数据是不是被删了。读 data file1 的时候，发现 <seq=0, jack, appple>这条数据被删了，就不读出来。读 data file2 的时候，没有数据被删了，读出 <jack, orange>。

##### Deletion vector 维护

可以看到 Deletion vector 需要对数据进行反查，得到这条数据所在的文件及其 pos，开销会比较大，所以并不会在写入的时候就生成 Deletion vector 。数据一开始会首先写入 L0层，Paimon 不会为 L0 层 的数据生成 deletion vector。Paimon 会在将 L0 层的数据 Compact 到更下层的时候才为其生成 deletion vector。

如下所示：

![](images/img_05.png)

Compaction 之前，没有 DV，Compaction 之后才会 DV。这也就意味着，如果没有经过 Compaction，L0层还是有数据文件的话，Paimon 是没办法通过 DV 为读取进行多并发读取加速的，只能退化到单并发读取。

下图是一个更复杂的例子：

![](images/img_06.png)

## 一致性模型

我们主要考虑如下两种写入 Topology 来看 Paimon 是否会存在一致性问题；

- 多 writer 写不同的 bucket：多个 writer ，每个 writer 写不同的 bucket，writer 写的时候会进行 compaction，并且每个 writer 都会进行单独进行 commit

- 多 writer 写相同的 bucket：多个 writer ，每个 writer 写相同的 bucket，writer 写的时候会进行 compaction，并且每个 writer 都会进行单独进行 commit

### 多 writer 写不同的 bucket

Topology 如下所示：

![](images/img_07.png)

假设 comact 操作和 write 操作同时在进行，则有如下的流程：

1. 写数据文件

   - Writer

      - 写数据文件

   - Compactor

      - 读当前snapshot 的数据文件，compact 成新的数据文件

2. 读最新的 snapshot，假设为 1，build 下一个 snapshot 2 文件

   - Writer 和 Compactor 都基于最新的 snapshot 1 来 build 下一个 snapshot，写base minifest 文件， delta minifest 文件，一个临时的 snapshot 文件

3. 原子性 rename snapshot 文件，以提交 snapshot

   - 重命名这个临时的 snapshot 文件为正式的 snapshot 2 文件

      - 如果 Writer 先 rename 成功了，则 Compactor rename 将会失败，跳转到第2步，重新进行提交，不会有一致性问题

ii. 如果 Compactor 先 rename 成功，Writer 的 rename 将会失败，跳转到第2步，重新进行提交，也不会有一致性问题

由于总是原子性基于最新的 snapshot 来提交 snapshot，提交 snapshot 这个动作永远都是可序列化的，避免了元数据的冲突。并且不同 Writer 写入的都是不同 bucket 的数据，避免了数据冲突，所以不会存在一致性的问题。

### 多 writer 写相同的 bucket

Topology 如下所示：

![](images/img_08.png)

在这种多 writer 写相同 bucket 的 Topology 是会存在一致性的问题。

#### 一致性问题1：更新丢失

由于没有像 Iceberg 一样的数据冲突检测，Paimon 会存在数据丢失的问题。考虑如下的两种 case：

##### Case1：

![](images/img_09.png)

假设一开始有一条数据 {‘Jack’, ‘Yellow’, ‘Hiking’}

1. Writer1 基于快照1读出数据 {‘Jack’, ‘Yellow’, ‘Hiking’}

2. Writer2 基于快照1读出数据 {‘Jack’, ‘Yellow’, ‘Hiking’}

3. Writer1 准备将 FavColor 字段修改为 ‘Blue’，写一条 +U 的数据，{‘Jack’, ‘Blue’, ‘Hiking’}

4. Writer1 提交成功，表的最新快照变为快照2

5. Writer2 准备将 FavHobby 字段修改为 ‘Cycling’，写一条 +U 的数据，{‘Jack’, ‘Yellow’, ‘Cycling’}，注意，它是基于 snapshot 1 的数据进行修改的，所以 FavColor 字段 还是 ‘Yellow’

6. Writer2 准备提交，提交失败，因为当前快照已经变为 2

7. Writer2 refresh 一下得到最新的 快照2，因为没有数据冲突检测，提交成功。

最终这条数据变为 writer2 写的这条 {‘Jack’, ‘Yellow’, ‘Cycling’} 数据会把 writer1 写的这条 {‘Jack’, ‘Blue’, ‘Hiking’} 数据覆盖掉，于是只剩 {‘Jack’, ‘Yellow’, ‘Cycling’} 这条数据，导致 Writer1 的写入丢失了。

##### Case2：

![](images/img_10.png)

假设一开始有一条数据 {‘Jack’, 1}，Writer1 和 Writer2 都准备给 ‘Jack’ 的积分加1

假设一开始有一条数据 {‘Jack’, 1}

1. Writer1 基于快照1读出数据 {‘Jack’, 1}

2. Writer2 基于快照1读出数据 {‘Jack’, 1}

3. Writer1 准备将积分字段加1，修改为 {‘Jack’, 2}，写一条 +U 的数据， {‘Jack’, 2}

4. Writer1 提交成功，表的最新快照变为 快照2

5. Writer2 准备将 积分字段加1，修改为 {‘Jack’, 2}，写一条 +U 的数据， {‘Jack’, 2}，注意，它是基于 snapshot 1 的数据进行修改的

6. Writer2 准备提交，提交失败，因为当前快照已经变为 2

7. Writer2 refresh 一下得到最新的 快照2，因为没有数据冲突检测，提交成功。

最终这条数据变为 {‘Jack’, 2}，破坏了可序列化隔离性

#### 一致性问题2：悬空的 deletion vector

如果两个 compaction 作业同时进行，可能会存在悬空 deletion vector ，即 deletion vector 文件指向一个已经被标记为删除的数据文件。

1. Compactor1 将 L0 层的数据 compact 到 L1 层，并创建一个 deletion vector 指向 L2 层的这个有相同 PK 的老数据，假设为 data-file1

2. Compactor2 compaction L2 层 的数据，重写 data-file1 到另外一个 data-file2，虽然这个时候 data-file1 其实已经被标记为删除了，但是还是有 deletion vector 指向这个 data-file1

### 总结

Paimon 在每个 writer 写不同的 bucket 的 topology 下，不存在数据一致性问题，但是会在多 writer 写相同的 bucket 的 topology 下存在数据一致性问题，但是这样的 topology 并不是 Paimon 推荐的 topology， Paimon 社区也没怎么遇到过。不过如果要支持这样的 topology，做一下数据冲突检测也不会花费很大工作量。

但是其实也有好处，就是如果你用了正确 topology 写 Paimon，就不会有数据冲突，数据冲突对使用体验，性能还是有很大的影响。
