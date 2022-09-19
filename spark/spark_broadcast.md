## 简介

什么是broadcast？

指driver将Broadcast变量分发到job相关的executor上。

Broadcast变量在每个机器上保存一个副本，而不是为每个task传输一个副本。

它可以用于为每个node提供一个大的dataset的副本。spark还尝试使用高效的broadcast算法来分发broadcast变量，降低传输成本。

可以通过SparkContext.broadcast()创建broadcast变量，然后调用Broadcast.value()方法获取它的值。

Broadcast变量被broadcast后，它的值不能被修改，以此保证各个node有相同的值。

## 主要类

spark 3.2.1

### Broadcast

主要实现类为TorrentBroadcast，实现了一个类似BitTorrent机制的Broadcast对象。

TorrentBroadcast主要机制为：

- driver将序列化的对象分成多个小块，然后存储到driver端的BlockManager。
- 每个executor，首先尝试从本地的BlockManager获取broadcast对象。如果本地不存在，就从driver或其他executor远程获取这些小的块。
- 一旦executor获取到这些块，就存储到它自己的BlockManager，其他executor可以从它这里获取数据块。

### BroadcastManager

管理Broadcast变量，cache本地Broadcast变量。

### BlockManager

提供Block管理服务，数据可以存储到内存、磁盘或远程存储里。

### BlockManagerMaster

位于BlockManager中，负责与BlockManagerMasterEndpoint的交互，如BlockManager的注册、上报block状态、获取block所在位置等。

## 主要流程

### Broadcast变量的使用

可以参考BroadcastTest类。

```
// BroadcastTest.scala
    val sc = spark.sparkContext

    val slices = if (args.length > 0) args(0).toInt else 2
    val num = if (args.length > 1) args(1).toInt else 1000000

    val arr1 = (0 until num).toArray

    for (i <- 0 until 3) {
      println(s"Iteration $i")
      println("===========")
      val startTime = System.nanoTime
      val barr1 = sc.broadcast(arr1) // 创建broadcast变量
      val observedSizes = sc.parallelize(1 to 10, slices).map(_ => barr1.value.length) // 通过value方法获取broadcast变量
      // Collect the small RDD so we can print the observed sizes locally.
      observedSizes.collect().foreach(i => println(i))
      println("Iteration %d took %.0f milliseconds".format(i, (System.nanoTime - startTime) / 1E6))
    }
```

### Broadcast变量的存储、获取、转发流程

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_broadcast_structure.drawio.png)

如上图所示，为Broadcast变量的存储和使用过程。

左侧，driver端创建Broadcast变量时，将其存储到BlockManager中。BlockManager存储完成后，将block的位置report给BlockManagerMasterEndpoint。

右侧，executor中Task执行时，调用Broadcast.getValue()方法时，首先从BroadcastManager获取本地cache的Broadcast变量。
本地不存在时，通过BlockManager从远程（driver或其他executor）获取。BlockManager向driver发请求获取block位置，然后通过BlockTransferService获取block数据。获取到Broadcast变量的block后，将状态report给driver，之后其他executor也可以从本executor获取block。