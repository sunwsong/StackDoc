# 概述

在一个较高的概念上来说，每一个 Spark 应用程序由一个在集群上运行着用户的  `main`  函数和执行各种并行操作的  _driver program_（驱动程序）组成。Spark 提供的主要抽象是一个_弹性分布式数据集_（RDD），它是可以执行并行操作且跨集群节点的元素的集合。RDD 可以从一个 Hadoop 文件系统（或者任何其它 Hadoop 支持的文件系统），或者一个在 driver program（驱动程序）中已存在的 Scala 集合，以及通过 transforming（转换）来创建一个 RDD。用户为了让它在整个并行操作中更高效的重用，也许会让 Spark persist（持久化）一个 RDD 到内存中。最后，RDD 会自动的从节点故障中恢复。

在 Spark 中的第二个抽象是能够用于并行操作的  _shared variables_（共享变量），默认情况下，当 Spark 的一个函数作为一组不同节点上的任务运行时，它将每一个变量的副本应用到每一个任务的函数中去。有时候，一个变量需要在整个任务中，或者在任务和 driver program（驱动程序）之间来共享。Spark 支持两种类型的共享变量 :  _broadcast variables_（广播变量），它可以用于在所有节点上缓存一个值，和  _accumulators_（累加器），他是一个只能被 “added（增加）” 的变量，例如 counters 和 sums。

# Spark依赖

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5Mjc5NjU5ODQsLTgzOTY0OTA2MF19
-->