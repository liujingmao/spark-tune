## ###【重难点梳理】Spark资源参数调优



#### 1 Spark 任务提交

```shell
spark-shell  \
--master yarn-client  \
--num-executors 3 \
--executor-cores 2 \
--executor-memory 3g \
--driver-memory 10g \
--conf spark.executor.memoryOverhead=1024m
```

上面的spark任务， 申请了**3个executor**， 每个executor有2个core ，每个executor内存大小为3g，Driver内存的大小为10G。executor的堆外内存大小为**1024M**。

![图片描述](http://szimg.mukewang.com/62d3d36b09bca63c06140480.png)

Executor分布在节点Node2和Node3， Node2运行一个Executor， Node3运行两个Executor，每个Executor分配了两个CPU vCore，每个Executor的内存大小为3G，每个Executor的对外内存大小为1024MB。

Driver运行在Node1节点，占用内存大小为10G。

**常用参数说明**

- –num-executors： 启动的executor数量，即执行spakr计算任务启动的java jvm的数量， 默认是2个。

- –executor-memory：executor内存大小，默认1G。

- –executor-cores： 每个executor使用的内核数，默认为1。

- –driver-memory： Driver程序使用内存大小。

- –jars： 依赖的第三方jar包。

- –name: Application名称。

- –class: 主类名称，包含完整的包名加类名。

- –queue: 提交应用程序给哪个YARN的队列，默认是default队列。

- –conf PROP=VALUE: 任意的Spark属性， 例如，spark.executor.memoryOverhead=1024m 。

  

#### 2 Spark任务运行的核心组件

Spark任务运行的核心组件有三个：Driver、Executor和ApplicationMaster。

Driver： Spark的驱动节点，用于执行Spark的main方法。简单理解，Driver是驱动整个应用运行起来的程序。

Executor：Spark任务的执行节点。

ApplicationMaster：是Application的管理者，用于申请任务执行所需要的资源、跟踪任务的状态等。

这三个进程，在不同的提交模式下，有不同的分布

- Yarn Client模式提交任务
  Driver在任务提交的本地机器运行。
  ApplicationMaster在集群的某一台NodeManager节点运行。
  Executor在多台NodeManager节点运行。
- Yarn Cluster模式提交任务

Driver和ApplicationMaster是同一个进程， 在集群的某一台NodeManager节点运行。
Executor在多台NodeManager节点运行。

#### 3 Spark内存模型

我们来看下Spark的内存模型，从而更好的理解Spark的资源分配。我们以Spark的Executor为例，如下图所示：

![图片描述](http://szimg.mukewang.com/62d3d37909f85abf08470389.png)

Spark的Executor的Container内存有两大部分组成：堆外内存和堆内（Executor）内存。

堆外内存通过spark.executor.memoryOverhead参数设置，堆内内存通过参数spark.executor.memory（或者通过–executor-memory）设置， 如上图所示。

因此，**一个Executor进程（Executor Container）的内存=堆内内存+堆外内存**。

- 堆外内存(spark.executor.memoryOverhead)

主要用于JVM自身的开销、内部字符串以及其他原生的开销。 默认： MAX(executorMemory * 0.10, 384m)

- 堆内内存(spark.executor.memory)

用于聚合、排序等计算和缓冲相关的开销。

堆内内存分三部分组成：Execution内存 + Storage内存 + 其他内存。

- Execution: shuffle、排序、聚合等用于计算的内存
- Storage： 用于集群中缓冲和传播内部数据的内存（cache、广播变量）
- 其他内存： 用于用户的数据结构、Spark的元数据和预留防止OOM的内存

堆内内存有两个重要的参数：

- spark.memory.fraction

用于设置Execution和Storage内存在内存（这个内存是JVM的堆内存 - 300M， 这300M是预留内存）中的占比，默认是60%。即Execution和Storage内存大小之和占堆内存比例。

剩下的40%用于用户的数据结构、Spark的元数据和预留防止OOM的内存。

- spark.memory.storageFraction

表示Storage内存在Execution和Storage内存之和的占比。 设置这个参数可避免缓冲的数据块被清理出内存。

#### 4 堆外内存

我们以Spark的Executor进程为例， 介绍了Spark的内存模型。 Spark的Executor Container内存由堆内内存和堆外内存两部分组成。 Spark的Driver进程、ApplicationMaster进程与Executor类似，也是由堆外内存和堆内内存组成。

堆外内存对于任务运行是非常重要的，在任务运行时，通常都需要设置堆外内存。默认的堆外内存比较小， 不满足大数据量或长时间运行的任务。堆外内存不足将会导致运行任务的Container(也就是Executor)被Kill，错误信息类似于"Memory Overhead Exceeded"，当看到此类错误，就需要考虑增加堆外内存。

##### 4.1 Spark Executor的堆外内存

Spark Executor的堆外内存由两个参数控制：spark.executor.memoryOverhead和spark.executor.memoryOverheadFactor

1. spark.executor.memoryOverhead
       控制堆外内存的大小。默认值为：executorMemory * spark.executor.memoryOverheadFactor。 最小值为384MB。spark.executor.memoryOverheadFactor的默认值为0.1。
2. spark.executor.memoryOverheadFactor
       控制堆外内存的因子。默认值为0.1。但指定了spark.executor.memoryOverhead的大小，这个参数将被忽略。

##### 4.2 Driver的堆外内存

Spark Driver的堆外内存由两个参数控制：spark.driver.memoryOverhead和spark.driver.memoryOverheadFactor

1. spark.driver.memoryOverhead
       控制堆外内存的大小。默认值为：driverMemory* spark.driver.memoryOverheadFactor。 最小值为384MB。spark.driver.memoryOverheadFactor的默认值为0.1。
2. spark.driver.memoryOverheadFactor
       控制堆外内存的因子。默认值为0.1。但指定了spark.driver.memoryOverhead的大小，这个参数将被忽略。

##### 4.3 ApplicationMaster的堆外内存

在Yarn Cluster模式系， ApplicationMaster和Driver是同一个进程，因此堆外内存分片同Driver的堆外内存。

在Yarn Client模式下， ApplicationMaster和Driver是两个不同的进程， 因此， 针对ApplicationMaster的堆外内存，可通过参数spark.yarn.am.memoryOverhead进行设置。

spark.yarn.am.memoryOverhead的默认值：AM memory * 0.10, 最小值为384MB。AM即ApplicationMaster的缩写。 AM memory通过参数spark.yarn.am.memory设置（仅在Yarn Client模式）。

#### 5 Spark任务运行资源申请

```shell
spark-shell --master yarn \
--master yarn-client  \
--num-executors 4 \
--driver-memory 10g \
--executor-memory 3g \
--executor-cores 5
```

Yarn的管理界面，可以看到申请的资源情况：分配了5个Container， 21个Cpu Core， 15360M内存。

- 占用的CPU VCore数分析

VCore数量 = Executor的VCore数量 + AM的Vcore数量。其中，AM为ApplicationMaster。

Executor的VCore数量=4个Executor * 每个Executor的core数 = 4 * 5 = 20个。

AM的VCore数量 = 1。

因此，总的VCore数量 = 20 + 1 = 21个。

注意，这里没有计算Driver端的CPU VCore数量。

上面的任务是在yarn-client模式提交的任务， 请大家思考一个问题，在yarn-cluster模式， 占用的CPU Vcore是多少？

- 占用的Container数

占用的Container数 = 4个Excutor 进程 + 1个Applicatoin Master进程 = 5个Container

- 占用的内存分析

Excutor或者AM的Container内存： 堆外内存 + Excutor内存。

（1）、Excutor堆外内存

每个executor的堆外内存 = Max(executor-memory*0.10, 384) = Max(3g*0.10, 384) = 384MB。在Yarn分配内存时，以512MB的整数倍进行内存的扩容（通过规整因子的参数yarn.scheduler.increment-allocation-mb控制扩容的单位，此处yarn.scheduler.increment-allocation-mb=512MB），因此， 384向上规整为512MB。

4个Executor的堆外内存 = 4*512MB = 2048MB。

（2）、Excutor堆内内存

每个Executor Container的堆内内存大小 = 3G

4个Executor的Container的堆内内存大小 = 4 * 3G = 12288MB。

3）、AM的堆外内存
任务的提交参数并没有指定AM的内存， AM内存默认是512MB。
堆外内存 = Max(AM内存*0.10, 384) = Max(512MB*0.10, 384) = 384MB,根据规整因子512MB, 堆外内存取512MB。

（4）、AM的堆内内存
任务的提交参数并没有指定AM的内存， AM内存默认是512MB。

总的内存大小：Excutor的Container内存 + AM的Container内存 = Excutor堆外内存 + Excutor堆内内存 + AM的堆外内存 + AM的堆内内存 = 2048MB + 12288MB + 512MB + 512MB = 15360MB。

#### 6 Spark的资源分配调优

考虑如下的两个场景， Executor占用的资源均相同：Executor占用的CPU VCore的总数量为60， 占用的堆内内存为120G。
**场景1： **

```
--num-executors 1 --executor-cores 60 --executor-memory 120G
```

分配了1个executor，每个executor的内存为120G， CPU Vore为60。

**场景2：**

```plain
-num-executors 60 --executor-cores 1 --executor-memory 2G
```

分配了60个executor，每个executor的内存为2G， CPU Vore为60。

两种场景， Executor占用的资源相同，这两种分配方式都有各自的问题。

场景1的问题：单个executor内存过大，有120G。并且每个Executor的并行task太多，有60个。 存在两个问题：（1）、单个进程的内存过大， 将导致过多的垃圾回收， 影响性能。 （2）、并行task太多， 可能会造成资源的浪费。 比如，针对写HDFS数据源，一般5个task就可以实现全IO吞吐量，过多的task反而会导致吞吐量下降。

场景2的问题： executor数据量多， 但是每个executor的内存小，只有2G。 过多的executor数量导致启动的executor进程过多，启动executor开销大， 并且每个executor还需要分配堆外内存（最小384MB）。 此外， 广播变量是在每个executor上复制一次， 如果很多小的executor会导致更多的数据副本。

**资源分配建议：**

（1）、运行的一个executor内存过大通常会导致过多的垃圾回收延迟。对于单个executor来说，64GB是一个很好的上限。

（2）、executor的core数量保持在5个左右， 即每个executor可同时运行五个左右任务。