---
title: "VLDB-2025 Best Industry Paper - Ursa: A Lakehouse-Native Data Streaming Engine for Kafka"
date: 2025-09-03T10:12:08+08:00
draft: false
tags: ["Kafka", "存算分离", "论文"]
categories: ["论文阅读"]
summary: "Ursa是兼容Kafka协议的湖仓原生存算分离流引擎，通过将数据直接写入对象存储并内置Compaction服务，降低存储成本并支持高效分析。"
ShowToc: true
TocOpen: false
---

本来并不打算专门写篇文章介绍 Ursa，因为Ursa 的架构和之前我写的文章KIP-1150 浅浅解读：聊一聊 Kafka社区的存算分离提案的架构非常像，但是考虑到 [Ursa](https://www.vldb.org/pvldb/vol18/p5184-guo.pdf) 居然拿了 VLDB-2025 的 best industry paper，水一篇文章简单介绍一下。

其实在最早去年FFB上云邪老师介绍了 [Fluss](https://github.com/apache/fluss) 后，StreamNative 搞 Ursa 的那帮人很感兴趣，就找我们交流了一下 Fluss 。然后我们让他们分享了一下他们内部搞的 Ursa，于是我在那个时候就了解到了 Ursa。

## 背景

Kafka 虽然已经是流存储事实的标准了，但是 Kafka 也有很多问题：

- 成本高：存算一体架构导致依赖昂贵的本地磁盘，扩缩容的运维成本，ISR 复制协议带来的跨 AZ 复制数据的成本，

- 不能直接用来高效分析：Kafka 从来不是用来进行高效分析的，要分析 Kafka 里面的数据，都需要运维一个新的数据 pipeline，把 Kafka 的数据灌到湖仓中

Ursa（兼容 Kafka 协议） 就是专门用来解决这两个问题的：

- Ursa 是存算分离的架构，直接写对象存储

- Ursa 内置 Compaction service，自动将 Ursa 的数据 compact 成湖格式

## 架构介绍
Ursa 的架构图如下所示：

![](images/img_01.png)

Ursa 主要分为三大部分：

- Broker：无状态的 broker，负责接收 client 的读/写请求

- Stream Storage：数据的存储层，将存储与计算节点（Broker）解耦合，分为 WAL 和 LakeStorage。Broker 会直接将数据写到 WAL 中，WAL 是可插拔的，可以是低延迟的 BookKeeper，也可以是延迟稍微高一点，但更廉价的对象存储。此外，Ursa 还会有一个 Compact service 将 WAL 里面的数据 compact 成湖格式

- Metadata service：负责元数据的管理，以及一些协调工作。因为 Ursa 是 leader-less 的，所以可能会存在两个 leader broker 同时写入数据，所以需要一个中心化的协调者来协调 log offset 的分配工作，不然两个不同的 leader broker 写入的数据可能会有相同的 log offset。

## 读/写流程

### 写流程

写流程如下所示：

![](images/img_02.png)

1. Writers 首先将数据写入到 Broker 的 WAL Buffer

2. WAL Buffer 攒够了足够多的数据，或者等待了足够长的时间后，将数据持久化到 WAL（对象存储 WAL） 中

3. 提交到 metadata service，让 metadata service 为这批数据分配 log offset，记录对应 offset 到 WAL 文件所在的 pos。对于 Kafka 来说，写入一批数据后，需要为这一批数据分配 log offset， consumer 也需要可以根据这个 log offset 快速定位到数据并进行消费。这就是 Ursa 的 Metadata service 要干的事情， Ursa 为 log offset 构建了一个 kv-store 来 索引 log offset ，用来快速根据 log offset 来找到该数据在哪个文件的哪个 pos。其中 index-entry 的 Key 和 Value 的格式如下所示：

- Key：(StreamID, OffsetEnd, CumulativeSize)

其中：StreamID 可以理解为一个 Partition，Usra 为了唯一标识一个 Partition，引入一个 StreamID 来标识它

OffsetEnd 是这一批数据最大的 log end offset，CumulativeSize 记录了这个 Partition 从开始直到这个 index-entry 总共有多少bytes 的数据。论文并没有说 CumulativeSize 是用来干嘛的，我怀疑可能是为了快速 fetch 到指定 bytes 数据的 index-entry。

- Value: （Location, FileType, EntryCount, MessageCount, OffsetInObject, EntryOffsets）

其中：Location 为文件名，FileType 为文件类型（WAL 或者 lake File），MessageCount 为这一批数据的消息数量，OffsetInObject 为这批数据在文件上的具体 pos，EntryOffsets 我没太看懂是干啥的，看起来像是为这一批数据再次构建了一个小的 index，用来快速seek 这一批数据的某行数据。基于此，这个 EntryCount 其实表示这个小的 index 包含的 index-entry。

于是，提交到 metadata service 的具体流程如下：

![](images/img_03.png)

4. 通知 Writers 写入成功了

### 读流程

假设从 offset x 开始读取数据，则流程如下所示：

1. Broker 去 Metadata Service 查询第一个 OffsetEnd > x 的 index entry

2. Broker 通过这个 index entry 定位到对应的文件和pos，读取数据

3. 如果数据在 WAL中，直接读取即可。但是如果数据在 Lake 上，需要将列式的数据转成行式的方式（因为 Ursa 需要兼容 Kafka 协议，未来会考虑扩展 Kafka 协议，支持 Arrow 格式）。会带来一定的 CPU 开销，但是 Parquet 高压缩率会带来更少的 IO

另外 Ursa还搞了个基于一致性 Hash 的 cache 来 cache 数据，我的上一篇文章有提到过，就不再赘述了。

## 性能评估

其实我比较好奇的是 Usra 和 kafka 比的性能测试，但是并没有，只是简单 benchmark 了一下 Usra 在不同 workload 下的表现。结论如下：

- Topic & Partition 数量和消息的大小均不影响吞吐，维持在 5G/s

- 1G/s produce 的 workload 下，p99 延迟 1s 内，p99999延迟2，3秒

- consumer 大规模数据回拉下，produce 延迟表现依然很好，除了produce达到 2G/s 的时候会有个延迟陡增。很奇怪，论文也并没有解释为什么。

![](images/img_04.png)

## 总结

- 我喜欢 Ursa 的 diskless + lake native 的架构，简单，优雅。

- Ursa 提到了对于数据的更新，未来会支持生成 Changelog。（：这不就是 Fluss 的主键表

- Ursa 提到未来会支持上层计算引擎将 WAL 的数据和 Lake 的数据合并起来，实现 real-time 的 OLAP。目前上层计算引擎只能查询 compact 到 Lake 的数据，数据有一定的 delay，需要再等一次 compact后， 数据才对上层计算引擎可见。（：这不就是 Fluss 的Union Read
