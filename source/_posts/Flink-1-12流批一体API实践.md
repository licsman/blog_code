---
title: Flink-1.12 API上手实践（源码层面）
date: 2021-01-71 17:20:18
categories: flink
tags: [flink] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: Flink-1-12流批一体API实践
---

![img](/images/navbar-brand-logo.jpg)

> 官方地址：https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/datastream_execution_mode.html

<!--more-->

流批一体官方原话说明

> Apache Flink’s unified approach to stream and batch processing means that a DataStream application executed over bounded input will produce the same *final* results regardless of the configured execution mode. It is important to note what *final* means here: a job executing in `STREAMING` mode might produce incremental updates (think upserts in a database) while a `BATCH` job would only produce one final result at the end. The final result will be the same if interpreted correctly but the way to get there can be different.
>
> **By enabling `BATCH` execution, we allow Flink to apply additional optimizations that we can only do when we know that our input is bounded.** For example, different join/aggregation strategies can be used, in addition to a different shuffle implementation that allows more efficient task scheduling and failure recovery behavior. We will go into some of the details of the execution behavior below.

官方举例说明什么时候适合使用流批一体，batch execution mode的好处：

> - One obvious outlier is when you want to use a bounded job to bootstrap some job state that you then want to use in an unbounded job. For example, by running a bounded job using `STREAMING` mode, taking a savepoint, and then restoring that savepoint on an unbounded job. This is a very specific use case and one that might soon become obsolete when we allow producing a savepoint as additional output of a `BATCH` execution job.
> - Another case where you might run a bounded job using `STREAMING` mode is when writing tests for code that will eventually run with unbounded sources. For testing it can be more natural to use a bounded source in those cases.

## 1.流批执行模式的配置

- `STREAMING`: The classic DataStream execution mode (default)
- `BATCH`: Batch-style execution on the DataStream API
- `AUTOMATIC`: Let the system decide based on the boundedness of the sources

命令行模式提交：

```shell
$ bin/flink run -Dexecution.runtime-mode=BATCH examples/streaming/WordCount.jar
```

代码中配置：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setRuntimeMode(RuntimeExecutionMode.BATCH);

//官方推荐不要在程序中设置，最好在提交job的时候指定参数，这样更加灵活。因为流批一体以后不管作业使用何种执行方式，都可以产生结果
```

