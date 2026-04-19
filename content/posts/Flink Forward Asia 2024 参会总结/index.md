---
title: "Flink Forward Asia 2024 参会总结"
date: 2024-12-09T21:08:50+08:00
draft: false
tags: ["Flink"]
categories: ["会议总结"]
summary: "Apache Flink在诞生10周年之际，回顾了其从实验项目到流计算标准的历程，并介绍了Flink 2.0的新特性如存算分离、流批一体及AI集成，同时发布了Paimon 1.0 和Fluss 0.5，旨在提升实时数据分析的效率和灵活性。"
ShowToc: true
TocOpen: false
---

# 11月29日上午

## The Past, Present, and Future of Apache Flink

今年（2024年）是Apache Flink 诞生 10 周年，王峰回顾了 Apache Flink 发展的10年，从一开始只是柏林工业大学的一个实验性项目，到在阿里巴巴大规模生产落地， 到成为流计算领域的事实标准，再到如今孵化出Paimon 构建流式湖仓。

想到大学里面学的“新技术” Hadoop，Hive，HBase 现在的处境，不得不让人感慨，一个开源项目能坚持10年，并且直到现在还依然有勃勃生机也是不容易呀。技术的持续领先固然是一回事，不过主要还是有后面商业化公司的支撑，现在活得好好的开源项目基本上后面都有商业化公司支撑。

最后预告了一下 Flink 2.0 的功能，存储分离，业务层流批一体（Materized Table），以及拥抱 AI 大模型。

## Apache Flink 2.0: Streaming into the Future

Flink 2.0 有很多吸引人的亮点，首先 Flink 2.0 的 Release Manager 宋辛童 介绍了 Flink 2.0 的 break changes，包含移除了一些 connector api，source/sink function，不一而足，也是解决了不少技术债，刚好趁着没有兼容性保证的 2.0大版本 发布一起解决掉。

然后梅源介绍了 Flink 2.0 的 State 存算分离的 feature，我觉得算是 Flink 2.0 最吸引人的 feature了。现在大家都在谈云原生，存储分离，而 Flink 1.0 的 state 还是在本地，和本地磁盘强耦合，本地大 state 带来的成本，扩展性等问题也是饱受诟病和非议。在 Flink 2.0 版本，这个问题终于得到了解决。我下午还特地去听了 Flink 2.0 State 存算分离专门的 Talk，这里面的一些工作确实还挺有意思的。

最后李麟也介绍了 Flink 2.0 中引入的 Materized Table，在SQL 层面上的流批一体。用户写一段 SQL，只需要设置 freshness，Flink 就能自动选择流模式和批模式了，不需要用户介入。我觉得未来的一些规划，比如自动 discovery Materized Table，通过 Materized Table 进行 SQL 优化还挺值得期待。

## Paimon 1.0: Unified Lake Format for Data + Al

李劲松介绍了 Paimon 这一年的发展，毕业成为 Apache 顶级项目，以及在各行各业大规模落地等。主要分享了一些用 Paimon 的动机，离线加速，提升数据的时效性，更快更便宜的廉价消息队列（分钟级别）等。
没有讲太多内容，然后就邀请了淘天，抖音，vivo 的工程师分享了一些他们基于 Paimon + Flink 的湖仓落地，更偏业务和技术细节多一点，但大体都是数据的时效性的提升等。

本人有幸见证了 Paimoin 从 Flink Table Store 到如今的 Paimon--中文社区最受欢迎的湖格式。
如果说以前 Flink 用户的态度是已经有了 Hudi 或者 Iceberg，为什么要用 Paimon。现在可能就是如果没有特别强的理由不用 Paimon，那就用 Paimon。

## Fluss: Next-Gen Streaming Storage for Real-Time Analytics

对于离线走向实时化，Paimon 可以做到分钟级别，但在强实时的场景下，还是需要秒级存储。
于是伍翀分享了面向流分析场景全新研发的新一代流存储 Fluss。
一开始介绍传统流存储如 Kafka 在流分析场景面临的痛点问题，比如不支持更新，Flink 消费需要额外的去重算子，数据无法复用，难以修正数据，数据无法探查，数据回溯难，只能存储几天数据。

然后就分享了 Fluss可以很好地解决上述提到的痛点问题。
此外Fluss ：
- 采用列式存储：可以在服务端进行列裁剪，可以节省大量网络成本，流读的吞吐也随着裁剪的列成比例上升。
- 和数据湖的结合：Fluss 将数据湖作为 tiered storage，计算引擎可以直接在湖上的数据进行分析。和数据湖结合的 Feature有个专门的 Talk 分享。
- 通过 Fluss，Flink 的双流 Join 可以被改造成 Delta Join，减免 Large State，大规模 Join 更稳定，中间数据可查。

最后就是现场开源 Fluss 了，虽然我早已经知道了 Fluss 要在FFA 现场开源，但是还是有点期待，主要是也没怎么见过现场开源的，想看看是啥样的。可能大家都没见过，现场的反响还是很热烈的，坐我旁边一老哥也一脸震惊“不会就在现场开源吧”。开源后，旁边的同事想赶紧上去点个 Star，当个早期 Star 者，不过就卡一两分钟，再去点 Star 就已经排在 180名开外了。
Fluss 项目地址：
[https://github.com/alibaba/fluss](https://github.com/alibaba/fluss)

## AI时代下大数据技术未来路在何方？

最后是圆桌会，探讨 AI 时代大数据技术未来路在何方，算是回答了在这个 AI 时代，我们大数据工程师或多或少会有的焦虑吧。大佬们一通聊，我也是听得云里雾里，反正最后我就记得了两点： AI 时代，大数据会更重要；以及 OpenAI 收购了 Rockset 也佐证大数据的重要性

# 11月29日下午

## Paimon 1.0: Unified Lake Format for Data + Al

主要介绍了 Paimon 1.0 的功能和特性，记录下几个自己觉得有意思的点
- Object table
非结构化的数据纳入结构化的湖格式的管理，比如 oss 上的一堆非结构化数据的文件，文件名，文件大小，文件类型，文件url 等就可以作为一个表的列；然后就可以在这个表上根据这些列进行过滤，定位到对应文件的url，最后机器学习工具就可以分析这个文件。避免了在对象存储上直接进行的 list 。

- Partition Finish markdown
分区结束后 若干分钟 内没有数据达到，将分区done 的标志写到元数据；然后下游就可以根据这个标志调度对应的 job

- Branch 和 Tag
Branch：融合批和流的数据，一张相同的表具有批分支和流分支，为了数据的准确性，批分支需要回刷，查询也是以批分支为主。
比如批分支有dt = 11-20，dt=11-21，dt=11-22 分区，流分支有 dt=11-23 分区；等到数据回刷到主分支后， dt=11-23 就可以被清理掉了。主分支查不到的分区，去流分支查；
Tag：没有分区的话，可以打一个 tag，tag 作为分区

- iceberg兼容
思路比较直接，写Paimon snapshot 的时候也写一份 iceberg metadata；

## 基于 Flink 进行增量批计算的探索与实践

流计算时效性高，但是成本较高。增量计算的思路就是将用户的 Query 以批的方式执行，但是不需要重新计算所有数据。每次执行的时候只计算增量部分，增量部分指的是 当前时刻的数据 - 上次执行的数据。执行的过程中需要记录下执行进度，方便下次调度的时候知道从哪个位点开始读增量。

目前只支持了Append 流，Retact 流还不支持。Retact 流如果借助下游的表的 merge 能力，比如 Paimon 的 aggregate table， 还是比较好做的，不然可能还得把上次计算的结果作为状态，供下一次计算使用，有点复杂了。

增量批计算还是比较依赖 Flink source的能力，比如只计算增量部分就需要 Source 来告诉 Flink 增量的这部分数据。与 Paimon 的对接是通过 TimeTravel 的方式来支持的

## 流批一体向量化引擎Flex

Native + 向量化的魔爪终究还是伸向了 Flink 。
蚂蚁的刘勇分析的流批一体向量化引擎 Flex，基于 Flink + Velox 。不过目前支持的功能还比较有限，只支持 Cal 算子和 Native Source/Sink，有状态的算子还不支持。

主要讲了一个优化点就是 projection reorder，就是一个 projection，有引用字段（引用 input 的 f1，f2，f3），还有可向量化的 cal function。如果全部都作为一个 native cal 算子的话，需要额外将引用字段也进行行转列；通过projection reorder，生成两个算子，一个非native 算子，一个 native 算子，避免引用字段的行转列。

并不是全链路的向量化，会有一定行转列，列转行的开销，看作业的效果，有部分作业的性能还出现了回退。

## Flink 2.0 存算分离状态存储 —— ForSt DB

这一场技术干货比较多，State 存算分离的好处不言而喻，state 不再受本地磁盘的限制，启动速度快（不需要下载 state 到本地）， checkpoint 轻量（不需要在checkpoint的时候上传大量文件）。
不过，state 完全放到远端，性能还要和 state 放到本地持平甚至更优，是有不少工作要做的；
***框架层面***
- State 访问异步化
毫无疑问，State 访问一定要异步化，不然就会一直卡在网络访问上了，cpu 基本干不了活。

不过异步化也带来一个问题，乱序问题如何解决，对于相同的 key，一定要这个 key 的 state 访问结束了才可以开始下一次对这个key 的处理。将对同一个 key 的操作进行划分，先进先出，处理完一个操作后再处理下一个操作。

- 攒批
读写线程分离，读 + 写分别进行攒批；

- Unaligned checkpoint 的支持
之前的 Unaligned checkpoint 需要checkpoint in-flight data，在存算分离 state 下，还需要额外 checkpoint 当前尚未开始的 state request。

***State 层面***
For Steaming DB = Forst DB 存算分离版的 rocksdb，本地磁盘只作为 cache；

看起来像是利用了 RocksDB 抽象出来的 FileSystem 的接口，这么说也不准确，应该得益于 RocksDB 将自己对FileSystem的访问抽象出来了。

最后 Benchmark 结果表明：50 % 的 磁盘 cache 下，存算分离版本 state 性能和本地 state 性能差不多，略好一点点。

## 打破Watermark壁垒：如何实现(跨)Flink作业的实时进度感知与自动对齐

小红书的陈宇分析了对 Lookup join 的改造，玩得比较花。

双流 join state 大，left join 又会 join 不上，存在数据正确性的风险，延迟 join 也不好评估延迟时间。

Left join 比较好，但是数据可能会join不上，如果有办法保证 join 上就好办了。
核心思路是让 left join 的左表可以感知维表的进度，这样即使join不上，等一会去join就可以了。进度用 watermark 来衡量，这也意味着 左表 和 维表 需要在 event time 上是可比的，如果不可比就没有意义了。

Flink 同步作业同步数据到维表，会在维表中记录一下 watermark 的时间；然后左表 join 维表的时候，看一下自己的 watermark 和维表的 watermark，如果 维表的 watermark 更大，就认为 join 上了，不然就认为自己的进度比维表的进度慢，就等会再 join。

# 11月30日上午

上午的会场比较热闹和聚焦，美团，阿里云，腾讯，抖音都分享了湖上加速的工作，大家都不约而同地开展这方面工作，或许可以预见这未来会是个大趋势。

## 美团增量湖仓Beluga的架构设计与实践

看起来像是 Hudi + HBase 的结合体，HBase 提供高效的点查 + CDC 生成能力（流能力），Hudi 来支持批上的一些能力；

## 流存储Fluss：迈向湖流一体架构

本人的一个分享，主要是分享了基于流存储 Fluss 来构建湖流一体的架构；Fluss 开源的时候，很多人问 Fluss 和 Paimon 的关系，是不是竞争关系什么的，Fluss 和 Paimon 怎么选。这个talk 也算是回答了这个问题，Fluss 和 Paimon 是互为补充的一个关系。

只需要分钟级延迟：Paimon
需要秒级延迟，不关心复杂查询能力：Fluss
需要秒级延迟，并且需要复杂查询能力：Fluss + Paimon

理想的架构应该是 Fluss 和 Paimon 都有，但是 Fluss 把 Paimon 管起来了，用户其实只看到了 Fluss。

这个 Talk 主要还是分析了一下目前 湖存储搞一套架构，然后流存储搞一套架构的问题，引出了 Fluss 如何统一湖流，以及在 Fluss 统一湖流架构上可以带来什么好处。

分享完之后，大家的反响还是比较热烈，下来之后也有不少社区朋友交流了一些问题，这下没有人再有问 Fluss 和 Paimon 的区别了，问题主要集中技术细节，性能，自己的业务能不能用上，对 Iceberg 的支持等。

欢迎找我交流！！

## 腾讯大数据天穹流批一体建设之流批一体存储BSS

其实没听这个 talk，因为我讲完之后就一直被拉过去回答问题了，所以错过了这个 talk，不过我从 PPT 推测一下实现。

痛点也还是湖格式无法实现秒级延迟，然后就提出了 BSS，提供秒级流读 + 兼容数据湖。架构大概是 Plusar + Iceberg；数据写到 Bookie 中，这是秒级延迟的部分，然后有个 Lake Sync 来将数据同步到 Iceberg 中；不得不说，和 Fluss 还是很像的。 PPT 里面还一堆复杂的架构图，看得脑壳疼，我实在 YY 不来了。

最后未来展望还讲了和 Fluss 的融合，我其实就很好奇，这看起来差不多的东西，是要咋融合.....

## BTS - 抖音集团流批一体存储服务

痛点也差不多，湖上无法做到秒级延迟；思路也差不多，在湖上架个 server 提供秒级延迟。
基于 Hudi 做的，数据首先buffer 在内存中，然后 flush 到 HDFS 上，然后也会有个单独的 service 来将数据转成 湖格式。然后讲了BTS的内部实践收益：
- 降低成本 40%：主要是以前的 MQ（实时链路） + Hive（离线链路） 干掉了，换成了 BTS
- 单表可同时支持实时写和批读批写

# 11月30日下午

下午实在听不动了！

## Flink+Paimon+Hologres，面向未来的一体化实时湖仓平台架构设计

讲了一下 Paimon 作为 Hologres 的外表以及 Hologress 的 dynamic table。

## 基于 Flink 和 Paimon 构建 Pulsar 的大规模消息追踪平台

反正基本上都在介绍Pulsar，然后顺便讲了一下他们内部将 Pulsar 的数据同步到 Paimon 的实践，没太多技术干货。

## 基于 Paimon x Spark 构建极速湖仓分析

讲了不少 Paimon + Spark 的优化
- Cache: catalog cache,plan cache
- deletion vector：解决merge 读的并非限制，非主键字段的过滤条件无法下推
- Scan IO 优化：各种 pushdown，各种裁剪
- bucket join

## Flink + Doris 的实时湖仓解决方案

基本上都在介绍Doris，Doris 的存算分离架构，compute layer 不存数据，只有 cache，storage layer 存数据，基于 s3/oss，然后有个专门的 meta layer 来管理元数据等，然后讲了一下 Doris 的 LakeHouse 的解决方案，Flink + Paimon + Doris；
