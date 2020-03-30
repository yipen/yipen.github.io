最近看到一篇不错的关于Spark内存调优的blog,分享一下：
https://idk.dev/best-practices-for-successfully-managing-memory-for-apache-spark-applications-on-amazon-emr/
这篇blog主要提出了几种Spark内存调优方式（基于的是Amazon EMR总结的，但是我觉得通用性还是很强），的确是平时会遇到的情况，在这里就不做通篇的翻译了，我在这里结合自己遇到的情况大概总结一些，详细的大家可以自己去看原文。

## Spark调优基本配置参数
以下几个参数是最基本的job调优参数，只有把这几个参数设置的比较合适之后，我们才有更进一步的优化。
| property name | Default | Meaning |
| ------ | ------ | ------ |
|spark.executor.memory| 1g | Amount of memory to use per executor process, in the same format as JVM memory strings with a size unit suffix ("k", "m", "g" or "t") (e.g. 512m, 2g). |
|spark.driver.memory| 1g | Amount of memory to use for the driver process, i.e. where SparkContext is initialized, in the same format as JVM memory strings with a size unit suffix ("k", "m", "g" or "t") (e.g. 512m, 2g).
Note: In client mode, this config must not be set through the SparkConf directly in your application, because the driver JVM has already started at that point. Instead, please set this through the --driver-memory command line option or in your default properties file.|
|spark.executor.cores|1 in YARN mode, all the available cores on the worker in standalone and Mesos coarse-grained modes.|The number of cores to use on each executor. In standalone and Mesos coarse-grained modes|
|spark.driver.cores|1|Number of cores to use for the driver process, only in cluster mode.|
|spark.default.parallelism|For distributed shuffle operations like reduceByKey and join, the largest number of partitions in a parent RDD. For operations like parallelize with no parent RDDs, it depends on the cluster manager | Default number of partitions in RDDs returned by transformations like join, reduceByKey, and parallelize when not set by user.|
|spark.executor.instances| 2 in Yarn mode | The number of executors for static allocation. With spark.dynamicAllocation.enabled, the initial set of executors will be at least this large.|

## 设置timeout和retry
有时候在log里会看到timeout, lostNodeFail, lose connect这样的关键字，我们可以考虑去配置timeout的threshold，已经增加一些retry
比较常见的：
| property name | Default | Meaning |
| ------ | ------ | ------ |
|spark.network.timeout|120s|Default timeout for all network interactions. This config will be used in place of spark.core.connection.ack.wait.timeout, spark.storage.blockManagerSlaveTimeoutMs, spark.shuffle.io.connectionTimeout, spark.rpc.askTimeout or spark.rpc.lookupTimeout if they are not configured.|
|spark.executor.heartbeatInterval|10s|Interval between each executor's heartbeats to the driver. Heartbeats let the driver know that the executor is still alive and update it with metrics for in-progress tasks. spark.executor.heartbeatInterval should be significantly less than spark.network.timeout|
|spark.task.maxFailures|4|Number of failures of any particular task before giving up on the job. The total number of failures spread across different tasks will not cause the job to fail; a particular task has to fail this number of attempts. Should be greater than or equal to 1. Number of allowed retries = this value - 1.|
|spark.shuffle.io.maxRetries|3|(Netty only) Fetches that fail due to IO-related exceptions are automatically retried if this is set to a non-zero value. This retry logic helps stabilize large shuffles in the face of long GC pauses or transient network connectivity issues.|
|spark.yarn.maxAppAttempts|yarn.resourcemanager.am.max-attempts in YARN|The maximum number of attempts that will be made to submit the application. It should be no larger than the global number of max attempts in the YARN configuration.|
|spark.rpc.numRetries|3|Number of times to retry before an RPC task gives up. An RPC task will run at most times of this number.|
|spark.rpc.retry.wait|3s|Duration for an RPC ask operation to wait before retrying.|

## 设置JVM参数帮助调优
JVM的各种参数都可以通过spark.executor.extraJavaOptions/spark.driver.extraJavaOptions来设置。其实我们有时候发现我们的job突然跑的很慢，一方面可以去看看Yarn上的资源分配情况，另一方面也可以没看看是不是有大量的时间用来做GC导致的。
这里给出的example呢，是设置了G1来进行gc，并且打印一些GC的log.

```
"spark.executor.extraJavaOptions": "-XX:+UseG1GC -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=35 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:OnOutOfMemoryError='kill -9 %p' -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC",
"spark.driver.extraJavaOptions": "-XX:+UseG1GC -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=35 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:OnOutOfMemoryError='kill -9 %p' -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC",
```

## spark.dynamicAllocation.enabled
这个是Spark自带的一个功能，一般来说都是你的spark Job已经调优的很不错了，你希望能让自己的cluster有更好的资源利用率的情况下，会去enable的,算是进阶调优啦。
关于个功能，我之后会单独做个介绍。


