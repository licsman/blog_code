---
title: Flink-1.12版本解读（源码层面）
date: 2021-01-14 17:10:02
categories: flink
tags: [flink,hive] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: Flink-1-12版本解析，减少了什么，又增加了什么
---

![img](/images/navbar-brand-logo.jpg)

## 1. API

**移除掉 ExecutionConfig 中过期的方法**

<!--more-->

- 移除掉了 `ExecutionConfig#isLatencyTrackingEnabled` 方法, 你可以使用 `ExecutionConfig#getLatencyTrackingInterval` 方法代替.
- 移除掉了 `ExecutionConfig#enable/disableSysoutLogging`、`ExecutionConfig#set/isFailTaskOnCheckpointError` 过期的方法。
- 移除掉了 `-q` CLI 参数。

**移除掉过期的 `RuntimeContext#getAllAccumulators` 方法**

过期的 `RuntimeContext#getAllAccumulators` 方法被移除掉了，请使用 `RuntimeContext#getAccumulator` 方法作为代替。

**由于数据丢失的风险把 `CheckpointConfig#setPreferCheckpointForRecovery` 方法标为过期**

`CheckpointConfig#setPreferCheckpointForRecovery` 方法标记为过期了, 因为作业在进行恢复时，如果使用较旧的 Checkpoint 状态而不使用新的 Save point 状态数据，可能会导致数据丢失。

**FLIP-134: DataStream API 的批处理执行**

- 允许在 `KeyedStream.intervalJoin()` 的配置时间属性，在 Flink 1.12 之前 `KeyedStream.intervalJoin()` 算子的时间属性依赖于全局设置的时间属性。在 Flink 1.12 中我们可以在 IntervalJoin 方法后加上 `inProcessingTime()` 或 `inEventTime()` ，这样 Join 就不再依赖于全局的时间属性。

- 在 Flink 1.12 中将 DataStream API 的 `timeWindow()` 方法标记为过期，请使用 `window(WindowAssigner)`、`TumblingEventTimeWindows`、 `SlidingEventTimeWindows`、`TumblingProcessingTimeWindows` 或者 `SlidingProcessingTimeWindows`。
- 将 `StreamExecutionEnvironment.setStreamTimeCharacteristic()` 和 `TimeCharacteristic` 方法标记为过期。在 Flink 1.12 中，默认的时间属性改变成 EventTime 了，于是你不再需要该方法去开启 EventTime 了。在 EventTime 时间属性下，你使用 processing-time 的 windows 和 timers 也都依旧会生效。如果你想禁用水印，请使用 `ExecutionConfig.setAutoWatermarkInterval(long)` 方法。如果你想使用 `IngestionTime`，请手动设置适当的 WatermarkStrategy。如果你使用的是基于时间属性更改行为的通用 ‘time window’ 算子(eg: `KeyedStream.timeWindow()`)，请使用等效操作明确的指定处理时间和事件时间。
- 允许在 CEP PatternStream 上显式配置时间属性在 Flink 1.12 之前，CEP 算子里面的时间依赖于全局配置的时间属性，在 1.12 之后可以在 PatternStream 上使用 `inProcessingTime()` 或 `inEventTime()` 方法。

**API 清理**

- 移除了 UdfAnalyzer 配置，移除了 `ExecutionConfig#get/setCodeAnalysisMode` 方法和 `SkipCodeAnalysis` 类。
- 移除了过期的 `DataStream#split` 方法，该方法从很早的版本中已经标记成为过期的了，你可以使用 Side Output 来代替。
- 移除了过期的 `DataStream#fold()` 方法和其相关的类，你可以使用更加高性能的 `DataStream#reduce`。

**扩展 CompositeTypeSerializerSnapshot 以允许复合序列化器根据外部配置迁移**

不再推荐使用 CompositeTypeSerializerSnapshot 中的 `isOuterSnapshotCompatible(TypeSerializer)` 方法，推荐使用 `OuterSchemaCompatibility#resolveOuterSchemaCompatibility(TypeSerializer)` 方法。

**将 Scala 版本升级到 2.1.1**

Flink 现在依赖 Scala 2.1.1，意味着不再支持 Scala 版本小于 2.11.11。

## 2. SQL

**对 aggregate 函数的 SQL DDL 使用新类型推断**

aggregate 函数的 `CREATE FUNCTION` DDL 现在使用新类型推断，可能有必要将现有实现更新为新的反射类型提取逻辑，将 `StreamTableEnvironment.registerFunction` 标为过期。

**更新解析器模块 FLIP-107**

现在 `METADATA` 属于保留关键字，记得使用反引号转义。

**将内部 aggregate 函数更新为新类型**

使用 COLLECT 函数的 SQL 查询可能需要更新为新类型的系统。

## 3. Connectors 和 Formats

**移除 Kafka 0.10.x 和 0.11.x Connector**

在 Flink 1.12 中，移除掉了 Kafka 0.10.x 和 0.11.x Connector，请使用统一的 Kafka Connector（适用于 0.10.2.x 版本之后的任何 Kafka 集群），你可以参考 Kafka Connector 页面的文档升级到新的 Flink Kafka Connector 版本。

**CSV 序列化 Schema 包含行分隔符**

`csv.line-delimiter` 配置已经从 CSV 格式中移除了，因为行分隔符应该由 Connector 定义而不是由 format 定义。如果用户在以前的 Flink 版本中一直使用了该配置，则升级到 Flink 1.12 时，应该删除该配置。

**升级 Kafka Schema Registry Client 到 5.5.0 版本**

`flink-avro-confluent-schema-registry` 模块不再在 fat-jar 中提供，你需要显式的在你自己的作业中添加该依赖，SQL-Client 用户可以使用`flink-sql-avro-confluent-schema-registry` fat jar。

**将 Avro 版本从 1.8.2 升级到 1.10.0 版本**

`flink-avro` 模块中的 Avro 版本升级到了 1.10，如果出于某种原因要使用较旧的版本，请在项目中明确降级 Avro 版本。

**注意**：我们观察到，与 1.8.2 相比，Avro 1.10 版本的性能有所下降，如果你担心性能，并且可以使用较旧版本的 Avro，那么请降级 Avro 版本。

**为 SQL Client 打包 `flink-avro` 模块时会创建一个 uber jar**

SQL Client jar 会被重命名为 `flink-sql-avro-1.12.jar`，以前是 `flink-avro-1.12-sql-jar.jar`，而且不再需要手动添加 Avro 依赖。

## 4. Deployment（部署）

**默认 Log4j 配置了日志大小超过 100MB 滚动**

默认的 log4j 配置现在做了变更：除了在 Flink 启动时现有的日志文件滚动外，它们在达到 100MB 大小时也会滚动。Flink 总共保留 10 个日志文件，从而有效地将日志目录的总大小限制为 1GB（每个 Flink 服务记录到该目录）。

**默认在 Flink Docker 镜像中使用 jemalloc**

在 Flink 的 Docker 镜像中，jemalloc 被用作默认的内存分配器，以减少内存碎片问题。用户可以通过将 `disable-jemalloc` 标志传递给 `docker-entrypoint.sh` 脚本来回滚使用 glibc。有关更多详细信息，请参阅 Docker 文档上的 Flink。

**升级 Mesos 版本到 1.7**

将 Mesos 依赖版本从 1.0.1 版本升级到 1.7.0 版本。

**如果 Flink 进程在超时后仍未停止，则发送 SIGKILL**

在 Flink 1.12 中，如果 SIGTERM 无法成功关闭 Flink 进程，我们更改了独立脚本的行为以发出 SIGKILL。

**介绍非阻塞作业提交**

提交工作的语义略有变化，提交调用几乎立即返回，并且作业处于新的 INITIALIZING 状态，当作业处于该状态时，对作业做 Savepoint 或者检索作业详情信息等操作将不可用。

一旦创建了该作业的 JobManager，该作业就处于 CREATED 状态，并且所有的调用均可用。

## 5. Runtime

**FLIP-141: Intra-Slot Managed Memory 共享**

`python.fn-execution.buffer.memory.size` 和 `python.fn-execution.framework.memory.size` 的配置已删除，因此不再生效。除此之外，`python.fn-execution.memory.managed` 默认的值更改为 `true`， 因此默认情况下 Python workers 将使用托管内存。

**FLIP-119 Pipelined Region Scheduling**

从 Flink 1.12 开始，将以 pipelined region 为单位进行调度。pipelined region 是一组流水线连接的任务。这意味着，对于包含多个 region 的流作业，在开始部署任务之前，它不再等待所有任务获取 slot。取而代之的是，一旦任何 region 获得了足够的任务 slot 就可以部署它。对于批处理作业，将不会为任务分配 slot，也不会单独部署任务。取而代之的是，一旦某个 region 获得了足够的 slot，则该任务将与所有其他任务一起部署在同一区域中。

可以使用 `jobmanager.scheduler.scheduling-strategy：legacy` 启用旧的调度程序。

**RocksDB optimizeForPointLookup 导致丢失时间窗口**

默认情况下，我们会将 RocksDB 的 ReadOptions 的 setTotalOrderSeek 设置为true，以防止用户忘记使用 optimizeForPointLookup。 同时，我们支持通过RocksDBOptionsFactory 自定义 ReadOptions。如果观察到任何性能下降，请将 setTotalOrderSeek 设置为 false（根据我们的测试，这是不可能的）。

**自定义 OptionsFactory 设置似乎对 RocksDB 没有影响**

过期的 OptionsFactory 和 ConfigurableOptionsFactory 类已移除，请改用 RocksDBOptionsFactory 和 ConfigurableRocksDBOptionsFactory。 如果有任何扩展 DefaultConfigurableOptionsFactory 的类，也请重新编译你的应用程序代码。

