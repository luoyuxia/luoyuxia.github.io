---
title: "Kafka 复制协议不可不知的技术内幕 - 关于 Kafka 踩过的坑"
date: 2025-01-19T21:32:52+08:00
draft: false
tags: ["Kafka", "分布式系统"]
categories: ["消息队列"]
summary: "Kafka通过主从复制的协议对消息进行备份以实现高可靠的分布式系统，但是在如何正确地实现复制的协议中，Kafka作为一款公认的稳定可靠的分布式消息队列，也踩了不少坑。本文首先深入介绍了Kafka的复制协议，然后引出了Kafka在这套复制协议上踩的若干坑和修复方案。通过理解Kafka踩的坑和解决这些问题的思路和方案，希望可以对读者的在分布式系统设计上有所借鉴和启发。"
ShowToc: true
TocOpen: false
---

## 摘要

Kafka 通过主从复制的协议对消息进行备份以实现高可靠的分布式系统，但是在如何正确地实现复制的协议中，Kafka作为一款公认的稳定可靠的分布式消息队列，也踩了不少坑。 本文首先深入介绍了 Kafka 的复制协议，然后引出了 Kafka 在这套复制协议上踩的若干坑和修复方案。 通过理解 Kafka 踩的坑和解决这些问题的思路和方案，希望可以对读者的在分布式系统设计上有所借鉴和启发。

## Kafka复制协议 Overview

Kafka 是一个分布式，高可靠的消息系统。为了实现高可靠的分布式系统，需要在多台机器上保存相同数据的副本，这样即使某一台机器挂了，其他机器则可以及时接管，提供这部分数据。而复制协议则是解决如何在多台机器上保存相同数据的副本，以及机器挂了其他机器如何接管的问题。

数据的复制通常有两种策略，一种是主从复制，另一种是基于 Quorum 的复制，典型的如Paxos，Raft。这两种策略都需要选定一个主副本，客户所有的写请求都首先需要写入到主副本，然后主副本再将数据同步到从副本，副本同步成功了就给客户返回 ACK。

但是什么时候认为副本同步成功了，这两种策略的行为则不一样，主从复制需要数据同步到所有副本，而基于 Quorum 的复制则只需要同步到大部分的副本（如果共有 n 个副本，则大部分副本数量为 n / 2 + 1）。

Kafka 采用的是主从复制的机制，在 [Building a Replicated Logging System with Apache Kafka](https://www.vldb.org/pvldb/vol8/p1654-wang.pdf) 这篇 2015年的论文也说了原因：“如果要容忍 F 个副本丢失，基于主从复制只需要 F + 1 个副本就可以了，但是基于 Quorum 的复制则需要 2F + 1 个副本，虽然基于 Quorum 的复制协议只需要同步到 F + 1 个副本，所以其复制延迟更低，也可以避免网络延迟/慢节点的影响，但是 Kakfa 通常是部署在相同的数据中心，网络延迟不大，我们认为节省副本数更重要”。

根据 Kafka 的[官方文档：](https://kafka.apache.org/documentation/)Kafka 的复制协议的实现主要来源于论文 [PacificA: Replication in Log-Based Distributed Storage Systems](https://research.microsoft.com/apps/pubs/default.aspx?id=66814) 的思路。不过在实现上稍微有些不同，Kafka 的复制协议可总结如下：

1. 数据首先写入到主副本，然后再由主副本同步到所有从副本，同步成功后给客户返回 ACK

2. 数据需要同步到所有从副本才认为成功，但是如果某一个从副本挂了，或者就是同步得很慢怎么办？

如果还是等待的话，客户端写数据到收到 ACK之间的延迟就很大或完全写不进去了（在从副本挂了的情况下）。于是 Kafka 就引入了 ISR（In Sync Replica） 的概念了，ISR 是若干副本的集合，是所有副本的一个子集，数据只要同步到 ISR 集合中的这些副本中就认为写成功了。

一开始 ISR 是所有的副本，但是如果某个副本 R1挂了，或者同步数据很慢，Kafka 将会将这个副本从 ISR 集合中剔除，这样数据不需要同步到这个副本 R1 也可以认为写成功了。对应地，有一个 参数 min.insync.replicas（Topic 级别的参数，默认为1） 来控制同步到多少个在 ISR 中的副本就认为写成功了。

为了实现数据的高可靠，一个典型的配置是数据的副本数设置为3，min.insync.replicas 设置为2，这样数据同步到2个副本就认为写成功了。

3. 如果主副本挂了怎么办？

如果主副本挂了，Kafka 的元数据控制中心 Controller 会从 ISR 中选择一个其他的副本来当作主副本，对外提供服务。值得一提的是，Controller 也必须是高可靠的，Kafka 之前是基于 Zookeeper（基于 Paxos协议），后面自己基于 Raft 协议实现了 KRaft Controller。 所以 Kafka 弄的这套复制协议不是自举的，anyway 都需要额外的一个复制协议来实现元数据的高可靠。

## 深入理解Kafka复制协议

接下来我们来深入了解一下 Kafka 的复制协议，这对我们后面理解 Kafka 在复制协议上踩过的坑至关重要。

### Kafka 架构图

Kafka 为了对消息分类，引入了 Topic 的概念，类似数据库中表的概念。

为了提高系统的吞吐和可扩展性，在 Topic 的基础上，引入了 Partition（分区），一个 Topic 会被划分成多个 Partition，一个 Topic 的多个 Partition 会被放到多个 Broker 节点中。

为了服务的高可靠，引入了 Replica（副本）的概念，一个 Partition 包含多个 Replica，Replica 是一主多从的关系，有一个 Leader Replica 和 若干个 Follower Replica，Replica 分别在不同的 Broker 节点上。Leader Replica 负责读/写请求，Follower Replica只负责同步数据，Follower Replica 会主动向 Leader Replica 发起 fetch 请求从 Leader Replica 读数据，写入到本地存储中。

同时，由一个 Controller 来控制整个 Kafka 集群，做一些协调工作，比如 Leader Replica 挂了的话，Controller会从其他 Follower Replica 中选取一个作为新 Leader，对外提供服务。

整体架构如下图所示：

![](images/img_01.png)

### Kafka 日志复制流程

首先我们需要理解两个非常重要的概念，LEO（Log End Offset），HW（High Watermark）。

LEO 表示分区中的下一个被写入消息的偏移量（offset），用于记录 Leader Replica 和 Follower Replica 之间的数据同步进度，正常情况下，Leader Replica 的 LEO 总是要大于等于Follower Replica。

HW 是 ISR （In Sync Replica）中最小的 LEO，其表示 ISR 中的 Replica 都已经复制 HW 之前的消息了，即这些消息都认为是已经写成功的。消费者可以且只能消费到 HW 之前的数据。

注：之前我们提到过，数据只要写到 ISR 集合中的 Replica 中，就认为消息写成功了。ISR 集合一开始是全部 Replica ，如果有 Replica 挂了，该Replica 就会从 ISR 集合中剔除；比如一开始 Partition A 分配了 三个Replica {0, 1, 2}，其 ISR 也为 {0, 1, 2}，但 Replica 1 挂了的话，其 ISR 变为 {0, 2}。如果之后 Replica 1 恢复回来，且追上了 Leader Replica 的话，其 ISR 就将变为{0, 1, 2} 了。

下面来理解一下 Kafka 日志复制流程和对应的 LEO 和 HW 更新流程，假设有三个 Replica：

1. 初始状态，三个 Replica 各有 m0 和 m1 两条消息，LEO 都是 2，表示下一条要写入的消息的偏移量（offset），m0 和 m1 消息的offset 分别是 0 和 1。HW 也都是 2，表示 Leader Replica 中的所有消息已经全部同步到 Follower Replica 中，消费者可以消费 m0和 m1这两条消息。如下图所示：

![](images/img_02.png)

2. 接下来，生产者向 Leader Replica 中发送两条消息，m2，m3，此时 Leader Replica 的 LEO 的值增加2，变成4。但是由于 Follower Replica 还没同步到者两条数据，所以 HW 和 Follower Replica 的 LEO 的值都没有发生变化。消费者还是只能消费 m0和 m1这两条消息

![](images/img_03.png)

3. Followe1 和 Follower2 都向 Leader replica 拉取数据，同步到自己本地，但是同步速率不同，Follower1 已经同步到 m2 和 m3，但是 Follower2 只同步到了 m2。此时 Leader 的 LEO 和 Follower1 的 LEO 都是4，但是 Follower2 的 LEO 是3。同时 HW 代表 Replica 中最小的 LEO，所以还是 3，因为 Follower2 的 LEO 最小，为3。消费者可以消费 m0，m1，m2这三条消息。

![](images/img_04.png)

4. Follower2 也同步 m3 到本地了，此时所有 replica 的 LEO 都是 4，并且 HW 也都更新到4了。此时消费者可以消费到 m0，m1，m2，m3这四条消息。

![](images/img_05.png)

至此，Kafka 的复制协议的基本流程就讲完了，但在实际的实现中，并没那么简单。Kafka 也是对自己的复制协议打了很多的 Patch ，接下来我们来看看 Kafka 在复制协议上踩的坑以及提出的解决方案。

## Kafka复制协议踩过的坑

### [KIP-101 - Alter Replication Protocol to use Leader Epoch rather than High Watermark for Truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation)

#### 问题

上述介绍的复制协议可能会出现数据丢失或者数据发散的情况，考虑如下两个场景：

- Case1 数据丢失

Kafka 的复制协议有两个阶段：

第一阶段，follower 向 leader fetch 消息，假设 fetch 到了消息 m2，并append 到本地。follower 在下一轮的 fetch 请求中，follower 会告诉 leader 自己收到了消息 m2，Leader 就可以更新 HW（High Watermark）。 Leader 会在之后 follower 的 fetch 请求的 response 中把 HW 带上，这样 follower 就知道 HW 是多少了。

第二阶段：follower 初始化的时候，follower 会将自己本地的消息截断到它自己记录的 HW 中，然后再从 leader fetch 消息。但是这可能会导致一些已经被认为写成功（返回给客户端 ack）的消息被截断了，造成数据的丢失。

假设我们有两个 broker：A & B，A 是 follower，B 是 leader。

1. 一开始 A 从 B 中 fetch 到了消息 m2，然后发起下一轮 fetch，告诉 B m2 已经被同步了，然后 B更新自己的 HW 为 2

![](images/img_06.png)

注意此时 A 并不知道 HW 被更新为 2 了，需要在A 发起下一轮的fetch 请求，B 才会告诉 A 其HW 为2，这个时候 A 才能更新自己的 HW 为 2.

2. Broker A 重启了，它把自己本地的消息truncate 到 HW 1 了，注意：这个时候 m2 在 A 中就被删掉了

![](images/img_07.png)

3. 然后 A 从 Leader B fetch 数据，但是假设 Leader B 也挂了，此时 A 就是新的 Leader 的，但是 m2 却丢失了

![](images/img_08.png)

看起来原因就是 follower 的 HW 和 leader的 HW 更新步率不一致，follower 的 HW 需要再一轮 fetch 请求才可以更新它的 HW。一个直接的办法是 follower 更新 HW 后，leader 再去更新 HW，但这样就会让 leader HW 需要再多等一轮 fetch 才能被更新，会增加复制协议的延迟。并且也依然不能解决下面的数据发散的问题。

- Case2 数据发散

首先有个背景是，Kafka 的 Broker 将数据写到本地，实际上只是写到 page cache 中，所以如果 Broker 所在的机器直接挂了，这部分 page cache 中的数据在该 Broker 上就是丢失的。考虑如下的 case：

1. 一开始 A 是 leader，写了m1，m2 两条数据。follower B 也fetch 了 m1，m2 两条数据，于是 A 的 HW 更新成2。但是 follower B 写的 m2 这条数据并没有 flush 到磁盘。假设 A，B 都挂了，此时 B 同步的 m2 这条数据丢失了。

![](images/img_09.png)

2. 此时 B 恢复过来了，成为了 Leader，并且接受了 m3 这条消息

![](images/img_10.png)

3. A 也恢复过来了，truncate 数据到 HW 2，所以不会truncate掉 offset 为1 的 m2 消息。但是 Leader 的 offset 为1 的消息却是 m3。这样消息在不同 replica 之间就发散了，数据就不一致了。

![](images/img_11.png)

#### 解决方案

核心问题就是 Follower 不应该直接根据自己记录的 HW 来将消息进行截断，而是应该和 Leader 进行交互来知道自己应该 truncate 哪些数据。Leader 直接返回 HW 可以解决 case 1，但是解决不了 case 2，case2 的问题在于在 leader 会发生切换的情况下， Follwer 无法知道相同 offset 的一条 message 是不是相同的 message，也就无法进行将相同 offset 下与 Leader 不同的数据 truncate 掉

于是，Kafka 引入了一个 partition leader epoch 的概念来标识一次 leader 任期，每发生一次 leader 的切换，该 partition 的 leader epoch 就会加一，相同 leader epoch 下 append 的 相同 offset 的 message 也就是相同的。

每一个 replica 维护一个 \[leaderEpoch -> StartOffset\] 的映射来标记每个leader 任期的起始消息的offset，有了这个，Replica 就可以知道某个 epoch 下 append 的最后一条消息的log offset。 follower 恢复的时候，带上自己记录的当前的 leaderEpoch 向 Leader 发送 OffsetForLeaderEpoch请求（虽然图片上是 名字是 LeaderEpochRequest，但是实际代码实现用的名字是 OffsetForLeaderEpoch） ，Leader 返回该 leaderEpoch下 append 的最后一条消息的 log offset ，follower truncate 到该 offset，然后再向 Leader 发送fetch 请求进行消息同步；

考虑Case1 数据丢失的问题：

![](images/img_12.png)

1. 一开始 Replica A 的 HW 是1，Replica B 的 HW 是2；他们记录的 leader epoch 和 log start offset 都是 0；

2. 然后 Broker A 重启了，A 带着自己记录的 leader epoch 0 向 B 发送 OffsetForLeaderEpoch 请求，B 收到该请求后，发现该 epoch 和自己记录的 leader epoch 0 一样，返回该 leader epoch 0 下 append 的消息的 end offset，即自己的 log end offset，返回 offset 2 给 follower

3. follower 收到 offset 2 后，不进行 truncate，数据不会丢失

4. 之后 Broker B 挂了，Replica A 成为 leader 后，leader epoch 从 0 变成 1，收到消息 m3，leader epoch 1对应的 log start offset 也变成了 2

考虑Case2 数据发散的问题：

![](images/img_13.png)

1. 一开始 Replica A 的 HW 是 2，Replica B 的 HW 是1；他们记录的 leader epoch 和 log start offset 都是 0；Broker A 和 Broker B 都挂了

2. Broker B 重启了，成为 leader，收到消息m3，leader epoch 变成了 1，其对应的 log start offset 变成1

3. Broker A 启动了，Replica A 成为 Follower，带着 epoch 0 向 Leader B 发送 LeadEpoch 请求，B返回 epoch 0 下 append 的消息的 end offset，即 log offset 1；A 于是 truncate 掉 offset >= 1 的消息，然后再从 Leader fetch offset = 1 的消息 m3。自己记录的 leader epoch 变为 1，其对应的 start offset 也变为1

### [KIP-274: Fix log divergence between leader and follower after fast leader fail over](https://cwiki.apache.org/confluence/display/KAFKA/KIP-279%3A+Fix+log+divergence+between+leader+and+follower+after+fast+leader+fail+over)

#### 问题

提出 KIP-101 后，又发现 KIP-101 无法解决如下的 corner case：

1. Step 1

假设有两个 Broker A 和 B；一开始A 是 leader，leader epoch = 1；append 了两批数据到这个 leader，第一批数据的 log offset 是\[0, 10\]，第二批数据的 log offset 是 \[11, 20\]。第一批数据被同步到了 Broker B，但是第二批数据没有；Broker 中的数据如下所示：

![](images/img_14.png)

注意：不同批次的message 用不同的颜色标识

2. Step2

然后 Broker B 由于 preferred leader election 被选为 Leader 了，此时 A 和 B 都在 ISR 中，然后 B append 了一批数据，对应 log offset \[11, n\] 到本地中，现在Broker 中的数据如下所示：

![](images/img_15.png)

3. Step3

Broker A 成为 Follower，正常情况下，我们希望 Broker A truncate掉 offset 为 \[11, 20\] 的这批数据。但是此时，Broker B 下线了，Broker A 成为了 Leader，并且又 append 了一批数据，对应 offset \[21, 30\]。现在Broker 中的数据如下所示：

![](images/img_16.png)

4. Step4

然后 B 恢复过来，成为 Follower，于是带着 epoch 2 向 Broker A 发送 OffsetForLeaderEpoch 请求， Broker A 返回21（大于 epoch 2 的 leader epoch 对应的start offset，在这里即为21）。

- 如果 n <= 20，它不会进行 truncate，于是会从 n 开始向 leader fetch 数据。但此时数据已经发散了

如下图所示：

![](images/img_17.png)

- 如果 n > 20，它会 truncate 到 offset 21，但是由于 offset 21 是这一批数据的中间部分，于是会继续truncate，直到这一批数据的起始位点，即 11，此时 Broker B 中的数据只有 \[0, 10\] 了，然后从 offset 11 开始向 Leader A fetch 数据，数据不再发散，保持了一致性，如下图所示：

![](images/img_18.png)

#### 解决方案

其实核心的问题就是 Broker A 只有 epoch 1 和 3，它无法告诉 B 在 leader epoch 2 下append 的最后一条数据的 offset。在这种情况下，Broker B 应该用 leader epoch 1 去向 Broker A 发送 OffsetForLeaderEpoch 请求，然后 truncate掉自己在 epoch 2 下append 的数据。

所以 Kafka 提出修改 Follower 的整个恢复流程如下所示：

1. Follower 恢复的时候，发送 OffsetForLeaderEpoch 请求，并带上自己记录的最新的 leader epoch 给 leader

2. Leader 给 Follower 回复一个自己记录的 小于或等于 OffsetForLeaderEpoch 请求中的 leader epoch 的最大的一个 LeaderEpoch 和其对应的 end pffset

3. 如果 Follower 自己也记录了这个 Leader 回复的这个 LeaderEpoch，跳到第4步，否则

   - Follower truncate 掉所有 epoch 大于 LeaderEpoch 的消息

   - Follower 再用一个小于 LeaderEpoch 最大的一个 epoch 向 Leader 发送 OffsetForLeaderEpoch 请求

   - 重复步骤2和3

4. Follower truncate 到 end offset 对应的消息

5. Follower 继续向 Leader fetch数据

于是就可以解决上面提到的问题了，考虑上问题的 Step3：

![](images/img_19.png)

Broker B 成为 Follower 后，带着 leader epoch 2 发送 OffsetForLeaderEpoch 给 Broker A，Broker A 找到自己记录的最大的一个 epoch <= 2的 epoch，和其对应的 end offset: {leader\_epoch = 1, end offset =21}，返回给 Broker B。Broker B truncate 掉所有大于 leader\_epoch = 1 的数据，在这里就是 \[m11, n\]，然后再从 offset 11 开始从 Broker A fetch，这样 Broker A 和 Broker B 的数据就一致了。

### [KIP-320: Allow fetchers to detect and handle log truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-320%3A+Allow+fetchers+to+detect+and+handle+log+truncation)

#### 问题

考虑如下的 case，一个 Partition 有三个 Replica1，Replica2，Replica3。

1. 在 Leader epoch 0，Replica1 是 leader，消息的 end offset 为 50，Replica2 也复制到了 offset 50，但是 Replica3 只复制到了 offset 40 ，此时 high watermark 为 40

2. 此时 Replica2与 Controller 失去联系，但是 Replica2 还是可以从 Replica1 fetch 数据

3. 不管什么原因，Replica 3 被选为 Leader，leader epoch 为1，Replica1成为 Follower，truncate 消息到 offset 40

4. Replica 3 写了20条消息，end offset 变为 60，Replica 1 也复制到了 offset 60

5. Replica2继续从Replica1 fetch 数据，因为 Replica1不再是 leader 了，Replica1 返回 NOT\_LEADER\_FOR\_PARTITION 的异常，Replica2 将重试

6. Replica1 又变成 Leader 了，leader epoch 为2。Replica1从 offset 60 开始 append 消息。Replica2 继续重试向 Replica1 fetch 消息。Replica2 的当前 offset 是50，Replica2本来应该将消息 truncate 到位点 40，但是Replica2不知道Leader 变更了，所以不会truncate消息，而是继续从 offset 50 开始向 Leader fetch 数据，于是 Replica2 的offset 40 ～ 50 的消息就有问题了。

另外，这个 KIP 还提到的一个问题是，虽然 Replica2 被 Controller 标记为 Offline，但是由于 Replica2 还是可以及时同步 Replica1 的数据，这样 Replica1 又会把 Replica2 加入 ISR 中了。但是 Replica2 又被 Controller 标记为 Offline 了，不会被 Controller 选为 Leader，破坏了 ISR 的语义

#### 解决方案

核心问题就是 Follower 不知道 Leader 发生变化了，解决方案也很直接，follower 发送 fetch request 的时候需要带上自己记录的 leader epoch，而 Leader 知道自己的 leader epoch，这样 Leader 就能告诉 follower leader 是不是发生变化了。

Leader 和 Follower 侧的修改如下：

- Leader 侧

Leader 侧收到 fetch 请求后，会比对自己的 leader epoch 和 fetch 请求中带上的 leader epoch，只有当两个 epoch 一样，fetch 请求才会正常返回；如果 fetch 请求中的 leader epoch 小于自己的 leader epoch，返回 FENCED\_LEADER\_EPOCH 异常，如果 fetch 请求中的 leader epoch 大于自己的 leader epoch，则返回 UNKNOWN\_LEADER\_EPOCH。

- Follower

Follower 向 Leader 发送 fetch 请求的时候会带上自己记录的 leader epoch，如果收到 FENCED\_LEADER\_EPOCH 异常，就不再从 Leader fetch 数据了，如果收到 UNKNOWN\_LEADER\_EPOCH 异常，就继续重试。

我们来看，这个方案如何解决上面提到的问题，考虑step 6，Replica2 带着 leader epoch 0 向 leader Replica1 发送 fetch 请求，Replica1 发现自己的 leader epoch 大于 fetch 请求的 leader epoch 0，于是直接返回 FENCED\_LEADER\_EPOCH 异常，Replica2 就会停止fetch，不再作为 follower。

### [KIP-380: Detect outdated control requests and bounced brokers using broker generation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-380%3A+Detect+outdated+control+requests+and+bounced+brokers+using+broker+generation)

#### 问题

目前 Kafka 的 Controller 与 Broker 交互是通过发送如下的控制请求给 Broker 的：

- LeaderAndIsrRequest：某个 Partition 的 replica 被选为 leader 或 follower 了，通知 Broker 进行相应的操作，比如选为 follower 后，fetch 线程需要向 leader 所在的 Broker fetch 数据

- UpdateMetadataRequest：将集群的 metadata 信息同步给 Broker

- StopReplicaRequest：通知 Broker 停止 serving 或者删除某个 Partition 的 Replica

并且 Controller 是通过监听 zookeeper 来感知Broker 的上线和下线的，Broker 上线的时候会在 zookeeper路径 /brokers/ids/znode 创建一个临时节点，Broker 是监听路径 /brokers/ids的变化来知道 Broker 的上线和下线的

但是会出现以下几个问题：

1. Broker1 向 controller 发送 ControlledShutdownRequest 告诉 controller 自己要 close 了，然后 controller 会发送一些控制请求，比如 StopReplicaRequest等给 Broker1，在这个时候 Broker1快速重启了，然后就接收到这些过期的（该 Broker 以前触发的）控制请求并进行处理，这样Broker1 就处于一个不正常的状态了

2. 分区 p1 和 p2 都在 broker1上，controller 发送 p1 的 LeaderAndIsrRequest 请求给 broker1，但是在broker1收到这个请求的时候，broker1 重启了，然后 broker1 收到了这个 LeaderAndIsrRequest 请求，这是 broker1 收到的第一个 LeaderAndIsrRequest 请求，它会重写 high watermark checkpoint 文件，由于 LeaderAndIsrRequest 请求只有p1，所以 p2 的 high watermark信息 就丢失了

3. 如果一个broker 1快速重启，controller 监听到了 /brokers/ids的变化，但是这个时候 broker 已经将自己注册到到了路径 /brokers/ids/1 中，controller 去list 路径 /brokers/ids，发现 broker 1还在，所以就会忽略 broker 1 的重启，也就不会发任何 request 给该 broker 1 让其进行初始化，导致 broker 1 永远无法初始化自己的 leader/follower Replica

#### 解决方案

核心原因就是没办法知道一个 broker 是没重启过还是经过了快速重启。于是 Kafka 引入一个 broker epoch 的概念， broker epoch 是唯一且递增的，每次 broker 重启，它的 broker epoch 就会增加，然后Controller 每次发送的控制消息中都会带上 该 broker epoch。这样：

对于问题1和2，Broker发现控制消息中的 epoch 小于自己的 epoch，就直接 reject 这个消息。

对于问题3，Controller 通过 epoch 的值来判断 broker是不是重启过，Controller 会比对自己 cache 中的 broker 的 epoch 值和该 broker 在 zookeeper中 记录的 epoch，如果 zookeeper 中记录的 epoch 更大，则表示是经过了快速重启，会把它当成一次正常的 broker shutdown 和启动来对待。

### [KIP-903: Replicas with stale broker epoch should not be allowed to join the ISR](https://cwiki.apache.org/confluence/display/KAFKA/KIP-903%3A+Replicas+with+stale+broker+epoch+should+not+be+allowed+to+join+the+ISR)

#### 问题

考虑如下的 case：

1. 一开始一个 partition 有两个 replica A 和 B。A 是 leader，B 是 follower，且 ISR 为 {A}

2. Broker B 追上了 A，然后 A 发送 AlterPartition 请求给 Controller 试图将 B 添加到 ISR 中

3. 在 AlterPartition 被 Controller 收到前，Broker B 直接挂了，注意此时在 page cache 中的数据还没 flush 下去

4. 这个时候 Broker B 恢复过来了，在 page cache 中的那部分数据丢失了

5. Controller 知也收到了 AlterPartition 请求，并且知道 Broker B 是online 的，于是将 B 添加到 ISR 中，此时 ISR 为 {A, B}

但是实际上 B 不应该添加到 ISR 中，因为它的数据丢失了。

#### 解决方案

核心问题是在 Broker B 挂了再恢复后，这个 AlterPartition 其实是一个过期的请求，这个请求是针对 重启前的 BrokerB的，而不是重启后的 Broker B。Controller 应该reject 这个过期的 AlterPartition 请求，但是目前 Controller 并没有办法知道这是个过期的请求。

想法也很直接，在[KIP-380](https://cwiki.apache.org/confluence/display/KAFKA/KIP-380%3A+Detect+outdated+control+requests+and+bounced+brokers+using+broker+generation)后， Broker 已经有了 epoch了，AlterPartition 请求带上 broker 的 epoch 就可以了，Controller 发现 broker 当前的 epoch 比 AlterPartition 请求的 epoch 要大，就认为这是个过期的 AlterPartition 请求，可以直接 reject。

方案的修改如下：

1. Follower 侧

因为 AlterPartition 请求是 Leader 发出的，所以 Leader 需要知道要添加到 ISR 的 replica 所在的 broker 的 epoch，所以 Follower 在向 Leader 发送 fetch 请求的时候就会带上自己的 broker epoch

2. Leader 侧

   - Leader 需要记录下 fetch 请求带过来的 broker epoch

   - Leader 在发送 AlterPartition 请求 的时候会带上 broker epoch

3. Controller 侧

Controller 将验证 AlterPartition 请求中要加入ISR 的 replica 所在的 broker 的 epoch 和元数据中的 broker 的epoch是否保持一致，如果不一致的话，就 reject 这次 AlterPartition 请求，并返回 INELIGIBLE\_REPLICA 的异常

### [KIP-966: Eligible Leader Replicas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-966%3A+Eligible+Leader+Replicas)

比较惊讶的是这个问题在最新的 Kafka 版本中还是存在，预计要在 Kafka 4.0 修复。

比较有意思的是，RedPanda（日志的复制是基于 Raft 协议，Raft 协议要求 flush 到磁盘）2013年5月写了篇[博客](https://www.redpanda.com/blog/why-fsync-is-needed-for-data-safety-in-kafka-or-non-byzantine-protocols)说 Kafka 这种写到 page cache，不强制 flush 到磁盘的行为是有问题的，会导致数据的丢失。并且还提供了代码和demo流程来演示数据的丢失。

然后 Kafka 社区就提了这个 KIP 来解决 RedPanda 说的这个问题了。

PS：关于 Kafka 的复制协议为什么不需要强制 flush 到磁盘，而Raft 协议需要强制 flush 到磁盘的文章可以参考[Why Apache Kafka doesn't need fsync to be safe](https://jack-vanlightly.com/blog/2023/4/24/why-apache-kafka-doesnt-need-fsync-to-be-safe)这篇文章，写得很好，我觉得解释得很清楚了。

#### 问题

首先有个背景是 Kafka 选 Leader 的时候只会（不考虑 unclean election 的情况）从 ISR 中选，当 ISR 集合中只有一个 Replica 的话，即使这个 Replica 挂了，也不会从 ISR 中移除掉。如果移除掉的话，就不知道选哪个作为 leader 了，因为 ISR 为空了。

基于这样的背景，这个问题会在最后一个在 ISR 中的 replica 挂了的时候发生。考虑如下的 case：

一开始一个 Partition 有三个 replica，ISR 为{0, 1, 2}，并且 min.isr 为 2

- T0 时刻，broker 0 与Kafka 集群失联了，从 ISR中剔除了，此时 ISR 为 {1, 2}

- T1 时刻，broker 1 也与Kafka 集群失联了，从 ISR中剔除了，此时 ISR 为 {2}

- T2 时刻，broker 2 直接挂了，在 page cache 中的数据还没flush 到磁盘，这部分在 page cache 中的数据丢了，但是 Kafka 并不会把 broker 2 从 ISR 剔除，因为 Kafka 要避免 ISR 为 空

- T3 时刻，broker 0 和 broker 1 恢复过来，但是 broker0和 broker1 都不在 ISR 中，controller 不会把他们选为 leader

- T4 时刻，broker 2 重启了，controller 把 broker 2 选为 leader。然后 broker 0 和 broker 1就开始 truncate + fetch 数据。于是这部分还没flush 到磁盘的数据就丢了

#### 解决

针对上面提到的 case，其实 broker 1 也可以被选为 leader，只要 T1时刻后 high watermark 不能被推进，不然 broker 1 的 high watermark为 10，broker 2 high watermark 为 12，消费者消费到了 offset 为 11 的消息。如果 broker 1 被选为 leader 后，消费者就再也消费不到 offset 为 11 的消息了。

并且，我们需要知道一个 broker 是不是 unclean shutdown 的，避免选择一个 unclean shutdown 的 broker 作为 leader，unclean shutdown 指是还没有 flush page cache 的数据到磁盘就直接挂掉。

如何知道一个 Broker 是不是 unclean shutdown 的

Kafka 通过在 server close 的时候写一个 CleanShutDownFile 来表示是不是clean 的shutdown，如果有这个文件，则表示是 clean 的shutdown，否则不是。

如何保证T1时刻后时刻后 high watermark 不能被推进

Kafka 提出一个更严格的限制 high watermark 前进的规则：只有当 ISR 的 replica 数量大于或等于 min.insync.replica 的数量时，High watermark 才可以前进。

之前的high watermark 前进规则是，只要 ISR 的中的 Replica 都复制到了某个 offset，high watermark 就可以前进到这个 offset。现在还需要 ISR 的 replica 数量满足 >= min.insync.replica 这个条件。

这样 T1 时刻后，ISR为1，无法满足 >= min.insync.replica 这个条件，high watermark 也就不会前进了。

如何让 Broker 1 也可以被选为 leader？

Kafka 提出来一个 eligible leader replica 的概念，简称 ELR；之前只能从 ISR 中选一个 leader，现在还能从 ELR 中选一个 leader 出来。Kafka 使用 ELR 来记录不在 ISR 中，但是 high watermark 之前的消息都有的replica。

有了 ELR 这个概念后，broker 1 虽然不在 ISR 中，但会在 ELR 中，这样可以从 ELR 中把 broker1 选出来作为 leader。

对于 ISR 和 ELR，Kafka 保证如下的行为：

ISR invariants：

ISR 可以为 empty 了，ISR 的行为和之前的行为保持一致。

ELR invariants

- ELR 中的 replica 一定不在 ISR

- ELR 一定有 high watermark 之前的消息

- ELR 可能存在消息复制的滞后

- 如果 ELR 不是空，那么 ISR 中 replica 的数量 就要小于 min.insync.replica

- 除非有 unclean shutdown，否则 Contoller ELR + ISR 的 size 永远不会小于 min.insync.replica ，

- 如果某个replica 有 unclean 的 shutdown，Controller 会把它从 ELR 中移除

修改主要体现在 controller 侧，当 Broker 向 controller 提出修改 ISR 的 proposal 的时候，controller 进行如下的操作：

1. 如果提出的 ISR 大于或等于 min isr，controller 接受提出的 ISR， 并清空 ELR

2. 如果提出的 ISR小于 min isr 的话，controller 将

   - 维持目前 ELR 的replica

   - 把（当前 ISR - proposed ISR）添加到 ELR 中

   - 把同时在 ISR 和 ELR 中的 replica 从 ELR 中移除掉

只有当提出的 ISR小于 min isr 的话， controller 才会把 replica 添加到 ELR 中，考虑到之前提出的限制，只有当 ISR 的 replica 数量大于或等于 min.insync.replica 的数量时，High watermark 才可以前进，这个时候 High watermark 也就不会前进了，并且 ELR 也一定包含high watermark 之前的消息，ELR 可以作为一个有效的 leader。

考虑上面提到的例子：

- T1 时刻，ISR 变成 {2}，由于 ISR 的数量小于 min isr，Broker 1 加入到 ELR 中，ELR 变成 {1}

- T2 时刻，Broker2 挂了，ISR 为空，ELR变成 {1, 2}

- T3 时刻，broker 0 和 broker 1 恢复过来，Controller 从 ELR 中选择一个 online broker 作为 Leader，Broker 1 成为 leader ，ISR 为{1}，ELR 为 {2}

- T4 时刻，broker 2 重启了，但是 Controller 发现它是一个 unclean shutdown，将其从 ELR 中移除，此时 ELR 为空集合

broker 1 作为了 leader，不就出现上面的问题

下面是一个例子来演示 ELR ， ISR 的变化和 Leader 的选举情况，假设有4 个 broker，min isr 为 3

![](images/img_20.png)

- T0 时刻，所有 replica 都在线

![](images/img_21.png)

- T1 时刻，b3 和 b4 同步 消息比较慢，leader b1 提出将 b3 ，b4 剔除 ISR 中，Controller 将 ISR 修改为 {b1, b2}，ELR 也更新为 {b3, b4}

![](images/img_22.png)

- T2 时刻，b3 追上来了，leader b1提出 将 b3 加入到 ISR 中，Controller 将 ISR 修改为 {b1, b2，b3}，此时 ISR 的数量大于等于 min isr 了，这个时候要将 ELR 清空。因为这个时候 high watermark 是可以前进的， high watermark 前进后，ELR 就不能被选为 leader 了

![](images/img_23.png)

- T3 时刻，b2 和 b3 掉线了，Controller 将 b2 和b3从 ISR 中移除，放到 ELR 中，此时 ISR 为 {b1}，ELR 为 {b2, b3}

![](images/img_24.png)

- T4 时刻，b4追上来了， leader b1 提出 将 b4 加入到 ISR 中，此时 Controller 修改 ISR 为 {b1, b4}，ELR 为 {b2, b3}

![](images/img_25.png)

- T5 时刻，b1，b4 掉线了，Controller 将 b1, b4 从 ISR {b1, b4}中移除，并把他们添加到 ELR ，并且此时 b3 恢复过来了，但是检测到是一个 unclean shutdown，应该从 ELR 中移除；于是 ISR 就变成 empty，ELR 为 {b1, b2, b4}

![](images/img_26.png)

- T6 时刻，b1恢复过来了，但是检测到是一个 unclean shutdown，controller 将其从 ELR 中移除， b2 也恢复过来了。因为此时 ISR 为空，所以 controller 将 ELR 中的 b2 选为 leader，添加到 ISR 中，并从 ELR 中移除，这个时候 ELR 为 {b4}

![](images/img_27.png)

- T7 时刻，b1，b3 都追上了 leader b2，leader b2提出 将 b1，b3 都加入到 ISR 中，此时 ISR 为 {b1, b2, b3}，由于 ISR 中 replica 的数量大于 min.isr，于是 ELR 也被清空了

![](images/img_28.png)

## 总结

写到这，总算是尽我所能，把我所知道的 Kafka 复制协议踩过的坑讲完了，当然可能还有一些我不知道的坑，不过上面的内容应该也可以覆盖大部分 Kafka 在复制协议踩过的坑了。不得不说，Kafka 踩过的坑还真不少呀～

不过虽然 Kafka 踩过不少坑，Kafka 在业界还是公认非常稳定可靠的分布式消息队列系统的。
