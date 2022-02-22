---
title: Spark分区管理
date: 2021-03-22 17:47:38
categories: flink
tags: [spark] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: Spark分区管理的方式
---

当我们使用Spark加载数据源并进行一些列转换时，Spark会将数据拆分为多个分区Partition，并在分区上并行执行计算。所以理解Spark是如何对数据进行分区的以及何时需要手动调整Spark的分区，可以帮助我们提升Spark程序的运行效率。

<!--more-->

## 1. 什么是分区

关于什么是分区，其实没有什么神秘的。我们可以通过创建一个DataFrame来说明如何对数据进行分区：

```scala
scala> val x = (1 to 10).toList
x: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
scala> val numsDF = x.toDF("num")
numsDF: org.apache.spark.sql.DataFrame = [num: int]
```

创建好DataFrame之后，我们再来看一下该DataFame的分区，可以看出分区数为4：

```scala
scala> numsDF.rdd.partitions.size
res0: Int = 4
```

当我们将DataFrame写入磁盘文件时，再来观察一下文件的个数，

```scala
scala> numsDF.write.csv("file:///opt/modules/data/numsDF")
```

可以发现，上述的写入操作会生成4个文件

![image-20210322181006944](/images/image-20210322181006944.png)



## 2. coalesce操作

### 源码

```scala
def coalesce(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = false, planWithBarrier)
  }
```

### 解释

在减少分区时，返回一个新的分区数为指定`numPartitions`的DataSet，在增大分区时，则分区数保持不变。值得注意的是，该操作生成的是窄依赖，**所以不会发生shuffle**。然而，如果是极端的操作，比如numPartitions = 1，这样会导致只在一个节点进行计算。为了避免这种情况发生，可以使用repartition方法，**该方法会发生shuffle操作**，这就意味着当前的上游分区可以并行执行

### 示例

#### 减少分区操作

coalesce方法可以用来减少DataFrame的分区数。以下操作是将数据合并到两个分区：

```scala
scala> val numsDF2 = numsDF.coalesce(2)
numsDF2: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [num: int]
```

我们可以验证上述操作是否创建了只有两个分区的新DataFrame：可以看出，分区数变为了2

```scala
scala> numsDF2.rdd.partitions.size 
res13: Int = 2
```

将numsDF2写入文件存储，观察文件数量

```scala
numsDF2.write.csv("file:///opt/modules/data/numsDF2")
```

可以发现，上述的写入操作会生成2个文件

![image-20210322181535098](/images/image-20210322181535098.png)

上述每个分区的数据如下：

```scala
part-00000: 1, 2, 3, 4, 5
part-00001: 6, 7, 8, 9, 10
```

对比减少分区之前的数据存储，可以看出：在减少分区时，并没有对所有数据进行了移动，仅仅是在原来分区的基础之上进行了合并而已，这样的操作可以减少数据的移动，所以效率较高。

#### 增加分区操作

从上面的源码可以看出，如果使用coalesce方法进行增加分区，将不会生效。我们可以尝试通过coalesce来增加分区的数量，观察一下具体结果：

```scala
scala> val numsDF3 = numsDF.coalesce(6)
numsDF3: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [num: int]

scala> numsDF3.rdd.partitions.size
res16: Int = 4
```

可以看出，即使我们尝试使用coalesce(6)来创建6个分区，numsDF3的分区数依然是4，并没有发生变化，coalesce算法通过将数据从某些分区移动到现有分区来更改分区数，该方法显然不能增加分区数。

## 3. repartition操作

### 源码

```scala
  /**
   * 返回一个分区数为`numPartitions`的新的DataSet
   * @group typedrel
   * @since 1.6.0
   */
  def repartition(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = true, planWithBarrier)
  }
```

从源码中可以看出，该方法可以用于减少或者增加分区的数量，并且会发生Shuffle操作。

### 示例

#### 减少分区操作

已知numsDF有4个分区，现在将其分区置为2，观察结果

```scala
scala> val numsDF4 = numsDF.repartition(2)
numsDF4: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [num: int]
scala> numsDF4.rdd.partitions.size
res19: Int = 2
```

可以看出，分区确实减少了，我们在来看一下每个分区的数据：

```scala
numsDF4.write.csv("file:///opt/modules/data/numsDF4")
```

上面的操作会产生两个文件，每个分区文件的数据为：

```scala
part-00000:2,3,4,7,10,9
part-00000:1,5,6,8
```

从上面的数据分布可以看出，数据被Shuffle了。这也印证了源码中说的，repartition操作会将所有数据进行Shuffle，并且将数据均匀地分布在不同的分区上，并不是像coalesce方法一样，会尽量减少数据的移动。

#### 增加分区操作

repartition操作方法不仅可以用于减少分区操作，也可以用于增加分区数量。

```scala
scala> val numsDF5 = numsDF.repartition(6)
numsDF5: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [num: int]
scala> numsDF5.rdd.partitions.size
res22: Int = 6
```

### coalesce 与repartition之间的区别

repartition算法对数据进行了Shuffle操作，并创建了大小相等的数据分区。coalesce操作合并现有分区以避免Shuffle。除此之外，coalesce操作仅能用于减少分区，不能用于增加分区操作。

### 按照列字段进行repartition

- 源码

```scala
  def repartition(partitionExprs: Column*): Dataset[T] = {
    repartition(sparkSession.sessionState.conf.numShufflePartitions, partitionExprs: _*)
  }
```

- 解释

返回一个按照指定分区列的新的DataSet，**具体的分区数量有参数`spark.sql.shuffle.partitions`默认指定，该默认值为200**，该操作与HiveSQL的DISTRIBUTE BY操作类似。

repartition除了可以指定具体的分区数之外，还可以指定具体的分区字段。我们可以使用下面的示例来探究如何使用特定的列对DataFrame进行重新分区。

首先创建DataFrame：

```scala
val people = List(
("jack","male"),
("Alice","female"),
("tom","male"),
("Angela","female"),
("tony","male")
)
val peopleDF = people.toDF("name","gender")
```

让我们按gender列对DataFrame进行分区：

```scala
scala> val genderDF = peopleDF.repartition($"gender")
genderDF: org.apache.spark.sql.Dataset[org.apache.spark.sql.Row] = [name: string, gender: string]
```

按列进行分区时，Spark默认会创建200个分区。此示例将有两个带有数据的分区,其他分区将没有数据。

```scala
scala> genderDF.rdd.partitions.size
res23: Int = 200
```

## 4. 重点

### 合理设置分区数量

假设我们要对一个大数据集进行操作，该数据集的分区数也比较大，那么当我们进行一些操作之后，比如filter过滤操作、sample采样操作，这些操作可能会使结果数据集的数据量大量减少。但是Spark却不会对其分区进行调整，由此会造成大量的分区没有数据，并且向HDFS读取和写入大量的空文件，效率会很低，这种情况就需要我们重新调整分数数量，以此来提升效率。

通常情况下，结果集的数据量减少时，其对应的分区数也应当相应地减少。那么该如何确定具体的分区数呢？

- **分区过少**：将无法充分利用群集中的所有可用的CPU core
- **分区过多**：产生非常多的小任务，从而会产生过多的开销

在这两者之间，第一个对性能的影响相对比较大。对于小于1000个分区数的情况而言，调度太多的小任务所产生的影响相对较小。但是，如果有成千上万个分区，那么Spark会变得非常慢。

spark中的shuffle分区数是静态的。它不会随着不同的数据大小而变化。上文提到：默认情况下，控制shuffle分区数的参数**spark.sql.shuffle.partitions**值为**200**，这将导致以下问题

- 对于较小的数据，200是一个过大的选择，由于调度开销，通常会导致处理速度变慢。
- 对于大数据，200很小，无法有效使用群集中的所有资源

一般情况下，我们可以通过将集群中的CPU数量乘以2、3或4来确定分区的数量。如果要将数据写出到文件系统中，则可以选择一个分区大小，以创建合理大小的文件。

**该使用哪种方法进行重分区呢？**

对于大型数据集，进行Shuffle操作是很消耗性能的，但是当我们的数据集比较小的时候，可以使用repartition方法进行重分区，这样可以尽量保证每个分区的数据分布比较均匀(使用coalesce可能会造成数据倾斜)，对于下游使用者来说效率更高。

### 如何将数据写入到单个文件

通过使用repartition(1)和coalesce(1))可用于将DataFrame写入到单个文件中。通常情况下，不会只将数据写入到单个文件中，因为这样效率很低，写入速度很慢，在数据量比较大的情况，很可能会出现写入错误的情况。所以，只有当DataFrame很小时，我们才会考虑将其写入到单个文件中。

### 何时考虑重分区

一般对于在对比较大的数据集进行过滤操作之后，产生的较小数据集，通常需要对其考虑进行重分区，从而提升任务执行的效率。

## 5. 总结

本文主要介绍了Spark是如何管理分区的，分别解释了Spark提供的两种分区方法，并给出了相应的使用示例和分析。最后对分区情况及其影响进行了讨论，并给出了一些实践的建议。希望本文对你有所帮助。