# 概述

在一个较高的概念上来说，每一个 Spark 应用程序由一个在集群上运行着用户的  `main`  函数和执行各种并行操作的  _driver program_（驱动程序）组成。Spark 提供的主要抽象是一个_弹性分布式数据集_（RDD），它是可以执行并行操作且跨集群节点的元素的集合。RDD 可以从一个 Hadoop 文件系统（或者任何其它 Hadoop 支持的文件系统），或者一个在 driver program（驱动程序）中已存在的 Scala 集合，以及通过 transforming（转换）来创建一个 RDD。用户为了让它在整个并行操作中更高效的重用，也许会让 Spark persist（持久化）一个 RDD 到内存中。最后，RDD 会自动的从节点故障中恢复。

在 Spark 中的第二个抽象是能够用于并行操作的  _shared variables_（共享变量），默认情况下，当 Spark 的一个函数作为一组不同节点上的任务运行时，它将每一个变量的副本应用到每一个任务的函数中去。有时候，一个变量需要在整个任务中，或者在任务和 driver program（驱动程序）之间来共享。Spark 支持两种类型的共享变量 :  _broadcast variables_（广播变量），它可以用于在所有节点上缓存一个值，和  _accumulators_（累加器），他是一个只能被 “added（增加）” 的变量，例如 counters 和 sums。

# Spark依赖

Spark 2.2.0 默认使用 Scala 2.11 来构建和发布直到运行。（当然，Spark 也可以与其它的 Scala 版本一起运行）。为了使用 Scala 编写应用程序，您需要使用可兼容的 Scala 版本（例如，2.11.X）。

要编写一个 Spark 的应用程序，您需要在 Spark 上添加一个 Maven 依赖。Spark 可以通过 Maven 中央仓库获取:

```
groupId = org.apache.spark
artifactId = spark-core_2.11
version = 2.2.0
```

此外，如果您想访问一个 HDFS 集群，则需要针对您的 HDFS 版本添加一个  `hadoop-client`（hadoop 客户端）依赖。

```
groupId = org.apache.hadoop
artifactId = hadoop-client
version = <your-hdfs-version>
```

最后，您需要导入一些 Spark classes（类）到您的程序中去。添加下面几行:

```scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
```

(在 Spark 1.3.0 之前，您需要明确导入  `org.apache.spark.SparkContext._`  来启用必要的的隐式转换。)

# 初始化Spark

Spark 程序必须做的第一件事情是创建一个`SparkContext`对象，它会告诉 Spark 如何访问集群。要创建一个  `SparkContext`，首先需要构建一个包含应用程序的信息的`SparkConf` 对象。

每一个 JVM 可能只能激活一个 SparkContext 对象。在创新一个新的对象之前，必须调用  `stop()`该方法停止活跃的 SparkContext。

```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

这个  `appName`  参数是一个在集群 UI 上展示应用程序的名称。  `master`  是一个  [Spark, Mesos 或 YARN 的 cluster URL](http://spark.apachecn.org/docs/cn/2.2.0/submitting-applications.html#master-urls)，或者指定为在 local mode（本地模式）中运行的 “local” 字符串。在实际工作中，当在集群上运行时，您不希望在程序中将 master 给硬编码，而是用  [使用  `spark-submit`  启动应用](http://spark.apachecn.org/docs/cn/2.2.0/submitting-applications.html)  并且接收它。然而，对于本地测试和单元测试，您可以通过 “local” 来运行 Spark 进程。

# 弹性分布式数据集（RDDs）

Spark 主要以一个 _弹性分布式数据集_（RDD）的概念为中心，它是一个容错且可以执行并行操作的元素的集合。有两种方法可以创建 RDD : 在你的 driver program（驱动程序）中 _parallelizing_ 一个已存在的集合，或者在外部存储系统中引用一个数据集，例如，一个共享文件系统，HDFS，HBase，或者提供 Hadoop InputFormat 的任何数据源。

## 并行集合

可以在您的 driver program (a Scala  `Seq`) 中已存在的集合上通过调用  `SparkContext`  的  `parallelize`  方法来创建并行集合。该集合的元素从一个可以并行操作的 distributed dataset（分布式数据集）中复制到另一个 dataset（数据集）中去。例如，这里是一个如何去创建一个保存数字 1 ~ 5 的并行集合。

```scala
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```

在创建后，该 distributed dataset（分布式数据集）（`distData`）可以并行的执行操作。例如，我们可以调用  `distData.reduce((a, b) => a + b`) 来合计数组中的元素。后面我们将介绍 distributed dataset（分布式数据集）上的操作。

并行集合中一个很重要参数是  _partitions_（分区）的数量，它可用来切割 dataset（数据集）。Spark 将在集群中的每一个分区上运行一个任务。通常您希望群集中的每一个 CPU 计算 2-4 个分区。一般情况下，Spark 会尝试根据您的群集情况来自动的设置的分区的数量。当然，您也可以将分区数作为第二个参数传递到  `parallelize`  (e.g.  `sc.parallelize(data, 10)`) 方法中来手动的设置它。注意: 代码中的一些地方会使用 term slices (a synonym for partitions) 以保持向后兼容.

## 外部Datasets（数据集）

## RDD操作

### 基础

### 传递Functions（函数）给Spark

### 与Key-Value Pairs一起使用

### Transformations（转换）

### Action（动作）

### Shuffle操作

#### Background（幕后）

#### 性能影响

## RDD Persistence（持久化）

### 存储级别
### 删除数据

# 共享变量

## 广播变量
##
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAwOTYxOTQ4Miw4MzA3ODUyODEsLTgzOT
Y0OTA2MF19
-->