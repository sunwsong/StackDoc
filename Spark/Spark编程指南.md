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



### Action（动作）

### Shuffle操作

#### Background（幕后）

#### 性能影响

## RDD Persistence（持久化）

### 存储级别
### 删除数据

# 共享变量

## 广播变量
## Accumulators（累加器）

# 布署应用到集群中


<!--stackedit_data:
eyJoaXN0b3J5IjpbNDY2MTg4Nzk0LC0xMDEyNzU3NDI3LDM1OD
E2MDE5MCw4MzA3ODUyODEsLTgzOTY0OTA2MF19
-->