---
title: "QCon 北京参会总结"
date: 2025-04-12T23:54:45+08:00
draft: false
tags: ["AI"]
categories: ["会议总结"]
summary: "QCon北京大会大模型正在重新定义软件参会总结"
ShowToc: true
TocOpen: false
---

这几天去北京参加了 [QCon 大会](https://qcon.infoq.cn/2025/beijing/schedule)，主题是大模型正在重新定义软件，现在我满脑子都是 AI Agent。

本着“参会不总结，等于没总结” 的原则，总结一下这几天的一些 talk 吧

## [从代码到文明：AI 安全的技术深渊与治理边界](https://qcon.infoq.cn/2025/beijing/presentation/6421)

聊了一下 大模型的安全问题，比如通过提示词注入，不安全模型加载和文件处理，不安全的反序列化与处理（分布式训练/模型推理序列化机制）。关于不安全的反序列化有一个例子比较有意思，可以通过提示词让 AI 使用一种编解码方式（甚至是一种之前完成不存在的编解码方式）。

然后讲了一下大模型在腾讯内部网络攻防的应用：

- 漏洞挖掘，大模型辅助进行二进制逆向分析

- 渗透过程，用大模型辅助自动化部分渗透路径

- 大模型辅助钓鱼攻击

最后提了一个 AI 应用隐私保护的一个流程：

1. 终端小模型对用户的输入进行脱敏信息

2. 云端大模型进行处理，返回处理结果

3. 终端小模型对处理结果反脱敏，返回给用户

## [生成式 AI 驱动的软件开发生产力变革](https://qcon.infoq.cn/2025/beijing/presentation/6417)

主要就是讲了一下 Amazon Q Developer CLI，可以理解为一个 AI code agent，提示一个任务，Q Developer CLI 帮你把代码写好。甚至还提了一个例子，Amazon Q Developer CLI 说他们有一个 fetaure，完全由一个不懂 Rust 人通过 Q Developer CLI开发。看起来自己要失业了。

这个 Amazon Q Developer CLI 其实不是由一个 agent 构成，由多个单一功能的 agent组成的，比如输入 /dev 调起写正式代码的 agent，输入 /test 调起写测试代码的 agent。

我比较认可他的一个观点是 ，AI 帮助我们开发，开发者去学习背后底层技术，花更多时间提示自己的认知。

不过我还是很焦虑，因为我并不想花时间提升自己的认知....

## [Al Vision Shape the Future](https://qcon.infoq.cn/2025/beijing/presentation/6385)

微软的一个分享，听完这个 talk，感慨 AI Agent 居然已经在国外发展这么快了吗。

微软云平台提供了多个不同的 agent，分别对应完成不同的 sub-task，用户在微软的平台拖拉拽多个不同 sub-task 的agent协助来完成task。

以一个企业为例，企业的每个部门都可以抽象成一个 agnet，如人力资源，IT等。对于员工办理入职，有个 plan agent 去生成 plan，生成执行计划，然后多个 agent 互相通过聊天的方式进行交互。不过这期间还是需要人介入，来选择要不要执行这些步骤。

另外举了一个他们的 use case，用户要买一个音箱， agent 自己去操作web浏览器，自动完成一系列操作。

## [Agent 元年，关于知识管理的新思考](https://qcon.infoq.cn/2025/beijing/presentation/6401)

有一个观点我挺认同的，想让 Agent 帮你打工并不容易，比如写周报，但是你需要告诉 Agent 你这周做了什么，Agent 需要更加了解你。

于是他们搞了个 Remio，一站式管理你的文件，网页和文件自动入库，了解你的一切信息。

## [Fluss 湖流一体：Lakehouse 架构实时化演进](https://qcon.infoq.cn/2025/beijing/presentation/6297)

没什么好多说的，因为是我自己的一个分享。小米的老师会后找我要了份 PPT，说要带回去在他们内部分享下。

## [StarRocks x Iceberg: 探索Lakehouse架构极致查询性能](https://qcon.infoq.cn/2025/beijing/presentation/6328)

OLAP 也太卷了吧。

StarRocks的老师 分享了一下他们在 Iceberg 上做的优化，不少干货。

- Metata 解析开销大，Plan 耗时长 
  - 分布式 Metdata Plan，把元数据的解析当作 starrocks的一个query，发给 BE

- 统计信息不足导致 plan 恶化 
  - 查询触发统计信息收集

- 冷数据 IO 访问开销大 
  - 针对 AWS client 进行优化，poco client，poco 连接池

- Cache 不够 smart，访问频次不高，但是延迟敏感 
  - IO 自适应，磁盘（local cache）达到瓶颈，远端访问也许更快

- Cache miss 引起抖动 
  - Cache 共享，新节点的加入，从其他节点访问cache 数据

- 字符串执行效率低，难以向量化，内存占用高，传递开销大 
  - 低基数字符串优化，类似字典压缩；数据分析场景，80%是低基数字符串

- Parquet 文件解析开销大 
  - 高效谓词优先（优先执行能过滤更多数据的 predicate ），Filter 下推到 Decoder

- 湖上物化视图 
  - 自动查询改写

## [从碎片到统一：如何用元数据湖解决多 Lakehouse 治理难题](https://qcon.infoq.cn/2025/beijing/presentation/6311)

主要是讲了一下 Gravitino，数据统一视图，统一访问和治理，没有太多技术细节。

其实我一开始是比较好奇他们是怎么管理非结构化的数据，Gravitino 定义了一种 FileSet 类型，本身并不存储非结构化数据，只是记录一个文件引用，如下所示：

```
FileSet {
  name: string,
  storgeLocation: string,
  type: Type
}
```

## [关于人工智能大模型的几点思考](https://qcon.infoq.cn/2025/beijing/presentation/6376)

郑纬民院士讲了构建人工智能大模型的关键技术点和挑战，然后说了一下他们搞得一些技术或者系统如何来解决这些挑战。我觉得还是很有参考价值的。

- 大模型训练需要的数据量大，小文件多，元数据管理难 
  - 他们搞的 Super Fs： 分布式管理元数据，并且目录元数据和文件元数据分开

- 数据预处理

他们搞了个叫诸葛弩的东西，看起来是 C++写的一个 Spark，兼容PySpark编程接口，提供基于 C++ RDD 编程接口

- 模型训练 
  - 十万张卡训练模型，平均每个小时会发生一次硬件，软件错误，所以训练过程需要对模型参数记录到检查点文件，避免发生故障后重新训练。大模型检查点文件的读写对存储系统提出了更高的要求，于是他们搞了个分布式检查点，数据被均匀分布到所有参与训练的检查点

## [从指令到 Agent：基于大语言模型构建智能编程助手](https://qcon.infoq.cn/2025/beijing/presentation/6294)

讲了一下字节跳动搞的 code agent。印象比较深的是他们如何管理多轮对话与记忆。与 agent 交互的时候，随着交互轮数变多，需要的context 也就更大，消耗的token 也就越多，所以需要对 context 进行压缩。他们试过用 大语言模型对信息进行摘要，但是大模型很贵，于是他们最终选择通过规则和策略决定那些信息决定面小，然后将这些信息丢失掉，成本也可控。

## [Agentic RAG 的现在与未来：从使用工具到重构知识系统](https://qcon.infoq.cn/2025/beijing/presentation/6323)

强调了对于 Agent，上下文是关键。举了个自己使用 Code Copilot 的感受，使用 Github 的 Copilot，跨多个文件进行 code 的效果不是很好，但是使用 Cursor，跨多个文件进行 code 效果很好，原因Cursor有更多的 context，可以捕获到整个项目的context。

我觉得他提到一个 Agentic RAG 系统架构还是很有参考价值的：

- 有一个 Agent 控制器，负责规划，行动，反思

- 有一个检索模块，支持语义检索，多轮动态检索

- 有一个工具调用模块，调用外部工具来执行任务，搜索，数据库操作等

- 状态管理 & 日志模块，追踪整个 Agent 的过程，思考过程，记笔记（用户的这个问题通过什么方式解决了，Agent 在检索过程中缺少了什么数据）等对 Agent 进行迭代，反馈

## [云端 AI Agents 系统开发的探索与实践](https://qcon.infoq.cn/2025/beijing/presentation/6399)

核心还是要有多个 AI Agent 之间的协同，编排。有一个 Super Agent 进行 Plan，Sub Agent 并行化进行处理，Super Agent 再进行归纳总结。

## [面向 AI Agents 的高性能数据基座：架构和工程实践](https://qcon.infoq.cn/2025/beijing/presentation/6377)

在 AI Agent 时代。一个统一的支持多模态的，低延迟的数据存储很重要。因为 AI agent 需要多轮交互，每一轮交互都必须快速响应，并且，AI 去检索知识总归是需要去不同类型的数据库进行检索，图数据库，文档数据库，向量数据库等。

于是他们想做的中间（上层是计算引擎，下层是物理存储）的这一层数据 Cache 以支持高速访问， Cache 兼容上层计算引擎 APi，如 Mongo API 等。

我觉得这位老师讲的思路很清晰，不过总感觉有点“虎头蛇尾”，讲了这么多统一的数据存储，最后只是展示了一些他们同时做的 KV 存储 EloqKV，说和 RocksDB 比性能更好。我问了一下性能好的原因，他说他们分析了一下，是因为他们用了 协程（减少了线程上下文切换的开销） + 异步 IO（IO Uring，SPDK）。

## [Data Warebase-- 一体化数据平台的云原生实践](https://qcon.infoq.cn/2025/beijing/presentation/6338)

忘了是前多少任老板的一个分享了，主要是分享他们公司做的 ProtonBase，一个一体化的分布式数据库，支持OLAP，OLTP，向量检索等。

我主要记了三点值得关注下 
- 如何水平扩展 
  - 水平扩展就是通过分shard 来解决，但是有 hash 的方式分shard，有range 的方式分shard，他们选择来 range 的方式。理由是从用户的角度看，range 分片范围查询效率高，扩缩容可自动进行（hash 分片需要预先定义到分多少个片，并且要做大量数据的迁移），虽然工程实现上要复杂点

- 如何做存算分离 
  - 传统的方式就是直接写对象存储，但是说写对象存储不能满足他们的高并发实时写入（s3天然不支持低延迟，频繁调用）。所以他们抽象出来一个统一的存储层，使用高速本地盘或云盘，内置 raft 协议进行高可靠。下面还有个对象存储来进行冷存以节省成本。看起来像是 Pangu + OSS ？毕竟副总裁是前盘古团队架构负责人（雾

- 如何做 HATP 架构 
  - 传统方式就是 OLTP（行存） 一套存储，OLAP（列存） 一套存储；然后自动从 OLTP 到 OLAP。他们搞了个混合存储只要一套存储同时满足 OLTP 和 OLAP 的需求。没讲太多细节，我问了下是咋做到的，直观看起来一套存储满足两种需求不太 make sense。前多少任老板说是它们内部搞了一种文件格式，可以满足两者的需求，具体文件格式啥样没讲太多。不过个人感觉可能还是列存本身，只不过可能加了索引什么的可以满足高性能实时写入，高性能点查吧。我估计真正应对 OLTP 场景，数据频繁单行单行修改，并且也要保证读/写毫秒延迟也够呛。

## 个人总结

- AI Agent 应该是目前最火爆的方向了，我觉得也是真正让大模型更近一步落地的方向。相比训练出一个通用的大而全的 Agent，训练出多个专攻特定领域的 Agent 更切实际并且效果要更好。多 Agent 之间的协作，编排是值得继续关注的方向

- Context 对于 Agent 来说非常重要，如果高效管理 Context 是个值得关注的方向

- 虽然大家都在说多模态数据，但其实目前貌似我也没有看到太多关于如何管理多模态数据的案例分享

- 随着 AI 对数据有不同的访问需求，统一的元数据层是比较明确的一个趋势。虽然也有在提统一的存储系统，不过我还是觉得底层存储还是比较难统一，因为不同存储确实有他自己擅长的领域
