---
title: "浅浅学习一下 Mooncake - 如何让 Postgres 的每一次写入，Iceberg 都能实时看见"
date: 2026-04-14T21:44:51+08:00
draft: false
tags: ["数据库", "数据湖"]
categories: ["数据库"]
summary: "Mooncake通过GlobalIndex实时生成DeletionVector替代低效EqualityDelete，并结合UnionRead将内存Arrow批次、磁盘Parquet与多级删除信息统一查询，实现Postgres到Iceberg的毫秒级实时同步与分析。"
ShowToc: true
TocOpen: false
---

## 背景

[Mooncake Labs](https://github.com/Mooncake-Labs) 团队核心成员来自 SingleStore,SingleStore（前身 MemSQL）是一家做 HTAP（Hybrid Transactional/Analytical Processing）的数据库公司，试图在一个引擎里同时搞定 OLTP 和 OLAP。

有意思的是，从 SingleStore 出来后，Mooncake 团队选择了一条不同的路线：不做 HTAP，而是让 Postgres 专注 OLTP，通过实时同步把数据镜像到 Iceberg 列存，分析交给 DuckDB。 两个引擎各做各自擅长的事，中间用 Mooncake 做实时桥接，实现 Postgres 上的实时分析。

2025 年 10 月，Databricks 宣布收购 Mooncake Labs，将其技术整合进 Lakebase——Databricks 正在构建的基于 Postgres 的 OLTP 数据库，面向 AI Agent 场景。收购的核心价值在于 Mooncake 的实时同步能力：Postgres 的变更实时镜像到 Lakehouse，应用、分析和 AI 共享同一份新鲜数据，不需要 ETL。

> 因为之前 Mooncake 在 CMU 分享的时候 cue 到了 Fluss，说借鉴了 Fluss 和 Paimon 的 UnionRead 的思路，所以一直想找一个机会学习下 Mooncake 的原理。刚好最近开始在看 Fluss 与 Paimon/Iceberg 主键表 deletion vector 的结合，解锁主键表的极致查询，所以浅浅学习一下。

## 解决的痛点

把 Postgres CDC 实时写进 Iceberg，用 Flink 就能做。但核心痛点有两个：

1. 删除只能靠 Equality Delete，查询越来越慢

Iceberg 支持两种删除方式：Equality Delete（按值删，记录"id=42 被删了"）和 Deletion Vector（按位置删，标记"第 105 行被删了"）。Deletion Vector 查询时只需按位过滤，效率远高于 Equality Delete。

但 Flink 流式写入生成不了 Deletion Vector——因为数据写进 Parquet 后就"忘了"每行的位置，收到一条 `DELETE id=42` 时，根本不知道 id=42 在哪个文件的第几行。所以只能退回到 Equality Delete：每个 checkpoint 周期把累积的删除攒成一个 Equality Delete 文件，查询时要和数据文件做 JOIN。checkpoint 频率越高，Equality Delete 文件堆积越快，查询代价线性增长。

2. 实时写入的数据不可查

CDC 流到 Iceberg 的数据要等 checkpoint 提交（通常是分钟级）后才对查询可见，中间这段"实时数据"查不到，实时分析也无从谈起

## 怎么解决

Mooncake 针对这两个问题给出了方案：用 Deletion Vector 替代 Equality Delete，用 Union Read 让实时数据也能被查到，整体架构如下图所示：

```
Postgres WAL
     |
     | CDC
     v
+-----------+    flush     +----------------+   persist   +------------------+
| Arrow     | -----------> | Parquet + DV   | --------->  | Iceberg (S3/GCS) |
| in-memory |              | on NVMe        |             | Parquet + Puffin |
+-----------+              +----------------+             +------------------+
     |                            |                              |
     |   BatchDeletionVector      |   Position Delete            |  Deletion Vector
     |   (MemIndex)               |   (FileIndex)                |  (Puffin)
     |                            |                              |
     +----------------------------+------------------------------+
                                  |
                           Union Read (DuckDB)
```

### 怎么实时生成 Deletion Vector

前面说了，Flink 生成不了 Deletion Vector 是因为写完就忘了行在哪。Mooncake 的思路很直接：写入时就建索引，记住每行在哪个文件的第几行，删除时查索引拿到位置，直接生成 Deletion Vector。

核心是一个 GlobalIndex （FileIndex）——全局哈希索引，维护 `主键 → (file_id, row_offset)` 的映射。每当一行数据落盘写入 Parquet 文件，就在索引里记录它的位置。索引热数据在内存，冷数据持久化到 NVMe。

当一条 `DELETE id=42` 从 CDC 流过来，流程就是：

1. 查 GlobalIndex：`id=42 → (data-file-7, 第 105 行)`
2. 在 data-file-7 对应的 Deletion Vector 中标记第 105 行已删除

Parquet 是不可变的，不能原地改，所以只能通过外挂的 Deletion Vector 来标记删除。每个 Parquet 文件对应一个 Deletion Vector（位图）。

UPDATE 被拆成 DELETE + INSERT：先查索引在旧位置标记删除，再在新位置追加一行并更新索引。

### GlobalIndex

#### Data 写入后的 GlobalIndex 的更新

GlobalIndex 本质是一个持久化的分桶哈希表（Bucket Hash Map），维护 `主键 → (file_id, row_offset)` 的映射。

什么时候生成：每当内存中的 Arrow 数据 flush 到 Parquet 文件时，同步构建这批数据的索引。flush 过程中把每行的 `(主键哈希值, 文件内序号, 行号)` 收集起来， 构建一个索引块（IndexBlock）。每次 flush 产生一个新的索引块，注册到全局索引中。

内部结构：

```
主键 id=42
    │
    │ splitmix64 哈希
    ▼
哈希值: 0xA3B1...  (64 bit)
    │
    ├── 高 N 位 → 桶编号 (bucket index)
    └── 低 (64-N) 位 → 存入桶内，用于精确匹配

桶数组 (长度 = num_buckets，2 的幂次)
┌─────────┬─────────┬─────────┬─────────┐
│ bucket 0│ bucket 1│ bucket 2│   ...   │
└────┬────┴─────────┴────┬────┴─────────┘
     │                   │
     ▼                   ▼
 ┌──────────────┐   ┌──────────────┐
 │ hash_low_bits│   │ hash_low_bits│
 │ file_id      │   │ file_id      │
 │ row_offset   │   │ row_offset   │
 └──────────────┘   └──────────────┘
```

桶的数量根据这批数据的行数动态计算：`num_buckets = next_power_of_two(num_rows / 4 + 2)`。比如 flush 了 1000 行，桶数就是 256（`(1000/4+2)` 向上取 2 的幂）。每个桶平均约 4 个条目，保证查询时桶内遍历很短。

怎么定桶：桶数量一定是 2 的幂，所以 `N = log2(桶数量)`，取哈希值的高 N 位就是桶编号。比如 256 个桶，N=8，取高 8 位。位移即可，不需要取模。每个索引块在构建时把自己的 N 记录为 `hash_upper_bits`，查询时按这个值取高位定桶。

每次 flush 产生一个独立的索引块，索引块之间互不影响。如果存在多个索引块，查询时需要逐个索引块查——在每个索引块内部是 O(1) 哈希定桶，但索引块多了查询次数就多。所以需要后台做索引合并：合并后只剩一个索引块，查一次就够了。

桶内每个条目存三个字段：`(哈希低位, file_id, row_offset)`，全部用位压缩紧凑编码——file_id 只用 `log2(文件数)` 个 bit，row_offset 固定 32 bit——尽量压缩索引体积。整个索引块序列化到磁盘文件，运行时通过 mmap 映射到内存，查询时不需要额外的磁盘 I/O。

写入、查询、合并的完整流程：

用一个例子串起来。假设系统陆续 flush 了三批数据：

```
第 1 次 flush: 1000 行 → parquet-1 + IndexBlock-1 (256 桶, N=8)
第 2 次 flush:  500 行 → parquet-2 + IndexBlock-2 (128 桶, N=7)
第 3 次 flush: 2000 行 → parquet-3 + IndexBlock-3 (512 桶, N=9)
```

每次 flush 都是独立的：产生一个新的 Parquet 文件和一个新的索引块，索引块只覆盖这一批数据，桶数量根据行数独立计算。已有的索引块不会被修改——和 Parquet 一样，写完就不可变。

现在收到一条 `DELETE id=42`，查询流程：

```
splitmix64(id=42) → 哈希值 0xA3B1...

  查 IndexBlock-1 (N=8): 取高 8 位 → bucket 163 → 桶内遍历 → 未命中
  查 IndexBlock-2 (N=7): 取高 7 位 → bucket 81  → 桶内遍历 → 未命中
  查 IndexBlock-3 (N=9): 取高 9 位 → bucket 326 → 桶内遍历 → 命中！
                          → (parquet-3, 第 105 行)
```

每个索引块内部定桶是 O(1)（位移取高位），桶内平均 4 个条目，遍历很快。但索引块越多，要查的次数越多。

索引合并就是解决这个问题：Mooncake 后台把多个小索引块归并成一个大索引块。

```
合并前: IndexBlock-1 (256桶) + IndexBlock-2 (128桶) + IndexBlock-3 (512桶)
           ↓ build_from_merge：归并迭代器按哈希值有序归并
合并后: IndexBlock-merged (1024桶, N=10)
```

合并后所有数据在同一个索引块里，查询一次定桶就够了。这个思路和 LSM-Tree 的 compaction 一样：写入只追加新块，读取多路查找，后台合并减少读放大。

#### Data Compaction 后 GlobalIndex 的更新

如果只考虑写入，GlobalIndex 的更新比较简单——每次 flush 追加一个新索引块就行。

但 Compaction 后就复杂了：多个小 Parquet 文件合并成大文件，行的物理位置全变了，索引里记录的 `(file_id, row_offset)` 全部失效。

Mooncake 自己的 Compaction 流程同时处理数据文件和索引：

1. 合并数据文件 ：读取多个小 Parquet，应用各自的 Deletion Vector 过滤掉已删除的行，把存活的行写入新的大 Parquet 文件。写入过程中记录每一行的位置映射：`旧 (file_id, row_offset) → 新 (file_id, row_offset)`。

2. 重建索引 ：拿到位置映射表后重建索引。用一个具体例子：

```
Compaction 前：
  parquet-1: [row0: id=10, row1: id=20, row2: id=30(已删除)]
  parquet-2: [row0: id=40, row1: id=50]

  IndexBlock-1:  id=10 → (parquet-1, row0)
                 id=20 → (parquet-1, row1)
                 id=30 → (parquet-1, row2)
  IndexBlock-2:  id=40 → (parquet-2, row0)
                 id=50 → (parquet-2, row1)

          ↓ 合并数据文件，应用 Deletion Vector，id=30 被过滤

Compaction 后：
  parquet-new: [row0: id=10, row1: id=20, row2: id=40, row3: id=50]

  位置映射表：
    (parquet-1, row0) → (parquet-new, row0)   ✓ 存活
    (parquet-1, row1) → (parquet-new, row1)   ✓ 存活
    (parquet-1, row2) → 无映射                ✗ 已删除
    (parquet-2, row0) → (parquet-new, row2)   ✓ 存活
    (parquet-2, row1) → (parquet-new, row3)   ✓ 存活

          ↓ 遍历旧索引条目，查映射表，替换位置

  IndexBlock-new: id=10 → (parquet-new, row0)
                  id=20 → (parquet-new, row1)
                  id=30 → 跳过（映射表中不存在）
                  id=40 → (parquet-new, row2)
                  id=50 → (parquet-new, row3)
```

重建索引的详细流程：

第一步：构建归并迭代器。 Compaction 不会重写所有文件，只选一部分小文件（或删除比例高的文件）合并。选中数据文件的同时，也会选中这些文件关联的索引块。把这些被选中的旧索引块合在一起，创建一个归并迭代器（MergingIterator），按哈希值顺序依次吐出每一个旧条目 `(hash, old_file_id, old_row_offset)`。没被选中的文件和索引块不受影响。

第二步：逐条处理。 对迭代器吐出的每一个旧条目：

- 用 `(old_file_id, old_row_offset)` 去查位置映射表
- 查到了 → 拿到新位置 `(new_file_id, new_row_offset)`，写入新索引块
- 查不到 → 说明这行在 Compaction 中已被 Deletion Vector 过滤，直接跳过

第三步：生成新索引块。 所有旧条目处理完后，新索引块构建完成——只包含存活行，指向 Compaction 后的新 Parquet 文件。

```
旧索引块                        位置映射表                     新索引块
┌──────────────┐
│ IndexBlock-1 │──┐
└──────────────┘  │
                  ├→ 归并迭代器 ─→ 逐条输出旧条目
┌──────────────┐  │                    │
│ IndexBlock-2 │──┘                    │
└──────────────┘                       ▼
                            ┌─────────────────────┐
                 旧条目 ──→  │ 查位置映射表          │
                            └──────┬──────────────┘
                                   │
                        ┌──────────┴──────────┐
                        │                     │
                     查到映射               未查到
                  (行存活)              (行已删除)
                        │                     │
                        ▼                     ▼
                 替换为新位置              直接跳过
                 写入新索引块
                        │
                        ▼
               ┌────────────────┐
               │ IndexBlock-new │  只包含存活行
               │ 指向 parquet-new│  的全新索引块
               └────────────────┘
```

整个过程是一次线性扫描：归并迭代器保证每个旧条目只访问一次，映射表查询是 O(1) 的哈希查找，所以重建索引的时间复杂度是 O(旧条目总数)，和数据量成正比，没有额外的放大。

3. 原子替换 ：新的数据文件和索引块准备好后，在原子地替换掉旧的文件和索引。旧文件随后被清理。

所以 Compaction 不只是合并数据文件，而是数据文件和索引的联动更新。这也是 Mooncake 选择自己做 Compaction 而不依赖外部 Spark 的原因之一——外部引擎没法同步更新 Mooncake 的 GlobalIndex。

目前 Mooncake 只用自己的引擎做 Compaction，好处是数据文件和索引可以在同一个流程里联动更新，实现简洁。但代价是 Compaction 的吞吐受限于单机——数据量大了之后，靠自己做 Compaction 会成为瓶颈。

### 怎么做 Union Read

前面解决了第一个痛点（用 GlobalIndex 实时生成 Deletion Vector 替代 Equality Delete）。Union Read 解决第二个痛点：让还没落盘到 Iceberg 的实时数据也能被查到。

#### 三个数据源

一次查询需要合并三个数据源：

```
                       DuckDB Query
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
     Iceberg Parquet   Arrow Batches   Deletions
      (persisted)      (in-memory)        |
                                   +------+------+
                                   |      |      |
                                   v      v      v
                               DV     PD     BatchDV
                            (Puffin) (list) (in-memory)

DV     = Deletion Vector, persisted in Puffin
PD     = Position Delete, committed but not persisted
BatchDV = BatchDeletionVector, in-memory batch bitmap
```

1. Iceberg Parquet 文件 ：已经持久化到对象存储的数据，这是存量。

2. 内存中的 Arrow 批次 ：Postgres CDC 过来的数据先写到内存的 Arrow 缓冲区，还没 flush 到 Parquet。这部分是"实时增量"。

3. 删除信息 ：一条 DELETE 来了之后，需要先找到目标行在哪。前面讲过 FileIndex（GlobalIndex）负责定位磁盘上的行，但内存中的行它管不了。所以 Mooncake 的索引实际上是两层：
   - FileIndex （前面详细讲过的 GlobalIndex）：维护 `主键 → (file_id, row_offset)`，定位磁盘 Parquet 文件中的行。持久化的分桶哈希表，mmap 到内存。
   - MemIndex ：维护 `主键 → (batch_id, row_offset)`，定位内存 Arrow 批次中的行。纯内存的哈希表（hashbrown::HashTable），每个 Arrow 批次对应一个 MemIndex，行写入内存时同步插入。

   DELETE 来了先查 MemIndex，再查 FileIndex，找到目标行后根据位置不同产生三种删除：
   - Deletion Vector （已持久化）：已经写入 Iceberg Puffin 文件的删除位图，查询时按位过滤。针对磁盘上的 Parquet 文件。
   - Position Delete （已提交但未持久化）：FileIndex 命中，查到目标行在磁盘文件的 `(file_id, row_offset)`，但还没来得及写入 Iceberg Puffin。以 `(文件编号, 行号)` 列表的形式传给 DuckDB。
   - BatchDeletionVector （内存删除）：MemIndex 命中，目标行还在内存 Arrow 批次中，直接在该批次的位图里标记删除。这种删除不需要传给 DuckDB——内存批次序列化到临时 Parquet 的过程中会应用这个位图，已删除的行直接被过滤掉，不会出现在临时文件里。

#### ReadState：把三个源打包成一个快照

Mooncake 在收到查询请求时，会组装一个 ReadState ，把上述三个数据源打包成一个一致性快照：

1. 收集 Iceberg 数据文件 ：从当前快照的 `disk_files` 中取出所有已持久化的 Parquet 文件路径。

2. 序列化内存数据 ：把内存中已提交的 Arrow 批次写到一个临时 Parquet 文件。注意只包含已提交的部分——根据 `last_commit` 位置截断，未提交的事务不可见。

> 注意：是的，这里是将Arrow 批次写到一个临时 Parquet 文件。嗯，就是这么粗暴，直接。

3. 收集删除信息 ：两种删除合在一起——已持久化的 Deletion Vector（Puffin 文件引用）和未持久化的 Position Delete（`(文件编号, 行号)` 列表）。

最终 ReadState 包含：`data_files`（Iceberg 文件 + 临时内存文件）、`puffin_files`（Puffin 文件路径）、`deletion_vectors`（已持久化的删除位图引用）、`position_deletes`（未持久化的删除列表）。序列化后通过 Unix Socket 发给 DuckDB。

#### DuckDB 怎么用 ReadState

DuckDB 拿到 ReadState 后：

1. 打开所有 `data_files`（Iceberg 远程文件 + 本地临时文件），统一当作 Parquet 扫描。
2. 对每个数据文件，合并两种删除信息：把 Deletion Vector（Roaring Bitmap）和 Position Delete 合并成一个完整的删除集合。
3. 用删除集合构建 Access Plan ——在读 Parquet 之前就跳过已删除的行，不是读出来再过滤，而是直接不读。

这样 DuckDB 看到的就是一份完整的、包含实时数据、排除已删除行的一致视图。

## 总结

Mooncake 选择的这个方向（值得一提是的 Mooncake 最早专注做数据摄入，后面才开始投入 union read 方向）是吸引人的，但更聚焦于 Postgres 生态。整体设计选择了"简单优先"，在小规模场景下跑得通，但受限于单机执行和 compaction，可以预见在大规模数据下会存在瓶颈。

## 个人感想

就海外而言，Iceberg 已经成为事实标准——Snowflake、Databricks、AWS 都在积极拥抱 Iceberg。Data Infra 只有积极拥抱 Iceberg，围绕它做周边（实时写入、查询加速、索引、Compaction、CDC 同步……），才能搭上这波生态的快车。Mooncake 就是一个典型：在 Iceberg 之上补齐实时能力，最终被 Databricks 看中收购。

所幸的是 Fluss 也在这个方向上——走"湖流一体"，用一套实时存储同时承接流式消费和湖上分析，数据实时写入 Fluss，后台自动沉降到 Paimon/Iceberg 湖存储，流和湖共享同一份数据。并且同样通过 Union Read 满足秒级新鲜度的数据查询需求。

和 Mooncake 的思路异曲同工：都是在 湖生态上补齐实时能力，只是入口不同——Mooncake 从 Postgres CDC 切入，Fluss 从流计算切入，满足更多的数据接入。而且 Fluss 是分布式架构，天然可以支撑更大规模的数据量，不会像 Mooncake 那样受限于单机 Compaction 的瓶颈。
