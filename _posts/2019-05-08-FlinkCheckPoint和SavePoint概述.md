---
layout: post
title: "Flink中Checkpoint和Savepoint相关概念"
date: 2019-05-08
updated: 2019-05-08
description: 本文主要关于Flink中Checkpoint和Savepoint相关概念，基于Flink1.7.1版本。
categories:
- BigData
tags:
- Flink
---
> 本文主要关于Flink中Checkpoint和Savepoint相关概念，基于Flink1.7.1。  
  
# 基础概念一览-Checkpoint和Savepoint的区别  
对于Flink容错保障中，Checkpoint和Savepoint是两个关键的概念，注意区分和应用。  
### 使用场景  
Checkpoint仅用于恢复意外失败的作业（走flink的job默认恢复机制）。  
SavePoint用于用户计划的、手动的备份和恢复（代码版本更新、更改并行度、jog graph、2个程序同时运行red/blue deployment）。  
### 保留时间  
Checkpoint默认情况下是作业终止会被删除可配置保留）；Savepoint作业终止后继续存在。  
### 设计目标  
Checkpoint设计目标：轻量级创建；快速恢复。  
SavePoint：生成和恢复成本更高，关注可移植性和对先前提到的作业更改的支持。  
### 维护  
Checkpoint由flink自动维护（ created, owned, and released by Flink），Savepoint需用户维护（ created, owned, and released by user）。  
### 其他  
存储的数据格式与state backend密切相关；使用RocksDB state backend，Checkpoint支持增量存储（轻量）；Savepoint是全量Snapshot。  
  
# Checkpoint  
## Checkpoint配置例子  
```scala  
val env: StreamExecutionEnvironment = if (mode == "local") {
  StreamExecutionEnvironment.createLocalEnvironment()
} else {
  StreamExecutionEnvironment.getExecutionEnvironment
}
var stateBackend: RocksDBStateBackend = new RocksDBStateBackend(checkpointDir, false)
val rocksdbConf = new Configuration
rocksdbConf.setString(RocksDBOptions.TIMER_SERVICE_FACTORY, RocksDBStateBackend.PriorityQueueStateType.ROCKSDB.toString)
stateBackend.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)
stateBackend = stateBackend.configure(rocksdbConf)
env.setStateBackend(stateBackend)
env.enableCheckpointing(triggerInterval * 1000)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.AT_LEAST_ONCE)
env.getCheckpointConfig.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION)
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(3000)
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
```  
**说明**:  
通过env.setStateBackend(stateBackend)，设置使用的state backend。  
通过env.enableCheckpointing开启checkpoint并设置时间间隔为triggerInterval * 1000。  
通过ExternalizedCheckpointCleanup配置checkpoints在job取消的时候是否清除：ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION表示一旦Flink处理程序被cancel后，会保留Checkpoint数据，以便后续需要的话根据实际需要恢复到指定的Checkpoint处理，需要手动清除checkpoint。ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION表示默认的，当job停止后删除checkpoint数据。  
  
**其他配置**:  
默认情况下，如果设置了Checkpoint选项，则Flink只保留最近成功生成的1个Checkpoint，由于Flink的job恢复机制只需要使用最新一个有效的checkpoint，因此在新的checkpoint生成后就可以安全移除其余旧的checkpoint了，当Flink程序失败时，可以从最近的这个Checkpoint来进行恢复。  
如果需要保留多个Checkpoint，需要在Flink的配置文件conf/flink-conf.yaml中，添加如下配置，指定最多需要保存Checkpoint的个数，这样在HDFS的相应文件夹下面会产生多个checkpoint文件：  
```  
state.checkpoints.num-retained: 20
```
  
## 从checkpoint恢复  
checkpoint由元数据文件、额外的数据文件（与state backend相关）组成。在保留checkpoint的情况下，使用命令:  
```  
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
```  
从checkpoint的元数据文件进行数据恢复，注意若元数据文件中信息不够，那么jobmanager就需要使用相关的数据文件来恢复作业。  
例如：bin/flink run -s hdfs://checkpoint目录/JobID/chk-21(checkpint点)/_metadata 其他运行命令，成功启动的job对应的Checkpoint编号会从该次运行基于的编号继续连续生成：chk-22、chk-23、chk-24等。  
  
# SavePoint  
avepoint使用Flink的Checkpoint机制创建流作业的全量（非增量）状态快照，包含Streaming程序的状态，并且将checkpoint数据和元数据写出到外部文件系统。  
## 算子UID  
Flink程序中包含两种状态数据，一种是用户定义的状态（User-defined State），他们是基于Flink的Transformation函数来创建或者修改得到的状态数据；另一种是系统状态（System State），他们是指作为Operator计算一部分的数据Buffer等状态数据，比如在使用Window Function时，在Window内部缓存Streaming数据记录。为了能够在创建Savepoint过程中，唯一识别对应的Operator的状态数据，Flink提供了API来为程序中每个Operator设置ID，这样可以在后续更新/升级程序的时候，可以在Savepoint数据中基于Operator ID来与对应的状态信息进行匹配，从而实现恢复。  
当然，如果我们不指定Operator ID，Flink也会我们自动生成对应的Operator状态ID。自动生成的ID依赖于程序的结构，并且非常容易受到程序变化的影响。因此，强烈推荐手动指定ID。手动为每个Operator设置ID，即使未来Flink应用程序可能会改动很大，比如替换原来的Operator实现、增加新的Operator、删除Operator等等，至少我们有可能与Savepoint中存储的Operator状态对应上。另外，保存的Savepoint状态数据，毕竟是基于当时程序及其内存数据结构生成的，所以如果未来Flink程序改动比较大，尤其是对应的需要操作的内存数据结构都变化了，可能根本就无法从原来旧的Savepoint正确地恢复。  
**官网设置例子**：  
```scala
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id") // ID for the source operator
  .shuffle()
  // Stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id") // ID for the mapper
  // Stateless printing sink
  .print(); // Auto-generated ID
```  
## Savepoint state  
可以将savepoint想象成为保存了每个有状态的算子的“算子ID->状态”映射的集合。  
Operator ID | State  
------------+------------------------  
source-id   | State of StatefulSource  
mapper-id   | State of StatefulMapper  
在上述示例中，print结果表是无状态的，因此不是savepoint状态的一部分。默认情况下，我们试图将savepoint的每条数据，都映射到新的程序中。  
  
## Savepoint操作  
可以通过命令行操作Savepoint，Flink版本>=1.2.0可以使用 webui 从savepoint恢复作业 。  
### 创建一个Savepoint  
创建一个Savepoint，需要指定对应Savepoint基础目录，此job对应的新的savepoint目录将在其内被创建并存储数据和元数据。  
有两种方式来指定Savepoint目录:  
1. 配置Savepoint的默认路径，在Flink的配置文件conf/flink-conf.yaml中，添加如下配置：state.savepoints.dir:hdfs://hdfscluster/flink-savepoints  
2. 在手动执行savepoint命令的时候，指定Savepoint存储目录，命令格式如下所示：bin/flink savepoint :jobId [:targetDirectory]  
  
如果你既不配置默认目标目录，也不指定自定义目标目录，触发savepoint将失败。  
注意： 目标目录位置必须可以被JobManager和TaskManager访问，比如位于分布式文件系统中。  
**例子**：  
例如，正在运行的Flink Job对应的ID为，使用默认state.savepoints.dir配置指定的Savepoint目录，执行如下命令:bin/flink savepoint 40be7c0160b73041dbceafc6256cb37d,可以看到，在目录hdfs://hdfscluster/flink-savepoints/savepoint-40be7c-8201276a3c10下面生成了ID为40be7c0160b73041dbceafc6256cb37d的Job的Savepoint数据。  
为正在运行的Flink Job指定一个目录存储Savepoint数据，执行如下命令:bin/flink savepoint 40be7c0160b73041dbceafc6256cb37d hdfs://hdfscluster/tmp/flink/savepoints,可以看到，在目录 hdfs://hdfscluster/tmp/flink/savepoints/savepoint-40be7c-c19253t1b376下面生成了ID为40be7c0160b73041dbceafc6256cb37d的Job的Savepoint数据。  
#### YARN环境中触发Savepoint  
```
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId  
```  
这将为ID为:yarnAppId的YARN应用中，为ID为:jobId的作业触发一份savepoint，并返回savepoint的路径。  
#### 取消作业时触发savepoint  
```  
$ bin/flink cancel -s [:targetDirectory] :jobId  
```  
这将为ID为:jobId的作业自动触发一份savepoint，并取消作业。此外，你可以指定目标目录用于保存savepoint。此目录需要可以被JobManager和TaskManager访问。  
  
### savepoint目录结构  
以FsStateBackend或RocksDBStateBackend为例：  
```  
# Savepoint target directory
# savepoint目标目录
/savepoints/
# Savepoint directory
# savepoint目录
/savepoints/savepoint-:shortjobid-:savepointid/
# Savepoint file contains the checkpoint meta data
# savepoint元数据文件
/savepoints/savepoint-:shortjobid-:savepointid/_metadata
# Savepoint state
# savepoint状态
/savepoints/savepoint-:shortjobid-:savepointid/...
```  
**注意**： 虽然看起来savepoint可能被移动，但是由于_metadata文件中的绝对路径，目前这是不可能的。如果你使用MemoryStateBackend，元数据和savepoint状态将被存储于_metadata文件。因为它是自完备的，你可以移动此文件并从任何位置恢复。  
**例子**：  
```  
hdfs dfs -ls /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b
Found 5 items
-rw-r--r--   3 hadoop supergroup       4935 2018-09-02 01:21 /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b/50231e5f-1d05-435f-b288-06d5946407d6
-rw-r--r--   3 hadoop supergroup       4599 2018-09-02 01:21 /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b/7a025ad8-207c-47b6-9cab-c13938939159
-rw-r--r--   3 hadoop supergroup       4976 2018-09-02 01:21 /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b/_metadata
-rw-r--r--   3 hadoop supergroup       4348 2018-09-02 01:21 /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b/bd9b0849-aad2-4dd4-a5e0-89297718a13c
-rw-r--r--   3 hadoop supergroup       4724 2018-09-02 01:21 /flink-1.5.3/flink-savepoints/savepoint-11bbc5-bd967f90709b/be8c1370-d10c-476f-bfe1-dd0c0e7d498a
```  
如上面列出的HDFS路径中，11bbc5是Flink Job ID字符串前6个字符，后面bd967f90709b是随机生成的字符串，然后savepoint-11bbc5-bd967f90709b作为存储此次Savepoint数据的根目录，最后savepoint-11bbc5-bd967f90709b目录下面_metadata文件包含了Savepoint的元数据信息，其中序列化包含了savepoint-11bbc5-bd967f90709b目录下面其它文件的路径，这些文件内容都是序列化的状态信息。  
  
### 从savepoint恢复  
```  
$ bin/flink run -s :savepointPath [:runArgs]
```  
这将提交作业并从指定的savepoint目录恢复作业。你可以指定savepoint的目录或_metadata 文件路径。  
#### 允许不恢复状态-Allowing Non-Restored State  
默认情况下，恢复操作将试图将savepoint中所有状态条目映射到你计划恢复的程序中。如果你删除了一个算子，通过--allowNonRestoredState(简写：-n)参数，将允许忽略某个不能映射到新程序的状态：  
```  
$ bin/flink run -s :savepointPath -n [:runArgs]
```  
  
### 删除savepoint  
```  
$ bin/flink savepoint -d :savepointPath
```  
这将删除存储于:savepointPath的savepoint。  
注意，同样可以通过常规的文件系统操作手动的删除savepoint，而不影响其他的savepoint或checkpoint（回想下每个savepoint都是自完备的）。在FLink 1.2版本之前，执行以上savepoint命令曾是一个更繁琐的任务。  
  
## Savepoint FAQ  
### 是否需要为我作业中的所有算子都分配ID？  
根据经验，是的。严格来说，在作业中通过uid方法仅仅为那些有状态的算子分配ID更高效。savepoint仅仅包含这些算子的状态，而无状态算子则不是savepoint的一部分。  
在实践中，推荐为所有算子分配ID，因为某些Flink的内置算子如窗口同样是有状态的，但是哪些算子实际上是有状态的、哪些是没有状态的却并不明显。如果你非常确定某个算子是无状态的，你可以省略uid方法。  
### 如果在我的作业中增加一个需要状态的新算子将会发生什么？  
当你在作业中增加一个新算子，它将被初始化为无任何状态。savepoint包含每个有状态算子的状态。无状态算子不是savepoint的一部分。新算子表现为类似于无状态算子。  
### 如果从作业中删除一个有状态算子将会发生什么？  
默认情况下，从savepoint恢复时作业时，将试图匹配savepoint中所有operatorid和state条目。如果你的新作业将某个有状态的算子删除了，那么从savepoint恢复作业会失败。  
你可以通过使用--allowNonRestoredState (简写：-n)参数运行命令来允许这种情况下对作业恢复状态。  
$ bin/flink run -s :savepointPath -n [:runArgs]  
### 如果在我的作业中重排序（reorder）有状态的算子将会发生什么？  
如果你为这些算子分配了ID，它们将正常的被恢复。  
如果你没有分配ID，在重排序后，有状态算子自动生成的ID很可能改变。这将可能导致你无法从之前的savepoint中恢复。  
### 如果在我的作业中增加、删除或重排序无状态的算子将会发生什么？  
如果你为有状态算子分配了ID，在savepoint恢复时无状态算子将不受影响。  
如果你没有分配ID，在重排序后，这些有状态算子自动生成的ID很可能改变。这将可能导致你无法从之前的savepoint中恢复。  
### 在恢复时当我改变程序的并发时将会发生什么？  
如果savepoint是在Flink 1.2.0及以上版本触发的，并且没有使用deprecated state API如Checkpointed，你可以指定新的并发并从savepoint恢复程序。  
如果作业要从低于flink 1.2.0版本触发的savepoint中恢复，或使用了已经deprecated APIs，则在改并发之前，必须将作业和savepoint迁移到Flink 1.2.0及以上版本。  
更新详见[Upgrading Applications and Flink Versions](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/upgrading.html)。  
### 我可以在稳定存储上移动保存点文件吗？  
这个问题的快速答案目前是“否”，因为元数据文件出于技术原因将稳定存储中的文件作为绝对路径引用。详细回答答案是：如果出于某种原因必须移动文件，有两种可能的解决方法。首先，更简单但可能更危险，您可以使用编辑器在元数据文件中查找旧路径，并将其替换为新路径。其次，可以使用 class SavepointV2Serializer作为以编程方式读取、操作和用新路径重写元数据文件的起点。  
  
  
本文参考:  
[flink使用checkpoint等知识](https://www.jianshu.com/p/8e74c7cdd463)  
[flink官网以及相关官网翻译](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/savepoints.html)  
  
  
---    
至此，本篇内容完成。  
