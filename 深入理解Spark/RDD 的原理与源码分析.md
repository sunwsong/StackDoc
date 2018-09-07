本文链接：[http://blog.csdn.net/u011239443/article/details/53894611](http://blog.csdn.net/u011239443/article/details/53894611)
该论文来自 Berkeley 实验室，英文标题为：Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing。下面的翻译，我是基于博文 [http://shiyanjun.cn/archives/744.html](http://shiyanjun.cn/archives/744.html) 翻译基础上进行优化、修改、补充注释和源码分析。如果翻译措辞或逻辑有误，欢迎批评指正。

# 摘要

本文提出了分布式内存抽象的概念——弹性分布式数据集（RDD，Resilient Distributed Datasets），它具备像 MapReduce 等数据流模型的容错特性，并且允许开发人员在大型集群上执行基于内存的计算。现有的数据流系统对两种应用的处理并不高效：一是迭代式算法，这在图应用和机器学习领域很常见；二是交互式数据挖掘工具。这两种情况下，将数据保存在内存中能够极大地提高性能。为了有效地实现容错，RDD 提供了一种高度受限的共享内存，即 RDD 是只读的，并且只能通过其他 RDD 上的批量操作来创建（**_注：还可以由外部存储系数据集创建，如 HDFS_**）。尽管如此，RDD 仍然足以表示很多类型的计算，包括 MapReduce 和专用的迭代编程模型（如 Pregel）等。我们实现的 RDD 在迭代计算方面比 Hadoop 快 20 多倍，同时还可以在 5-7 秒内交互式地查询 1TB 数据集。

# 1\. 引言

无论是工业界还是学术界，都已经广泛使用高级集群编程模型来处理日益增长的数据，如 MapReduce 和 Dryad。这些系统将分布式编程简化为自动提供位置感知性调度、容错以及负载均衡，使得大量用户能够在商用集群上分析超大数据集。

大多数现有的集群计算系统都是基于非循环的数据流模型。从稳定的物理存储（如分布式文件系统）(**_注：即磁盘_**) 中加载记录，记录被传入由一组确定性操作构成的 DAG，然后写回稳定存储。DAG 数据流图能够在运行时自动实现任务调度和故障恢复。

尽管非循环数据流是一种很强大的抽象方法，但仍然有些应用无法使用这种方式描述。我们就是针对这些不太适合非循环模型的应用，它们的特点是在多个并行操作之间重用工作数据集。这类应用包括：（1）机器学习和图应用中常用的迭代算法（每一步对数据执行相似的函数）(**_注：有许多机器学习算法需要将这次迭代权值调优后的结果数据集作为下次迭代的输入，而使用 MapReduce 计算框架经过一次 Reduce 操作后输出数据结果写回磁盘，速度大大的降低了_**)；（2）交互式数据挖掘工具（用户反复查询一个数据子集）。基于数据流的框架并不明确支持工作集，所以需要将数据输出到磁盘，然后在每次查询时重新加载，这带来较大的开销。

我们提出了一种分布式的内存抽象，称为弹性分布式数据集（RDD，Resilient Distributed Datasets）。它支持基于工作集的应用，同时具有数据流模型的特点：自动容错、位置感知调度和可伸缩性。RDD 允许用户在执行多个查询时显式地将工作集缓存在内存中，后续的查询能够重用工作集，这极大地提升了查询速度。

RDD 提供了一种高度受限的共享内存模型，即 RDD 是只读的记录分区的集合，只能通过在其他 RDD 执行确定的转换操作（如 map、join 和 group by）而创建，然而这些限制使得实现容错的开销很低。与分布式共享内存系统需要付出高昂代价的检查点和回滚机制不同，RDD 通过 Lineage 来重建丢失的分区：一个 RDD 中包含了如何从其他 RDD 衍生所必需的相关信息，从而不需要检查点操作就可以重构丢失的数据分区。尽管 RDD 不是一个通用的共享内存抽象，但却具备了良好的描述能力、可伸缩性和可靠性，但却能够广泛适用于数据并行类应用。

第一个指出非循环数据流存在不足的并非是我们，例如，Google 的 Pregel[21]，是一种专门用于迭代式图算法的编程模型；Twister[13] 和 HaLoop[8]，是两种典型的迭代式 MapReduce 模型。但是，对于一些特定类型的应用，这些系统提供了一个受限的通信模型。相比之下，RDD 则为基于工作集的应用提供了更为通用的抽象，用户可以对中间结果进行显式的命名和物化，控制其分区，还能执行用户选择的特定操作（而不是在运行时去循环执行一系列 MapReduce 步骤）。RDD 可以用来描述 Pregel、迭代式 MapReduce，以及这两种模型无法描述的其他应用，如交互式数据挖掘工具（用户将数据集装入内存，然后执行 ad-hoc 查询）。

Spark 是我们实现的 RDD 系统，在我们内部能够被用于开发多种并行应用。Spark 采用 Scala 语言 [5] 实现，提供类似于 DryadLINQ 的集成语言编程接口[34]，使用户可以非常容易地编写并行任务。此外，随着 Scala 新版本解释器的完善，Spark 还能够用于交互式查询大数据集。我们相信 Spark 会是第一个能够使用有效、通用编程语言，并在集群上对大数据集进行交互式分析的系统。

我们通过微基准和用户应用程序来评估 RDD。实验表明，在处理迭代式应用上 Spark 比 Hadoop 快高达 20 多倍，计算数据分析类报表的性能提高了 40 多倍，同时能够在 5-7 秒的延时内交互式扫描 1TB 数据集。此外，我们还在 Spark 之上实现了 Pregel 和 HaLoop 编程模型（包括其位置优化策略），以库的形式实现（分别使用了 100 和 200 行 Scala 代码）。最后，利用 RDD 内在的确定性特性，我们还创建了一种 Spark 调试工具 rddbg，允许用户在任务期间利用 Lineage 重建 RDD，然后像传统调试器那样重新执行任务。

本文首先在第 2 部分介绍了 RDD 的概念，然后第 3 部分描述 Spark API，第 4 部分解释如何使用 RDD 表示几种并行应用（包括 Pregel 和 HaLoop），第 5 部分讨论 Spark 中 RDD 的表示方法以及任务调度器，第 6 部分描述具体实现和 rddbg，第 7 部分对 RDD 进行评估，第 8 部分给出了相关研究工作，最后第 9 部分总结。

# 2\. 弹性分布式数据集（RDD）

本部分描述 RDD 和编程模型。首先讨论设计目标（2.1），然后定义 RDD（2.2），讨论 Spark 的编程模型（2.3），并给出一个示例（2.4），最后对比 RDD 与分布式共享内存（2.5）。

## 2.1 目标和概述

我们的目标是为基于工作集的应用（即多个并行操作重用中间结果的这类应用）提供抽象，同时保持 MapReduce 及其相关模型的优势特性：即自动容错、位置感知性调度和可伸缩性。RDD 比数据流模型更易于编程，同时基于工作集的计算也具有良好的描述能力。

在这些特性中，最难实现的是容错性。一般来说，分布式数据集的容错性有两种方式：即**_数据检查点_**和**_记录数据的更新_**。我们面向的是大规模数据分析，数据检查点操作成本很高：需要通过数据中心的网络连接在机器之间复制庞大的数据集，而网络带宽往往比内存带宽低得多，同时还需要消耗更多的存储资源（在内存中复制数据可以减少需要缓存的数据量，而存储到磁盘则会拖慢应用程序）。所以，我们选择记录更新的方式。但是，如果更新太多，那么记录更新成本也不低。因此，**_RDD 只支持粗粒度转换，即在大量记录上执行的单个操作。将创建 RDD 的一系列转换记录下来（即 Lineage），以便恢复丢失的分区。_**

虽然只支持粗粒度转换限制了编程模型，但我们发现 RDD 仍然可以很好地适用于很多应用，特别是支持数据并行的批量分析应用，包括数据挖掘、机器学习、图算法等，因为这些程序通常都会在很多记录上执行相同的操作。RDD 不太适合那些异步更新共享状态的应用，例如并行 web 爬虫。因此，我们的目标是为大多数分析型应用提供有效的编程模型，而其他类型的应用交给专门的系统。

## 2.2 RDD 抽象

RDD 是只读的、分区记录的集合。RDD 只能基于在稳定物理存储中的数据集和其他已有的 RDD 上执行确定性操作来创建。这些确定性操作称之为转换，如 map、filter、groupBy、join（转换不是程序开发人员在 RDD 上执行的操作（**_注：这句话的意思可能是，转换操作并不会触发 RDD 真正的 action。由于惰性执行，当进行 action 操作的时候，才会回溯去执行前面的转换操作_**））。

RDD 不需要物化。RDD 含有如何从其他 RDD 衍生（即计算）出本 RDD 的相关信息（即 Lineage），据此可以从物理存储的数据计算出相应的 RDD 分区。

## 2.3 编程模型

在 Spark 中，RDD 被表示为对象，通过这些对象上的方法（或函数）调用转换。

定义 RDD 之后，程序员就可以在动作（**_注：即 action 操作_**）中使用 RDD 了。动作是向应用程序返回值，或向存储系统导出数据的那些操作，例如，count（返回 RDD 中的元素个数），collect（返回元素本身），save（将 RDD 输出到存储系统）。在 Spark 中，只有在动作第一次使用 RDD 时，才会计算 RDD（即延迟计算）。这样在构建 RDD 的时候，运行时通过管道的方式传输多个转换。

程序员还可以从两个方面控制 RDD，即缓存和分区。用户可以请求将 RDD 缓存，这样运行时将已经计算好的 RDD 分区存储起来，以加速后期的重用。缓存的 RDD 一般存储在内存中，但如果内存不够，可以写到磁盘上。

另一方面，RDD 还允许用户根据关键字（key）指定分区顺序，这是一个可选的功能。目前支持哈希分区和范围分区。例如，应用程序请求将两个 RDD 按照同样的哈希分区方式进行分区（将同一机器上具有相同关键字的记录放在一个分区），以加速它们之间的 join 操作。在 Pregel 和 HaLoop 中，多次迭代之间采用一致性的分区置换策略进行优化，我们同样也允许用户指定这种优化。
（**_注:_**
![](https://img-blog.csdn.net/20161227135626363?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTIzOTQ0Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

）

## 2.4 示例：控制台日志挖掘

本部分我们通过一个具体示例来阐述 RDD。假定有一个大型网站出错，操作员想要检查 Hadoop 文件系统（HDFS）中的日志文件（TB 级大小）来找出原因。通过使用 Spark，操作员只需将日志中的错误信息装载到一组节点的内存中，然后执行交互式查询。首先，需要在 Spark 解释器中输入如下 Scala 代码：

```
    lines = spark.textFile("hdfs://...")
    errors = lines.filter(_.startsWith("ERROR"))
    errors.cache()
```

第 1 行从 HDFS 文件定义了一个 RDD（即一个文本行集合），第 2 行获得一个过滤后的 RDD，第 3 行请求将 errors 缓存起来。注意在 Scala 语法中 filter 的参数是一个闭包 (**_什么是闭包？[https://zhuanlan.zhihu.com/p/21346046](https://zhuanlan.zhihu.com/p/21346046)_**)。

这时集群还没有开始执行任何任务。但是，用户已经可以在这个 RDD 上执行对应的动作，例如统计错误消息的数目：

```
errors.count()
```

用户还可以在 RDD 上执行更多的转换操作，并使用转换结果，如：

```
// Count errors mentioning MySQL:
errors.filter(_.contains("MySQL")).count()
// Return the time fields of errors mentioning
// HDFS as an array (assuming time is field
// number 3 in a tab-separated format):
errors.filter(_.contains("HDFS"))
    .map(_.split('\t')(3))
    .collect()
```

使用 errors 的第一个 action 运行以后，Spark 会把 errors 的分区缓存在内存中，极大地加快了后续计算速度。注意，最初的 RDD lines 不会被缓存。因为错误信息可能只占原数据集的很小一部分（小到足以放入内存）。

最后，为了说明模型的容错性，图 1 给出了第 3 个查询的 Lineage 图。在 lines RDD 上执行 filter 操作，得到 errors，然后再 filter、map 后得到新的 RDD，在这个 RDD 上执行 collect 操作。Spark 调度器以流水线的方式执行后两个转换，向拥有 errors 分区缓存的节点发送一组任务。此外，如果某个 errors 分区丢失，Spark 只在相应的 lines 分区上执行 filter 操作来重建该 errors 分区。
![](https://img-blog.csdn.net/20161227142938658?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTIzOTQ0Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# <a></a>2.5 RDD 与分布式共享内存

为了进一步理解 RDD 是一种分布式的内存抽象，表 1 列出了 RDD 与分布式共享内存（DSM，Distributed Shared Memory）[24] 的对比。在 DSM 系统中，应用可以向全局地址空间的任意位置进行读写操作。（注意这里的 DSM，不仅指传统的共享内存系统，还包括那些通过分布式哈希表或分布式文件系统进行数据共享的系统，比如 Piccolo[28]（**_注: Spark 生态系统中有一名为 Alluxio 的分布式内存文件系统，它通常可作为 Spark 和 HDFS 的中间层存在_** ））DSM 是一种通用的抽象，但这种通用性同时也使得在商用集群上实现有效的容错性更加困难。

RDD 与 DSM 主要区别在于，不仅可以通过批量转换创建（即 “写”）RDD，还可以对任意内存位置读写。也就是说，RDD 限制应用执行批量写操作，这样有利于实现有效的容错。特别地，RDD 没有检查点开销，因为可以使用 Lineage 来恢复 RDD。而且，失效时只需要重新计算丢失的那些 RDD 分区，可以在不同节点上并行执行，而不需要回滚整个程序。

表 1 RDD 与 DSM 对比

| 对比项目 | RDD | 分布式共享内存（DSM） |
| --- | :-: | --: |
| 读 | 批量或细粒度操作 | 细粒度操作 |
| 写 | 批量转换操作 | 细粒度操作 |
| 一致性 | 不重要（RDD 是不可更改的） | 取决于应用程序或运行时 |
| 容错性 | 细粒度，低开销（使用 Lineage） | 需要检查点操作和程序回滚 |
| 落后任务的处理 | 任务备份 | 很难处理 |
| 任务安排 | 基于数据存放的位置自动实现 | 取决于应用程序（通过运行时实现透明性） |
| 如果内存不够 | 与已有的数据流系统类似 | 性能较差 |

注意，通过备份任务的拷贝，RDD 还可以处理落后任务（即运行很慢的节点），这点与 MapReduce[12] 类似。而 DSM 则难以实现备份任务，因为任务及其副本都需要读写同一个内存位置。

与 DSM 相比，RDD 模型有两个好处。**_第一，对于 RDD 中的批量操作，运行时将根据数据存放的位置来调度任务，从而提高性能。第二，对于基于扫描的操作，如果内存不足以缓存整个 RDD，就进行部分缓存。把内存放不下的分区存储到磁盘上，此时性能与现有的数据流系统差不多。_**

最后看一下读操作的粒度。RDD 上的很多动作（如 count 和 collect）都是批量读操作，即扫描整个数据集，可以将任务分配到距离数据最近的节点上。同时，RDD 也支持细粒度操作，即在哈希或范围分区的 RDD 上执行关键字查找。

# 3. Spark 编程接口

Spark 用 Scala[5] 语言实现了 RDD 的 API。Scala 是一种基于 JVM 的静态类型、函数式、面向对象的语言。我们选择 Scala 是因为它简洁（特别适合交互式使用）、有效（因为是静态类型）。但是，RDD 抽象并不局限于函数式语言，也可以使用其他语言来实现 RDD，比如像 Hadoop[2] 那样用类表示用户函数。

要使用 Spark，开发者需要编写一个 driver 程序，连接到集群以运行 Worker，如图 2 所示。Driver 定义了一个或多个 RDD，并调用 RDD 上的动作。Worker 是长时间运行的进程，将 RDD 分区以 Java 对象的形式缓存在内存中。

![](https://img-blog.csdn.net/20161227163227217?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTIzOTQ0Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
图 2 Spark 的运行时。用户的 driver 程序启动多个 worker，worker 从分布式文件系统中读取数据块，并将计算后的 RDD 分区缓存在内存中。

再看看 2.4 中的例子，用户执行 RDD 操作时会提供参数，比如 map 传递一个闭包（closure，函数式编程中的概念）。Scala 将闭包表示为 Java 对象，如果传递的参数是闭包，则这些对象被序列化，通过网络传输到其他节点上进行装载。Scala 将闭包内的变量保存为 Java 对象的字段。例如，var x = 5; rdd.map(_ + x) 这段代码将 RDD 中的每个元素加 5。总的来说，Spark 的语言集成类似于 DryadLINQ。

RDD 本身是静态类型对象，由参数指定其元素类型。例如，RDD[int] 是一个整型 RDD。不过，我们举的例子几乎都省略了这个类型参数，因为 Scala 支持类型推断。

虽然在概念上使用 Scala 实现 RDD 很简单，但还是要处理一些 Scala 闭包对象的反射问题。如何通过 Scala 解释器来使用 Spark 还需要更多工作，这点我们将在第 6 部分讨论。不管怎样，我们都不需要修改 Scala 编译器。

## 3.1 Spark 中的 RDD 操作

表 2 列出了 Spark 中的 RDD 转换和动作。每个操作都给出了标识，其中方括号表示类型参数。前面说过转换是延迟操作，用于定义新的 RDD；而动作启动计算操作，并向用户程序返回值或向外部存储写数据。

![](https://img-blog.csdn.net/20161227164534142?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTIzOTQ0Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

注意，有些操作只对键值对可用，比如 join。另外，函数名与 Scala 及其他函数式语言中的 API 匹配，例如 map 是一对一的映射，而 flatMap 是将每个输入映射为一个或多个输出（与 MapReduce 中的 map 类似）。

除了这些操作以外，用户还可以请求将 RDD 缓存起来。而且，用户还可以通过 Partitioner 类获取 RDD 的分区顺序，然后将另一个 RDD 按照同样的方式分区。有些操作会自动产生一个哈希或范围分区的 RDD，像 groupByKey，reduceByKey 和 sort 等。

# 4. 应用程序示例

现在我们讲述如何使用 RDD 表示几种基于数据并行的应用。首先讨论一些迭代式机器学习应用（4.1），然后看看如何使用 RDD 描述几种已有的集群编程模型，即 MapReduce（4.2），Pregel（4.3），和 Hadoop（4.4）。最后讨论一下 RDD 不适合哪些应用（4.5）。

## 4.1 迭代式机器学习

很多机器学习算法都具有迭代特性，运行迭代优化方法来优化某个目标函数，例如梯度下降方法。如果这些算法的工作集能够放入内存，将极大地加速程序运行。而且，这些算法通常采用批量操作，例如映射和求和，这样更容易使用 RDD 来表示。

例如下面的程序是逻辑回归 [15] 的实现。逻辑回归是一种常见的分类算法，即寻找一个最佳分割两组点（即垃圾邮件和非垃圾邮件）的超平面 w。算法采用梯度下降的方法：开始时 w 为随机值，在每一次迭代的过程中，对 w 的函数求和，然后朝着优化的方向移动 w。

``` scala
val points = spark.textFile(...)
     .map(parsePoint).persist()
var w = // random initial vector
for (i <- 1 to ITERATIONS) {
     val gradient = points.map{ p =>
          p.x * (1/(1+exp(-p.y*(w dot p.x)))-1)*p.y
     }.reduce((a,b) => a+b)
     w -= gradient
}
```

首先定义一个名为 points 的缓存 RDD，这是在文本文件上执行 map 转换之后得到的，即将每个文本行解析为一个 Point 对象。然后在 points 上反复执行 map 和 reduce 操作，每次迭代时通过对当前 w 的函数进行求和来计算梯度。7.1 小节我们将看到这种在内存中缓存 points 的方式，比每次迭代都从磁盘文件装载数据并进行解析要快得多。

已经在 Spark 中实现的迭代式机器学习算法还有：kmeans（像逻辑回归一样每次迭代时执行一对 map 和 reduce 操作），期望最大化算法（EM，两个不同的 map/reduce 步骤交替执行），交替最小二乘矩阵分解和协同过滤算法。Chu 等人提出迭代式 MapReduce 也可以用来实现常用的学习算法 [11]。

## 4.2 使用 RDD 实现 MapReduce

MapReduce 模型 [12] 很容易使用 RDD 进行描述。假设有一个输入数据集（其元素类型为 T），和两个函数 myMap: T => List[(Ki, Vi)] 和 myReduce: (Ki; List[Vi]) ) List[R]，代码如下：

``` scala
data.flatMap(myMap)
    .groupByKey()
    .map((k, vs) => myReduce(k, vs))
```

如果任务包含 combiner，则相应的代码为：

``` scala
data.flatMap(myMap)
    .reduceByKey(myCombiner)
    .map((k, v) => myReduce(k, v))
```

ReduceByKey 操作在 mapper 节点上执行部分聚集，与 MapReduce 的 combiner 类似。

## 4.3 使用 RDD 实现 Pregel

略

## 4.4 使用 RDD 实现 HaLoop

略

## 4.5 不适合使用 RDD 的应用

在 2.1 节我们讨论过，RDD 适用于具有批量转换需求的应用，并且相同的操作作用于数据集的每一个元素上。在这种情况下，RDD 能够记住每个转换操作，对应于 Lineage 图中的一个步骤，恢复丢失分区数据时不需要写日志记录大量数据。RDD 不适合那些通过异步细粒度地更新来共享状态的应用，例如 Web 应用中的存储系统，或者增量抓取和索引 Web 数据的系统，这样的应用更适合使用一些传统的方法，例如数据库、RAMCloud[26]、Percolator[27] 和 Piccolo[28]。我们的目标是，面向批量分析应用的这类特定系统，提供一种高效的编程模型，而不是一些异步应用程序。

# 5. RDD 的描述及作业调度

我们希望在不修改调度器的前提下，支持 RDD 上的各种转换操作，同时能够从这些转换获取 Lineage 信息。为此，我们为 RDD 设计了一组小型通用的内部接口。

简单地说，每个 RDD 都包含：（1）一组 RDD 分区（partition，即数据集的原子组成部分）；（2）对父 RDD 的一组依赖，这些依赖描述了 RDD 的 Lineage；（3）一个函数，即在父 RDD 上执行何种计算；（4）元数据，描述分区模式和数据存放的位置。例如，一个表示 HDFS 文件的 RDD 包含：各个数据块的一个分区，并知道各个数据块放在哪些节点上。而且这个 RDD 上的 map 操作结果也具有同样的分区，map 函数是在父数据上执行的。表 3 总结了 RDD 的内部接口。
表 3 Spark 中 RDD 的内部接口

| 操作 | 含义 |
| --- | :-: |
| partitions() | 返回一组 Partition 对象 |
| preferredLocations(p) | 根据数据存放的位置，返回分区 p 在哪些节点访问更快 |
| dependencies() | 返回一组依赖 |
| iterator(p, parentIters) | 按照父分区的迭代器，逐个计算分区 p 的元素 |
| partitioner() | 返回 RDD 是否 hash/range 分区的元数据信息 |

设计接口的一个关键问题就是，如何表示 RDD 之间的依赖。我们发现 RDD 之间的依赖关系可以分为两类，即：（1）窄依赖（narrow dependencies）：子 RDD 的每个分区依赖于常数个父分区（即与数据规模无关）；（2）宽依赖（wide dependencies）：子 RDD 的每个分区依赖于所有父 RDD 分区。例如，map 产生窄依赖，而 join 则是宽依赖（除非父 RDD 被哈希分区）。另一个例子见图 5。

![](https://img-blog.csdn.net/20161227211626421?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMTIzOTQ0Mw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（**_注：我们可以这样认为：_**

> 窄依赖指的是：每个 parent RDD 的 partition 最多被 child RDD 的一个 partition 使用
> 宽依赖指的是：每个 parent RDD 的 partition 被多个 child RDD 的 partition 使用

**_窄依赖每个 child RDD 的 partition 的生成操作都是可以并行的，而宽依赖则需要所有的 parent partition shuffle 结果得到后再进行。_**

**_下面我们来看下，我们来看下 org.apache.spark.Dependency.scala 的源码_**

抽象类 Dependency：

``` scala
abstract class Dependency[T] extends Serializable {
  def rdd: RDD[T]
}
```

**_Dependency 有两个子类，一个子类为窄依赖：NarrowDependency；一个为宽依赖 ShuffleDependency_**

**_NarrowDependency 也是一个抽象类，它实现了 getParents 重写了 rdd 函数，它有两个子类，一个是 OneToOneDependency，一个是 RangeDependency_**

``` scala
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
  /**
   * Get the parent partitions for a child partition.
   * @param partitionId a partition of the child RDD
   * @return the partitions of the parent RDD that the child partition depends upon
   */
  def getParents(partitionId: Int): Seq[Int]

  override def rdd: RDD[T] = _rdd
}
```

**_OneToOneDependency，可以看到 getParents 实现很简单，就是传进一个 partitionId: Int，再把 partitionId 放在 List 里面传出去，即去 parent RDD 中取与该 RDD 相同 partitionID 的数据_**

``` scala
class OneToOneDependency[T](rdd: RDD[T]) extends NarrowDependency[T](rdd) {
  override def getParents(partitionId: Int): List[Int] = List(partitionId)
}
```

**_RangeDependency，用于 union。与上面不同的是，这里我们要算出该位置，设某个 parent RDD 从 inStart 开始的 partition，逐个生成了 child RDD 从 outStart 开始的 partition，则计算方式为： partitionId - outStart + inStart_**

``` scala
class RangeDependency[T](rdd: RDD[T], inStart: Int, outStart: Int, length: Int)
  extends NarrowDependency[T](rdd) {

  override def getParents(partitionId: Int): List[Int] = {
    if (partitionId >= outStart && partitionId < outStart + length) {
      List(partitionId - outStart + inStart)
    } else {
      Nil
    }
  }
}
```

**_ShuffleDependency，需要进行 shuffle_**

```
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner,
    val serializer: Serializer = SparkEnv.get.serializer,
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None,
    val mapSideCombine: Boolean = false)
  extends Dependency[Product2[K, V]] {

  override def rdd: RDD[Product2[K, V]] = _rdd.asInstanceOf[RDD[Product2[K, V]]]

  private[spark] val keyClassName: String = reflect.classTag[K].runtimeClass.getName
  private[spark] val valueClassName: String = reflect.classTag[V].runtimeClass.getName
  // Note: It's possible that the combiner class tag is null, if the combineByKey
  // methods in PairRDDFunctions are used instead of combineByKeyWithClassTag.
  private[spark] val combinerClassName: Option[String] =
    Option(reflect.classTag[C]).map(_.runtimeClass.getName)
//获取shuffleID
  val shuffleId: Int = _rdd.context.newShuffleId()
//向注册shuffleManager注册Shuffle信息
  val shuffleHandle: ShuffleHandle = _rdd.context.env.shuffleManager.registerShuffle(
    shuffleId, _rdd.partitions.length, this)

  _rdd.sparkContext.cleaner.foreach(_.registerShuffleForCleanup(this))
}
```

）
区分这两种依赖很有用。首先，窄依赖允许在一个集群节点上以流水线的方式（pipeline）计算所有父分区。例如，逐个元素地执行 map、然后 filter 操作；而宽依赖则需要首先计算好所有父分区数据，然后在节点之间进行 Shuffle，这与 MapReduce 类似。第二，窄依赖能够更有效地进行失效节点的恢复，即只需重新计算丢失 RDD 分区的父分区，而且不同节点之间可以并行计算；而对于一个宽依赖关系的 Lineage 图，单个节点失效可能导致这个 RDD 的所有祖先丢失部分分区，因而需要整体重新计算。

通过 RDD 接口，Spark 只需要不超过 20 行代码实现便可以实现大多数转换。5.1 小节给出了例子，然后我们讨论了怎样使用 RDD 接口进行调度（5.2），最后讨论一下基于 RDD 的程序何时需要数据检查点操作（5.3）。

## <a></a>5.2 Spark 任务调度器

可见：[http://blog.csdn.net/u011239443/article/details/53911902](http://blog.csdn.net/u011239443/article/details/53911902)

## <a></a>5.3 检查点

尽管 RDD 中的 Lineage 信息可以用来故障恢复，但对于那些 Lineage 链较长的 RDD 来说，这种恢复可能很耗时。例如 4.3 小节中的 Pregel 任务，每次迭代的顶点状态和消息都跟前一次迭代有关，所以 Lineage 链很长。如果将 Lineage 链存到物理存储中，再定期对 RDD 执行检查点操作就很有效。

一般来说，Lineage 链较长、宽依赖的 RDD 需要采用检查点机制。这种情况下，集群的节点故障可能导致每个父 RDD 的数据块丢失，因此需要全部重新计算 [20]。将窄依赖的 RDD 数据存到物理存储中可以实现优化，例如前面 4.1 小节逻辑回归的例子，将数据点和不变的顶点状态存储起来，就不再需要检查点操作。

当前 Spark 版本提供检查点 API，但由用户决定是否需要执行检查点操作。今后我们将实现自动检查点，根据成本效益分析确定 RDD Lineage 图中的最佳检查点位置。

值得注意的是，因为 RDD 是只读的，所以不需要任何一致性维护（例如写复制策略，分布式快照或者程序暂停等）带来的开销，后台执行检查点操作。
（**_注：_**
**_我们来阅读下 org.apache.spark.rdd.ReliableCheckpointRDD 中的 def writePartitionToCheckpointFile 和 def writeRDDToCheckpointDirectory：_**

**_writePartitionToCheckpointFile: 把 RDD 一个 Partition 文件里面的数据写到一个 Checkpoint 文件里面_**

```
  def writePartitionToCheckpointFile[T: ClassTag](
      path: String,
      broadcastedConf: Broadcast[SerializableConfiguration],
      blockSize: Int = -1)(ctx: TaskContext, iterator: Iterator[T]) {
    val env = SparkEnv.get
    //获取Checkpoint文件输出路径
    val outputDir = new Path(path)
    val fs = outputDir.getFileSystem(broadcastedConf.value.value)

    //根据partitionId 生成 checkpointFileName
    val finalOutputName = ReliableCheckpointRDD.checkpointFileName(ctx.partitionId())
    //拼接路径与文件名
    val finalOutputPath = new Path(outputDir, finalOutputName)
    //生成临时输出路径
    val tempOutputPath =
      new Path(outputDir, s".$finalOutputName-attempt-${ctx.attemptNumber()}")

    if (fs.exists(tempOutputPath)) {
      throw new IOException(s"Checkpoint failed: temporary path $tempOutputPath already exists")
    }
    //得到块大小，默认为64MB
    val bufferSize = env.conf.getInt("spark.buffer.size", 65536)
    //得到文件输出流
    val fileOutputStream = if (blockSize < 0) {
      fs.create(tempOutputPath, false, bufferSize)
    } else {
      // This is mainly for testing purpose
      fs.create(tempOutputPath, false, bufferSize,
        fs.getDefaultReplication(fs.getWorkingDirectory), blockSize)
    }
    //序列化文件输出流
    val serializer = env.serializer.newInstance()
    val serializeStream = serializer.serializeStream(fileOutputStream)
    Utils.tryWithSafeFinally {
    //写数据
      serializeStream.writeAll(iterator)
    } {
      serializeStream.close()
    }

    if (!fs.rename(tempOutputPath, finalOutputPath)) {
      if (!fs.exists(finalOutputPath)) {
        logInfo(s"Deleting tempOutputPath $tempOutputPath")
        fs.delete(tempOutputPath, false)
        throw new IOException("Checkpoint failed: failed to save output of task: " +
          s"${ctx.attemptNumber()} and final output path does not exist: $finalOutputPath")
      } else {
        // Some other copy of this task must've finished before us and renamed it
        logInfo(s"Final output path $finalOutputPath already exists; not overwriting it")
        if (!fs.delete(tempOutputPath, false)) {
          logWarning(s"Error deleting ${tempOutputPath}")
        }
      }
    }
  }
```

**_writeRDDToCheckpointDirectoryWrite，将一个 RDD 写入到多个 checkpoint 文件，并返回一个 ReliableCheckpointRDD 来代表这个 RDD_**

```
 def writeRDDToCheckpointDirectory[T: ClassTag](
      originalRDD: RDD[T],
      checkpointDir: String,
      blockSize: Int = -1): ReliableCheckpointRDD[T] = {

    val sc = originalRDD.sparkContext

    // 生成 checkpoint文件 的输出路径
    val checkpointDirPath = new Path(checkpointDir)
    val fs = checkpointDirPath.getFileSystem(sc.hadoopConfiguration)
    if (!fs.mkdirs(checkpointDirPath)) {
      throw new SparkException(s"Failed to create checkpoint path $checkpointDirPath")
    }

    // 保存文件，并重新加载它作为一个RDD
    val broadcastedConf = sc.broadcast(
      new SerializableConfiguration(sc.hadoopConfiguration))
    sc.runJob(originalRDD,
      writePartitionToCheckpointFile[T](checkpointDirPath.toString, broadcastedConf) _)

    if (originalRDD.partitioner.nonEmpty) {
      writePartitionerToCheckpointDir(sc, originalRDD.partitioner.get, checkpointDirPath)
    }

    val newRDD = new ReliableCheckpointRDD[T](
      sc, checkpointDirPath.toString, originalRDD.partitioner)
    if (newRDD.partitions.length != originalRDD.partitions.length) {
      throw new SparkException(
        s"Checkpoint RDD $newRDD(${newRDD.partitions.length}) has different " +
          s"number of partitions from original RDD $originalRDD(${originalRDD.partitions.length})")
    }
    newRDD
  }
```

以上源码有可以改进的地方，因为重新计算 RDD 其实是没有必要的。

**_RDD checkpoint 之后得到了一个新的 RDD，那么 child RDD 如何知道 parent RDD 有没有被 checkpoint 过呢？ 看 RDD 的源码，我们可以发现：_**

```
private var dependencies_ : Seq[Dependency[_]] = null
```

dependencies_ 用来存放 checkpoint 后的结果的，如为 null，则就判断没 checkpoint：

```
 final def dependencies: Seq[Dependency[_]] = {
    checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
      if (dependencies_ == null) {
        dependencies_ = getDependencies
      }
      dependencies_
    }
  }
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkyNDIxOTUyOV19
-->