[前言](#前言) </br>
[Sparklens如何使用](#Sparklens如何使用) </br>
[Sparklens生成的报告](#Sparklens生成的报告) </br>
[参考](#参考) </br>

## 前言
[Sparklens](http://sparklens.qubole.com/)是一个可以帮助你了解你的Spark job效率的开源工具。Spark是个近些年来非常受欢迎的基于内存并行计算框架架，它有丰富的API支持，还支持 Spark SQL，MLlib，GraphX和Spark Streaming。在提交Spark Job的时候，我们会需要设置一些config, 虽然这极大的增加了开发者对于Spark job的控制灵活性，但是也会给优化Spark Job带来一些困难。我们平时写Spark Job的时候最长苦恼的应该就是如果和调节memoery,vcore这些参数，资源申请少了会造成job的失败，多了就会造成资源的浪费。所以Sparklens就是这么一个让你更加了解你的job运行情况，从而有效的进行Spark tuning的工具。

## Sparklens如何使用

### 和你的Job一起运行
这种方式是指当你的job在运行的时候，sparklens会实时的开始收集job的运行信息，最后打印结果。只要你的node能联网，那么你只需要在你的提交命令里面加上如下两行（当然如果没有网，你也可以从网上下载package之后打包运行）：
~~~
--packages qubole:sparklens:0.1.2-s_2.11
--conf spark.extraListeners=com.qubole.sparklens.QuboleJobListener
~~~

### Job跑完之后使用
这种方式会在你的job运行时生成一个Json文件在HDFS(默认)上，路径默认是/tmp/Sparklens。如果你想要设置自己的路径，可以通过配置spark.sparklens.data.dir来完成。
1. 首先你需要在运行你自己的spark job的时候在提交命令里面加上：
~~~
--packages qubole:sparklens:0.3.1-s_2.11
--conf spark.extraListeners=com.qubole.sparklens.QuboleJobListener
--conf spark.sparklens.reporting.disabled=true
~~~
2. 之后根据生成的Json文件，另外启动一个Job来获取分析你的Sparklens的报告结果
~~~
./bin/spark-submit --packages qubole:sparklens:0.3.1-s_2.11 --class com.qubole.sparklens.app.ReporterApp qubole-dummy-arg <filename>
~~~

### 从event-log里面获取
这种方式就是根据event log来生成Sparklens的报告，当然你需要在你的spark job里面enable event-log(之前的文章有提到event-log的相关知识)。
~~~
./bin/spark-submit --packages qubole:sparklens:0.3.1-s_2.11 --class com.qubole.sparklens.app.ReporterApp qubole-dummy-arg <filename> source=history
~~~

## Sparklens生成的报告
这里给出一个我自己例子来具体分析下。这里我跑了一个wordcount的spark Job,申请了2个executor(Spark on Yarn，相当于Yarn的container),1G的memory。 应用的是第一种Sparklens的使用方式，在job结束之后，从输出的log里拿到了Sparklens的报告结果。
~~~
Printing application meterics. These metrics are collected at task-level granularity and aggregated across the app (all tasks, stages, and jobs).
AggregateMetrics (Application Metrics) total measurements 2 
                NAME                        SUM                MIN           MAX                MEAN         
 diskBytesSpilled                            0.0 KB         0.0 KB         0.0 KB              0.0 KB
 executorRuntime                             4.0 ss       130.0 ms         3.9 ss              2.0 ss
 inputBytesRead                              0.0 KB         0.0 KB         0.0 KB              0.0 KB
 jvmGCTime                                 162.0 ms         0.0 ms       162.0 ms             81.0 ms
 memoryBytesSpilled                          0.0 KB         0.0 KB         0.0 KB              0.0 KB
 outputBytesWritten                          0.0 KB         0.0 KB         0.0 KB              0.0 KB
 peakExecutionMemory                       996.8 KB       415.6 KB       581.2 KB            498.4 KB
 resultSize                                109.7 KB         1.8 KB       108.0 KB             54.9 KB
 shuffleReadBytesRead                       44.8 KB         0.0 KB        44.8 KB             22.4 KB
 shuffleReadFetchWaitTime                    0.0 ms         0.0 ms         0.0 ms              0.0 ms
 shuffleReadLocalBlocks                           1              0              1                   0
 shuffleReadRecordsRead                       4,913              0          4,913               2,456
 shuffleReadRemoteBlocks                          0              0              0                   0
 shuffleWriteBytesWritten                   44.8 KB         0.0 KB        44.8 KB             22.4 KB
 shuffleWriteRecordsWritten                   4,913              0          4,913               2,456
 shuffleWriteTime                            4.4 ms         0.0 ms         4.4 ms              2.2 ms
 taskDuration                                4.7 ss       198.0 ms         4.5 ss              2.4 ss
Total Hosts 2, and the maximum concurrent hosts = 2
Host BN4SCH103041416 startTime 07:28:38:743 executors count 1
Host BN4SCH103041424 startTime 07:28:19:242 executors count 1
Done printing host timeline
======================
Printing executors timeline....
Total Executors 2, and maximum concurrent executors = 2
At 07:28 executors added 2 & removed  0 currently available 2
Done printing executors timeline...
============================
Printing Application timeline 
07:28:10:548 app started 
07:28:45:349 JOB 0 started : duration 00m 05s 
[      0   ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||          ]
[      1                                                                             ||| ]
07:28:45:482      Stage 0 started : duration 00m 04s 
07:28:49:987      Stage 0 ended : maxTaskTime 3869 taskCount 1
07:28:50:403      Stage 1 started : duration 00m 00s 
07:28:50:601      Stage 1 ended : maxTaskTime 130 taskCount 1
07:28:50:607 JOB 0 ended 
07:28:50:653 app ended 
Checking for job overlap...
JobGroup 1  SQLExecID (-1)
 Number of Jobs 1  JobIDs(0)
 Timing [07:28:45:349 - 07:28:50:607]
 Duration  00m 05s
 
 JOB 0 Start 07:28:45:349  End 07:28:50:607
 
No overlapping jobgroups found. Good
Time spent in Driver vs Executors
 Driver WallClock Time    00m 34s   86.89%
 Executor WallClock Time  00m 05s   13.11%
 Total WallClock Time     00m 40s
      
Minimum possible time for the app based on the critical path (with infinite resources)   00m 38s
Minimum possible time for the app with same executors, perfect parallelism and zero skew 00m 36s
If we were to run this app with single executor and single core                          00h 00m
Total cores available to the app 2
OneCoreComputeHours: Measure of total compute power available from cluster. One core in the executor, running
                      for one hour, counts as one OneCoreComputeHour. Executors with 4 cores, will have 4 times
                      the OneCoreComputeHours compared to one with just one core. Similarly, one core executor
                      running for 4 hours will OnCoreComputeHours equal to 4 core executor running for 1 hour.
Driver Utilization (Cluster idle because of driver)
Total OneCoreComputeHours available                             00h 01m
 Total OneCoreComputeHours available (AutoScale Aware)           00h 00m
 OneCoreComputeHours wasted by driver                            00h 01m
AutoScale Aware: Most of the calculations by this tool will assume that all executors are available throughout
                  the runtime of the application. The number above is printed to show possible caution to be
                  taken in interpreting the efficiency metrics.
Cluster Utilization (Executors idle because of lack of tasks or skew)
Executor OneCoreComputeHours available                  00h 00m
 Executor OneCoreComputeHours used                       00h 00m        38.03%
 OneCoreComputeHours wasted                              00h 00m        61.97%
App Level Wastage Metrics (Driver + Executor)
OneCoreComputeHours wasted Driver               86.89%
 OneCoreComputeHours wasted Executor             8.12%
 OneCoreComputeHours wasted Total                95.01%
App completion time and cluster utilization estimates with different executor counts
Real App Duration 00m 40s
 Model Estimation  00m 38s
 Model Error       3%
NOTE: 1) Model error could be large when auto-scaling is enabled.
       2) Model doesn't handles multiple jobs run via thread-pool. For better insights into
          application scalability, please try such jobs one by one without thread-pool.
Executor count     1  ( 50%) estimated time 00m 38s and estimated cluster utilization 10.29%
 Executor count     1  ( 80%) estimated time 00m 38s and estimated cluster utilization 10.29%
 Executor count     2  (100%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     2  (110%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     2  (120%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     3  (150%) estimated time 00m 38s and estimated cluster utilization 3.43%
 Executor count     4  (200%) estimated time 00m 38s and estimated cluster utilization 2.57%
 Executor count     6  (300%) estimated time 00m 38s and estimated cluster utilization 1.72%
 Executor count     8  (400%) estimated time 00m 38s and estimated cluster utilization 1.29%
 Executor count    10  (500%) estimated time 00m 38s and estimated cluster utilization 1.03%
Total tasks in all stages 2
Per Stage  Utilization
Stage-ID   Wall    Task      Task     IO%    Input     Output    ----Shuffle-----    -WallClockTime-    --OneCoreComputeHours---   MaxTaskMem
          Clock%  Runtime%   Count                               Input  |  Output    Measured | Ideal   Available| Used%|Wasted%                                  
       0   95.00   96.75         1    0.0    0.0 KB    0.0 KB    0.0 KB   44.8 KB    00m 04s   00m 01s    00h 00m   42.9   57.1  581.2 KB 
       1    4.00    3.25         1    0.0    0.0 KB    0.0 KB   44.8 KB    0.0 KB    00m 00s   00m 00s    00h 00m   32.8   67.2  415.6 KB 
Max memory which an executor could have taken = 581.2 KB
Stage-ID WallClock  OneCore       Task   PRatio    -----Task------   OIRatio  |* ShuffleWrite% ReadFetch%   GC%  *|
          Stage%     ComputeHours  Count            Skew   StageSkew                                                
      0   95.79         00h 00m       1    0.50     1.00     0.86     0.00     |*   0.11           0.00     4.19  *|
      1    4.21         00h 00m       1    0.50     1.00     0.66     0.00     |*   0.00           0.00     0.00  *|
PRatio:        Number of tasks in stage divided by number of cores. Represents degree of
               parallelism in the stage
TaskSkew:      Duration of largest task in stage divided by duration of median task.
               Represents degree of skew in the stage
TaskStageSkew: Duration of largest task in stage divided by total duration of the stage.
               Represents the impact of the largest task on stage time.
OIRatio:       Output to input ration. Total output of the stage (results + shuffle write)
               divided by total input (input data + shuffle read)
These metrics below represent distribution of time within the stage
ShuffleWrite:  Amount of time spent in shuffle writes across all tasks in the given
               stage as a percentage
ReadFetch:     Amount of time spent in shuffle read across all tasks in the given
               stage as a percentage
GC:            Amount of time spent in GC across all tasks in the given stage as a
               percentage
If the stage contributes large percentage to overall application time, we could look into
these metrics to check which part (Shuffle write, read fetch or GC is responsible)
~~~

我们重点看看几个在Spark UI上没有的东西：
1. Executor的数量
~~~
App completion time and cluster utilization estimates with different executor counts
Real App Duration 00m 40s
Model Estimation  00m 38s
Model Error       3%
NOTE: 1) Model error could be large when auto-scaling is enabled.
       2) Model doesn't handles multiple jobs run via thread-pool. For better insights into
          application scalability, please try such jobs one by one without thread-pool.
Executor count     1  ( 50%) estimated time 00m 38s and estimated cluster utilization 10.29%
 Executor count     1  ( 80%) estimated time 00m 38s and estimated cluster utilization 10.29%
 Executor count     2  (100%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     2  (110%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     2  (120%) estimated time 00m 38s and estimated cluster utilization 5.15%
 Executor count     3  (150%) estimated time 00m 38s and estimated cluster utilization 3.43%
 Executor count     4  (200%) estimated time 00m 38s and estimated cluster utilization 2.57%
 Executor count     6  (300%) estimated time 00m 38s and estimated cluster utilization 1.72%
 Executor count     8  (400%) estimated time 00m 38s and estimated cluster utilization 1.29%
 Executor count    10  (500%) estimated time 00m 38s and estimated cluster utilization 1.03%
~~~
以上Sparklens给出了，如果你申请N个Executor,你的Job的运行和资源使用效率情况。这个可以帮助我们更好的确定executor的申请数量。

2. Executor Vcore数量
~~~
App Level Wastage Metrics (Driver + Executor)
Executor OneCoreComputeHours available                  00h 00m
Executor OneCoreComputeHours used                       00h 00m        38.03%
OneCoreComputeHours wasted                              00h 00m        61.97%
~~~
以上我们可以分析是否需要增加或者减少Vcore申请数量，来优化Spark job的效率


3. Spark是否高效
这个高效不仅仅是你的资源使用情况，还可能和你本身Job的效率有关系
~~~
Time spent in Driver vs Executors
 Driver WallClock Time    00m 34s   86.89%
 Executor WallClock Time  00m 05s   13.11%
 Total WallClock Time     00m 40s
~~~
Spraklens建议最大限度的减小driver的wall clock时间和占比。我们都知道, 如果你的driver在busy的情况下，executor就没有在做任何实质性的工作。如果你的driver busy了一分钟，而你又10个executor，那么你的spark job就相当于有10分钟的浪费。Sparklens建议通过把更多的执行process放到executor上去做，尽量避免和减少driver的busy。

## 参考
1. **Sparklens blog**: https://www.qubole.com/blog/introducing-quboles-spark-tuning-tool/
2. **Sparklens github**: https://github.com/qubole/sparklens
