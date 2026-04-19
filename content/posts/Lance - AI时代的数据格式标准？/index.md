---
title: "Lance - AI时代的数据格式标准？"
date: 2025-04-26T18:17:56+08:00
draft: false
tags: ["数据格式", "AI"]
categories: ["数据格式"]
summary: "Lance是一种专为机器学习和AI优化的列式数据格式，通过摒弃RowGroup、引入DataPage及内置索引，解决Parquet在随机访问、超大列、大宽表支持上的不足，更好适配AI工作负载并对接主流AI生态。"
ShowToc: true
TocOpen: false
---

前段时间学习了一下 [Lance](https://github.com/lancedb/lance)，最近随着 Lance 被提及的越来越频繁，写篇文章聊一下自己对 Lance 的理解。

## 介绍

Lance 宣称自己是为机器学习和大语言模型（LLM）而生的列式数据格式。Lance 包含两部分定义，一部分是文件格式，另一部分是表格式。

其中 Lance 的文件格式类比 Parquet，定义的是怎么将数据组织成文件，以适应上层引擎访问的需求。而表格式类比于 Iceberg，定义的是怎么将这些文件组织起来，提供 ACID 语义，二级索引等。

表格式的这部分与 Iceberg 差不多，更重要的是 Lacne 的定义的文件格式，所以这篇文章主要聚集于 Lance 的文件格式的定义。

## Why lance

为什么需要一个新的文件格式，已有的 Parquet 格式不好吗？Parquet 格式虽然好，但是它是为大数据分析而生的，数据的组织也是为了适配上层引擎分析的需求，并不能很好地适应机器学习/AI 的 workload，主要分为以下几个方面。

### Parquet 不能很好支持随机访问

#### 为什么需要随机访问

随机访问指的是随机访问整个数据集的某几条数据。随机访问在 AI workload 是比较重要的，因为我们经常需要将整个训练集进行 shuffle，划分为训练集和验证集，而 shuffle 就依赖根据数据 id 快速访问数据的能力。并且在训练过程中，我们经常需要看一下第多少多少条数据是什么样子的，这也依赖随机访问的能力。

#### 为什么 Parquet 不能很好支持随机访问呢？

在 Parquet 中，如果要访问一条数据，假设通过某一列 id 来访问这条数据。我们需要读取 Parquet 文件的 footer 的统计信息，找到这条数据在哪个 Parquet 文件中。然后通过 footer 的 index，找到这条数据在这个文件的那个 RowGroup 中，最终再加载 整个 RowGroup 的数据，找到这条数据。

即使是访问一条数据，也需要访问加载整个 RowGroup，显然是不够高效的。

#### Lance 如何支持随机访问

Index!!!

在 Lance 中，每一条数据都会分配一个 row\_id 列，这是个递增的系统列。然后 Lance 会记录下每个文件包含的 row\_id，比如 \[1, 42, 3\] 表示的是这个文件的第一条数据，第二条数据，第三条数据的 row\_id 分别是 1，42，3。

这样就可以知道某个 row\_id 是在哪个文件的第几条数据，对于每一列，lance 文件的 footer 都记录了

- 每个数据块（一列的数据会组织成多个数据块）所在文件的 offset

- 每个数据块中数据的数量

通过 每个数据块中数据的数量 可以知道这条数据在哪个数据块中，通过数据块的 offset 信息，定位到该数据块。

接下来就变成了访问该数据块的第 i 条数据，为了快速访问到这条数据，数据块本身会记录 index 信息，然后就可以定位到这一条数据。

注：关于 Index 信息，这里多说几句，如果不压缩的话，Lance 本身是以 arrow 格式来存储的，即会做对齐。

- 对于 int 类型，总是使用 4 个字节来存储，所以如果要访问第 i 条数据，直接访问 i \* 4 这个offset 的数据即可。所以 lance 对于 int 类型，并没有存额外的 index 信息，是通过 4 字节对齐做的。

- 对于 string 变长类型，lance 会存储一个数组来记录每条数据的偏移量，这样也可以通过呢这个偏移量来直接定位到这一条数据

但是如果压缩的话，其实还是需要将整个数据块读出来进行解压。

### Parquet 不能很好支持超大列

超大列指的是这一列的每个 value 的 的 size 都很大，这在 AI 场景中特别常见，比如有一列直接来存储 embedding 的向量，甚至图像本身。

#### 为什么 Parquet 不能很好支持超大列

在 Parquet 文件中，有 RowGroup（行组）的概念，即先数据水平切分为若干个行组，然后再在 RowGroup 里面按列存放数据。数据读取的粒度也是以 RowGroup 来进行读取。这在某些列是超大列的情况下，会陷入如下的两难境地：

1. 正常的 Row Group size，如下所示：

![](images/img_01.png)

我们还是希望和以前一样，采用正常的 RowGroup，但是我们会发现，一个 Row Group 存储的数据更小了，对于窄列来说，需要读取更多的 Row Group了，更多次 IO 了，会存在大量小 IO，读取性能并不是很好。

2. 用一个很大的 RowGroup，如下所示：

![](images/img_02.png)

另外一种方式我用更大的 RowGroup，保证RowGroup能存储更多的数据。但是这样的问题也很大，我们需要在内存中 buffer 更多的数据才能写成 RowGroup，并且读数据的时候，并发数也变小。

#### Lance 如何支持超大列

No RowGroup!!!

Lance 直接把 RowGroup 干掉了，Lance 数据文件结构如下所示：

![](images/img_03.png)

Lance 不再有 RowGroup 的概念了，而是引入一个 DataPage 的概念。DataPage 存储的是某一列的数据，写某一列数据的时候，攒满一定 bytes 数后就 flush 成 Data Page，以此类推。不同的列可以有不同数量的 Data Page，超大列会有更多的 DataPage。

虽然 Lance 没有 RowGroup 了，但是也 Parquet 类似，也还是会有 footer 来记录列的 metadata，帮助我们快速定位到 Data Page。

### Parquet 不能很好支持大宽表

大宽表指的是一个表有大量（上万）列，在 AI 场景下，大宽表是非常常见的。

#### 为什么 Parquet 不能很好支持超大列

虽然 Parquet 作为一种列存格式文件，可以有些地支持列裁剪。但是即使对于只访问一列，也需要加载文件 footer 所有列的 metadata，这在列的数量很多的情况下也是个开销，大致如下所示：

![](images/img_04.png)

#### 核心原因是 Parquet 是按 RowGroup 来组织 metadata 的，如下图所示：

![](images/img_05.png)

图来自于： [https://parquet.apache.org/docs/file-format/metadata/](https://parquet.apache.org/docs/file-format/metadata/)

#### Lance 如何支持

Lance 没有 Rowgroup 的概念，各列单独存储统计信息和所在文件的 pos，这样要访问某一列直接读对应列的信息即可。我们看一下 Lance 的文件 layout 就可以理解了。

```
// ├──────────────────────────────────┤
// | Data Pages                       |
// |   Data Buffer 0*                 |
// |   ...                            |
// |   Data Buffer BN*                |
// ├──────────────────────────────────┤
// | Column Metadatas                 |
// | |A| Column 0 Metadata*           |
// |     Column 1 Metadata*           |
// |     ...                          |
// |     Column CN Metadata*          |
// ├──────────────────────────────────┤
// | Column Metadata Offset Table     |
// | |B| Column 0 Metadata Position*  |
// |     Column 0 Metadata Size       |
// |     ...                          |
// |     Column CN Metadata Position  |
// |     Column CN Metadata Size      |
// ├──────────────────────────────────┤
// | Global Buffers Offset Table      |
// | |C| Global Buffer 0 Position*    |
// |     Global Buffer 0 Size         |
// |     ...                          |
// |     Global Buffer GN Position    |
// |     Global Buffer GN Size        |
// ├──────────────────────────────────┤
// | Footer                           |
// | A u64: Offset to column meta 0   |
// | B u64: Offset to CMO table       |
// | C u64: Offset to GBO table       |
// |   u32: Number of global bufs     |
// |   u32: Number of columns         |
// |   u16: Major version             |
// |   u16: Minor version             |
// |   "LANC"                         |
// ├──────────────────────────────────┤
```

假如要读第 i 列的话，首先通过文件的 footer 定位到第 i 列的 Metadata，然后通过这个第 i 列的 metadata 得到该列数据对应数据块的信息。

## 总结

1. 相比于 Parquet，Lance 格式可以算是极致的“列裁剪”了，在 AI 领域确实有其独特的价值，但是也是个 trade offset，可以想见，其在传统的 OLAP 分析领域上还是没法和 Parquet 比的，比如 Parquet 的高压缩率，各种 fiter pushdown，大部分列读取等

2. Lance支持多模态数据的方式是直接在数据文件中存储多模态数据，相比于 Gravitino 的存储一个图片路径的方式，我更喜欢 Lance 这种可以直接存图片本身的方式。

3. Lance 格式内置了索引以支持向量检索在 Agent 时代还是很有用的

4. 我觉得最重要的一点是 Lance 很好地对接了 AI 生态，如 Pytorch，Tensorflow，Huggingface 等，几行代码就可以让 Lance 的文件作为 Pytorch 模型的训练集。并且 Lance 的 co-founder 也是 Pandas 的核心贡献者，个人还是看好 Lance 成为未来的一个 AI 文件格式的事实标准

## 参考链接

1. [https://lancedb.github.io/lance/format.html#](https://lancedb.github.io/lance/format.html#)

2. [https://blog.lancedb.com/lance-v2/](https://blog.lancedb.com/lance-v2/)
