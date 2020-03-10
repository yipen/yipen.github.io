[Yarn的调度器](#Yarn的调度器) </br>
[Priority在Yarn中的使用](#Priority在Yarn中的使用) </br>
[SparkOnYarn支持priority](#SparkOnYarn支持priority) </br>
[参考](#参考) </br>

## Yarn的调度器
在Yarn中，提供了Capacity scheduler和Fair scheduler，它们都支持priority的。这里我们简单介绍下概念，不做过多的描述。
### Capacity Scheduler
Capacity scheuler设计的目的是为了让Hadoop上的applications可以以一个多租户的形式下分享资源运行，这种调度器一般应用在有一个较大的公有集群，按照队列来分配资源给特定的用户组。我们可以简单的通过配置就可以设定队列在cluster中资源或者用户在队列中的的使用限制（最低保障和最高上限等），当一个队列的资源空余的时候，Yarn可以暂时利用剩余的资源分享给其他需要的队列。
### Fair Scheduler
Fair scheduler就如同它的名字一样，他在分配资源的时候，是秉承着公平原则，applications在一段时间内分配到的平均资源会趋于相等。如果一个只有一个application在集群上运行的时候，资源都可供这一个application使用。如果有另外的application被提交到集群上时，空闲的资源就会被分配给新提交的application上，这样最后每个运行的application都会分配到相等的资源。

## Priority在Yarn中的使用
### Capacity Scheduler
Capacity scheduler支持对应用的priority的设置。Yarn的priority是整数型，更大的数就代表更高的优先级，这个功能只支持在FIFO（默认）的策略下进行。priority可以针对cluster或者queue级别进行设置。
1. cluster level: 如果你的application设置的priority超过了cluster最大值，那按照最大的cluster priority对待。
2. queue level: 队列有一个默认的priority值，queue下的applications如果没有设置具体的priority会被设置成该默认值。如果application变更了queue，它的priority值不会更改。
### Fair Scheduler
Fair scheduler支持把一个正在运行的application迁移到另一个priority不同的queue里，这样这个application获取资源的权重就会跟着queue变化。被迁移得application的资源就会算在新的queue上，如果所需资源超过了新的queue的最大限制，迁移就会失败。

## SparkOnYarn支持priority
### 如何为Spark app设置priority
只需要再SparkConf里进行设置即可，遵循Yarn对于priority的定义，数值越大，priority越高，在同一时间提交的job会有更高的优先级获取资源：
``` java
val sparkConf = new SparkConf()
      .set(APPLICATION_TAGS.key, ",tag1, dup,tag2 , ,multi word , dup")
      .set(MAX_APP_ATTEMPTS, 42)
      .set("spark.app.name", "foo-test-app")
      .set(QUEUE_NAME, "staging-queue")
      .set(APPLICATION_PRIORITY, 1)
```

### Spark源码
Spark目前已经有了对于Yarn的priority官方支持，这里给出一个在Jira上closed的[SPARK-10879](https://issues.apache.org/jira/browse/SPARK-10879)。这个Jira是很早以前的一个版本，diff仅供参考，用于让大家理解Spark on Yarn如何设置priority的基本流程。
其实需要支持priority很简单，一是需要在submit的时候提供priority参数的设置,官方是放在了SparkConf里去设置；另一个是需要在createApplicationSubmissionContext的时候，调用setPriority将priority传入到Yarn。这里给出关键的地方的代码：
1. /resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/config.scala
``` java
  private[spark] val APPLICATION_PRIORITY = ConfigBuilder("spark.yarn.priority")
    .doc("Application priority for YARN to define pending applications ordering policy, those" +
      " with higher value have a better opportunity to be activated. Currently, YARN only" +
      " supports application priority when using FIFO ordering policy.")
    .intConf
    .createOptional
```
2. /resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala中对于createApplicationSubmissionContext函数的修改：
``` java
/**
   * Set up the context for submitting our ApplicationMaster.
   * This uses the YarnClientApplication not available in the Yarn alpha API.
   */
  def createApplicationSubmissionContext(
      newApp: YarnClientApplication,
      containerContext: ContainerLaunchContext): ApplicationSubmissionContext = {

    val componentName = if (isClusterMode) {
      config.YARN_DRIVER_RESOURCE_TYPES_PREFIX
    } else {
      config.YARN_AM_RESOURCE_TYPES_PREFIX
    }
    val yarnAMResources = getYarnResourcesAndAmounts(sparkConf, componentName)
    val amResources = yarnAMResources ++
      getYarnResourcesFromSparkResources(SPARK_DRIVER_PREFIX, sparkConf)
    logDebug(s"AM resources: $amResources")
    val appContext = newApp.getApplicationSubmissionContext
    appContext.setApplicationName(sparkConf.get("spark.app.name", "Spark"))
    appContext.setQueue(sparkConf.get(QUEUE_NAME))
    appContext.setAMContainerSpec(containerContext)
    appContext.setApplicationType("SPARK")

    sparkConf.get(APPLICATION_TAGS).foreach { tags =>
      appContext.setApplicationTags(new java.util.HashSet[String](tags.asJava))
    }
    sparkConf.get(MAX_APP_ATTEMPTS) match {
      case Some(v) => appContext.setMaxAppAttempts(v)
      case None => logDebug(s"${MAX_APP_ATTEMPTS.key} is not set. " +
          "Cluster's default value will be used.")
    }

    sparkConf.get(AM_ATTEMPT_FAILURE_VALIDITY_INTERVAL_MS).foreach { interval =>
      appContext.setAttemptFailuresValidityInterval(interval)
    }

    val capability = Records.newRecord(classOf[Resource])
    capability.setMemory(amMemory + amMemoryOverhead)
    capability.setVirtualCores(amCores)
    if (amResources.nonEmpty) {
      ResourceRequestHelper.setResourceRequests(amResources, capability)
    }
    logDebug(s"Created resource capability for AM request: $capability")

    sparkConf.get(AM_NODE_LABEL_EXPRESSION) match {
      case Some(expr) =>
        val amRequest = Records.newRecord(classOf[ResourceRequest])
        amRequest.setResourceName(ResourceRequest.ANY)
        amRequest.setPriority(Priority.newInstance(0))
        amRequest.setCapability(capability)
        amRequest.setNumContainers(1)
        amRequest.setNodeLabelExpression(expr)
        appContext.setAMContainerResourceRequest(amRequest)
      case None =>
        appContext.setResource(capability)
    }

    sparkConf.get(ROLLED_LOG_INCLUDE_PATTERN).foreach { includePattern =>
      try {
        val logAggregationContext = Records.newRecord(classOf[LogAggregationContext])
        logAggregationContext.setRolledLogsIncludePattern(includePattern)
        sparkConf.get(ROLLED_LOG_EXCLUDE_PATTERN).foreach { excludePattern =>
          logAggregationContext.setRolledLogsExcludePattern(excludePattern)
        }
        appContext.setLogAggregationContext(logAggregationContext)
      } catch {
        case NonFatal(e) =>
          logWarning(s"Ignoring ${ROLLED_LOG_INCLUDE_PATTERN.key} because the version of YARN " +
            "does not support it", e)
      }
    }
    appContext.setUnmanagedAM(isClientUnmanagedAMEnabled)

    sparkConf.get(APPLICATION_PRIORITY).foreach { appPriority =>
      appContext.setPriority(Priority.newInstance(appPriority))
    }
    appContext
  }
```

## 参考
Yarn Capacity Scheduler: https://hadoop.apache.org/docs/r3.1.1/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html </br>
Yarn Fair Scheduler: https://hadoop.apache.org/docs/r3.1.1/hadoop-yarn/hadoop-yarn-site/FairScheduler.html </br>
Spark Jira: https://issues.apache.org/jira/browse/SPARK-10879 </br>
Hadoop Yarn API: https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/yarn/api/records/ApplicationSubmissionContext.html </br>