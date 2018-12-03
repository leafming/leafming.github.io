---
layout: post
title: "实时流处理系统反压机制（BackPressure）综述[转]"
date: 2018-11-15
updated: 2018-12-03
description: 本文主要关于实时流处理系统反压机制，最近看到反压问题看到此文章很好，在此分享并mark一下。
categories:
- BigData
tags:
- Spark
- Flink
- Storm
---
> 本文主要关于实时流处理系统反压机制，最近看到反压问题看到此文章很好，在此分享并mark一下。  
 (￢_￢)ﾉ最近菜叶子没自己写见谅。  
> 本文转自 [实时流处理系统反压机制（BackPressure）综述](https://blog.csdn.net/qq_21125183/article/details/80708142)  
> https://blog.csdn.net/qq_21125183/article/details/80708142  
> [开启Back Pressure使生产环境的Spark Streaming应用更稳定、有效](https://www.jianshu.com/p/28fcd51d4edd)  

反压机制（BackPressure）被广泛应用到实时流处理系统中，流处理系统需要能优雅地处理反压（backpressure）问题。反压通常产生于这样的场景：短时负载高峰导致系统接收数据的速率远高于它处理数据的速率。许多日常问题都会导致反压，例如，垃圾回收停顿可能会导致流入的数据快速堆积，或者遇到大促或秒杀活动导致流量陡增。反压如果不能得到正确的处理，可能会导致资源耗尽甚至系统崩溃。反压机制就是指系统能够自己检测到被阻塞的Operator，然后系统自适应地降低源头或者上游的发送速率。目前主流的流处理系统 Apache Storm、JStorm、Spark Streaming、S4、Apache Flink、Twitter Heron都采用反压机制解决这个问题，不过他们的实现各自不同。

![实时流处理系统反压机制01](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制01.png?raw=true)

不同的组件可以不同的速度执行（并且每个组件中的处理速度随时间改变）。 例如，考虑一个工作流程，或由于数据倾斜或任务调度而导致数据被处理十分缓慢。 在这种情况下，如果上游阶段不减速，将导致缓冲区建立长队列，或导致系统丢弃元组。 如果元组在中途丢弃，那么效率可能会有损失，因为已经为这些元组产生的计算被浪费了。并且在一些流处理系统中比如Strom，会将这些丢失的元组重新发送，这样会导致数据的一致性问题，并且还会导致某些Operator状态叠加。进而整个程序输出结果不准确。第二由于系统接收数据的速率是随着时间改变的，短时负载高峰导致系统接收数据的速率远高于它处理数据的速率的情况，也会导致Tuple在中途丢失。所以实时流处理系统必须能够解决发送速率远大于系统能处理速率这个问题，大多数实时流处理系统采用反压（BackPressure）机制解决这个问题。下面我们就来介绍一下不同的实时流处理系统采用的反压机制：

# Strom 反压机制

## Storm 1.0 以前的反压机制

对于开启了acker机制的storm程序，可以通过设置conf.setMaxSpoutPending参数来实现反压效果，**如果下游组件(bolt)处理速度跟不上导致spout发送的tuple没有及时确认的数超过了参数设定的值，spout会停止发送数据**，这种方式的缺点是很难调优conf.setMaxSpoutPending参数的设置以达到最好的反压效果，设小了会导致吞吐上不去，设大了会导致worker OOM；有震荡，数据流会处于一个颠簸状态，效果不如逐级反压；另外对于关闭acker机制的程序无效；

## Storm Automatic Backpressure

新的storm自动反压机制(Automatic Back Pressure)通过监控bolt中的接收队列的情况，当超过高水位值时专门的线程会将反压信息写到 Zookeeper ，Zookeeper上的watch会通知该拓扑的所有Worker都进入反压状态，最后Spout降低tuple发送的速度。

![实时流处理系统反压机制02](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制02.png?raw=true)

每个Executor都有一个接受队列和发送队列用来接收Tuple和发送Spout或者Bolt生成的Tuple元组。每个Worker进程都有一个单的的接收线程监听接收端口。它从每个网络上进来的消息发送到Executor的接收队列中。Executor接收队列存放Worker或者Worker内部其他Executor发过来的消息。Executor工作线程从接收队列中拿出数据，然后调用execute方法，发送Tuple到Executor的发送队列。Executor的发送线程从发送队列中获取消息，按照消息目的地址选择发送到Worker的传输队列中或者其他Executor的接收队列中。最后Worker的发送线程从传输队列中读取消息，然后将Tuple元组发送到网络中。

1. 当Worker进程中的Executor线程发现自己的接收队列满了时，也就是接收队列达到high watermark的阈值后，因此它会发送通知消息到背压线程。
2. 背压线程将当前worker进程的信息注册到Zookeeper的Znode节点中。具体路径就是 /Backpressure/topo1/wk1下
3. Zookeepre的Znode Watcher监视/Backpreesure/topo1下的节点目录变化情况，如果发现目录增加了znode节点说明或者其他变化。这就说明该Topo1需要反压控制，然后它会通知Topo1所有的Worker进入反压状态。
4. 最终Spout降低tuple发送的速度。

# JStorm 反压机制

 Jstorm做了两级的反压，第一级和Jstorm类似，通过执行队列来监测，但是不会通过ZK来协调，而是通过Topology Master来协调。在队列中会标记high water mark和low water mark，当执行队列超过high water mark时，就认为bolt来不及处理，则向TM发一条控制消息，上游开始减慢发送速率，直到下游低于low water mark时解除反压。

 此外，在Netty层也做了一级反压，由于每个Worker Task都有自己的发送和接收的缓冲区，可以对缓冲区设定限额、控制大小，如果spout数据量特别大，缓冲区填满会导致下游bolt的接收缓冲区填满，造成了反压。

![实时流处理系统反压机制03](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制03.png?raw=true)

限流机制：jstorm的限流机制， 当下游bolt发生阻塞时， 并且阻塞task的比例超过某个比例时（现在默认设置为0.1），触发反压

限流方式：计算阻塞Task的地方执行线程执行时间，Spout每发送一个tuple等待相应时间，然后讲这个时间发送给Spout，  于是， spout每发送一个tuple，就会等待这个执行时间。

Task阻塞判断方式：在jstorm 连续4次采样周期中采样，队列情况，当队列超过80%（可以设置）时，即可认为该task处在阻塞状态。

# SparkStreaming 反压机制

## 为什么引入反压机制Backpressure

默认情况下，Spark Streaming通过Receiver以生产者生产数据的速率接收数据，计算过程中会出现batch processing time > batch interval的情况，其中batch processing time 为实际计算一个批次花费时间， batch interval为Streaming应用设置的批处理间隔。这意味着Spark Streaming的数据接收速率高于Spark从队列中移除数据的速率，也就是数据处理能力低，在设置间隔内不能完全处理当前接收速率接收的数据。如果这种情况持续过长的时间，会造成数据在内存中堆积，导致Receiver所在Executor内存溢出等问题（如果设置StorageLevel包含disk, 则内存存放不下的数据会溢写至disk, 加大延迟）。  
Spark 1.5以前版本，用户如果要限制Receiver的数据接收速率，可以通过设置静态配制参数“spark.streaming.receiver.maxRate”的值来实现，此举虽然可以通过限制接收速率，来适配当前的处理能力，防止内存溢出，但也会引入其它问题。比如：producer数据生产高于maxRate，当前集群处理能力也高于maxRate，这就会造成资源利用率下降等问题。为了更好的协调数据接收速率与资源处理能力，Spark Streaming 从v1.5开始引入反压机制（back-pressure）,通过动态控制数据接收速率来适配集群数据处理能力。

## 反压机制Backpressure

Spark Streaming Backpressure:  根据JobScheduler反馈作业的执行信息来动态调整Receiver数据接收率。通过属性“spark.streaming.backpressure.enabled”来控制是否启用backpressure机制，默认值false，即不启用。
```
sparkConf.set("spark.streaming.backpressure.enabled",”true”)
```  

SparkStreaming 架构图如下所示: 

![实时流处理系统反压机制04](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制04.png?raw=true)

SparkStreaming 反压过程执行如下图所示：

在原架构的基础上加上一个新的组件RateController,这个组件负责监听“OnBatchCompleted”事件，然后从中抽取processingDelay 及schedulingDelay信息.  Estimator依据这些信息估算出最大处理速度（rate），最后由基于Receiver的Input Stream将rate通过ReceiverTracker与ReceiverSupervisorImpl转发给BlockGenerator（继承自RateLimiter）.

![实时流处理系统反压机制05](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制05.png?raw=true)
  
## direct模式-BackPressure(此部分详细说明了direct模式接收：转自-[开启Back Pressure使生产环境的Spark Streaming应用更稳定、有效](https://www.jianshu.com/p/28fcd51d4edd))  
当Spark Streaming与Kafka使用Direct API集群时，我们可以很方便的去控制最大数据摄入量--通过一个被称作spark.streaming.kafka.maxRatePerPartition的参数。根据文档描述，他的含义是：Direct API读取每一个Kafka partition数据的最大速率（每秒读取的消息量）。  
配置项spark.streaming.kafka.maxRatePerPartition，对防止流式应用在下边两种情况下出现流量过载时尤其重要：  
1.Kafka Topic中有大量未处理的消息，并且我们设置是Kafka auto.offset.reset参数值为smallest，他可以防止第一个批次出现数据流量过载情况。  
2.当Kafka 生产者突然飙升流量的时候，他可以防止批次处理出现数据流量过载情况。  
  
但是，配置Kafka每个partition每批次最大的摄入量是个静态值，也算是个缺点。随着时间的变化，在生产环境运行了一段时间的Spark Streaming应用，每批次每个Kafka partition摄入数据最大量的最优值也是变化的。有时候，是因为消息的大小会变，导致数据处理时间变化。有时候，是因为流计算所使用的多租户集群会变得非常繁忙，比如在白天时候，一些其他的数据应用（例如Impala/Hive/MR作业）竞争共享的系统资源时（CPU/内存/网络/磁盘IO）。  
背压机制可以解决该问题。背压机制是呼声比较高的功能，他允许根据前一批次数据的处理情况，动态、自动的调整后续数据的摄入量，这样的反馈回路使得我们可以应对流式应用流量波动的问题。  
Spark Streaming的背压机制是在Spark1.5版本引进的，我们可以添加如下代码启用改功能：  
```
sparkConf.set("spark.streaming.backpressure.enabled",”true”)
```  
那应用启动后的第一个批次流量怎么控制呢？因为他没有前面批次的数据处理时间，所以没有参考的数据去评估这一批次最优的摄入量。在Spark官方文档中有个被称作spark.streaming.backpressure.initialRate的配置，看起来是控制开启背压机制时初始化的摄入量。其实不然，该参数只对receiver模式起作用，并不适用于direct模式。推荐的方法是使用spark.streaming.kafka.maxRatePerPartition控制背压机制起作用前的第一批次数据的最大摄入量。我通常建议设置spark.streaming.kafka.maxRatePerPartition的值为最优估计值的1.5到2倍，让背压机制的算法去调整后续的值。请注意，spark.streaming.kafka.maxRatePerPartition的值会一直控制最大的摄入量，所以背压机制的算法值不会超过他。  
另一个需要注意的是，在第一个批次处理完成前，紧接着的批次都将使用spark.streaming.kafka.maxRatePerPartition的值作为摄入量。通过Spark UI可以看到，批次间隔为5s，当批次调度延迟31秒时候，前7个批次的摄入量是20条记录。直到第八个批次，背压机制起作用时，摄入量变为5条记录。  
  

# Heron 反压机制

![实时流处理系统反压机制06](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制06.png?raw=true)

当下游处理速度跟不上上游发送速度时，一旦StreamManager 发现一个或多个Heron Instance 速度变慢，立刻对本地spout进行降级，降低本地Spout发送速度, 停止从这些spout读取数据。并且受影响的StreamManager  会发送一个特殊的start backpressure message 给其他的StreamManager ，要求他们对spout进行本地降级。 当其他StreamManager  接收到这个特殊消息时，他们通过不读取当地Spout中的Tuple来进行降级。一旦出问题的Heron Instance 恢复速度后，本地的SM 会发送stop backpressure message 解除降级。

很多Socket Channel与应用程序级别的Buffer相关联，该缓冲区由high watermark 和low watermark组成。 当缓冲区大小达到high watermark时触发反压，并保持有效，直到缓冲区大小低于low watermark。 此设计的基本原理是防止拓扑在进入和退出背压缓解模式之间快速振荡。

# Flink 反压机制

 Flink 没有使用任何复杂的机制来解决反压问题，因为根本不需要那样的方案！它利用自身作为纯数据流引擎的优势来优雅地响应反压问题。下面我们会深入分析 Flink 是如何在 Task 之间传输数据的，以及数据流如何实现自然降速的。
 Flink 在运行时主要由 operators 和 streams 两大组件构成。每个 operator 会消费中间态的流，并在流上进行转换，然后生成新的流。对于 Flink 的网络机制一种形象的类比是，Flink 使用了高效有界的分布式阻塞队列，就像 Java 通用的阻塞队列（BlockingQueue）一样。还记得经典的线程间通信案例：生产者消费者模型吗？使用 BlockingQueue 的话，一个较慢的接受者会降低发送者的发送速率，因为一旦队列满了（有界队列）发送者会被阻塞。Flink 解决反压的方案就是这种感觉。
 在 Flink 中，这些分布式阻塞队列就是这些逻辑流，而队列容量是通过缓冲池来（LocalBufferPool）实现的。每个被生产和被消费的流都会被分配一个缓冲池。缓冲池管理着一组缓冲(Buffer)，缓冲在被消费后可以被回收循环利用。这很好理解：你从池子中拿走一个缓冲，填上数据，在数据消费完之后，又把缓冲还给池子，之后你可以再次使用它。

## Flink 网络传输中的内存管理

如下图所示展示了 Flink 在网络传输场景下的内存管理。网络上传输的数据会写到 Task 的 InputGate（IG） 中，经过 Task 的处理后，再由 Task 写到 ResultPartition（RS） 中。每个 Task 都包括了输入和输入，输入和输出的数据存在 Buffer 中（都是字节数据）。Buffer 是 MemorySegment 的包装类。

![实时流处理系统反压机制07](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制07.png?raw=true)

1. TaskManager（TM）在启动时，会先初始化NetworkEnvironment对象，TM 中所有与网络相关的东西都由该类来管理（如 Netty 连接），其中就包括NetworkBufferPool。根据配置，Flink 会在 NetworkBufferPool 中生成一定数量（默认2048个）的内存块 MemorySegment（关于 Flink 的内存管理，后续文章会详细谈到），内存块的总数量就代表了网络传输中所有可用的内存。NetworkEnvironment 和 NetworkBufferPool 是 Task 之间共享的，每个 TM 只会实例化一个。
2. Task 线程启动时，会向 NetworkEnvironment 注册，NetworkEnvironment 会为 Task 的 InputGate（IG）和 ResultPartition（RP） 分别创建一个 LocalBufferPool（缓冲池）并设置可申请的 MemorySegment（内存块）数量。IG 对应的缓冲池初始的内存块数量与 IG 中 InputChannel 数量一致，RP 对应的缓冲池初始的内存块数量与 RP 中的 ResultSubpartition 数量一致。不过，每当创建或销毁缓冲池时，NetworkBufferPool 会计算剩余空闲的内存块数量，并平均分配给已创建的缓冲池。注意，这个过程只是指定了缓冲池所能使用的内存块数量，并没有真正分配内存块，只有当需要时才分配。为什么要动态地为缓冲池扩容呢？因为内存越多，意味着系统可以更轻松地应对瞬时压力（如GC），不会频繁地进入反压状态，所以我们要利用起那部分闲置的内存块。
3. 在 Task 线程执行过程中，当 Netty 接收端收到数据时，为了将 Netty 中的数据拷贝到 Task 中，InputChannel（实际是 RemoteInputChannel）会向其对应的缓冲池申请内存块（上图中的①）。如果缓冲池中也没有可用的内存块且已申请的数量还没到池子上限，则会向 NetworkBufferPool 申请内存块（上图中的②）并交给 InputChannel 填上数据（上图中的③和④）。如果缓冲池已申请的数量达到上限了呢？或者 NetworkBufferPool 也没有可用内存块了呢？这时候，Task 的 Netty Channel 会暂停读取，上游的发送端会立即响应停止发送，拓扑会进入反压状态。当 Task 线程写数据到 ResultPartition 时，也会向缓冲池请求内存块，如果没有可用内存块时，会阻塞在请求内存块的地方，达到暂停写入的目的。
4. 当一个内存块被消费完成之后（在输入端是指内存块中的字节被反序列化成对象了，在输出端是指内存块中的字节写入到 Netty Channel 了），会调用 Buffer.recycle() 方法，会将内存块还给 LocalBufferPool （上图中的⑤）。如果LocalBufferPool中当前申请的数量超过了池子容量（由于上文提到的动态容量，由于新注册的 Task 导致该池子容量变小），则LocalBufferPool会将该内存块回收给 NetworkBufferPool（上图中的⑥）。如果没超过池子容量，则会继续留在池子中，减少反复申请的开销。  

## Flink 反压机制

下面这张图简单展示了两个 Task 之间的数据传输以及 Flink 如何感知到反压的：

![实时流处理系统反压机制08](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制08.png?raw=true)

1. 记录“A”进入了 Flink 并且被 Task 1 处理。（这里省略了 Netty 接收、反序列化等过程）
2. 记录被序列化到 buffer 中。
3. 该 buffer 被发送到 Task 2，然后 Task 2 从这个 buffer 中读出记录。

**不要忘了：记录能被 Flink 处理的前提是，必须有空闲可用的 Buffer。**

结合上面两张图看：Task 1 在输出端有一个相关联的 LocalBufferPool（称缓冲池1），Task 2 在输入端也有一个相关联的 LocalBufferPool（称缓冲池2）。如果缓冲池1中有空闲可用的 buffer 来序列化记录 “A”，我们就序列化并发送该 buffer。

这里我们需要注意两个场景：

- 本地传输：如果 Task 1 和 Task 2 运行在同一个 worker 节点（TaskManager），该 buffer 可以直接交给下一个 Task。一旦 Task 2 消费了该 buffer，则该 buffer 会被缓冲池1回收。如果 Task 2 的速度比 1 慢，那么 buffer 回收的速度就会赶不上 Task 1 取 buffer 的速度，导致缓冲池1无可用的 buffer，Task 1 等待在可用的 buffer 上。最终形成 Task 1 的降速。
- 远程传输：如果 Task 1 和 Task 2 运行在不同的 worker 节点上，那么 buffer 会在发送到网络（TCP Channel）后被回收。在接收端，会从 LocalBufferPool 中申请 buffer，然后拷贝网络中的数据到 buffer 中。如果没有可用的 buffer，会停止从 TCP 连接中读取数据。在输出端，通过 Netty 的水位值机制来保证不往网络中写入太多数据（后面会说）。如果网络中的数据（Netty输出缓冲中的字节数）超过了高水位值，我们会等到其降到低水位值以下才继续写入数据。这保证了网络中不会有太多的数据。如果接收端停止消费网络中的数据（由于接收端缓冲池没有可用 buffer），网络中的缓冲数据就会堆积，那么发送端也会暂停发送。另外，这会使得发送端的缓冲池得不到回收，writer 阻塞在向 LocalBufferPool 请求 buffer，阻塞了 writer 往 ResultSubPartition 写数据。

这种固定大小缓冲池就像阻塞队列一样，保证了 Flink 有一套健壮的反压机制，使得 Task 生产数据的速度不会快于消费的速度。我们上面描述的这个方案可以从两个 Task 之间的数据传输自然地扩展到更复杂的 pipeline 中，保证反压机制可以扩散到整个 pipeline。

## 反压实验

另外，官方博客中为了展示反压的效果，给出了一个简单的实验。下面这张图显示了：随着时间的改变，生产者（黄色线）和消费者（绿色线）每5秒的平均吞吐与最大吞吐（在单一JVM中每秒达到8百万条记录）的百分比。我们通过衡量task每5秒钟处理的记录数来衡量平均吞吐。该实验运行在单 JVM 中，不过使用了完整的 Flink 功能栈。

![实时流处理系统反压机制09](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制09.png?raw=true)

首先，我们运行生产task到它最大生产速度的60%（我们通过Thread.sleep()来模拟降速）。消费者以同样的速度处理数据。然后，我们将消费task的速度降至其最高速度的30%。你就会看到背压问题产生了，正如我们所见，生产者的速度也自然降至其最高速度的30%。接着，停止消费task的人为降速，之后生产者和消费者task都达到了其最大的吞吐。接下来，我们再次将消费者的速度降至30%，pipeline给出了立即响应：生产者的速度也被自动降至30%。最后，我们再次停止限速，两个task也再次恢复100%的速度。总而言之，我们可以看到：生产者和消费者在 pipeline 中的处理都在跟随彼此的吞吐而进行适当的调整，这就是我们希望看到的反压的效果。

## Flink 反压监控

在 Storm/JStorm 中，只要监控到队列满了，就可以记录下拓扑进入反压了。但是 Flink 的反压太过于天然了，导致我们无法简单地通过监控队列来监控反压状态。Flink 在这里使用了一个 trick 来实现对反压的监控。如果一个 Task 因为反压而降速了，那么它会卡在向 LocalBufferPool 申请内存块上。那么这时候，该 Task 的 stack trace 就会长下面这样：

```java
java.lang.Object.wait(Native Method)
o.a.f.[...].LocalBufferPool.requestBuffer(LocalBufferPool.java:163)
o.a.f.[...].LocalBufferPool.requestBufferBlocking(LocalBufferPool.java:133) <--- BLOCKING request
[...]
```

那么事情就简单了。通过不断地采样每个 task 的 stack trace 就可以实现反压监控。

![实时流处理系统反压机制10](https://github.com/leafming/bak/blob/master/images/bigdata-total/2018-11-15-实时流处理系统反压机制10.png?raw=true)

Flink 的实现中，只有当 Web 页面切换到某个 Job 的 Backpressure 页面，才会对这个 Job 触发反压检测，因为反压检测还是挺昂贵的。JobManager 会通过 Akka 给每个 TaskManager 发送TriggerStackTraceSample消息。默认情况下，TaskManager 会触发100次 stack trace 采样，每次间隔 50ms（也就是说一次反压检测至少要等待5秒钟）。并将这 100 次采样的结果返回给 JobManager，由 JobManager 来计算反压比率（反压出现的次数/采样的次数），最终展现在 UI 上。UI 刷新的默认周期是一分钟，目的是不对 TaskManager 造成太大的负担。

# 总结  
  
Flink不需要一种特殊的机制来处理反压，因为Flink 中的数据传输相当于已经提供了应对反压的机制。因此，Flink 所能获得的最大吞吐量由其 pipeline 中最慢的组件决定。相对于 Storm/JStorm 的实现，Flink 的实现更为简洁优雅，源码中也看不见与反压相关的代码，无需 Zookeeper/TopologyMaster 的参与也降低了系统的负载，也利于对反压更迅速的响应。
  
> 本文转自 [实时流处理系统反压机制（BackPressure）综述](https://blog.csdn.net/qq_21125183/article/details/80708142)  
> https://blog.csdn.net/qq_21125183/article/details/80708142  
> [开启Back Pressure使生产环境的Spark Streaming应用更稳定、有效](https://www.jianshu.com/p/28fcd51d4edd)  

---    
至此，本篇内容完成。  
