---
title: "浅浅聊一聊四大湖格式的内部机制和一致性模型 - Iceberg 篇"
date: 2025-09-06T18:49:52+08:00
draft: false
tags: ["数据湖"]
categories: ["数据湖"]
summary: "本文深入解析Iceberg数据湖格式的内部机制与一致性模型，涵盖写入流程、快照管理、并发控制及冲突检测机制，确保多写者场景下的数据一致性。"
ShowToc: true
TocOpen: false
---

## 前言

最近这段时间致力于 [Fluss](https://github.com/apache/fluss) 与各大数据湖进行结合的工作，一直很想找个时间快速地，系统地学习下目前主流的数据湖格式。直到后来我看到了Jack 老哥的专题文章 [The ultimate guide to table format internals - all my writing so far](https://jack-vanlightly.com/blog/2024/10/28/the-ultimate-guide-to-table-format-internals)，浅浅阅读了一下，根据 Jack 老哥的一系列文章和自己的理解，梳理总结一下，帮助自己理解各大数据湖格式。

对于每一种数据湖格式，都会覆盖数据湖格式的内部机制（如何将数据组成成文件，如何将文件组织成完整的表数据），以及一致性模型（处理多 writer 写的情况，如何保证数据的一致性）。

PS：我看了 Jack 的很多文章，写的都非常好，建议大家可以去读一读。

首先来聊一下 Iceberg：

## 内部机制

### 写入流程

Iceberg 写入分为如下三步：

1. 将数据写入数据文件

2. 写一个 snpashot 文件来记录所有的数据文件，包括之前写入的数据文件和这次写入的数据文件

3. 将这个 snpashot 文件的 path 提交到 Catalog 中，Catalog 作为 source of truth，记录当前数据的版本。查询引擎从 Catalog 得到当前数据版本的 snpashot 文件的 path，从这个 snpashot 文件 list 出所有的数据文件，进行读取。

对于第2步写 snapshpt 文件：

其中 snpashot 文件记录数据文件的方式如下所示：

![](images/img_01.png)

Snapshot 会指向一个 manifest list 文件，这个manifest list 文件包含若干个 manifest，每个 manifest 会指向多个数据文件，一个 manifest 会包含多个 manifest entry，每个 manifest entry 执行一个数据文件

下面是一个 Iceberg 表有多个 snapshot 的例子：

![](images/img_02.png)

对于第3步提交 snapshot 文件 path 到 catalog 这一步，要求必须是原子 CAS 操作，不然就会出现数据丢失的情况。

比如当前 catalog 记录的 snapshot 是 snapshot1-<uuid>，这个时候有两个 writer，W1，W2 同时进行写入，W1 基于 snapshot1 写了一个 snapshot-2-<uuid> 文件，W2 也基于 snapshot1 写一个snapshot-2-<uuid> 文件，如果提交不是原子 CAS 操作，那么 W1 可以提交成功，W2 也可以提交成功，这就会导致 W1 的写入被 W2 覆盖了，即 W1 这次写入的数据丢失了。

Iceberg 通过原子 CAS 的提交（通过外部的 catalog lock）避免这个问题，W2 提交的时候发现当前 catalog 记录的 snapshot 是 snapshot2了，就会abort 自己，不进行提交，如下所示：

![](images/img_03.png)

### 如何记录数据文件的添加和删除

所有的数据湖格式都需要记录每个 snapshot 添加了哪些文件，删除了哪些文件。Iceberg 通过 Manifest 文件来进行记录。对于每个 Manifest 文件：

- 会有一个字段 `added_in_snapshot` 来记录这个 Manifest 文件是在哪个 snapshot 添加的。

- Manifest 文件有多个 manifest entry，每个 manifest entry 指向具体的文件，会记录这个具体的文件的 status - ADDED（新增的文件），EXISTING（之前 snapshot 写入的文件），DELETED（删除的文件）

下面是一个具体的示例：

![](images/img_04.png)

### Copy-on-write & merge on read

为了支持数据的删除和修改，即主键表模型，Iceberg 提供了两种模式：

#### Copy-on-write（COW）

任何数据的修改都需要重写整个数据文件。写不友好，因为需要重写整个文件。读友好，不需要额外的 Merge 操作，直接读重写后的数据文件即可。

大致流程如下所示：

![](images/img_05.png)

Iceberg metadata 如下所示：

![](images/img_06.png)

注意：Snapshot 2 的 metadata 需要把 data-1 文件标记为删除。

#### Merge-on-Read（MOR）

数据的修改只需要写新的文件来记录数据的修改。写友好，只需要在写的文件中写被修改的数据。读不友好，读的时候需要进行额外的 merge，将新的文件和之前的数据文件 merge 起来，得到最终的数据。

Iceberg 将数据的 Update 抽象成一次 delete，一次 insert，所以 iceberg 需要标识一下哪条数据被删除了，也需要写新的数据。写新的数据比较简单，就是直接写一个 data file 就可以了。

对于 “标识哪条数据被删除了”，iceberg 提供有两类 delete file 来进行标识：

- position delete file，就是在 position delete file 中记录哪个数据文件的第几条记录被删除了。

如下所示：

![](images/img_07.png)

注意：这里会写两个文件，一个是 delete 文件，来标识老的数据被删除了。一个是新的 data（数据） 文件，记录新的数据。

Iceberg metadata 如下所示：

![](images/img_08.png)

- Equality delete file：虽然 position delete file 很高效，但是 position delete file 的一个问题是计算引擎需要知道要删除的数据所在文件的 pos，写的时候需要反查，代价比较高。Equality delete file 则是用来解决这个问题的，Equality delete file 就是记录 “删除的数据的过滤条件”，任何满足这个过滤条件的数据都会被认为删掉了。

如下所示：

![](images/img_09.png)

注意，现在 delete 文件的内容变成 要删除的数据的过滤条件了，即 "name = jack"

Iceberg metadata 如下所示：

![](images/img_10.png)

我们注意到，manifest 额外多了一个 seq\_no 的字段，这是用来避免 Equality delete file 把不应该被删除的数据标记为删除。比如在 snapshot 3，我新增了一个 name=jack 的数据，如果还对这条新的数据应用 Equality delete file 就会导致这条数据不会被读到，Equality delete file 只应该 apply seq\_no < 2 的 data file 的数据。

#### Compaction

Iceberg 支持 compaction action 通过重写数据文件来减少数据文件数量，提高读取效率，如下图所示：

![](images/img_11.png)

数据最终被 compaction 成一个数据文件

## 一致性模型

### 深入理解 iceberg writer 写入流程

Iceberg writer 的写入流程（只考虑 MERGE，DELETE，UPDATE 的 SQL 语句）如下图所示：

![](images/img_12.png)

1. Scan 当前快照的 DataFile，记录下当前快照的 snapshot id，这个 snapshot id 是用于后面冲突检测的

2. 写数据文件

3. 刷新 metadata，得到此时最新的 metadata

4. 进行数据的冲突检测（后面会详细解释），看是否存在数据写入冲突，有的话，就直接 abort 这次写入

5. 写一个新的 snapshot 文件，来记录这次的写入的文件

6. 提交这个 snapshot 到 catalog，这个一个原子的 CAS 的操作，看 catalog 当前记录的 snapshot 是不是这个 writer 认为最新的 snapshot，如果是的话，提交成功。如果不是的话，返回第 3 步，进行重试。

### 数据冲突检测

Iceberg 通过数据冲突的检测来保证多 writer 同时进行写入的情况下，将有数据冲突的写入 abort，从而提供数据的一致性。

接下来我们来看下 Iceberg 支持的数据操作，以及其潜在的数据冲突；

#### AppendFiles 操作

这个操作比较简单，是针对非主键表而言的，就是向 Iceberg 表中追加数据文件，这个是不会有冲突的，所以 Iceberg 不会对 AppednFiles 操作进行冲突检测。

#### Overwrite（Copy-on-write） 操作

Overwrite 操作会 添加新的数据文件 && 将已有的数据文件标记为已删除。该操作可能存在以下的冲突：

1. 两个 Overwrite 操作都将相同的数据文件标记为删除，然后都写了不同的新数据文件，如下图所示：

![](images/img_13.png)

这样，data-2 和 data-3 都会被认为是要读的数据文件，造成数据的重复和不一致。

2. Overwrite 操作和 RowDelta（Merge-on-read） 操作的冲突。RowDelta 首先通过 delete file 将一个数据文件 data1-file 的某条数据 标记为已删除。然后这个时候 Overwrite 操作重写了数据文件 data1-file 成 data2-file，将 data1-file 的这个数据文件标记为已删除。于是在 Iceberg 中，虽然 数据文件 data1-file 被标记为已删除，但是 delete file 中的 delete entry 还依然引用着数据文件 data1-file，出现不一致的情况。

Iceberg 通过如下三种校验来避免 Overwrite 的冲突：

- **Fail missing delete paths validation**

  思路很简单，就是看一下这个操作要删除的文件是不是已经被标记为删除了。以上面的第一种冲突为例，Operation B 提交的时候，会发现它要删除的文件 data1-file 已经被标记为删除了，于是就检测到冲突了。

- **No new deletes for data files validation**

  检测从这个 Overwrite 操作开始时对应的 snapshot id 开始，到当前最新的 snapshot id 是否有同时满足如下条件的 delete 文件，如果有，则认为数据冲突了：

   - delete 文件的 sequence number > Overwrite 操作开始时对应的 sequence number（表示是在Overwrite 操作开始后添加的）
   - delete 文件是由 Delete 或者 OverWrite 操作添加的
   - delete 文件 匹配 Overwrite 操作用到的过滤条件（可下推的），比如 update xxx where = jack，其中 where = jack 就是这个过滤条件。Iceberg 会通过文件的统计信息看是否匹配。注意会存在 统计信息认为匹配，但是其实并不匹配的情况，这个没办法避免。如果过滤条件不设置的话，就会认为所有的 delete 文件都匹配。 如果 delete file 没有统计信息，比如 pos delete file，也会认为匹配。
   - delete 文件引用了一个被这个 Overwrite 操作标记为删除的数据文件

  这个可以解决上面提到的第二种冲突。

- **Added data file validation**

  检测从这个 Overwrite 操作开始时对应的 snapshot id 开始，到当前最新的 snapshot id 是否有同时满足如下条件的数据文件，如果有，则认为数据冲突了。

   - 被 APPEND 或者 OverWrite 操作添加
   - 数据文件匹配 Overwrite 操作用到的过滤条件（可下推的），比如 update xxx where = jack，其中 where = jack 就是这个过滤条件。Iceberg 会通过数据文件的统计信息看是否匹配。注意会存在 统计信息认为匹配，但是其实并不匹配的情况，这个没办法避免。如果过滤条件不设置的话，就会认为所有的数据文件都匹配。

Iceberg 的这三种冲突检测实际是需要上层引擎手动调用进行检测，Spark 基于上述的冲突检测，实现了快照隔离和可序列化隔离。

Spark 实现快照隔离：调用 validateNoConflictingDeletes 开启 Fail missing delete paths validation 和 No new deletes for data files validation 检测。只检查是否有新的删除文件影响了它要覆盖的数据，不检查是否有新的数据文件出现在它要覆盖的范围内。但是会存在如下的一个问题，其实也是快照隔离级别会出现的写倾斜的问题：

考虑一个经典的医生值班的例子，假设医院规定，任何时候至少必须有一位医生在值班，表的数据如下所示：

表：`doctors_on_call`

| doctor\_id | name | is\_on\_call |
|------------|------|-------------|
| 1 | Alice | true |
| 2 | Bob | true |

Alice 和 Bob 通过 MERGE INTO 语句执行操作，如果还有其他人在值班，我就不值班。

**MERGE INTO 语句**

```sql
-- Alice 执行的语句
MERGE INTO doctors_on_call target
USING (SELECT 1 as doctor_id) source
ON (target.doctor_id = source.doctor_id)
WHEN MATCHED AND EXISTS (
    SELECT 1 FROM doctors_on_call 
    WHERE is_on_call = true AND doctor_id != 1 -- 确保还有别人在值班
) THEN
  UPDATE SET target.is_on_call = false;

-- Bob 执行的语句
MERGE INTO doctors_on_call target
USING (SELECT 2 as doctor_id) source
ON (target.doctor_id = source.doctor_id)
WHEN MATCHED AND EXISTS (
    SELECT 1 FROM doctors_on_call 
    WHERE is_on_call = true AND doctor_id != 2 -- 确保还有别人在值班
) THEN
  UPDATE SET target.is_on_call = false;
```

于是：

1. Alice 基于当前快照1 执行语句，Bob 还在值班，将自己的 is\_on\_call 设置为 false。即 Iceberg 将 Alice 所在数据文件 data-file1 标记为删除，写一个新的数据文件 data-file3 ，记录 <Alice, false> ，准备提交

2. Bob 也基于当前快照1 执行语句，Alice 还在值班，将自己的 is\_on\_call 设置为 false。即将Iceberg 将 Bob 所在数据文件 data-file2 标记为删除，写一个新的数据文件 data-file4，记录 <Bob, false> ，准备提交

3. Bob 提交成功，没有任何冲突

4. Alice 也提交成功，因为它也没检测到任何冲突

   - Fail missing delete paths validation，检测通过，因为 Alice 要标记删除的 data-file1 并没有被 Bob 标记为删除

   - No new deletes for data files validation，检测通过，因为没有任何 delete 文件

Spark 实现可序列化隔离：调用 validateNoConflictingDeletes 和 validateNoConflictingData 开启 Fail missing delete paths validation ， No new deletes for data files validation ，Added data file validation 检测。既检查是否有新的删除文件，也检查是否有新的数据文件出现在它要操作的范围内。对于上面提到的医生值班的例子，Alice 提交的时候，进行 Added data file validation 检测，发现 Bob 添加了新的数据文件，于是直接 fail。注意这里没办法用到过滤条件，因为过滤条件是 exists，不能下推。

#### RowDelta（Merge-On-Read） 操作

RowDelta 操作会 添加新的数据文件 + 新的 delete 文件来标记数据被删除了。该操作可能存在以下的冲突：

1. RowDelta 操作创建了一个新的 delete 文件来标记某个数据文件 data1-file 的某条数据被删除了，但是同时另外一个 Overwrite 操作已经把 data1-file 标记为已删除。如下图所示：

![](images/img_14.png)

如果 Opeation B 不做任何冲突检测的话，最后会变成：

![](images/img_15.png)

于是会有两条 name = sarah 的数据， <sarah, orange>，<sarah, cherry>

2. 两个 RowDelta 修改相同的一条数据，一个 RowDelta 删除这条数据，另外一个 RowDelta 更新这条数据。于是就会导致两个 delete 文件都指向了相同的一条数据，一个数据文件包含更新后的这条数据。出现冲突的 case 如下所示：

   - Writer 1 开始 update 操作，UPDATE Fruits SET FavFruit = 'banana' WHERE Name = 'jack'
   - Writer 2 开始 delete 操作，DELETE FROM Fruits WHERE Name = 'jack'
   - Writer 1 添加一个 delete 文件，标记现有的 data-1 文件中的 'jack' 这一条数据无效，然后添加一条数据 <"jack", "banana"> 到 data-2 文件
   - Writer 2 也添加一个 delete 文件，标记现有的 data-1 文件中的 'jack' 这一条数据无效
   - Writer 1 成功提交
   - Writer 2 也成功提交

于是 Reader 仍然能读到 <“jack”, “banana”> ，但是不论是 Writer 1 先执行，然后 Writer 2 后执行；还是 Writer 2 先执行，Writer 1 后执行。都不应该读到 <“jack”, “banana”> 这条数据。

Iceberg 通过如下三种校验来避免 RowDelta 的冲突：

- **Data files exist validation**

  检测从这个 RowDelta 操作开始时对应的 snapshot id 开始，到当前最新的 snapshot id 是否有同时满足如下条件的数据文件，如果有，则认为数据冲突了：

   - 已经被标记为删除了
   - 是由 Overwrite 操作将其标记为删除的
   - RowDelta 操作的 delete file 会引用这个数据文件

  在上面的 case1 中，Opeation B 的 delete file 引用了 data-1 file，但是 data-1 file 已经在 snapshot-2 中被 Overwrite 操作标记为删除了，于是检测到冲突了，Opeation B 会 abort。

- **No new delete files validation**

  检测从这个 RowDelta 操作开始时对应的 snapshot id 开始，到当前最新的 snapshot id 是否有同时满足如下条件的 delete 文件，如果有，则认为数据冲突了：

   - delete 文件的 status 为 ADDED，表示是这期间新增的delete 文件
   - delete 文件的 sequence number 大于 RowDelta 操作开始时对应的 sequence number（表示是在RowDelta 操作开始后添加的）
   - delete 文件是由 Delete 或者 OverWrite 操作添加的
   - delete 文件 匹配 RowDelta 操作用到的过滤条件

  对于上面提到的 case 2的情况，Writer 2 的时候发现 writer1 新增加了一个 delete 文件，就会直接 abort 掉自己的提交。

- **Added data file validation**

  和 Overwrite 的 Added data file validation 完全一样，这里重复一下检测流程：

  检测从这个 RowDelta 操作开始时对应的 snapshot id 开始，到当前最新的 snapshot id 是否有同时满足如下条件的数据文件，如果有，则认为数据冲突了。

   - 被 APPEND 或者 OverWrite 操作添加
   - 数据文件匹配 RowDelta 操作用到的过滤条件，比如 update xxx where = jack，其中 where = jack 就是这个过滤条件。Iceberg 会通过数据文件的统计信息看是否匹配。注意会存在 统计信息认为匹配，但是其实并不匹配的情况，这个没办法避免。如果过滤条件不设置的话，就会认为所有的数据文件都匹配。

#### RewriteFiles 操作（即 compaction）

执行如下两种冲突检测来避免 RewriteFiles 操作和 OverwriteFiles，RowDelta，RewriteFiles 操作冲突

- Fail missing delete paths validation

- No new deletes for data files validation
