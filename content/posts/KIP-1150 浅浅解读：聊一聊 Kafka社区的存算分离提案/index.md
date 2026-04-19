---
title: "KIP-1150 浅浅解读：聊一聊 Kafka社区的存算分离提案"
date: 2025-08-15T09:35:09+08:00
draft: false
tags: ["Kafka", "存算分离"]
categories: ["消息队列"]
summary: "KIP-1150提出Kafka存算分离架构，通过将数据存储至远程对象存储（如S3）并采用无Leader设计，降低跨可用区复制成本，提升可扩展性与成本效率。"
ShowToc: true
TocOpen: false
---

有段时间没更新了，前段时间 Aiven 公司在 Kafka 社区提了一个 Kafka 存算分离的 [KIP-1150](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics)，最近基于这个 KIP，用 Rust 把 [Fluss](https://github.com/apache/fluss) 的 Tablet Server 简单 POC 了一下（PS：等完善了写篇文章和大家分享一下），代码写累了，沉淀一下，写篇文章聊下 [KIP-1150](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics)。

注： [KIP-1150](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics) 也还在讨论中，不一定会被 Kafka 社区接受，但我觉得整体的思路是 OK 的，值得参考学习。

## KIP-1150 要解决的问题

简单来说，KIP-1150 就是让 Kafka 不要在本地存数据，直接用远程的对象存储如 S3 来存数据，即存算分离架构。关于存算分离的好处，之前写过一篇文章介绍过，建议先阅读一下。但是我发现我一直忽略了一个很重要的好处，这也是 KIP-1150 反复强调的一个好处，那就是可以节省 跨可用区数据复制 的网络成本。

什么是跨可用区数据复制的成本呢？如果我们需要部署高可用 Kafka 集群的话，通常需要把其中一个从副本放在另外一个可用区上，实现跨可用区的容灾，但是从副本需要跨可用区从主副本复制数据，而这跨可用区的网络流量是需要付费的（Aws，Google Cloud收费，Azure 免费），这就是 跨可用区数据复制的成本。这个成本是非常高的，根据 Aiven 公司写的 [Diskless Kafka](https://aiven.io/blog/diskless-apache-kafka-kip-1150)博客，整个成本占总成本的 90%。

而对象存储本身就是跨可用区高可用的，直接写到对象存储就自动实现了跨可用区容灾，也就没有了跨可用区数据复制的费用。

## KIP-1150 解决思路

#### KIP-1150 整体架构

Aiven 公司提出的这个架构和 WarpStream（现在已经被 Confluent 收购），StreamNative 公司搞的 Ursa 非常像， 至少我现在还并不能搞清楚它们这三个在架构上的区别。我合理怀疑是 WarpStream 提了这套架构，延迟也能干到1秒内，证明了这套架构的可行性，然后大家借鉴参考......

不同与传统 Kafka，KIP-1150 提出的这个架构是 leader-less 的，即没有明确的一个主节点，主节点在 Kafka 里意味着 producer 只能向这个主节点 produce 数据，而在 leader-less 下，则意味着 producer 可以向任何节点produce 数据。这也是为了减少跨可用区的网络开销，试想一下，如果主节点在可用区 AZ1，但是用户的 producer 却在可用区 AZ2，那么就需要承担 AZ2 到 AZ1 的网络开销了。

整体架构如下所示：

![](images/img_01.png)

Producer 将数据发送到 Broker，Broker 将数据上传到 Object Storage，然后再通过一个 Batch Coordinator 分配数据对应的 log offset，将数据对应的 metadata 信息，比如在 Object Storage 的哪个文件上，文件 offset 等信息也持久化到 Batch Coordinator中。metadata 信息持久化到了 Batch Coordinator 则认为数据写成功了。

Consumer 消费数据的时候，根据要消费数据的位点从 Batch Coordinator 拿到数据在哪个文件上，文件 offset 等信息，然后根据这些信息去文件上读数据。

注：这个 Batch Coordinator 我们可以先将其看作是一个持久化的 Broker 共享的 KV，后面会详细介绍这个 Batch Coordinator。

##### Write Path

写数据的整体流程如下所示：

![](images/img_02.png)

1. Producers 将数据发送到任意的 Broker

2. Broker 将发过来的数据在内存中进行 buffer

3. Broker 等攒够了足够的数据（8M），或者超过250 ms 了，就上传到对象存储上；

4. Broker 与 ControlPlan（其实就是 BatchCoordinator）交互，让它为这一批数据分配 log offset，并将数据的 metadata 信息，如文件名，log offset 持久化

5. Broker 返回给 Producer 写入成功的 ACK

这个 KIP 做了一个 produce 数据的延迟分析：

- Buffer 数据: 最多 250ms

- 上传到对象存储: P99 ~200-400ms, P50 ~100ms

- 与 Batch Coordinates 交互（提交到 Batch Coordinates）: P99 ~20-50ms, P50 ~10ms

目标 Produce request 延迟： P50 ~500ms， P99 ~1-2 sec

##### Read Path

![](images/img_03.png)

1. Consumer 发送 fetch 请求到任意 Broker

2. Broker 请求 ControlPlan 得到要 fetch 的数据的 metadata 信息，比如文件名，所在文件的位点，其对应的 log offset 等信息

3. Broker 根据 metadata 信息去对象存储上读数据

4. Broker 将 log offset 信息填充到 Log Batch 中，返回给 Consumer

注意：传统 Kafka 中，log offset 信息是记录在数据块（record batch ）中的，作为数据块的 header 一部分存储在文件中。但是这个架构下。log offset 信息的 source of truth 是记录在 BatchCoordinator 中的，所以需要消费的时候，需要请求一下 BatchCoordinator 得到 source of truth 的 log offset。

#### KIP-1150 核心组件 - BatchCoordinator

Batch Coordinator 其实是个协调者的角色，协调在 leader-less 下，不同 broker 写相同的 partition，log offset 到底应该怎么分配，做一个全局的 log offset 分配，这些不同的 broker 最终都会请求到 Batch Coordinator，Batch Coordinator 为这些请求分配 log offset。当然 Batch Coordinator 还有一个主要的功能就是持久化元信息。

这个 KIP 提出用 Kafka 的 inner topic 来作为 Batch Coordinator，当然 Batch Coordinator 的实现是可插拔的，Aiven 开放出来的代码的 Batch Coordinator 底层实际上用的是 Postgres。

用 Kafka 的 inner topic 来作为 Batch Coordinator 大概如下所示：

![](images/img_04.png)

CommitBatch 这一类写请求发送到 Leader Broker，对于 FindBatch 这一类读请求，可以直接发送到 Follower Broker。注意 Kafka 的 Topic 是不能直接查询的，所以这个KIP 说同时需要在内存中保存这些信息，然后本地搞个 sqllite，周期性地持久化到 sqllite。个人感觉有点复杂，不如直接用 Postgress 好了。

#### KIP-1150 核心组件 - Object Cache

其实这个 KIP 并没有介绍 Object Cache 的实现，但是显然 Object Cache 的实现是非常重要的，这个 Object Cache 是把在对象存储上的数据 cache 起来，避免频繁地直接从对象存储上读数据，提升性能，并减少对象存储 API 调用的成本。

虽然这个 KIP目前还没有介绍 Object Cache 的实现，不过 WrapStream 的博客 [Minimizing S3 API Costs with Distributed mmap](https://www.warpstream.com/blog/minimizing-s3-api-costs-with-distributed-mmap)介绍了它们搞的 cache。

简单来说，他们是基于一致性哈希搞了个分布式的 Cache，对象存储的 object key 就是 hash key，每个 Broker cache 一部分数据。

如下图所示（Agent 其实就是 Broker）：

![](images/img_05.png)

比如 Agent1 要读 file3 的数据，基于一致性hash 计算发现应该去 Agent 2 拉数据，然后就向 Agent2 请求数据，Agent2 发现本地没有这个数据，就去对象存储上 fetch 数据，然后返回给 Agent1。

Agent3 也要读 file3 的数据，也向 Agent2 请求数据，Agent2 发现数据在本地，就直接返回数据给 Agent3

#### KIP-1150 核心组件 - Object Compact

上传到对象存储的文件会包含多个 Kafka Topic & Partition 的数据，因为 Broker 会将这一段时间 produce 到所有 Topic & Partition 的数据都组织成一个文件，上传到对象存储中，这个也是为了减少小文件的数量。但是一个文件包含多个 topic 的数据对消费显然是不友好的，因为 Consumer 消费通常都是只消费一个 topic 的数据，文件包含多个topic 的数据会不利于顺序读。

所以 Object Compact 是必须的，Object Compact 就是将文件重新组织，尽量让相同的 topic & partition 的数据都组织到一个文件中，提高顺序读的效率。

然而这个 KIP 还没有写完 Object compact，等写完了再来更新下吧。

## 其他

其实 Kafka 社区还提了个其他简单的，也能节省跨可用区数据复制的方案，Kafka 社区还在讨论最终采用哪个方案，感兴趣的可以去[讨论邮件](https://lists.apache.org/thread/07xl5m6b4k10rog6rsbfnhqcqtgrtzxl)里面追踪下。等这个讨论结束我也写篇文章和大家聊一下最终方案后面的思考和 trade off。
