---
layout: post
title: "记一次Flink Checkpoint超时问题排查"
date: 2019-05-16
updated: 2019-05-16
description: 本文主要关于Flink Checkpoint超时问题排查，基于Flink1.7.1版本。
categories:
- BigData
tags:
- Flink
---
> 本文主要关于Flink Checkpoint超时问题排查过程，基于Flink1.7.1版本。  
  
最近在使用Flink进行流处理开发，开发时间不长，但是经历了很长时间的调优过程。最近发现Checkpoint经常出现超时失败的情况，所以对这一现象进行了排查。  
  
针对Flink Checkpoint超时问题，已经有一个大神最近刚刚写了关于这个的排查思路给了很多的参考，我也跟着他的思路，来进行进一步排查。参考文章如下:[Flink常见Checkpoint超时问题排查思路](https://www.jianshu.com/p/dff71581b63b)，这个文章最开始对于checkpoint超时做了分析，后来给出了一般的排查方式，我也是对照他的排查方式来查找问题的。下面文章我将对此问题排查过程进行一下记录。  
  
# Flink Checkpoint超时问题排查  
最近程序经常出现checkpoint超时失败的问题，因此去排查了一下问题，查找资料的时候看到了上文提到的文章，[Flink常见Checkpoint超时问题排查思路](https://www.jianshu.com/p/dff71581b63b)，在他最初的分析中指出“可能是因为Barrier对齐原因导致的checkpoint超时”，我就对这这个思路去排查了下问题。  
首先，可以看到下图checkpoint开始失败(这个图是后来再截的补充的，所以时间和ck ID与后续内容不一致，为了表明现象补充的截图，现象是一样)。  
![FlinkUICheckpoint开始失败](https://github.com/leafming/bak/blob/master/images/flink/2019-04-25-FlinkUICheckpoint超时现象.png?raw=true)  
点开看下详细信息，发现"narrowReduce->Sink:es"整体显示的n/a,但是它的上游"wideReduce-toNarrow"可以看出是479/480,可以看到是卡在了一个task上。  
![2019-05-16-checkpoint超时详细信息](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时详细信息.png?raw=true)   
点开show subtasks,可以看一下，卡在了编号97的subtask上。  
![2019-05-16-checkpointSubTask详细信息](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpointSubTask详细信息.png?raw=true)  
这里有个问题，看参考文章里写的是“这个id并不是subtaskIndex”，这里显示的97，但是我理解此97确实为subtaskIndex,对于subtaskIndex的描述我看了源码里写的是：“The task's index in the subtask group.”，对于此我的理解是，我们程序中没有划分subtask group，都是默认的“default”，因此暂时可以用此subtaskIndex标识一个subtask，但是如果程序中划分了group，那这个id就不准确了。因此后续查看了一下，executionId也就是通过metric采集到的taskAttemptId，看它的描述是：“The execution Id uniquely identifying the executed task represented by this metrics group.”，因此后续对于查看subtask维度的监控，可以使用taskAttemptId作为标识。  
因此，假设我们不能使用此subtaskIndex ID作为标示去查询此task的信息，我们就继续跟随参考文章的步骤，去查看下JobManager的日志，查看到日志如下：  
![2019-05-16-checkpoint超时jobmanager日志](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时jobmanager日志.png?raw=true)  
从上面日志截图可以看到这个checkpoint的一些延迟的信息，可以看到一个taskAttemptID，然后我们可以去监控上查下相关信息，我们拿“0d2b9593f56af1f5719a00bf4f6”这个taskAttemptId去查一下:  
![2019-05-16-checkpoint超时排查监控id关系](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时排查监控id关系.png?raw=true)  
正好看到，taskIndexId显示的为96，因为Flink UI上ID从1开始算的，监控采集的指标ID从0开始算，因此此task即为我们上面看到的卡住的task，可以看到taskName也为“wideReduce-toNarrow”。  
参考文章中有说明，Checkpoint时间要么花费在barrier对齐，要么花费在异步状态遍历和写hdfs。而我们的状态并不是很大，因此可以判断下是否是barrier没有对其，barrier下游无法对齐的主要原因还是在于下游消费能力不足，会导致buffer堆积一段时间，但这时并不足以造成上游反压，因为反压需要下游channel持续无法写入，导致tcp阻塞，导致上游的outputbuffer占满才会引起反压。  
因此，我们去查看一下flink对于buffer的几个指标。flink针对与task buffers metircs的监控有inputQueueLength（队列输入缓冲区的数量）、outputQueueLength（队列输出缓冲区的数量）、inPoolUsage（输入缓冲区使用情况评估）、outPoolUsage（输出缓冲区使用情况评估）。其中input
QueueLength（队列输入缓冲区的数量）我们来参考一下。那么就去看一下具体的监控，如下图：  
![2019-05-16-checkpoint超时排查inputQueuelength监控](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时排查inputQueuelength监控.png?raw=true)  
可以看到，的确在那段时间，对应subTask的inputQueuelength值达到了一条横线，也就是达到了一个最大值，判断inputQueuelength满了，此时发送barrier无法对其引起了Checkpoint超时。此时可以断定下游消费能力不足，但是是因为数据倾斜导致的还是因为下游其他原因阻塞，我们还不能轻易下结论，需要继续排查。不过我们可以确定的是，我们需要计算一个inputQueuelength的最大值怎么计算，便于以后更好的监控和告警，快速定位。对于inputQueuelength最大值如何计算，后续会单独整理一个文章来说明。  
为了判断是否是数据倾斜原因还是其他原因导致的问题，可以具体继续分析一下，我们再继续看一下我们采集的此算子每秒records的监控，如下图：  
![2019-05-16-checkpoint超时问题排查numrecordInperSecond监控](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时问题排查numrecordInperSecond监控.png?raw=true)  
可以看到，的确在那段时间，数据量非常大，而且我们也可以观察到，后续的15:55左右也是不同的subTask的inputQueueLength满了，也可以发现它们的每秒in的数据量较多，并且我们keyby选用的key是有每5min的时间戳的，所以根据in数据量也可以看得出，subtask差不多是每5min变化一次，就可以初步判断出，可能是因为出现了热点导致了数据倾斜，后续我们需要根据这个问题进行一次热点处理操作。  
此次的checkpoint超时问题就可以暂时排查到这里，后续需要进行热点处理操作，对于针对这次热点打散的相关文章，如果有时间也会整理一下。  
  
---
# 后续  
之前的checkpoint超时问题可以判断可能是因为热点问题导致的，因此先去做了打散操作处理了热点，基本解决了问题。后续其他流也出现了类似问题，但是此次问题不是因为数据倾斜的问题，具体排查过程再来一次如下：  
通过UI页面看到卡在“Sink：job”的subtaskIndex为1的subtask处，这次直接去看了InputQueueLength的监控，我们通过taskName、TaskAttemptID、TaskIndex 3个维度看一下监控如下：  
**inputQueuelength taskName维度**  
![2019-05-16-2inputQueuelengthtaskName维度](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-2inputQueuelengthtaskName维度.png?raw=true)  
**inputQueuelength TaskAttemptID维度**  
![2019-05-16-2inputQueuelengthTaskAttemptID维度](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-2inputQueuelengthTaskAttemptID维度.png?raw=true)  
**inputQueueLength SubTaskIndex维度**  
![2019-05-16-2inputQueueLengthSubTaskIndex维度](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-2inputQueueLengthSubTaskIndex维度.png?raw=true)  
通过上面三个图可以看出，这次的问题较之前不一样，这次可以发现是某个subTask的InputQueueLength先达到了一个“满”的状态，并且一直是这一个subTask，并没有变化的情况，所以我们可以具体登陆到那台host上去看一下问题，可以直接通过监控查询到具体哪台host：  
![2019-05-16-2inputQueueLength数据查看](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-2inputQueueLength数据查看.png?raw=true)  
可以看到上面三个图的taskName、subtaskIndex、TaskAttemptId正好对应上，那么我们去它所在的“...169”机器上去看一下具体情况。其他排查过程省略，后来我截取了一下堆栈信息：jstack -l id > billing，如下：  
![2019-05-16-inputQueueLength排查堆栈信息截取](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-inputQueueLength排查堆栈信息截取.png?raw=true)  
通过此内容可以发现卡在BulkRequestHandler的88行，88行内容如下：  
![2019-05-16-checkpoint超时es问题](https://github.com/leafming/bak/blob/master/images/flink/2019-05-16-checkpoint超时es问题.png?raw=true)  
可以判断写入es的时候发生了阻塞，我们现在使用的是es6具体后续排查优化一下，有时间的话再写一篇详细继续说明一下。  
    
---    
至此，本篇内容完成。  
