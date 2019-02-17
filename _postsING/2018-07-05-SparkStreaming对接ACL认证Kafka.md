---
layout: post
title: "SparkStreaming对接ACL认证Kafka"
date: 2018-07-05
updated: 2018-07-05
description: 本文关于在使用Spark Streaming从ACL认证的Kafka消费数据的操作。
categories:
- BigData
tags:
- Spark
- Kafka
---

> 最近有其他合作厂商在使用我们的ACL权限认证的kafka0.10.1.1的过程中，总出现不会使用spark从带认证的kafka消费数据的情况，所以给他们写了连接示例并且也来分享一下。  
> **注意: 本文中使用的版本是spark2.2.1和kafka0.10.1.1**  
  
# 消费准备  
Kafka自0.9.0.0.版本引入安全性，Kafka0.9提供了SASL/Kerberos，在0.10中增加了更多的SASL功能，比如SASL/Plaintext。当前Kafka安全主要包含3大功能：认证(authentication)、信道加密(encryption)和授权(authorization)。  
1. 认证机制主要是指配置SSL或者SASL(Kerberos)协议来对客户端（生产者和消费者）或其他brokers/工具到brokers的连接进行身份认证。以及对brokers到ZooKeeper的连接进行身份认证。  
2. 信道加密就是利用SSL为client到broker、broker到broker以及工具脚本与broker之间的数据传输配置SSL。  
3. 授权是通过ACL接口命令来完成的，对客户端的读写进行权限控制，权限控制是可插拔的并且支持与外部的授权服务进行集成。  
生产环境中，用户若要一般使用SASL则配置Kerberos，而我们的kafka集群给其他厂商提供数据，Kerberos对其他厂商来说使用起来可能稍有难度，所以暂时还没有进行Kerberos配置，只是使用了kafka acl（SASL/Plaintext）功能。  
对于kafka Acl的其他内容，可以参考官网["SECURITY"](http://kafka.apache.org/0101/documentation.html#security)。  
对于kafka ACL集群的搭建，这里就先不整理了。  
下面，如果使用acl认证的kafka的话，先进行赋权操作。  
为用户user1赋予对于topic test的权限，消费组为user1。  
```
./kafka-acls.sh --authorizer-properties zookeeper.connect=132.111.111.11:2181,132.111.111.12:2181,132.111.111.13:2181/kafka --add --allow-principal User:user1 --consumer --topic test --group user1
其他命令补充-生产者赋权命令：
./kafka-acls.sh --authorizer-properties zookeeper.connect=132.90.130.11:2181,132.90.130.12:2181,132.90.130.13:2181/kafka --add --allow-principal User:user1 --producer --topic test
删除acl：
./kafka-acls.sh --authorizer-properties zookeeper.connect=132.90.130.11:2181,132.90.130.12:2181,132.90.130.13:2181/kafka3 --remove --allow-principal User:user1 --consumer --topic test2 --group group1
topic列表：
./kafka-topics.sh --list --zookeeper 132.90.130.11:2181,132.90.130.12:2181,132.90.130.13:2181/kafka
acl列表：
./kafka-acls.sh --authorizer-properties zookeeper.connect=132.90.130.11:2181,132.90.130.12:2181,132.90.130.13:2181/kafka --list
topic详细信息：
./kafka-topics.sh --describe --topic test2 --zookeeper 132.90.130.11:2181/kafka,132.90.130.12:2181/kafka,132.90.130.13:2181/kafka
指定partation从最近消费：
 ./kafka-console-consumer.sh --bootstrap-server 132.90.117.70:9092,132.90.117.71:9092,132.90.117.72:9092,132.90.117.73:9092,132.90.117.74:9092,132.90.117.75:9092 --topic 4g_info_original_op --partition 0 --offset latest --consumer.config=../config/consumer.properties
整体从开始消费：
./kafka-console-consumer.sh --bootstrap-server 132.90.117.70:9092,132.90.117.71:9092,132.90.117.72:9092,132.90.117.73:9092,132.90.117.74:9092,132.90.117.75:9092 --topic gn_info_234g_op --from-beginning --consumer.config=../config/consumer.properties

```  
# 程序连接  
这里，我们使用的是spark2.1.1版本的spark来对接kafka0.10版本，因为官方Spark2.*之后才适配kafka0.10。  
如果使用HDP2.6中的spark1.6来对接kafka0.10版本的话，可以参考文章("Spark-Streaming Kafka In Kerberos")[https://www.jianshu.com/p/a70e27029ec7]中说明的，使用HDP提供的spark-kafka-0-10-connector包（(官网说明)[https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.0/bk_spark-component-guide/content/using-spark-streaming.html#spark-streaming-jar]）。不过这种方式没有尝试，直接把之前的1.6.3版本的spark换成了spark2.1.1版本的来对接，有兴趣的可以看看。  
这里暂时留一个问题，既然Spark2.*之后适配的是kafka0.10，那么如何使用Spark2.*对接kafka0.8版本消费呢，这个在实际生产中我们也遇到了，所以后续会写一个文章来说明下。  
  
  
  
  
  
  
  
  
  
  
  
  
--- 
