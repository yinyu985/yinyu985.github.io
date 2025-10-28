---
title: Elasticsearch写入优化
slug: elasticsearch-write-optimization
tags: [ELK]
date: '2024-11-16T19:40:25+08:00'
---

## 收到工单

用户提工单到 L2 说写入消息堆积了，第一时间检查 Elasticsearch 这边的日志和监控，把一百多个节点的日志都遍历看了一遍，并没有看到任何异常的关键字。再把 Grafana 监控看了一个遍，也没有任何明显异常的指标，只有少数几个 data 节点超出 85% 水位。没办法，只能从写入端的日志去着手了。

刚开始就看到每分钟内多次的 HTTP 429 报错，还有 socket timeout 的报错。从下午排查到晚上，实在 Elasticsearch 这边没有任何异常。客户那边也上会了，问了下：“你们这个重试是怎样的？” 回答是等待十秒重试一次。另外，客户是通过 Bulk API 来批量发送请求的，但是每次 Bulk 发送的大小不固定：小于 2000 直接发送，大于 2000 截断发送（这个肯定是不合理的，后面再讲）。

这个等待重试策略，如果 Elasticsearch 因为写入请求太多导致阻塞，那么简单的等待重试肯定是解决不了任何问题的。当天就推荐用户修改为退避式重试。修改完后的写入端日志里，不再有 429 的报错了，但还是会有 socket timeout 的报错。

结合之前注意到的情况，猜想是请求发送的次数太多了。然而单个 Bulk 的数据量明显不到推荐值，没有被充分利用。官方推荐的 Bulk 大小是 5–15 MB，而目前的情况是，几十条甚至一两条的写入请求也会被直接发送出去。

推荐用户修改 Bulk 单次提交数据量后，仍然没有任何缓解效果，消息队列消费不完，还是经常性出现堆积。

data 节点的配置是 16C64G，检查了写入线程数也就是默认是 16，这个没法改，最大也就改到 17。写入线程队列是 6.x Elasticsearch 的默认值 200，这个改大也不能解决消费慢的问题，只能让更多的线程在排队。

其他方面，不同索引的刷新配置 `refresh_interval` 设置的 30–60 s，这也是一个合理的值。translog 落盘配置检查过也都是默认值。唯一一处疑点就是 mapping 的 `max_field` 设置的是 10000，`max_depth` 设置的是 20，这个不是正常值，但也不会导致写入慢吧？用户反映他们的 mapping 并不复杂，也不存在 20 层嵌套。逻辑上分析也没啥问题——我定义一个瓶子最大能装 1 吨水，也不是说我的瓶子真的要装 1 吨水。

## 硬着头皮

按理说，云厂商提供云产品服务，只提供服务和运维，调优能调的肯定在售卖给客户时已经是最优配置，不可能让用户买到的实例再去关闭 swap、调最大文件句柄这种东西。

那么现在的问题就是：Elasticsearch 这边没有任何异常，只是客户觉得写入慢了。没办法，价值观第一条就是 `客户第一`，谁叫客户是上帝呢？说到这，我就想这话传的这么广，到底是谁说的呢？于是就搜了下。

> 西方并不用上帝来形容顾客，“顾客就是上帝”是在日本歌手[三波春夫](https://baike.baidu.com/item/三波春夫/5908901?fromModule=lemma_inlink)的说法基础上意译而来。
>
> 在中国，“顾客就是上帝”似乎是一句随处可见的口号，但在这句貌似舶来品的短语中，“上帝”一词其实并没有特别对应的翻译。通常西方并不会用上帝（god）这个词来表示对顾客的尊重，往往代之以“顾客优先”（customer first）、“顾客总是对的”（The customer is always right），或者“顾客就是国王”（The Customer is King）等。
>
> 更为相近的说法来自于日本，但是有趣的是这说法并非出自制造业，也不是出自服务业，而是由日本歌手三波春夫于 1961 年的时候提出的。三波春夫的原话如果直接翻译，应该是“顾客就是神”，这个“神”对应日本神道教的多神。此言一出，即惹来争议，媒体将这句话列为“暴发户的拜金主义”的典型。为此三波春夫曾多次解释，说自己这句话的意思是指自己对观众唱歌的时候，就好像是在对神明祈祷一般。但不管本意如何，这句话迅速被服务业追捧成为口号，而在这个基础上，加上中国式意译，“顾客就是上帝”这个说法就产生了。

这话能传成这样，也是没谁了。顾客想听歌，所以有人上了天堂；顾客想看跳舞，所以有人上了天堂；顾客想打篮球，所以有人上了天堂。我想到一位故友，可以一换三！！！

一边是完全没有任何报错的 Elasticsearch 日志和没有任何异常的监控，一边是客户发出来的消息堆积曲线，只能被迫硬着头皮去接着分析写入端的日志。

在我看到打印的时间后面，每条日志都有一个 `[xxxxxx-pool-x]`，当时我就问：“你们这个是多线程写入的吗？” 因为我看到这个 pool。上帝当时就在会上，直接说：“你先别管怎么写的，你先解决 Elasticsearch 问题。” 我……您是上帝，我没脾气。

后面了解到，用户的业务需要，所以为每个文档指定了 ID，并且写入的数据有百分之 50 的概率是会被更新的。这就又需要提到另一个点了：[Elasticsearch 使用误区——频繁更新文档](https://developer.aliyun.com/article/1586035)。但是上帝要更新，没办法，只能推荐把 `refresh_interval` 设置得更大一些。

说不要指定文档 ID，但是上帝业务需要，又是一个不肯改。这还没完，上帝开始不爽了，说：“我这个集群运行了一年没出现问题，只保留六个月的数据，也不存在写入流量激增，为什么出问题了？”

### 为什么建议非必要不要给 Elasticsearch 文档指定 ID？

> - 指定 ID 会导致写入前检查是否存在，增加查询开销。
> - 指定 ID 需要分片间协调，增加分布式系统的复杂性。
> - 手动生成 ID 容易出现冲突，导致写入失败或重试。
> - 自动生成 ID 更高效，避免额外的冲突检测。
> - 指定 ID 会增加事务日志的大小，降低写入性能。
> - 自动 ID 利用哈希算法快速定位分片，减少写入延迟。

## 无能狂怒

上帝不但系统设计薄弱，使用最简单的等待重试，业务系统没有反压逻辑，还不熟练 Elasticsearch 的使用，在写入时不使用任何调优写入的方式。在每天写入一亿多条数据的情况下，还有百分之五十的更新几率。在种种骚操作加持下，我觉得支撑了一年多也是挺强了。

在我们绞尽脑汁想还有哪里可以优化的点时，上帝还在群里隔三差五叫几句：“消息又堆积了，四千万了”，“XX 云三倍的体量都没出现任何问题”，“请给出止血方案”，“怎么性能这么差，要不重启试试吧”。

在业务低峰期重启还好，写入量不大，分片能够迅速重新分配。但是上帝可不管你这么多，在写入高峰期重启，大量写入和更新，分片分配任务几乎是卡住不动，然后集群一天都是黄的。上帝又开始叫了，简直不知道肩膀上顶的是个什么物体。

让你一天做一亿个俯卧撑，然后还要抽空去拉屎吃饭，看你啥状态？七七八八的优化建议给了七八条，结果上帝表示：“你这个没有依据。” 我尼玛，真不能忍了，qnmlgb，技术显然没有人难解决啊，我敲！

## 躺平摆烂

如果你去医院看病的时候，也是这样一幅“顾客就是上帝”的熊样，那我真要给你竖大拇哥，你看医生会不会叫你滚出去？说程序员是世界上最单纯的人，眼里只有 0 和 1，我确实佩服一心钻研技术的大佬们，这种人就像苦行僧一样，与程序为伴以此为乐。

但有些写程序只为养家糊口的垃圾人，自己写出垃圾代码还一幅无知者无畏、“顾客就是上帝”的态度，对待云厂商的技术支持颐指气使，我实在不得不感叹人性的丑恶，真是百度搜不到你，得去搜狗啊。我想清楚了，狗咬人，人不咬狗。

> 郭德纲：内行要是与外行去辩论那是外行。比如我和火箭科学家说，你那火箭不行，燃料不好，我认为得烧柴，最好是煤，煤最好选精煤，水洗煤不好。如果那个科学家拿正眼看我一眼，那他就输了。

```java
"es[write]elasticsearch[tianwang-data-42][T#9]" #163 daemon prio=5 os_prio=0 cpu=820840.80ms elapsed=376684.96s tid=0x00007f1bca5ff800 nid=319 runnable  [0x00007f1b793fc000]
   java.lang.Thread.State: RUNNABLE
 // 线程正在执行中
 at java.util.Collections$SetFromMap.add(java.base@11.0.19.19-AJDK/Collections.java:5566)
 // 尝试将一个元素添加到 SetFromMap 中
 at java.util.Collections$SynchronizedCollection.add(java.base@11.0.19.19-AJDK/Collections.java:2040)
 - locked <0x0000000716b6cb08> (a java.util.Collections$SynchronizedSet)
 // SynchronizedCollection 是一个线程安全的集合，这里正在进行同步操作，锁住了 SynchronizedSet
 // 这个锁确保在同一时间只有一个线程可以修改这个集合
 at org.apache.lucene.index.IndexReader.registerParentReader(IndexReader.java:140)
 // IndexReader 正在注册一个父 Reader，这是 Lucene 搜索引擎的一部分，用于管理索引读取器
 at org.apache.lucene.index.BaseCompositeReader.<init>(BaseCompositeReader.java:77)
 // 初始化一个复合读取器，用于管理多个子读取器
 at org.apache.lucene.index.DirectoryReader.<init>(DirectoryReader.java:367)
 // 初始化一个目录读取器，用于读取存储在目录中的索引
 at org.apache.lucene.index.FilterDirectoryReader.<init>(FilterDirectoryReader.java:91)
 // 初始化一个过滤目录读取器，用于对目录读取器进行过滤
 at org.elasticsearch.index.shard.IndexSearcherWrapper$NonClosingReaderWrapper.<init>(IndexSearcherWrapper.java:113)
 at org.elasticsearch.index.shard.IndexSearcherWrapper$NonClosingReaderWrapper.<init>(IndexSearcherWrapper.java:110)
 // 初始化一个不关闭的读取器包装器，用于在 Elasticsearch 中管理索引读取器
 at org.elasticsearch.index.shard.IndexSearcherWrapper.wrap(IndexSearcherWrapper.java:76)
 // 包装索引读取器，使其具有特定的行为
 at org.elasticsearch.index.shard.IndexShard.acquireSearcher(IndexShard.java:1345)
 // 获取一个索引分片的搜索器，用于执行搜索操作
 at org.elasticsearch.index.shard.IndexShard$$Lambda$3197/0x0000000800e66c40.apply(Unknown Source)
 at org.elasticsearch.index.engine.Engine.getFromSearcher(Engine.java:628)
 // 从搜索器中获取文档
 at org.elasticsearch.index.engine.InternalEngine.get(InternalEngine.java:732)
 // 内部引擎获取文档
 at org.elasticsearch.index.shard.IndexShard.get(IndexShard.java:1018)
 // 索引分片获取文档
 at org.elasticsearch.index.get.ShardGetService.innerGet(ShardGetService.java:173)
 // 分片获取服务内部获取文档
 at org.elasticsearch.index.get.ShardGetService.get(ShardGetService.java:94)
 // 分片获取服务获取文档
 at org.elasticsearch.index.get.ShardGetService.getForUpdate(ShardGetService.java:108)
 // 分片获取服务获取文档并准备更新
 at org.elasticsearch.action.update.UpdateHelper.prepare(UpdateHelper.java:78)
 // 准备更新操作
 at org.elasticsearch.action.bulk.TransportShardBulkAction.executeBulkItemRequest(TransportShardBulkAction.java:185)
 // 执行批量请求中的单个项
 at org.elasticsearch.action.bulk.TransportShardBulkAction.performOnPrimary(TransportShardBulkAction.java:165)
 at org.elasticsearch.action.bulk.TransportShardBulkAction.performOnPrimary(TransportShardBulkAction.java:156)
 // 在主分片上执行批量操作
 at org.elasticsearch.action.bulk.TransportShardBulkAction.shardOperationOnPrimary(TransportShardBulkAction.java:143)
 at org.elasticsearch.action.bulk.TransportShardBulkAction.shardOperationOnPrimary(TransportShardBulkAction.java:82)
 // 在主分片上执行分片操作
 at org.elasticsearch.action.support.replication.TransportReplicationAction$PrimaryShardReference.perform(TransportReplicationAction.java:1059)
 at org.elasticsearch.action.support.replication.TransportReplicationAction$PrimaryShardReference.perform(TransportReplicationAction.java:1037)
 // 执行主分片上的操作
 at org.elasticsearch.action.support.replication.ReplicationOperation.execute(ReplicationOperation.java:104)
 // 执行复制操作
 at org.elasticsearch.action.support.replication.TransportReplicationAction$AsyncPrimaryAction.runWithPrimaryShardReference(TransportReplicationAction.java:433)
 // 使用主分片引用异步执行操作
 at org.elasticsearch.action.support.replication.TransportReplicationAction$AsyncPrimaryAction.lambda$doRun$0(TransportReplicationAction.java:374)
 at org.elasticsearch.action.support.replication.TransportReplicationAction$AsyncPrimaryAction$$Lambda$2951/0x0000000800e10c40.accept(Unknown Source)
 // 异步执行操作的 lambda 表达式
 at org.elasticsearch.action.ActionListener$1.onResponse(ActionListener.java:62)
 // 监听器响应操作完成
 at org.elasticsearch.index.shard.IndexShard.lambda$wrapPrimaryOperationPermitListener$14(IndexShard.java:2676)
 at org.elasticsearch.index.shard.IndexShard$$Lambda$2953/0x0000000800e11440.accept(Unknown Source)
 // 包装主分片操作许可监听器的 lambda 表达式
 at org.elasticsearch.action.ActionListener$1.onResponse(ActionListener.java:62)
 // 监听器响应操作完成
 at org.elasticsearch.index.shard.IndexShardOperationPermits.acquire(IndexShardOperationPermits.java:273)
 at org.elasticsearch.index.shard.IndexShardOperationPermits.acquire(IndexShardOperationPermits.java:240)
 // 获取主分片操作许可
 at org.elasticsearch.index.shard.IndexShard.acquirePrimaryOperationPermit(IndexShard.java:2651)
 // 索引分片获取主分片操作许可
 at org.elasticsearch.action.support.replication.TransportReplicationAction.acquirePrimaryOperationPermit(TransportReplicationAction.java:996)
 // 复制操作获取主分片操作许可
 at org.elasticsearch.action.support.replication.TransportReplicationAction$AsyncPrimaryAction.doRun(TransportReplicationAction.java:370)
 // 异步执行主分片操作
 at org.elasticsearch.common.util.concurrent.AbstractRunnable.run(AbstractRunnable.java:37)
 // 抽象可运行类运行
 at org.elasticsearch.action.support.replication.TransportReplicationAction$PrimaryOperationTransportHandler.messageReceived(TransportReplicationAction.java:325)
 at org.elasticsearch.action.support.replication.TransportReplicationAction$PrimaryOperationTransportHandler.messageReceived(TransportReplicationAction.java:312)
 // 处理主分片操作的消息接收
 at com.amazon.opendistroforelasticsearch.security.ssl.transport.OpenDistroSecuritySSLRequestHandler.messageReceivedDecorate(OpenDistroSecuritySSLRequestHandler.java:194)
 // OpenDistro 安全插件装饰消息接收
 at com.amazon.opendistroforelasticsearch.security.transport.OpenDistroSecurityRequestHandler.messageReceivedDecorate(OpenDistroSecurityRequestHandler.java:163)
 // OpenDistro 安全插件装饰消息接收
 at com.amazon.opendistroforelasticsearch.security.ssl.transport.OpenDistroSecuritySSLRequestHandler.messageReceived(OpenDistroSecuritySSLRequestHandler.java:116)
 // OpenDistro 安全插件处理消息接收
 at com.amazon.opendistroforelasticsearch.security.OpenDistroSecurityPlugin$7$1.messageReceived(OpenDistroSecurityPlugin.java:619)
 // OpenDistro 安全插件处理消息接收
 at org.elasticsearch.transport.RequestHandlerRegistry.processMessageReceived(RequestHandlerRegistry.java:66)
 // 请求处理器注册表处理消息接收
 at org.elasticsearch.transport.TransportService$7.doRun(TransportService.java:704)
 // 传输服务处理消息接收
 at org.elasticsearch.common.util.concurrent.ThreadContext$ContextPreservingAbstractRunnable.doRun(ThreadContext.java:778)
 // 保存上下文的抽象可运行类处理消息接收
 at org.elasticsearch.common.util.concurrent.AbstractRunnable.run(AbstractRunnable.java:37)
 // 抽象可运行类运行
 at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@11.0.19.19-AJDK/ThreadPoolExecutor.java:1128)
 // 线程池执行工作线程
 at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@11.0.19.19-AJDK/ThreadPoolExecutor.java:628)
 // 工作线程运行
 at java.lang.Thread.run(java.base@11.0.19.19-AJDK/Thread.java:991)
 // 线程运行
```

jstack 也都打出来了。写之前搜一遍，数据量越来越多不就是越搜越久，写入越慢？明明单个 Bulk 执行都超过了几分钟，没人关注，一个劲无脑继续往里怼。等十秒不行重试一次，重试十次后直接丢弃，真是 666。我情绪稳定，我呵呵一笑，我看你狗叫。