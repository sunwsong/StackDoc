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

Spark 可以从 Hadoop 所支持的任何存储源中创建 distributed dataset（分布式数据集），包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3  等等。 Spark 支持文本文件，SequenceFiles，以及任何其它的 Hadoop  `InputFormat`。

可以使用  `SparkContext`  的  `textFile`  方法来创建文本文件的 RDD。此方法需要一个文件的 URI（计算机上的本地路径 ，`hdfs://`，`s3n://`  等等的 URI），并且读取它们作为一个 lines（行）的集合。下面是一个调用示例:

```scala
scala> val distFile = sc.textFile("data.txt")
distFile: org.apache.spark.rdd.RDD[String] = data.txt MapPartitionsRDD[10] at textFile at <console>:26
```

在创建后，`distFile`  可以使用 dataset（数据集）的操作。例如，我们可以使用下面的 map 和 reduce 操作来合计所有行的数量:  `distFile.map(s => s.length).reduce((a, b) => a + b)`。

使用 Spark 读取文件时需要注意:

-   如果使用本地文件系统的路径，所工作节点的相同访问路径下该文件必须可以访问。复制文件到所有工作节点上，或着使用共享的网络挂载文件系统。
    
-   所有 Spark 基于文件的 input 方法, 包括  `textFile`, 支持在目录上运行, 压缩文件, 和通配符. 例如, 您可以使用  `textFile("/my/directory")`,  `textFile("/my/directory/*.txt")`, and  `textFile("/my/directory/*.gz")`.
    
-   `textFile`  方法也可以通过第二个可选的参数来控制该文件的分区数量. 默认情况下, Spark 为文件的每一个 block（块）创建的一 个 partition 分区（HDFS 中块大小默认是 128MB），当然你也可以通过传递一个较大的值来要求一个较高的分区数量。请注意，分区的数量不能够小于块的数量。
    

除了文本文件之外，Spark 的 Scala API 也支持一些其它的数据格式:

-   `SparkContext.wholeTextFiles`  可以读取包含多个小文本文件的目录, 并且将它们作为一个 (filename, content) pairs 来返回. 这与  `textFile`  相比, 它的每一个文件中的每一行将返回一个记录. 分区由数据量来确定, 某些情况下, 可能导致分区太少. 针对这些情况,  `wholeTextFiles`  在第二个位置提供了一个可选的参数用户控制分区的最小数量.
    
-   针对  SequenceFiles, 使用 SparkContext 的  `sequenceFile[K, V]`  方法，其中  `K`  和  `V`  指的是文件中 key 和 values 的类型. 这些应该是 Hadoop 的  `Writable`  接口的子类, 像 `IntWritable` and `Text`. 此外, Spark 可以让您为一些常见的 Writables 指定原生类型; 例如,  `sequenceFile[Int, String]`  会自动读取 IntWritables 和 Texts.
    
-   针对其它的 Hadoop InputFormats, 您可以使用  `SparkContext.hadoopRDD`  方法, 它接受一个任意的  `JobConf`  和 input format class, key class 和 value class. 通过相同的方法你可以设置你的 input source（输入源）. 你还可以针对 InputFormats 使用基于 “new” MapReduce API (`org.apache.hadoop.mapreduce`) 的  `SparkContext.newAPIHadoopRDD`.
    
-   `RDD.saveAsObjectFile`  和  `SparkContext.objectFile`  支持使用简单的序列化的 Java objects 来保存 RDD. 虽然这不像 Avro 这种专用的格式一样高效，但其提供了一种更简单的方式来保存任何的 RDD。

## RDD操作

RDDs support 两种类型的操作:  _transformations（转换）_, 它会在一个已存在的 dataset 上创建一个新的 dataset, 和  _actions（动作）_, 将在 dataset 上运行的计算后返回到 driver 程序. 例如,  `map`  是一个通过让每个数据集元素都执行一个函数，并返回的新 RDD 结果的 transformation,  `reduce`  reduce 通过执行一些函数，聚合 RDD 中所有元素，并将最终结果给返回驱动程序（虽然也有一个并行  `reduceByKey`  返回一个分布式数据集）的 action.

Spark 中所有的 transformations 都是  _lazy（懒加载的）_, 因此它不会立刻计算出结果. 相反, 他们只记得应用于一些基本数据集的转换 (例如. 文件). 只有当需要返回结果给驱动程序时，transformations 才开始计算. 这种设计使 Spark 的运行更高效. 例如, 我们可以了解到，`map`  所创建的数据集将被用在  `reduce`  中，并且只有  `reduce`  的计算结果返回给驱动程序，而不是映射一个更大的数据集.

默认情况下，每次你在 RDD 运行一个 action 的时， 每个 transformed RDD 都会被重新计算。但是，您也可用  `persist`  (或  `cache`) 方法将 RDD persist（持久化）到内存中；在这种情况下，Spark 为了下次查询时可以更快地访问，会把数据保存在集群上。此外，还支持持续持久化 RDDs 到磁盘，或复制到多个结点。

### 基础

为了说明 RDD 基础，请思考下面这个的简单程序:

```scala
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
```

第一行从外部文件中定义了一个基本的 RDD，但这个数据集并未加载到内存中或即将被行动:  `line`仅仅是一个类似指针的东西，指向该文件. 第二行定义了  `lineLengths`  作为  `map`  transformation 的结果。请注意，由于  `laziness`（延迟加载）`lineLengths`  不会被立即计算. 最后，我们运行  `reduce`，这是一个 action。此时，Spark 分发计算任务到不同的机器上运行，每台机器都运行在 map 的一部分并本地运行 reduce，仅仅返回它聚合后的结果给驱动程序.

如果我们也希望以后再次使用  `lineLengths`，我们还可以添加:

```scala
lineLengths.persist()
```

在  `reduce`  之前，这将导致  `lineLengths`  在第一次计算之后就被保存在 memory 中。

### 传递Functions（函数）给Spark

当 driver 程序在集群上运行时，Spark 的 API 在很大程度上依赖于传递函数。有 2 种推荐的方式来做到这一点:

-   [Anonymous function syntax（匿名函数语法）](http://docs.scala-lang.org/tutorials/tour/anonymous-function-syntax.html), 它可以用于短的代码片断.
-   在全局单例对象中的静态方法. 例如, 您可以定义  `object MyFunctions`  然后传递  `MyFunctions.func1`, 如下:

```scala
object MyFunctions {
  def func1(s: String): String = { ... }
}

myRdd.map(MyFunctions.func1)
```

请注意，虽然也有可能传递一个类的实例（与单例对象相反）的方法的引用，这需要发送整个对象，包括类中其它方法。例如，考虑:

```scala
class MyClass {
  def func1(s: String): String = { ... }
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
}
```

这里，如果我们创建一个  `MyClass`  的实例，并调用  `doStuff`，在  `map`  内有  `MyClass`  实例的  `func1`  方法的引用，所以整个对象需要被发送到集群的。它类似于  `rdd.map(x => this.func1(x))`

类似的方式，访问外部对象的字段将引用整个对象:

```scala
class MyClass {
  val field = "Hello"
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
}
```

相当于写  `rdd.map(x => this.field + x)`, 它引用  `this`  所有的东西. 为了避免这个问题, 最简单的方式是复制  `field`  到一个本地变量，而不是外部访问它:

```scala
def doStuff(rdd: RDD[String]): RDD[String] = {
  val field_ = this.field
  rdd.map(x => field_ + x)
}
```

### 与Key-Value Pairs一起使用

虽然大多数 Spark 操作工作在包含任何类型对象的 RDDs 上，只有少数特殊的操作可用于 Key-Value 对的 RDDs. 最常见的是分布式 “shuffle” 操作，如通过元素的 key 来进行 grouping 或 aggregating 操作.

在 Scala 中，这些操作在 RDD 上是自动可用，它包含了  [Tuple2](http://www.scala-lang.org/api/2.11.8/index.html#scala.Tuple2)  objects (the built-in tuples in the language, created by simply writing  `(a, b)`). 在  [PairRDDFunctions](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)  class 中该 key-value pair 操作是可用的, 其中围绕 tuple 的 RDD 进行自动封装.

例如，下面的代码使用的  `Key-Value`  对的  `reduceByKey`  操作统计文本文件中每一行出现了多少次:

```scala
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```

我们也可以使用  `counts.sortByKey()`  ，例如，在对按字母顺序排序，最后  `counts.collect()`  把他们作为一个数据对象返回给 driver 程序。

**Note（注意）:**  当在 key-value pair 操作中使用自定义的 objects 作为 key 时, 您必须确保有一个自定义的  `equals()`  方法有一个  `hashCode()`  方法相匹配. 有关详情, 请参阅  [Object.hashCode() documentation](http://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#hashCode())  中列出的约定.

### Transformations（转换）

下表列出了一些 Spark 常用的 transformations（转换）. 详情请参考 RDD API 文档 ([Scala](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.rdd.RDD),  [Java](http://spark.apachecn.org/docs/cn/2.2.0/api/java/index.html?org/apache/spark/api/java/JavaRDD.html),[Python](http://spark.apachecn.org/docs/cn/2.2.0/api/python/pyspark.html#pyspark.RDD),  [R](http://spark.apachecn.org/docs/cn/2.2.0/api/R/index.html)) 和 pair RDD 函数文档 ([Scala](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions),  [Java](http://spark.apachecn.org/docs/cn/2.2.0/api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)).

Transformation（转换）|Meaning（含义）|
--|--|
map(_func_)|返回一个新的 distributed dataset（分布式数据集），它由每个 source（数据源）中的元素应用一个函数  _func_  来生成.|

**filter**(_func_)

返回一个新的 distributed dataset（分布式数据集），它由每个 source（数据源）中应用一个函数  _func_  且返回值为 true 的元素来生成.

**flatMap**(_func_)

与 map 类似，但是每一个输入的 item 可以被映射成 0 个或多个输出的 items（所以  _func_  应该返回一个 Seq 而不是一个单独的 item）.

**mapPartitions**(_func_)

与 map 类似，但是单独的运行在在每个 RDD 的 partition（分区，block）上，所以在一个类型为 T 的 RDD 上运行时  _func_  必须是 Iterator<T> => Iterator<U> 类型.

**mapPartitionsWithIndex**(_func_)

与 mapPartitions 类似，但是也需要提供一个代表 partition 的 index（索引）的 interger value（整型值）作为参数的  _func_，所以在一个类型为 T 的 RDD 上运行时  _func_  必须是 (Int, Iterator<T>) => Iterator<U> 类型.

**sample**(_withReplacement_,  _fraction_,  _seed_)

样本数据，设置是否放回（withReplacement）, 采样的百分比（_fraction_）、使用指定的随机数生成器的种子（seed）.

**union**(_otherDataset_)

反回一个新的 dataset，它包含了 source dataset（源数据集）和 otherDataset（其它数据集）的并集.

**intersection**(_otherDataset_)

返回一个新的 RDD，它包含了 source dataset（源数据集）和 otherDataset（其它数据集）的交集.

**distinct**([_numTasks_]))

返回一个新的 dataset，它包含了 source dataset（源数据集）中去重的元素.

**groupByKey**([_numTasks_])

在一个 (K, V) pair 的 dataset 上调用时，返回一个 (K, Iterable<V>) .  
**Note:**  如果分组是为了在每一个 key 上执行聚合操作（例如，sum 或 average)，此时使用  `reduceByKey`  或  `aggregateByKey`  来计算性能会更好.  
**Note:**  默认情况下，并行度取决于父 RDD 的分区数。可以传递一个可选的  `numTasks`参数来设置不同的任务数.

**reduceByKey**(_func_, [_numTasks_])

在 (K, V) pairs 的 dataset 上调用时, 返回 dataset of (K, V) pairs 的 dataset, 其中的 values 是针对每个 key 使用给定的函数  _func_来进行聚合的, 它必须是 type (V,V) => V 的类型. 像  `groupByKey`  一样, reduce tasks 的数量是可以通过第二个可选的参数来配置的.

**aggregateByKey**(_zeroValue_)(_seqOp_,  _combOp_, [_numTasks_])

在 (K, V) pairs 的 dataset 上调用时, 返回 (K, U) pairs 的 dataset，其中的 values 是针对每个 key 使用给定的 combine 函数以及一个 neutral "0" 值来进行聚合的. 允许聚合值的类型与输入值的类型不一样, 同时避免不必要的配置. 像  `groupByKey`  一样, reduce tasks 的数量是可以通过第二个可选的参数来配置的.

**sortByKey**([_ascending_], [_numTasks_])

在一个 (K, V) pair 的 dataset 上调用时，其中的 K 实现了 Ordered，返回一个按 keys 升序或降序的 (K, V) pairs 的 dataset, 由 boolean 类型的  `ascending`  参数来指定.

**join**(_otherDataset_, [_numTasks_])

在一个 (K, V) 和 (K, W) 类型的 dataset 上调用时，返回一个 (K, (V, W)) pairs 的 dataset，它拥有每个 key 中所有的元素对。Outer joins 可以通过  `leftOuterJoin`,  `rightOuterJoin`  和  `fullOuterJoin`  来实现.

**cogroup**(_otherDataset_, [_numTasks_])

在一个 (K, V) 和的 dataset 上调用时，返回一个 (K, (Iterable<V>, Iterable<W>)) tuples 的 dataset. 这个操作也调用了  `groupWith`.

**cartesian**(_otherDataset_)

在一个 T 和 U 类型的 dataset 上调用时，返回一个 (T, U) pairs 类型的 dataset（所有元素的 pairs，即笛卡尔积）.

**pipe**(_command_,  _[envVars]_)

通过使用 shell 命令来将每个 RDD 的分区给 Pipe。例如，一个 Perl 或 bash 脚本。RDD 的元素会被写入进程的标准输入（stdin），并且 lines（行）输出到它的标准输出（stdout）被作为一个字符串型 RDD 的 string 返回.

**coalesce**(_numPartitions_)

Decrease（降低）RDD 中 partitions（分区）的数量为 numPartitions。对于执行过滤后一个大的 dataset 操作是更有效的.

**repartition**(_numPartitions_)

Reshuffle（重新洗牌）RDD 中的数据以创建或者更多的 partitions（分区）并将每个分区中的数据尽量保持均匀. 该操作总是通过网络来 shuffles 所有的数据.

**repartitionAndSortWithinPartitions**(_partitioner_)

根据给定的 partitioner（分区器）对 RDD 进行重新分区，并在每个结果分区中，按照 key 值对记录排序。这比每一个分区中先调用  `repartition`  然后再 sorting（排序）效率更高，因为它可以将排序过程推送到 shuffle 操作的机器上进行.

### Action（动作）

下表列出了一些 Spark 常用的 actions 操作。详细请参考 RDD API 文档 ([Scala](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.rdd.RDD),[Java](http://spark.apachecn.org/docs/cn/2.2.0/api/java/index.html?org/apache/spark/api/java/JavaRDD.html),  [Python](http://spark.apachecn.org/docs/cn/2.2.0/api/python/pyspark.html#pyspark.RDD),  [R](http://spark.apachecn.org/docs/cn/2.2.0/api/R/index.html))

和 pair RDD 函数文档 ([Scala](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions),  [Java](http://spark.apachecn.org/docs/cn/2.2.0/api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html)).

Action（动作）

Meaning（含义）

**reduce**(_func_)

使用函数  _func_  聚合 dataset 中的元素，这个函数  _func_  输入为两个元素，返回为一个元素。这个函数应该是可交换（commutative ）和关联（associative）的，这样才能保证它可以被并行地正确计算.

**collect**()

在 driver 程序中，以一个 array 数组的形式返回 dataset 的所有元素。这在过滤器（filter）或其他操作（other operation）之后返回足够小（sufficiently small）的数据子集通常是有用的.

**count**()

返回 dataset 中元素的个数.

**first**()

返回 dataset 中的第一个元素（类似于 take(1).

**take**(_n_)

将数据集中的前  _n_  个元素作为一个 array 数组返回.

**takeSample**(_withReplacement_,  _num_, [_seed_])

对一个 dataset 进行随机抽样，返回一个包含  _num_  个随机抽样（random sample）元素的数组，参数 withReplacement 指定是否有放回抽样，参数 seed 指定生成随机数的种子.

**takeOrdered**(_n_,  _[ordering]_)

返回 RDD 按自然顺序（natural order）或自定义比较器（custom comparator）排序后的前  _n_  个元素.

**saveAsTextFile**(_path_)

将 dataset 中的元素以文本文件（或文本文件集合）的形式写入本地文件系统、HDFS 或其它 Hadoop 支持的文件系统中的给定目录中。Spark 将对每个元素调用 toString 方法，将数据元素转换为文本文件中的一行记录.

**saveAsSequenceFile**(_path_)  
(Java and Scala)

将 dataset 中的元素以 Hadoop SequenceFile 的形式写入到本地文件系统、HDFS 或其它 Hadoop 支持的文件系统指定的路径中。该操作可以在实现了 Hadoop 的 Writable 接口的键值对（key-value pairs）的 RDD 上使用。在 Scala 中，它还可以隐式转换为 Writable 的类型（Spark 包括了基本类型的转换，例如 Int, Double, String 等等).

**saveAsObjectFile**(_path_)  
(Java and Scala)

使用 Java 序列化（serialization）以简单的格式（simple format）编写数据集的元素，然后使用  `SparkContext.objectFile()`  进行加载.

**countByKey**()

仅适用于（K,V）类型的 RDD 。返回具有每个 key 的计数的 （K , Int）pairs 的 hashmap.

**foreach**(_func_)

对 dataset 中每个元素运行函数  _func_  。这通常用于副作用（side effects），例如更新一个  [Accumulator](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#accumulators)（累加器）或与外部存储系统（external storage systems）进行交互。**Note**：修改除  `foreach()`之外的累加器以外的变量（variables）可能会导致未定义的行为（undefined behavior）。详细介绍请阅读  [Understanding closures（理解闭包）](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#understanding-closures-a-nameclosureslinka)  部分.

该 Spark RDD API 还暴露了一些 actions（操作）的异步版本，例如针对  `foreach`的  `foreachAsync`，它们会立即返回一个`FutureAction`  到调用者，而不是在完成 action 时阻塞。 这可以用于管理或等待 action 的异步执行。.

### Shuffle操作

Spark 里的某些操作会触发 shuffle。shuffle 是spark 重新分配数据的一种机制，使得这些数据可以跨不同的区域进行分组。这通常涉及在 executors 和 机器之间拷贝数据，这使得 shuffle 成为一个复杂的、代价高的操作。

#### Background（幕后）

为了明白  [`reduceByKey`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#ReduceByLink)  操作的过程，我们以  `reduceByKey`  为例。reduceBykey 操作产生一个新的 RDD，其中 key 所有相同的的值组合成为一个 tuple - key 以及与 key 相关联的所有值在 reduce 函数上的执行结果。面临的挑战是，一个 key 的所有值不一定都在一个同一个 paritition 分区里，甚至是不一定在同一台机器里，但是它们必须共同被计算。

在 spark 里，特定的操作需要数据不跨分区分布。在计算期间，一个任务在一个分区上执行，为了所有数据都在单个  `reduceByKey`  的 reduce 任务上运行，我们需要执行一个 all-to-all 操作。它必须从所有分区读取所有的 key 和 key对应的所有的值，并且跨分区聚集去计算每个 key 的结果 - 这个过程就叫做  **shuffle**.。

尽管每个分区新 shuffle 的数据集将是确定的，分区本身的顺序也是这样，但是这些数据的顺序是不确定的。如果希望 shuffle 后的数据是有序的，可以使用:

-   `mapPartitions`  对每个 partition 分区进行排序，例如,  `.sorted`
-   `repartitionAndSortWithinPartitions`  在分区的同时对分区进行高效的排序.
-   `sortBy`  对 RDD 进行全局的排序

触发的 shuffle 操作包括  **repartition**  操作，如  [`repartition`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#RepartitionLink)  和  [`coalesce`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#CoalesceLink),  **‘ByKey**操作 (除了 counting 之外) 像  [`groupByKey`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#GroupByLink)  和  [`reduceByKey`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#ReduceByLink), 和  **join**  操作, 像  [`cogroup`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#CogroupLink)  和  [`join`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#JoinLink).

#### 性能影响

该  **Shuffle**  是一个代价比较高的操作，它涉及磁盘 I/O、数据序列化、网络 I/O。为了准备 shuffle 操作的数据，Spark 启动了一系列的任务，_map_  任务组织数据，_reduce_  完成数据的聚合。这些术语来自 MapReduce，跟 Spark 的  `map`  操作和  `reduce`  操作没有关系。

在内部，一个 map 任务的所有结果数据会保存在内存，直到内存不能全部存储为止。然后，这些数据将基于目标分区进行排序并写入一个单独的文件中。在 reduce 时，任务将读取相关的已排序的数据块。

某些 shuffle 操作会大量消耗堆内存空间，因为 shuffle 操作在数据转换前后，需要在使用内存中的数据结构对数据进行组织。需要特别说明的是，`reduceByKey`  和  `aggregateByKey`  在 map 时会创建这些数据结构，`'ByKey`  操作在 reduce 时创建这些数据结构。当内存满的时候，Spark 会把溢出的数据存到磁盘上，这将导致额外的磁盘 I/O 开销和垃圾回收开销的增加。

shuffle 操作还会在磁盘上生成大量的中间文件。在 Spark 1.3 中，这些文件将会保留至对应的 RDD 不在使用并被垃圾回收为止。这么做的好处是，如果在 Spark 重新计算 RDD 的血统关系（lineage）时，shuffle 操作产生的这些中间文件不需要重新创建。如果 Spark 应用长期保持对 RDD 的引用，或者垃圾回收不频繁，这将导致垃圾回收的周期比较长。这意味着，长期运行 Spark 任务可能会消耗大量的磁盘空间。临时数据存储路径可以通过 SparkContext 中设置参数  `spark.local.dir`  进行配置。

shuffle 操作的行为可以通过调节多个参数进行设置。详细的说明请看  [Spark 配置指南](http://spark.apachecn.org/docs/cn/2.2.0/configuration.html)  中的 “Shuffle 行为” 部分。

## RDD Persistence（持久化）

Spark 中一个很重要的能力是将数据  _persisting_  持久化（或称为  _caching_  缓存），在多个操作间都可以访问这些持久化的数据。当持久化一个 RDD 时，每个节点的其它分区都可以使用 RDD 在内存中进行计算，在该数据上的其他 action 操作将直接使用内存中的数据。这样会让以后的 action 操作计算速度加快（通常运行速度会加速 10 倍）。缓存是迭代算法和快速的交互式使用的重要工具。

RDD 可以使用  `persist()`  方法或  `cache()`  方法进行持久化。数据将会在第一次 action 操作时进行计算，并缓存在节点的内存中。Spark 的缓存具有容错机制，如果一个缓存的 RDD 的某个分区丢失了，Spark 将按照原来的计算过程，自动重新计算并进行缓存。

另外，每个持久化的 RDD 可以使用不同的  _storage level_  存储级别进行缓存，例如，持久化到磁盘、已序列化的 Java 对象形式持久化到内存（可以节省空间）、跨节点间复制、以 off-heap 的方式存储在 Tachyon。这些存储级别通过传递一个  `StorageLevel`  对象 ([Scala](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.storage.StorageLevel),  [Java](http://spark.apachecn.org/docs/cn/2.2.0/api/java/index.html?org/apache/spark/storage/StorageLevel.html),  [Python](http://spark.apachecn.org/docs/cn/2.2.0/api/python/pyspark.html#pyspark.StorageLevel)) 给  `persist()`  方法进行设置。`cache()`方法是使用默认存储级别的快捷设置方法，默认的存储级别是  `StorageLevel.MEMORY_ONLY`（将反序列化的对象存储到内存中）。详细的存储级别介绍如下:

Storage Level（存储级别）

Meaning（含义）

MEMORY_ONLY

将 RDD 以反序列化的 Java 对象的形式存储在 JVM 中. 如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算. 这是默认的级别.

MEMORY_AND_DISK

将 RDD 以反序列化的 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取.

MEMORY_ONLY_SER  
(Java and Scala)

将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用  [fast serializer](http://spark.apachecn.org/docs/cn/2.2.0/tuning.html)  时会节省更多的空间，但是在读取时会增加 CPU 的计算负担.

MEMORY_AND_DISK_SER  
(Java and Scala)

类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算.

DISK_ONLY

只在磁盘上缓存 RDD.

MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc.

与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本.

OFF_HEAP (experimental 实验性)

类似于 MEMORY_ONLY_SER, 但是将数据存储在  [off-heap memory](http://spark.apachecn.org/docs/cn/2.2.0/configuration.html#memory-management)  中. 这需要启用 off-heap 内存.

**Note:**  _在 Python 中, stored objects will 总是使用  [Pickle](https://docs.python.org/2/library/pickle.html)  library 来序列化对象, 所以无论你选择序列化级别都没关系. 在 Python 中可用的存储级别有  `MEMORY_ONLY`,  `MEMORY_ONLY_2`,  `MEMORY_AND_DISK`,  `MEMORY_AND_DISK_2`,  `DISK_ONLY`, 和  `DISK_ONLY_2`._

在 shuffle 操作中（例如  `reduceByKey`），即便是用户没有调用  `persist`  方法，Spark 也会自动缓存部分中间数据.这么做的目的是，在 shuffle 的过程中某个节点运行失败时，不需要重新计算所有的输入数据。如果用户想多次使用某个 RDD，强烈推荐在该 RDD 上调用 persist 方法.

### 存储级别

Spark 的存储级别的选择，核心问题是在 memory 内存使用率和 CPU 效率之间进行权衡。建议按下面的过程进行存储级别的选择:

-   如果您的 RDD 适合于默认存储级别 (`MEMORY_ONLY`), leave them that way. 这是CPU效率最高的选项，允许RDD上的操作尽可能快地运行.
    
-   如果不是, 试着使用  `MEMORY_ONLY_SER`  和  [selecting a fast serialization library](http://spark.apachecn.org/docs/cn/2.2.0/tuning.html)以使对象更加节省空间，但仍然能够快速访问。 (Java和Scala)
    
-   不要溢出到磁盘，除非计算您的数据集的函数是昂贵的, 或者它们过滤大量的数据. 否则, 重新计算分区可能与从磁盘读取分区一样快.
    
-   如果需要快速故障恢复，请使用复制的存储级别 (e.g. 如果使用Spark来服务 来自网络应用程序的请求).  _All_  存储级别通过重新计算丢失的数据来提供完整的容错能力，但复制的数据可让您继续在 RDD 上运行任务，而无需等待重新计算一个丢失的分区.

### 删除数据

Spark 会自动监视每个节点上的缓存使用情况，并使用 least-recently-used（LRU）的方式来丢弃旧数据分区。 如果您想手动删除 RDD 而不是等待它掉出缓存，使用 `RDD.unpersist()` 方法。

# 共享变量

通常情况下，一个传递给 Spark 操作（例如 `map` 或 `reduce`）的函数 func 是在远程的集群节点上执行的。该函数 func 在多个节点执行过程中使用的变量，是同一个变量的多个副本。这些变量的以副本的方式拷贝到每个机器上，并且各个远程机器上变量的更新并不会传播回 driver program（驱动程序）。通用且支持 read-write（读-写） 的共享变量在任务间是不能胜任的。所以，Spark 提供了两种特定类型的共享变量 : broadcast variables（广播变量）和 accumulators（累加器）。

## 广播变量

Broadcast variables（广播变量）允许程序员将一个 read-only（只读的）变量缓存到每台机器上，而不是给任务传递一个副本。它们是如何来使用呢，例如，广播变量可以用一种高效的方式给每个节点传递一份比较大的 input dataset（输入数据集）副本。在使用广播变量时，Spark 也尝试使用高效广播算法分发 broadcast variables（广播变量）以降低通信成本。

Spark 的 action（动作）操作是通过一系列的 stage（阶段）进行执行的，这些 stage（阶段）是通过分布式的 “shuffle” 操作进行拆分的。Spark 会自动广播出每个 stage（阶段）内任务所需要的公共数据。这种情况下广播的数据使用序列化的形式进行缓存，并在每个任务运行前进行反序列化。这也就意味着，只有在跨越多个 stage（阶段）的多个任务会使用相同的数据，或者在使用反序列化形式的数据特别重要的情况下，使用广播变量会有比较好的效果。

广播变量通过在一个变量  `v`  上调用  `SparkContext.broadcast(v)`  方法来进行创建。广播变量是  `v`  的一个 wrapper（包装器），可以通过调用  `value`  方法来访问它的值。代码示例如下:

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```

在创建广播变量之后，在集群上执行的所有的函数中，应该使用该广播变量代替原来的 `v` 值，所以节点上的 `v` 最多分发一次。另外，对象 `v` 在广播后不应该再被修改，以保证分发到所有的节点上的广播变量具有同样的值（例如，如果以后该变量会被运到一个新的节点）。

## Accumulators（累加器）

Accumulators（累加器）是一个仅可以执行 “added”（添加）的变量来通过一个关联和交换操作，因此可以高效地执行支持并行。累加器可以用于实现 counter（ 计数，类似在 MapReduce 中那样）或者 sums（求和）。原生 Spark 支持数值型的累加器，并且程序员可以添加新的支持类型。

作为一个用户，您可以创建 accumulators（累加器）并且重命名. 如下图所示, 一个命名的 accumulator 累加器（在这个例子中是  `counter`）将显示在 web UI 中，用于修改该累加器的阶段。 Spark 在 “Tasks” 任务表中显示由任务修改的每个累加器的值.

![Accumulators in the Spark UI](http://spark.apachecn.org/docs/cn/2.2.0/img/spark-webui-accumulators.png "Accumulators in the Spark UI")

在 UI 中跟踪累加器可以有助于了解运行阶段的进度（注: 这在 Python 中尚不支持）.

可以通过调用  `SparkContext.longAccumulator()`  或  `SparkContext.doubleAccumulator()`  方法创建数值类型的  `accumulator`（累加器）以分别累加 Long 或 Double 类型的值。集群上正在运行的任务就可以使用  `add`  方法来累计数值。然而，它们不能够读取它的值。只有 driver program（驱动程序）才可以使用  `value`  方法读取累加器的值。

下面的代码展示了一个 accumulator（累加器）被用于对一个数组中的元素求和:

```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```

虽然此代码使用 Long 类型的累加器的内置支持, 但是开发者通过  [AccumulatorV2](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.util.AccumulatorV2)它的子类来创建自己的类型. AccumulatorV2 抽象类有几个需要 override（重写）的方法:  `reset`  方法可将累加器重置为 0,  `add`  方法可将其它值添加到累加器中,`merge`  方法可将其他同样类型的累加器合并为一个. 其他需要重写的方法可参考  [API documentation](http://spark.apachecn.org/docs/cn/2.2.0/api/scala/index.html#org.apache.spark.util.AccumulatorV2). 例如, 假设我们有一个表示数学上 vectors（向量）的  `MyVector`  类，我们可以写成:

```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}

// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1")
```

注意，在开发者定义自己的 AccumulatorV2 类型时， resulting type（返回值类型）可能与添加的元素的类型不一致。

累加器的更新只发生在  **action**  操作中，Spark 保证每个任务只更新累加器一次，例如，重启任务不会更新值。在 transformations（转换）中， 用户需要注意的是，如果 task（任务）或 job stages（阶段）重新执行，每个任务的更新操作可能会执行多次。

累加器不会改变 Spark lazy evaluation（懒加载）的模式。如果累加器在 RDD 中的一个操作中进行更新，它们的值仅被更新一次，RDD 被作为 action 的一部分来计算。因此，在一个像  `map()`  这样的 transformation（转换）时，累加器的更新并没有执行。下面的代码片段证明了这个特性:

```scala
val accum = sc.longAccumulator
data.map { x => accum.add(x); x }
// Here, accum is still 0 because no actions have caused the map operation to be computed.
```

# 布署应用到集群中


<!--stackedit_data:
eyJoaXN0b3J5IjpbMjExNTU1MDgzLC0xMjcwMzM2NTczLC0xMD
EyNzU3NDI3LDM1ODE2MDE5MCw4MzA3ODUyODEsLTgzOTY0OTA2
MF19
-->