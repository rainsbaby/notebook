Spark 启动（Standalone模式）

参考：[Spark Standalone Mode](https://spark.apache.org/docs/3.2.1/spark-standalone.html)

## 启动Cluster

### 启动Master

	./sbin/start-master.sh

Master类：org.apache.spark.deploy.master.Master



### 启动Worker

	./sbin/start-worker.sh spark://{master_host}:{master_port}

Worker类：org.apache.spark.deploy.worker.Worker


## 提交task


入口：org.apache.spark.deploy.SparkSubmit.main

参数：

```
--class
org.apache.spark.examples.JavaWordCount
--master
local[8]
examples/target/original-spark-examples_2.12-3.2.1.jar
examples/src/main/resources/people.txt
```

通过代码可以看出，最终通过反射调用JavaWordCount.main方法，执行计算逻辑。

## Task执行

### SparkContext

### SparkSession

### Spark基础架构

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_structure.png)

## Standalone模式

Master

Worker

SparkSubmit


## High Availability


## 问题记录

### java.lang.ClassNotFoundException: com.google.common.cache.CacheLoader

实际是有Guava这个jar的，但是是provided的，所以需要在debug选项中添加配置 “Add dependencies with "provided" scope to classpath”。



### 添加spark-version-info.properties文件

执行：
	 build/spark-build-info core/target/extra-resources 3.2.1-SNAPSHOT
	 
 