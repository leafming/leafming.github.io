---
layout: post
title: "SparkStreaming输出数据到Kafka--Kafka连接池的使用"
date: 2018-04-02
description: 本文关于在使用Spark Streaming输出数据到Kafka的过程中，使用Kafka连接池来提高输出效率。
categories: 
- BigData
tags: 
- Spark
- Kafka
---

> 最近把spark实时流处理的信令处理程序，由原来的向kafka0.8.2.1中写入数据，改成了向带ACL权限认证的kafka0.10.1.1中写入数据，因此在之前的基础上创建连接前会多一个认证过程，因此导致写入效率有些低下，所以使用Kafka连接池来优化之前程序。  
> **注意: 本文中使用的版本是spark2.2.1和kafka0.10.1.1**   

# 原写入方式描述
使用Spark Streaming从Kafka中接收数据时，可以使用Spark提供的统一的接口来接收数据，但是写入数据的时候，并没有spark官方提供的写入接口，需要自己写使用底层的kafka方法，使用producer写入。  
下面是原写入方式的程序示例:  
## 第一部分：Spark Streaming主程序部分  
```scala
    writeDStrem.foreachRDD(rdd => {
      if (rdd.isEmpty) {
        logInfo(" No Data in this batchInterval --------")
      } else {
        val start_time = System.currentTimeMillis()
        // 不能在这里创建KafkaProducer
	rdd.foreachPartition(iter=>{
          process.writeAsPartitionToKafka(proKafkaTopicName,iter,cacheNum,props)
        })
        competeTime(start_time, "Processed data write to KAFKA")
      }
    })
```
在每个partation中调用ProcessData类中的writeAsPartitionToKafka方法来向kafka写入数据。
> 特别说明：这里是通过调用ProcessData类中的方法，在ProcessData类中的方法中创建KafkaProducer来向kafka里写入的，如果直接写创建KafkaProducer，不能把将KafkaProducer的创建放在foreachPartition外边，因为KafkaProducer是不可序列化的（not serializable）。  

## 第二部分：主程序在rdd.foreachPartition中调用以下类中的方法把数据写入kafka中  
```scala  
    final class ProcessData extends Logging{
      /**
        * 数据发送
        * @param topic
        * @param messages
        * @param props
        */
      def sendMessages(topic: String, messages: String, props: Properties) {
        System.setProperty("java.security.auth.login.config","kafka_client_jaas.conf")
        val producer: Producer[String, String] = new KafkaProducer[String, String](props)
        try{
          val msg: ProducerRecord[String, String] = new ProducerRecord[String,String](topic,messages)
          producer.send(msg)
        }catch{
          case e: IOException =>
            logError("partition write data to  kafka exception : " + e.getMessage + "\n")
        } finally {
          producer.close
        }
      }
      /**
        * 处理数据，按N条写入一次
        * @param topicName
        * @param iter
        * @param cache 缓存数
        * @param props
        * @return
        */
      def writeAsPartitionToKafka(topicName: String,iter: Iterator[String], cache: Int,props:Properties): Try[Unit] = Try {
        var record_sum=""   //初始化空串
        var count_sum=0     //计数器
        var record=""
        while (iter.hasNext) {
          record = iter.next()
          record_sum += record+"\n"
          count_sum = count_sum + 1
          if (count_sum == cache) {
            sendMessages(topicName,record_sum,props)
            record_sum = ""
            count_sum = 0
          }
        }
        if (!record_sum.isEmpty) {
          sendMessages(topicName,record_sum,props)
        }
      }
    }
```  
此类中主要2个方法，writeAsPartitionToKafka是用于处理数据，sendMessages用来创建KafkaProducer来向kafka生产消息。  
writeAsPartitionToKafka方法为了“提高”写入效率，设置了一个cacheNum值，这个方法会将每个patation中的数据以cacheNum条和并成一条数据来向Kafka中写入，因此是cacheNum条数据以“\n”来分割合并成一条在Kafka里的消息，所以对下游数据处理也造成了麻烦。  
并且使用此方式，相当于对于每个partation的每cacheNum条记录（即每次调用sendMessages方法，发送1条kafka消息）都需要创建KafkaProducer，然后再利用producer进行输出操作
。还是需要较频繁的建立连接，因此使用kafka连接池来更改程序。   
  
# 新写入方式描述:使用Kafka连接池更改程序  
## 第一步：包装KafkaProducer-创建Kafka连接池  
创建class KafkaPool 以及object KafkaPool，将KafkaProducer以lazy val的方式进行包装。
```scala  
    import java.util.concurrent.Future
    import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord, RecordMetadata}
    //scala中，类名之后的括号中是构造函数的参数列表，() =>是传值传参，KafkaProducer
    class KafkaPool[K, V]( createProducer: () => KafkaProducer[K,V])  extends Serializable{
      //使用lazy关键字修饰变量后，只有在使用该变量时，才会调用其实例化方法
      //后续在spark主程序中使用时，将kafkapool广播出去到每个executor里面了，然后到每个executor中，当用到的时候，会实例化一个producer，这样就不会有NotSerializableExceptions的问题了。
      lazy val producer = createProducer()
      def send(topic: String, key: K, value: V): Future[RecordMetadata] = 
        producer.send(new ProducerRecord[K, V](topic, key, value))
      def send(topic: String, value: V): Future[RecordMetadata] = 
        producer.send(new ProducerRecord[K, V](topic, value))
    }
    object KafkaPool{
      import scala.collection.JavaConversions._
      def apply[K, V](config: Map[String, Object]): KafkaPool[K, V] = {
          val createProducerFunc = () => {
            //kafka权限认证
            System.setProperty("java.security.auth.login.config","kafka_client_jaas.conf")
            val producer = new KafkaProducer[K, V](config)
            sys.addShutdownHook {
	      //当发送ececutor中的jvm shutdown时，kafka能够将缓冲区的消息发送出去。
              producer.close()
            }
            producer
          }
          new KafkaPool(createProducerFunc)
      }
      def apply[K, V](config: java.util.Properties): KafkaPool[K, V] = apply(config.toMap)
    }
```    
> 补充说明：  
> 传值传参和传名的区别（()=>和:=>的区别）
> * 传值
> ```scala  
>     def test1(code: ()=>Unit){
>       println("start")
>       code() //要想调用传入的代码块，必须写成code()，否则不会调用。  
>       println("end")
>     }
>     test1 {//此代码块，传入后立即执行。  
>       println("when evaluated") //传入就打印
>       ()=>{println("bb")} // 执行code()才打印
>     }
> ```
> 结果:   
> when evaluated   
> start  
> bb  
> end  
> * 传名
> ```scala
>     def test2(code : => Unit){  
>       println("start")  
>       code // 这行才会调用传入的代码块，写成code()亦可  
>       println("end")  
>     }  
>     test2{// 此处的代码块不会马上被调用  
>       println("when evaluated")  //执行code的时候才调用
>       println("bb") //执行code的时候才调用  
>     }
> ```
> 结果:  
> start  
> when evaluated  
> bb  
> end  
 
## 第二步：利用广播变量下发KafkaProducer    
利用广播变量，给每一个executor自己的KafkaProducer，将KafkaProducer广播到每一个executor中。    
**注意：这里暂留一个问题，此种方式只可以Spark Streaming程序不用checkpoint的时候使用，否则，如果程序中断而重启程序，广播变量无法从checkpoint中恢复，会出现 `“java.lang.ClassCastException:B cannot be cast to KafkaPool”` 的问题，具体解决方式见下篇文章([SparkStreaming程序中checkpoint与广播变量兼容处理](https://leafming.github.io/bigdata/2018/04/04/SparkStreaming程序中checkpoint与广播变量兼容处理/))。现在先说明这种不用checkpoint的方式。**  
在spark主程序中加入如下代码：  
```scala  
    //利用广播变量的形式，将后端写入KafkaProducer广播到每一个executor 注意：这里写广播变量的话，与checkpoint一起用会有问题
    val kafkaProducer: Broadcast[KafkaPool[String, String]] = {
      val kafkaProducerConfig = {
        val props = new Properties()
        props.put("metadata.broker.list",proKafkaBrokerAddr)
        props.put("security.protocol","SASL_PLAINTEXT")
        props.put("sasl.mechanism","PLAIN")
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
        props.put("bootstrap.servers",proKafkaBrokerAddr)
        props
      }
      log.warn("kafka producer init done!")
      ssc.sparkContext.broadcast(KafkaPool[String, String](kafkaProducerConfig))
    }
```  
## 第三步：使用广播变量  
spark 主程序中，在每个executor中使用广播变量。  
```scala
    writeDStrem.foreachRDD(rdd => {
      if (rdd.isEmpty) {
        logInfo(" No Data in this batchInterval --------")
      } else {
        val start_time = System.currentTimeMillis()
        rdd.foreach(record=>{
           kafkaProducer.value.send(proKafkaTopicName,record)
        })
        competeTime(start_time, "Processed data write to KAFKA")
      }
    })
```
至此，更改完毕。  
# 结果对比  
使用Kafka连接池更改程序之前以及之后的处理速度对比如下图所示，写入90W条数据由原来的17953ms变为了1966ms，效率大大提高。  
![compare](https://github.com/leafming/bak/blob/master/images/2018-04-02SparkStreamingKafka.png?raw=true)  
  
内容即以上，会在下篇文章([SparkStreaming程序中checkpoint与广播变量兼容处理](https://leafming.github.io/bigdata/2018/04/04/SparkStreaming程序中checkpoint与广播变量兼容处理/))解决spark streaming中checkpoint和广播变量使用冲突的问题，敬请期待。  
如有问题，请发送邮件至leafming@foxmail.com联系我，谢谢～  
©商业转载请联系作者获得授权，非商业转载请注明出处。 
