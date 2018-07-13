---
layout: post
title: "SparkStreaming写入数据到HDFS（工作实际）"
date: 2018-07-02
updated: 2018-07-02
description: 本文关于在使用Spark Streaming输出数据到HDFS的几种方式。
categories:
- BigData
tags:
- Spark
- HDFS
---
  
> 之前的Spark实时流处理的数据处理程序，要求把数据从kafka接收之后，分2路分别写入kafka和hdfs，写入kafka的部分之前已经有过总结，现在回过头来把之前的写入HDFS的地方重新总结一下，整个过程从头到尾有一个写入方式的优化，不过时间有点长啦，尽量描述完整( ˘ ³˘)♥。  
> **注意: 本文中使用的版本是spark2.2.1和2.6.0-cdh5.11.0**  

# 背景  
在工作中，需要将从kafka收到的数据做一些处理，然后分2路存储到kafka和HDFS中，再供下游进行使用。  
所以，在通过Spark Streaming写入HDFS的时候根据业务需要，最后变革了好多次，才形成了最后的版本。  
  
# 最简单方式-直接使用saveAsTextFile  
对于将数据直接存入HDFS，spark提供了现成的api：**saveAsTextFile**。  
```scala
def saveAsTextFile(path: String): Unit
def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit
saveAsTextFile用于将RDD以文本文件的格式存储到文件系统中。
```  
不过，直接使用saveAsTextFile有很多需要注意的问题。首先，**saveAsTextFile要求保存的目录之前是没有的，否则会报错。**所以，最好程序中保存前先判断一下目录是否存在。后续代码例子先没有判断目录是否存在，需要自己调整。  
1. 文件内容会被覆盖掉  
对于初学spark的时候，第一次使用saveAsTextFile，可能会遇到文件内容会被覆盖掉的问题。  
在第一次使用的时候，可能会想要这么写的：  
```scala
        saveDstream.foreachRDD(rdd => {
          //EmptyRDD是没有分区的，所以调用partitions.isEmpty是true，所以这样可以避免写空文件
          if (rdd.partitions.isEmpty) {
            logInfo(" No Data in this batchInterval --------")
          } else {
            val start_time = System.currentTimeMillis()
            rdd.saveAsTextFile("hdfs://cdh5hdfs/savepath")
            competeTime(start_time, "Processed data write to hdfs")
          }
        })
```  
而一旦实际测试之后，你会发现savepath目录里的文件每次都会被覆盖掉，只保存着最后一次saveAsTextFIle的内容。  
这是因为foreachRDD默认保存的文件名就是part0000_ ... part0000n，每一个rdd都是这样，所以在Spark Streaming程序中，后面的批次数据过来在同一路径下后面的文件可能会覆盖前面的，因此会出现文件内容会被覆盖掉。所以来说，为了避免这个问题，可以在保存的时候在文件夹后面加上时间戳来解决。  
```scala
        saveDstream.foreachRDD(rdd => {
          //EmptyRDD是没有分区的，所以调用partitions.isEmpty是true，所以这样可以避免写空文件
          if (rdd.partitions.isEmpty) {
            logInfo(" No Data in this batchInterval --------")
          } else {
            val start_time = System.currentTimeMillis()
            val curDay=new Date(start_time)
            val date: String =dateFormat.format(curDay)
            rdd.saveAsTextFile("hdfs://cdh5hdfs/"+date+"/savepath"+start_time)
            competeTime(start_time, "Processed data write to hdfs")
          }
        })
```
即：rdd.saveAsTextFile("hdfs://cdh5hdfs/"+date+"/savepath"+start_time)，这样写之后，保存的路径会根据当前时间来生成，假设允许到此段程序的时间是“2018-06-30 18:30:20”，那么文件存储的位置就会是“hdfs://cdh5hdfs/2018-06-30/savepath1530354620”，这样暂且解决了文件内容会被覆盖掉的问题。不过，只是解决了这个还不够，此种方式还有其他需要注意的问题。  
2. 保存文件过多  
使用saveAsTextFile经过以上操作之后，不会再出现文件覆盖的问题。但是，当实际运行后会发现，会有很多很多文件，名称为part00000 ... part0000n,这样如果文件太小并且过多，就会浪费hdfs的block空间，需要尽可能把文件大小调整到和block大小差不多为好。  
之所以产生了这么多文件，这是因为在spark运行的时候，spark一般会按照执行task的多少生成多少个文件，spark把数据分成了很多个partation，每个partation对应一个task，一个partation的数据由task来执行保存动作，这样的话，在调用saveAsTextFile的时候，会把每个partation的数据保存为一个part0000n文件。  
所以，为了把文件保存为一个文件，可以在RDD上调用coalesce(1,true).saveAsTextFile()，意味着做完计算之后将数据汇集到一个分区，然后再执行保存的动作，显然，一个分区，Spark自然只起一个task来执行保存的动作，也就只有一个文件产生了。又或者，可以调用repartition(1)，它其实是coalesce的一个包装，默认第二个参数为true。即：  
```scala
rdd.coalesce(1,true).saveAsTextFile("hdfs://cdh5hdfs/"+date+"/savepath"+start_time)
//或
//rdd.repartition(1).saveAsTextFile("hdfs://cdh5hdfs/"+date+"/savepath"+start_time)
```  
不过这样写，在数据过多的时候有很大的隐患：一般情况下，Spark面对的是大量的数据，并且是并行执行的，在数据过多的时候，如果强行要求最后只有一个分区，必然导致大量的磁盘IO和网络IO产生，并且最终操作的节点的内存也会承受很大考验，可能会出现单节点内存不足的问题或者效率及其低下。因此，有人提出，可以采用HDFS磁盘合并操作，即HDFS的getmerge操作。  
```
//hadoop fs -getmerge 源路径 目的路径 
hadoop fs -getmerge /hdfs/output   /hdfs2/output.txt
//或者cat >操作
```  
3. 不能自定义名称，目录层级太多  
经过以上操作之后，就能把数据写到hdfs中，并且文件数量不会过多了。但是经过以上测试，我们会发现有个问题，为了防止文件内容被覆盖我们使用了时间戳，这样的话相当于生成的是一个不同的目录，但是在那个目录里面，仍然是part00000命名的文件，不能自定义名称，并且这样是每个批次生成了一个目录，这样的话，感觉目录太多了。  
# 直接使用HDFS API-append方法（测试，非生产）  
为了实现自定义文件名称、减少目录层级、追加写文件的需要，有文章提示说可以直接调用HDFS的api-append。但是实际上，有资料表明，hadoop的版本1.0.4以后，API中已经有了追加写入的功能，但不建议在生产环境中使用，不过我们也可以测试下。  
如果需要使用此方法，需要修改配置文件，开启此功能，把dfs.support.appen的参数设置为true，不然客户端写入的时候会报错：  
```
Exception in thread "main" org.apache.hadoop.ipc.RemoteException: java.io.IOException: Append to hdfs not supported. Please refer to dfs.support.append configuration parameter.
```  
修改namenode节点上的hdfs-site.xml：  
```xml
  <property>  
       <name>dfs.support.append</name>  
       <value>true</value>  
  </property>
```  
或者，可以直接在程序里面代码设置：  
```scala
private val conf = new Configuration()
conf.setBoolean("dfs.support.append", true)
//后续直接使用其创建fs如：
//var fs=FileSystem.get(uri, conf)
```  
并且需要注意的是，如果使用append追加写入文件，如果文件不存在，需求先创建。那么如果出现了这样的问题，你可能会直接像如下代码那么写（注意是错误的）：  
```scala
    val path=new Path(strpath)
    val uri = new URI(strpath)
    var fs=FileSystem.get(uri, conf)
    if (!fs.exists(path)) {
      fs.create(path)
    }
    val out = fs.append(path)
    out.write(record_sum1.getBytes())
```  
这样的话，会报如下错误：  
```
Exception in thread "main" org.apache.hadoop.ipc.RemoteException: org.apache.hadoop.hdfs.protocol.AlreadyBeingCreatedException: failed to create file /hdfs/testfile/file for DFSClient_-14565853217 on client 132.90.130.101 because current leaseholder is trying to recreate file.
```  
通过搜索发现(参考[Hadoop HDFS文件常用操作及注意事项（更新）遇到的问题2](https://www.cnblogs.com/byrhuangqiang/p/3926663.html)，[HDFS写入异常:追加文件第一次抛异常](https://blog.csdn.net/liu0bing/article/details/78951415))，发现可能和fs句柄有关，因此解决的时候，可以创建完文件之后，关闭流fs，再次获得一次新的fs：  
```scala
    val path=new Path(strpath)
    val uri = new URI(strpath)
    var fs=FileSystem.get(uri, conf)
    if (!fs.exists(path)) {
      fs.create(path)
      fs.close()
      fs=FileSystem.get(uri, conf)
    }
    val out = fs.append(path)
    out.write(record_sum1.getBytes())
```  
因此整体的写入HDFS的方法如下所示：  
```scala
  def writeToHDFS(strpath: String,iter: Iterator[(String, String, String)], cache: Int): Try[Unit] = Try {
    var record_sum1=""   //初始化空串
    var count_sum1=0     //计数器
    var record=""
    val path=new Path(strpath)
    val uri = new URI(strpath)
    var fs=FileSystem.get(uri, conf)
    if (!fs.exists(path)) {
      fs.create(path)
      fs.close()
      fs=FileSystem.get(uri, conf)
    }
    val out = fs.append(path)
    while (iter.hasNext) {
      record=iter.next()._3
      record_sum1 += record+"\n"
      count_sum1 = count_sum1 + 1

      if (count_sum1 == cache) {
        out.write(record_sum1.getBytes())
        record_sum1 = ""
        count_sum1 = 0
      }
    }
    if (!record_sum1.isEmpty) {
      out.write(record_sum1.getBytes())
    }
    out.flush()
    out.close()
    fs.close()
  }
```  
因此，在spark程序里面可以直接rdd处这样使用：  
```scala
      if (!rdd.isEmpty()){
      //注意这里， 因为不需要返回值，所以用foreachPartition直接落地存储了最好。foreachPartition是collect算子，mapPartitions是transofmation。
       rdd.foreachPartition(iter=>{
           writeToHDFS_equal(strpath_equals,iter,300)          
       })       
       //rdd.repartition(1).mapPartitions(iter=>{
       //   val str=List[String]()
       //   val strpath_equals="hdfs://dcoshdfs:8020/spark_log/test2/" + date +"/"+hour+"/"+creatFileName(hour.toString,sys_time)
       //   writeToHDFS_equal(strpath_equals,iter,300)
       //   str.iterator
       // }).collect()
      }
```  
但是使用此种方法，相当于一个partation建立一次HDFS连接，执行起来会特别慢，是否可以考虑向Kafka似的弄个连接池的方式提高效率，但是我没有这样实验，毕竟append被说是测试版本，不建议用于生产。而且，因为我们处理的数据也是从上游接收过来的，每条数据中都有记录此条数据生成的时间，但是我们实际收到的时间可能较生成有所延迟，所以后来增加了需求，不仅仅要求需要自定义存储的文件名，而且要根据数据内容（数据内容的生成时间字段），来决定将数据写入到哪个文件夹的哪个文件内，所以来说，直接用append有很大的麻烦。最终，再寻找了下面的最终方法。  
# 最终方法-saveAsHadoopFile+RDDMultipleTextOutputFormat  
## 分析  
我们使用的saveAsTextFile只能传入所需要的path，而不能根据我们需要进行自定义操作。  
不过我们可以看一下saveAsTextFile的源码：  
```scala
/**
   * Save this RDD as a text file, using string representations of elements.
   */
  def saveAsTextFile(path: String): Unit = withScope {
    // https://issues.apache.org/jira/browse/SPARK-2075
    //
    // NullWritable is a `Comparable` in Hadoop 1.+, so the compiler cannot find an implicit
    // Ordering for it and will use the default `null`. However, it's a `Comparable[NullWritable]`
    // in Hadoop 2.+, so the compiler will call the implicit `Ordering.ordered` method to create an
    // Ordering for `NullWritable`. That's why the compiler will generate different anonymous
    // classes for `saveAsTextFile` in Hadoop 1.+ and Hadoop 2.+.
    //
    // Therefore, here we provide an explicit Ordering `null` to make sure the compiler generate
    // same bytecodes for `saveAsTextFile`.
    val nullWritableClassTag = implicitly[ClassTag[NullWritable]]
    val textClassTag = implicitly[ClassTag[Text]]
    val r = this.mapPartitions { iter =>
      val text = new Text()
      iter.map { x =>
        text.set(x.toString)
        (NullWritable.get(), text)
      }
    }
    RDD.rddToPairRDDFunctions(r)(nullWritableClassTag, textClassTag, null)
      .saveAsHadoopFile[TextOutputFormat[NullWritable, Text]](path)
  }

  /**
   * Save this RDD as a compressed text file, using string representations of elements.
   */
  def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit = withScope {
    // https://issues.apache.org/jira/browse/SPARK-2075
    val nullWritableClassTag = implicitly[ClassTag[NullWritable]]
    val textClassTag = implicitly[ClassTag[Text]]
    val r = this.mapPartitions { iter =>
      val text = new Text()
      iter.map { x =>
        text.set(x.toString)
        (NullWritable.get(), text)
      }
    }
    RDD.rddToPairRDDFunctions(r)(nullWritableClassTag, textClassTag, null)
      .saveAsHadoopFile[TextOutputFormat[NullWritable, Text]](path, codec)
  }
```  
可以发现，saveAsTextFile函数是依赖于saveAsHadoopFile函数，由于saveAsHadoopFile函数接受PairRDD，所以在saveAsTextFile函数中利用rddToPairRDDFunctions函数转化为(NullWritable,Text)类型的RDD，然后通过saveAsHadoopFile函数实现相应的写操作。  
  
  
  
  
  
  

