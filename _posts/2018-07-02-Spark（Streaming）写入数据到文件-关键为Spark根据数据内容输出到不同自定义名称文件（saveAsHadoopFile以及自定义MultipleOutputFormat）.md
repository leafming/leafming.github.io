---
layout: post
title: "Spark(Streaming)写入数据到文件-关键为根据数据内容输出到不同自定义名称文件(saveAsHadoopFile以及自定义MultipleOutputFormat)"
date: 2018-07-02
updated: 2018-12-04
description: 本文关于在使用Spark或Spark Streaming输出数据到文件的几种方式。关键的内容是Spark Streaming中实现实时根据数据内容，将数据写入不同的文件存储，支持自定义输出的文件名称，主要使用saveAsHadoopFile以及自定义MultipleOutputFormat实现。本文的场景是数据写入hdfs。
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
  
# 最基础直接方式-直接使用saveAsTextFile  
对于将数据直接存入HDFS或本地文件等，spark提供了现成的算子：**saveAsTextFile**。  
```scala
def saveAsTextFile(path: String): Unit
def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit
saveAsTextFile用于将RDD以文本文件的格式存储到文件系统中。
```  
不过，直接使用saveAsTextFile有很多需要注意的问题。首先，**saveAsTextFile要求保存的目录之前是没有的，否则会报错。**所以，最好程序中保存前先判断一下目录是否存在。**下面的代码的例子先没有判断目录是否存在，需要自己调整**。  
下面我们说一下直接使用saveAsTextFile可能遇到的问题：  
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
这是因为foreachRDD中使用saveAsTextFile默认保存的文件名就是part-0000_ ... part-0000n，每一个rdd都是这样，所以在Spark Streaming程序中，后面的批次数据过来在同一路径下后面的文件可能会覆盖前面的，因此会出现文件内容会被覆盖掉。所以来说，为了避免这个问题，可以在保存的时候在文件夹后面加上时间戳来解决。  
```scala
        saveDstream.foreachRDD(rdd => {
          //EmptyRDD是没有分区的，所以调用partitions.isEmpty是true，所以这样可以避免写空文件
          if (rdd.partitions.isEmpty) {
            logInfo(" No Data in this batchInterval --------")
          } else {
            val start_time = System.currentTimeMillis()
            val curDay=new Date(start_time)
            val date: String =dateFormat.format(curDay)
            rdd.saveAsTextFile("hdfs://cdh5hdfs/savepath/"+date+"/"+start_time)
            competeTime(start_time, "Processed data write to hdfs")
          }
        })
```
即：rdd.saveAsTextFile("hdfs://cdh5hdfs/savepath/"+date+"/"+start_time)，这样写之后，保存的路径会根据当前时间来生成，假设运行到此段程序的时间是“2018-06-30 18:30:20”，那么文件存储的目录就会是“hdfs://cdh5hdfs/savepath/2018-06-30/1530354620”，如下显示：  
```
dcos@d8pccdsj3[~]$hadoop fs -ls /savepath/2018-06-30/1530354620
Found 8 items
-rw-r--r--   3 test cgroup          0 2018-06-30 18:30 /savepath/2018-06-30/1530354620/_SUCCESS
-rw-r--r--   3 test cgroup 2201736217 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00000
-rw-r--r--   3 test cgroup 2201037065 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00001
-rw-r--r--   3 test cgroup 2202157942 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00002
-rw-r--r--   3 test cgroup 2202523100 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00003
-rw-r--r--   3 test cgroup 2202310836 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00004
-rw-r--r--   3 test cgroup 2202639458 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00005
-rw-r--r--   3 test cgroup 2201906597 2018-06-30 18:30 /savepath/2018-06-30/1530354620/part-00006
```  
这样暂且解决了文件内容会被覆盖掉的问题。不过，这种情况下，我们会发现以日期命名的文件夹2018-06-30下，会根据每个批次生成一个时间戳命名的目录，并且目录下文件名称名称为part-0000n，所以说只是解决了这个还不够，此种方式还有其他需要注意的问题。  
2. 保存文件过多  
使用saveAsTextFile经过以上操作之后，不会再出现文件覆盖的问题。但是，当实际运行后会发现，在sparkStreming程序中使用这种方式，会根据每个批次生成一个时间戳命名的目录，目录太多；并且在目录下会有很多很多文件，名称为part-00000 ... part-0000n,这样如果文件太小并且过多，就会浪费hdfs的block空间，需要尽可能把文件大小调整到和block大小差不多为好。  
之所以产生了这么多文件，这是因为在spark运行的时候，spark一般会按照执行task的多少生成多少个文件，spark把数据分成了很多个partation，每个partation对应一个task，一个partation的数据由task来执行保存动作，这样的话，在调用saveAsTextFile的时候，会把每个partation的数据保存为一个part-0000n文件。  
所以，为了把文件保存为一个或较少文件，可以使用coalesce或repartition算子。如果生成一个文件，可以在RDD上调用coalesce(1,true).saveAsTextFile()，意味着做完计算之后将数据汇集到一个分区，然后再执行保存的动作，显然，一个分区，Spark自然只起一个task来执行保存的动作，也就只有一个文件产生了。又或者，可以调用repartition(1)，它其实是coalesce的一个包装，默认第二个参数为true。即：  
```scala
rdd.coalesce(1,true).saveAsTextFile("hdfs://cdh5hdfs/savepath/"+date+"/"+start_time)
//或
//rdd.repartition(1).saveAsTextFile("hdfs://cdh5hdfs/savepath/"+date+"/"+start_time)
```  
不过这样写，在数据过多的时候有很大的隐患：一般情况下，Spark面对的是大量的数据，并且是并行执行的，在数据过多的时候，如果强行要求最后只有一个分区，必然导致大量的磁盘IO和网络IO产生，并且最终操作的节点的内存也会承受很大考验，可能会出现单节点内存不足的问题或者效率及其低下。因此，有人提出，可以采用HDFS磁盘合并操作，即HDFS的getmerge操作。  
```
//hadoop fs -getmerge 源路径 目的路径 
hadoop fs -getmerge /hdfs/output   /hdfs2/output.txt
//或者cat >操作
```  
3. 不能自定义名称，目录层级太多  
经过以上操作之后，就能把数据写到hdfs中，并且文件数量不会过多了。但是经过以上测试，我们会发现有个问题，为了防止文件内容被覆盖我们使用了时间戳，这样的话相当于生成的是一个不同的目录，在sparkStreming程序中使用这种方式，会根据每个批次生成一个时间戳命名的目录，目录太多；并且在那个目录里面，仍然是part-0000n命名的文件，不能自定义名称。因此，我们就思考能不能从文件名称入手，自定义文件名称而不是目录名称呢？  
  
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
通过搜索发现可能和fs句柄有关，因此解决的时候，可以创建完文件之后，关闭流fs，再次获得一次新的fs：
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
经实践测试，上述方法仍可能报错：“org.apache.hadoop.hdfs.protocol.AlreadyBeingCreatedException”，异常的原因：FSDataOutputStream create(Path f) 产生了一个输出流，创建完后需要关闭。解决：创建完文件之后，关闭流FSDataOutputStream。因此可尝试使用如下：  
```scala
      //不重新读传入uri，则直接从操作系统环境变量获取
      var fs: FileSystem =FileSystem.get(conf)
      println("append to file:"+dirPath)
      if (!fs.exists(path)) {
        println("path not exist")
        // create(Path f) 产生了一个输出流，创建完后需要关闭,否则报错
        fs.create(path).close()
      }

      //fs.append不建议在生产环境使用
      val out: FSDataOutputStream = fs.append(path)
```  
> 参考[Hadoop HDFS文件常用操作及注意事项（更新）遇到的问题2](https://www.cnblogs.com/byrhuangqiang/p/3926663.html)，[HDFS写入异常:追加文件第
一次抛异常](https://blog.csdn.net/liu0bing/article/details/78951415)。  
 

因此整体的写入HDFS的方法，我们这么写（假设这里测试数据的Iterator[(String,String)]元组，需要写入第二个字段），在spark调用的时候，每cache条写入一次：  
```scala
  def writeToHDFS(strpath: String,iter: Iterator[(String,String)], cache: Int): Try[Unit] = Try {
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
      record=iter.next()._2
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
在spark程序里面可以直接rdd处这样使用：  
```scala
      if (!rdd.isEmpty()){
      //注意这里， 因为不需要返回值，所以用foreachPartition直接落地存储了最好。foreachPartition是collect算子，mapPartitions是transofmation。
        val start_time = System.currentTimeMillis()
        val curDay=new Date(start_time)
        val date: String =dateFormat.format(curDay)
        rdd.foreachPartition(iter=>{
           val strpath="hdfs://cdh5hdfs/savepath/"+date+"/"+start_time
           writeToHDFS(strpath,iter,300)          
       })       
       //rdd.repartition(1).mapPartitions(iter=>{
       //   val str=List[String]()
       //   val strpath="hdfs://cdh5hdfs/savepath/"+date+"/"+start_time
       //   writeToHDFS(strpath,iter,300)
       //   str.iterator
       // }).collect()
      }
```  
这样的话，目录层次就变成了“hdfs://cdh5hdfs/savepath/当前日期/时间戳文件”内容即直接追加写入了以时间戳命名的文件中，减少了目录层级，并且能够自定义文件名称了。
但是使用此种方法，相当于一个partation建立一次HDFS连接，执行起来会特别慢，是否可以考虑向Kafka似的弄个连接池的方式提高效率，但是我没有这样实验，毕竟append被说是测试版本，不建议用于生产。  
# 需求更改  
上面的操作均是是对数据没有任何区分，直接将数据写入文件中。但是，后来我们的需要因为业务需要而有所更改。  
我们处理的数据也是从上游接收过来的，每条数据中都有记录此条数据生成的时间，但是我们实际收到的时间可能较生成有所延迟，所以不仅仅要求需要自定义存储的文件名，而且要根据数据内容（数据内容的生成时间字段），来决定将数据写入到哪个文件夹的哪个文件内。  
比如如数据格式如下所示：  
```
0|18610000000|460010000000000|2018|07|28|16-21-35|41003|22002|35004007800300|0000|||||0|||2018-07-28 16:21:35
```  
因此，我们根据需求，生成的目录格式需要是：  
```
dcos@d8pccdsj3[~]$hadoop fs -ls /savepath/2018-07-28
Found 11 items
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:42 /savepath/2018-07-28/00
drwxr-xr-x   - user1 cgroup          0 2018-07-28 09:48 /savepath/2018-07-28/01
drwxr-xr-x   - user1 cgroup          0 2018-07-28 09:40 /savepath/2018-07-28/02
drwxr-xr-x   - user1 cgroup          0 2018-07-28 08:54 /savepath/2018-07-28/03
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:26 /savepath/2018-07-28/04
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:30 /savepath/2018-07-28/05
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:38 /savepath/2018-07-28/06
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:26 /savepath/2018-07-28/07
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:42 /savepath/2018-07-28/08
drwxr-xr-x   - user1 cgroup          0 2018-07-28 10:42 /savepath/2018-07-28/09
drwxr-xr-x   - user1 cgroup          0 2018-07-28 11:42 /savepath/2018-07-28/10
...
drwxr-xr-x   - user1 cgroup          0 2018-07-28 16:40 /savepath/2018-07-28/16
...
drwxr-xr-x   - user1 cgroup          0 2018-07-29 00:30 /savepath/2018-07-28/23
```
即目录要求为/savepath/日期/小时/文件名，文件名需要辨识当前此文件的写入时间（即系统时间），而目录名称的日期和小时需要根据数据里面的业务时间决定，即上述示例数据的第18个字段，那么上面那条数据就需要放在/savepath/2018-07-28/16目录下的某个文件内（如16-16-06-00）。  
所以来说，根据这个需求，我们是不能直接使用saveAsTextFile的，因为文件名称需要自定义；而也尝试直接用append的时候有很大的麻烦，需要根据每条数据内容来决定放入某个目录和文件，并且数据可能延迟比如说现在10点，能零零散散收到几条3点的数据，所以来说在使用append的时候把每条数据的提取日期等信息处理成了三元组的形式来在writeToHDFS方法里面提取时间决定目录和文件名称，但是使用起来还是很复杂并且处理和写入时间太长，影响实时性，并且append还是测试方法不适合投入生产。因此，最终再寻找了下面的最终方法。  
# 最终方法-saveAsHadoopFile+自定义的RDDMultipleTextOutputFormat  
## 分析  
为了解决上述需求，我需要另寻找解决办法。我们还是从spark本身的算子入手，可以先看一下saveAsTextFile的源码：  
```scala
/**
   * Save this RDD as a text file, using string representations of elements.
   */
  def saveAsTextFile(path: String): Unit = withScope {
    // https://issues.apache.org/jira/browse/SPARK-2075
    // NullWritable is a `Comparable` in Hadoop 1.+, so the compiler cannot find an implicit Ordering for it and will use the default `null`. However, it's a `Comparable[NullWritable]`in Hadoop 2.+, so the compiler will call the implicit `Ordering.ordered` method to create an Ordering for `NullWritable`. That's why the compiler will generate different anonymous classes for `saveAsTextFile` in Hadoop 1.+ and Hadoop 2.+.Therefore, here we provide an explicit Ordering `null` to make sure the compiler generate same bytecodes for `saveAsTextFile`.
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
> 参考：[【SparkJavaAPI】Action(6)—saveAsTextFile、saveAsObjectFile](https://www.jianshu.com/p/4d4a7fdaca5c)。  

并且我们通过saveAsTextFile的源码看到，可以看出Spark内部写文件方式其实调用的都是Hadoop那一套东西，当saveAsTextFile调用saveAsHadoopFile方法时候，默认OutputFormat使用的是TextOutputFormat[NullWritable, Text]。  
而对于TextOutputFormat，在Hadoop的MapReduce中多文件输出默认就是TextOutputFormat，因此默认输出为part-r-00000和part-r-00001依次递增的文件名，所有Mapreduce作业都输出一组文件，并没有和我们的需求一样，根据文件内容输出多组文件或者把一个数据集分为多个数据集（比如说将一个log里面属于不同业务线的日志分开来输出，并交给相关的业务线，或者像我们这种根据数据时间分文件存储）。而在Hadoop中，Hadoop的多文件输出根据Key或者Value的不同将属于不同的类型记录写到不同的文件中的需求，可以使用MultipleOutputFormat或者MultipleOutputs替换TextOutputFormat来解决。  
> 具体可以参考以下文章，虽然文章时间较长了，但是都写的非常好，可以参考学习：[“Hadoop多文件输出：MultipleOutputFormat和MultipleOutputs深究(一)”](https://www.iteblog.com/archives/842.html)、[“Hadoop多文件输出：MultipleOutputFormat和MultipleOutputs深究(二)”](https://www.iteblog.com/archives/848.html)、[“Hadoop的MultipleOutputFormat使用”](https://blog.csdn.net/dajuezhao/article/details/5799388)。  
  
因此，既然Spark内部写文件方式其实调用的都是Hadoop那一套东西，所以，我们就可以使用saveAsHadoopFile，自定义一个MultipleOutputFormat的OutputFormat类就可以了。  
> 题外话：到这里，就可以继续我们下面的工作直接解决问题了，不过看到这里我就有了个疑问，既然saveAsTextFile默认使用TextOutputFormat向hdfs输出文件，但是默认的文件输出文件名称为part-00000和part-00001依次递增的文件名，而hadoop默认输出的是part-r-00000，想知道文件名称是什么时候变化的呢，因此就追踪了一下spark源码学习学习，关于这个问题的源码分析，后续有时间再写文章说明。  
  
## 代码编写  
下面我们开始正式的如何使用spark实现多文件输出，根据文件内容划分文件存储并且自定义文件名称。  
> 参考：[“spark streaming实现根据文件内容自定义文件名并实现文件内容追加”](https://blog.csdn.net/qq_19917081/article/details/56841299)、[“Spark多文件输出(MultipleOutputFormat)”](https://www.iteblog.com/archives/1281.html)。  
  
### saveAsHadoopFile算子  
首先看看一下**saveAsHadoopFile**算子的官方说明：  
```scala
 /**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class supporting the key and value types K and V in this RDD.
   */
def saveAsHadoopFile[F <: OutputFormat[K, V]](path: String)(implicit fm: ClassTag[F]): Unit 
 /**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class supporting the key and value types K and V in this RDD. Compress the result with the supplied codec.
   */
 def saveAsHadoopFile[F <: OutputFormat[K, V]](path: String,codec: Class[_ <: CompressionCodec])(implicit fm: ClassTag[F]): Unit 
/**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class supporting the key and value types K and V in this RDD. Compress with the supplied codec.
   */
  def saveAsHadoopFile(
      path: String,
      keyClass: Class[_],
      valueClass: Class[_],
      outputFormatClass: Class[_ <: OutputFormat[_, _]],
      codec: Class[_ <: CompressionCodec]): Unit 
 /**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class supporting the key and value types K and V in this RDD.
   * @note We should make sure our tasks are idempotent when speculation is enabled, i.e. do not use output committer that writes data directly.
   * There is an example in https://issues.apache.org/jira/browse/SPARK-10063 to show the bad result of using direct output committer with speculation enabled.
   */
  def saveAsHadoopFile(
      path: String,
      keyClass: Class[_],
      valueClass: Class[_],
      outputFormatClass: Class[_ <: OutputFormat[_, _]],
      conf: JobConf = new JobConf(self.context.hadoopConfiguration),
      codec: Option[Class[_ <: CompressionCodec]] = None): Unit
```  
这里我们要使用的算子是:  
**def saveAsHadoopFile(path: String,keyClass: Class[_],valueClass: Class[_],outputFormatClass: Class[_ <: OutputFormat[_, _]],conf: JobConf = new JobConf(self.context.hadoopConfiguration),codec: Option[Class[_ <: CompressionCodec]] = None): Unit**  
这个算子里需要传入的参数依次是：文件路径、key类型、value类型、outputFormat方式。  
之前用的saveAsTextFile，在**org.apache.spark.rdd.RDD**类中，而saveAsHadoopFile算子属于**org.apache.spark.rdd.PairRDDFunctions**类，需要接收的参数是PairRDD，所以我们在使用前需要将原来的rdd做一下map操作，变成(key, value) 形式，这里先不详细说，在最后贴出来的代码之后再说一次。因此，我们暂且定（K，V）类型为classOf[String]、classOf[String]，再之后传入hdfs保存目录、类型，剩下的就是关键的需要传入OutputFormat，按照上面的分析，我们要自定义一个MultipleOutputFormat。  

### MultipleOutputFormat分析  
下面我们先看一下MultipleOutputFormat的源码：  
```java
/**
 * This abstract class extends the FileOutputFormat, allowing to write the
 * output data to different output files. There are three basic use cases for
 * this class. 
 * Case one: This class is used for a map reduce job with at least one reducer.
 * The reducer wants to write data to different files depending on the actual
 * keys. It is assumed that a key (or value) encodes the actual key (value)
 * and the desired location for the actual key (value).
 * Case two: This class is used for a map only job. The job wants to use an
 * output file name that is either a part of the input file name of the input
 * data, or some derivation of it.
 * Case three: This class is used for a map only job. The job wants to use an
 * output file name that depends on both the keys and the input file name,
 */
@InterfaceAudience.Public
@InterfaceStability.Stable
public abstract class MultipleOutputFormat<K, V>
extends FileOutputFormat<K, V> {
  /**
   * Create a composite record writer that can write key/value data to different
   * output files
   * @param fs
   *          the file system to use
   * @param job
   *          the job conf for the job
   * @param name
   *          the leaf file name for the output file (such as part-00000")
   * @param arg3
   *          a progressable for reporting progress.
   * @return a composite record writer
   * @throws IOException
   */
  public RecordWriter<K, V> getRecordWriter(FileSystem fs, JobConf job,
      String name, Progressable arg3) throws IOException {

    final FileSystem myFS = fs;
    final String myName = generateLeafFileName(name);
    final JobConf myJob = job;
    final Progressable myProgressable = arg3;

    return new RecordWriter<K, V>() {

      // a cache storing the record writers for different output files.
      TreeMap<String, RecordWriter<K, V>> recordWriters = new TreeMap<String, RecordWriter<K, V>>();

      public void write(K key, V value) throws IOException {

        // get the file name based on the key
        String keyBasedPath = generateFileNameForKeyValue(key, value, myName);

        // get the file name based on the input file name
        String finalPath = getInputFileBasedOutputFileName(myJob, keyBasedPath);

        // get the actual key
        K actualKey = generateActualKey(key, value);
        V actualValue = generateActualValue(key, value);

        RecordWriter<K, V> rw = this.recordWriters.get(finalPath);
        if (rw == null) {
          // if we don't have the record writer yet for the final path, create
          // one
          // and add it to the cache
          rw = getBaseRecordWriter(myFS, myJob, finalPath, myProgressable);
          this.recordWriters.put(finalPath, rw);
        }
        rw.write(actualKey, actualValue);
      };

      public void close(Reporter reporter) throws IOException {
        Iterator<String> keys = this.recordWriters.keySet().iterator();
        while (keys.hasNext()) {
          RecordWriter<K, V> rw = this.recordWriters.get(keys.next());
          rw.close(reporter);
        }
        this.recordWriters.clear();
      };
    };
  }
  /**
   * Generate the leaf name for the output file name. The default behavior does not change the leaf file name (such as part-00000) 
   * @param name
   *          the leaf file name for the output file
   * @return the given leaf file name
   */
  protected String generateLeafFileName(String name) {
    return name;
  }
  /**
   * Generate the file output file name based on the given key and the leaf file
   * name. The default behavior is that the file name does not depend on the
   * key. 
   * @param key
   *          the key of the output data
   * @param name
   *          the leaf file name
   * @return generated file name
   */
  protected String generateFileNameForKeyValue(K key, V value, String name) {
    return name;
  }
  /**
   * Generate the actual key from the given key/value. The default behavior is that
   * the actual key is equal to the given key 
   * @param key
   *          the key of the output data
   * @param value
   *          the value of the output data
   * @return the actual key derived from the given key/value
   */
  protected K generateActualKey(K key, V value) {
    return key;
  }
  /**
   * Generate the actual value from the given key and value. The default behavior is that
   * the actual value is equal to the given value 
   * @param key
   *          the key of the output data
   * @param value
   *          the value of the output data
   * @return the actual value derived from the given key/value
   */
  protected V generateActualValue(K key, V value) {
    return value;
  }
  /**
   * Generate the outfile name based on a given anme and the input file name. If
   * the {@link JobContext#MAP_INPUT_FILE} does not exists (i.e. this is not for a map only job),
   * the given name is returned unchanged. If the config value for
   * "num.of.trailing.legs.to.use" is not set, or set 0 or negative, the given
   * name is returned unchanged. Otherwise, return a file name consisting of the
   * N trailing legs of the input file name where N is the config value for
   * "num.of.trailing.legs.to.use". 
   * @param job
   *          the job config
   * @param name
   *          the output file name
   * @return the outfile name based on a given anme and the input file name.
   */
  protected String getInputFileBasedOutputFileName(JobConf job, String name) {
    String infilepath = job.get(MRJobConfig.MAP_INPUT_FILE);
    if (infilepath == null) {
      // if the {@link JobContext#MAP_INPUT_FILE} does not exists,
      // then return the given name
      return name;
    }
    int numOfTrailingLegsToUse = job.getInt("mapred.outputformat.numOfTrailingLegs", 0);
    if (numOfTrailingLegsToUse <= 0) {
      return name;
    }
    Path infile = new Path(infilepath);
    Path parent = infile.getParent();
    String midName = infile.getName();
    Path outPath = new Path(midName);
    for (int i = 1; i < numOfTrailingLegsToUse; i++) {
      if (parent == null) break;
      midName = parent.getName();
      if (midName.length() == 0) break;
      parent = parent.getParent();
      outPath = new Path(midName, outPath);
    }
    return outPath.toString();
  }
  /**
   * @param fs
   *          the file system to use
   * @param job
   *          a job conf object
   * @param name
   *          the name of the file over which a record writer object will be
   *          constructed
   * @param arg3
   *          a progressable object
   * @return A RecordWriter object over the given file
   * @throws IOException
   */
  abstract protected RecordWriter<K, V> getBaseRecordWriter(FileSystem fs,
      JobConf job, String name, Progressable arg3) throws IOException;
}
```  
在源码的最开始，我们看到对于MultipleOutputFormat<K, V>的描述，它可以将相似的记录输出到相同的数据集。我们可以看出，在写每条记录之前，MultipleOutputFormat将调用generateFileNameForKeyValue方法来确定需要写入的文件名。  
通过下图源码我们可以看出，在getRecordWriter中，对于文件名称的生成会先调用generateLeafFileName方法，而其只是传入了“name”来生成文件的leaf名称（图中标号1的myName），而此部分传入的name即默认的每个part生成的文件名称（如part-0000，关于这个name的值，也可以继续向父类FileOutputFormat或MultipleOutputs类继续挖掘，找出来为何这个默认值，或者参考后续的关于spark这里的源码说明文档，这里就不说明了）；而之后生成的myName会传入generateFileNameForKeyValue的方法，这个方法接受3个参数，可以根据k、v以及传入的name再次生成一个KeyBasePath即文件名称（图中标号2），之后获取到的KeyBasePath再作为参数传入getInputFileBasedOutputFileName方法生成finalPath。可以看到在默认情况下，我们看到generateLeafFileName方法和generateFileNameForKeyValue的方法均是直接**"return name"**，因此默认情况下得到name==myName==KeyBasePath，文件的名称即为part-0000。  
![MultipleOutputFormat-文件名称生成](https://github.com/leafming/bak/blob/master/images/spark/2018-07-26-SparkStreaming写入数据到HDFS-Outputformat方法说明.png?raw=true)  
所以，为了根据内容来确定写入的文件名称，generateLeafFileName只与name有关（后续写个示例中测试一下generateLeafFileName），而generateFileNameForKeyValue与内容key、value、name均有关，所以我们就直接在自己的类中重写generateFileNameForKeyValue方法即可。  
不过，我们需要写入的是文本，所以通常情况下，我们可以直接继承MultipleTextOutputFormat类，来完成实现generateFileNameForKeyValue方法以返回每个输出键/值对的文件名。MultipleTextOutputFormat也是继承的MultipleOutputFormat类，可以在官方文档的说明的[MultipleTextOutputFormat](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapred/lib/MultipleTextOutputFormat.html)也可以看到类的继承关系。  
MultipleTextOutputFormat的官方描述如下：  
```java
/**
 * This class extends the MultipleOutputFormat, allowing to write the output
 * data to different output files in Text output format.
 */
@InterfaceAudience.Public
@InterfaceStability.Stable
public class MultipleTextOutputFormat<K, V> extends MultipleOutputFormat<K, V> {
  private TextOutputFormat<K, V> theTextOutputFormat = null;

  @Override
  protected RecordWriter<K, V> getBaseRecordWriter(FileSystem fs, JobConf job,
      String name, Progressable arg3) throws IOException {
    if (theTextOutputFormat == null) {
      theTextOutputFormat = new TextOutputFormat<K, V>();
    }
    return theTextOutputFormat.getRecordWriter(fs, job, name, arg3);
  }
}  
```  
它默认能够将以text的格式将数据输出到不同的目录中，我们需要输出的也是文本，只不过是写入目录需要自定义一下，所以我们只需要继承MultipleTextOutputFormat来自定义一个类即可。  
### 测试generateLeafFileName（补充，可略过）  
上面我们说了MultipleOutputFormat中有两个决定文件名称的方法，generateLeafFileName和generateFileNameForKeyValue，我们在正式代码前，先直接写入文件系统测试下generateLeafFileName，可略过。  
1. 自定义一个OutputFormat名称为TestMultipleTextOutputFormat。  
```scala
package com.ileaf.test
import java.text.SimpleDateFormat
import java.util.Date
import org.apache.hadoop.mapred.lib.MultipleTextOutputFormat
class TestMultipleTextOutputFormat  extends MultipleTextOutputFormat[Any, Any]{
  private val HOURFORMAT = new SimpleDateFormat("HH-mm-ss")
  private val YMDFORMAT = new SimpleDateFormat("yyyy-mm-dd")
  private val start_time = System.currentTimeMillis()
  private val curDay=new Date(start_time)
  private val fileName=HOURFORMAT.format(curDay)
  private val dirName=YMDFORMAT.format(curDay)
  override def generateLeafFileName(name: String):String={
    val filename=dirName+"/"+fileName+"_"+name
    filename
  }
}
```  
2. spark程序中测试使用  
```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD
object TestSparkSave {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local").setAppName("flatMap Demo")
    val sc = new SparkContext(sparkConf)
    val rdd = sc.parallelize(List("0|18610000000|460010000000000|2018|07|21|16-21-35|41003|22002|35004007800300|0000|||||0|||2018-07-21 16:21:35",
      "0|18610000001|460010000000001|2018|07|21|15-21-35|41000|22000|35004007800301|0000|||||0|||2018-07-21 15:21:35",
      "0|18610000002|460010000000002|2018|07|21|15-20-35|41000|22001|35004007800302|0000|||||0|||2018-07-21 15:20:35",
      "0|18610000003|460010000000003|2018|07|21|17-21-35|41001|22002|35004007800303|0000|||||0|||2018-07-21 17:21:35"))
    val rdd1: RDD[(String, String)] =rdd.map(x=>(x,""))
    rdd1.rapartation(2).saveAsHadoopFile("/Users/yeziming/test/saveDir", classOf[String], classOf[String],classOf[RDDMultipleTextOutputFormat])
  }
}
```  
   
然后我们运行一下程序，可以看到生成的文件(日期有点问题。。不管了)：  
```
yezm:saveDir yeziming$ ls
2018-24-28    _SUCCESS
yezm:saveDir yeziming$ cd 2018-24-28/
yezm:2018-24-28 yeziming$ ls -l
total 16
-rw-r--r--  1 yeziming  staff  222  7 28 11:24 11-24-44_part-00000
-rw-r--r--  1 yeziming  staff  222  7 28 11:24 11-24-44_part-00001
yezm:2018-24-28 yeziming$ cat 11-24-44_part-00000
0|18610000000|460010000000000|2018|07|21|16-21-35|41003|22002|35004007800300|0000|||||0|||2018-07-21 16:21:35
0|18610000002|460010000000002|2018|07|21|15-20-35|41000|22001|35004007800302|0000|||||0|||2018-07-21 15:20:35
yezm:2018-24-28 yeziming$ cat 11-24-44_part-00001
0|18610000001|460010000000001|2018|07|21|15-21-35|41000|22000|35004007800301|0000|||||0|||2018-07-21 15:21:35
0|18610000003|460010000000003|2018|07|21|17-21-35|41001|22002|35004007800303|0000|||||0|||2018-07-21 17:21:35
```  
可以看到，我们文件名称的拼接使用的是“val filename=dirName+"/"+fileName+"_"+name”，因此是时间+name的方式传入的，所以可以看出，name的值果然是part-0000n，后面的数字为partation的标号，所以我们在使用的过程中，也可以留下来默认的name的值的标号来区分不同的partation。  
### 正式代码编写  
下面，我们开始根据业务的正式的代码编写，经过了以上分析，写起来也很简单。(业务需求见上述4需求更改部分)   
1. SparkStreaming程序中使用saveAsHadoopFile  
我们在SparkStreaming程序中，使用saveAsHadoopFile算子。由于之前的数据是rdd，而saveAsHadoopFile需要的是pairRDD，因此，我们使用map将数据转换一下，数据内容作为key，空串“”作为value，这里需要与后续我们自定义的RDDMultipleTextOutputFormat生成文件名称的时候相对应。  
后续我们自定义的RDDMultipleTextOutputFormat，与文件基础路径、key格式、value格式一同传入saveAsHadoopFile算子中。
```scala
//...部分内容
 val saveDstream: DStream[String] =writeDStrem.repartition(numHdfsFile_Repartition)
    //保存处理后的数据到hdfs
    saveDstream.foreachRDD(rdd => {
      // driver端运行，涉及操作：广播变量的初始化和更新
      // 这里的数据就是一个批次生成一次，然后下发到不同的patition的时候数据是一样的
      val start_time = System.currentTimeMillis()
      if (rdd.isEmpty) {
        logInfo(" No Data in this batchInterval --------")
      } else {
        //这里，因为saveAsHadoopFile需要接受pairRDD，所以用map转换一下
        val a: RDD[(String, String)] =rdd.map(x=>(x,""))
        a.saveAsHadoopFile(hdfsPath+"/", classOf[String], classOf[String],classOf[RDDMultipleTextOutputFormat])
        competeTime(start_time, "Processed data write to hdfs")
      }
    })//foreachRDD
//...
```  
2. 自定义RDDMultipleTextOutputFormat继承MultipleTextOutputFormat[Any, Any]  
因为我们这里继承的是MultipleTextOutputFormat，已经帮我把getRecordWriter重写好了，所以我们就很方便简单的重写一个generateFileNameForKeyValue，来根据数据内容划分文件目录就好了。  
```scala
import java.text.SimpleDateFormat
import java.util.Date
import org.apache.hadoop.mapred.lib.MultipleTextOutputFormat
class RDDMultipleTextOutputFormat  extends MultipleTextOutputFormat[Any, Any]{
  private val pre_flag="\\|"
  private val HOURFORMAT = new SimpleDateFormat("HH-mm-ss")
  private val start_time = System.currentTimeMillis()
  private val curDay=new Date(start_time)
  private val fileName=HOURFORMAT.format(curDay)
  override def generateFileNameForKeyValue(key: Any, value: Any, name: String):String ={
    //0|18610000000|460010000000000|2018|07|21|16-21-35|41003|22002|35004007800300|0000|||||0|||2018-07-21 16:21:35
    val line = key.toString
    val split=line.split(pre_flag)
    //2018-01-24 12:58:16:@TODO 这里有个问题：如果time是空可能会报错，有时间再处理，目前此字段经过之前处理都非空
    val time=split(18)
    val ymd=time.substring(0,time.indexOf(" "))
    val hour=time.substring(time.indexOf(" ")+1,time.indexOf(":"))
    val service_date=ymd+"/"+hour+"/"+fileName+"-"+name.substring(name.length()-2)//.split("-")(0)
    service_date
  }
}
```  
在generateFileNameForKeyValue方法中，我们根据上面用map转换的“rdd.map(x=>(x,""))”，则key为数据内容。所以对key进行切分，取第18个字段的日期来作为目录生成的依据。并且我们截取默认name的后两位，来作为partation编号，所以我们最后生成的文件路径名为“文件基础路径（即上述saveAsHadoopFile中传入的hdfsPath）/业务年月日/业务小时/文件写入时间_part编号”。  
  
所以，最终显示样式可如下：
```
//这次就是数据有延迟，12:12才写入12点目录中
dcos@d8pccdsj3[~]$hadoop fs -ls /encrypt_data/4g_info_c60/2018-07-28/12
Found 40 items
-rw-r--r--   3 user1 cgroup    6541003 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-05-00
-rw-r--r--   3 user1 cgroup    6555677 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-05-01
-rw-r--r--   3 user1 cgroup    6538441 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-05-02
-rw-r--r--   3 user1 cgroup    6567709 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-05-03
-rw-r--r--   3 user1 cgroup    6570481 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-05-04
-rw-r--r--   3 user1 cgroup   29486262 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-22-00
-rw-r--r--   3 user1 cgroup   29477123 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-22-01
-rw-r--r--   3 user1 cgroup   29476448 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-22-02
-rw-r--r--   3 user1 cgroup   29425074 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-22-03
-rw-r--r--   3 user1 cgroup   29435407 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-22-04
-rw-r--r--   3 user1 cgroup   66757368 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-43-00
-rw-r--r--   3 user1 cgroup   66862913 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-43-01
-rw-r--r--   3 user1 cgroup   66854597 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-43-02
-rw-r--r--   3 user1 cgroup   66809042 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-43-03
-rw-r--r--   3 user1 cgroup   66777279 2018-07-28 12:12 /encrypt_data/4g_info_c60/2018-07-28/12/12-12-43-04
-rw-r--r--   3 user1 cgroup   73288280 2018-07-28 12:13 /encrypt_data/4g_info_c60/2018-07-28/12/12-13-03-00
-rw-r--r--   3 user1 cgroup   73285173 2018-07-28 12:13 /encrypt_data/4g_info_c60/2018-07-28/12/12-13-03-01
-rw-r--r--   3 user1 cgroup   73355198 2018-07-28 12:13 /encrypt_data/4g_info_c60/2018-07-28/12/12-13-03-02
-rw-r--r--   3 user1 cgroup   73337561 2018-07-28 12:13 /encrypt_data/4g_info_c60/2018-07-28/12/12-13-03-03
-rw-r--r--   3 user1 cgroup   73320851 2018-07-28 12:13 /encrypt_data/4g_info_c60/2018-07-28/12/12-13-03-04
``` 
大功告成！  

# 补充-需求2实现-validateOutputSpecs参数-2018-09-03  
## 需求描述  
数据从很多接口接入，每条数据的第一个字段标明接口来源，需使用spark streaming程序，实时根据将数据根据不同的接口标注，写入到HDFS对应的目录的对应文件之中。  
数据样式：  
```
60,2018-08-14 10:16:50.062211990,2018-08-14 10:17:28.652398109,0,6......
61,2018-08-14 10:16:47.095155954,2018-08-14 10:17:21.435997962,0,52......
62,2018-08-14 10:17:05.457761049,2018-08-14 10:17:28.077723979,0,6......
...
```  
## 解决  
此需求跟上述问题解决方案一摸一样，也是重写自己的RDDMultipleTextOutputFormat即可。代码示例如下:  
### 重写的RDDMultipleTextOutputFormat 
```scala
import java.text.SimpleDateFormat
import java.util.Date
import org.apache.hadoop.mapred.lib.MultipleTextOutputFormat

class RDDMultipleTextOutputFormat  extends MultipleTextOutputFormat[Any, Any]{
  private final val PREFLAG=","
  private final val YMDFORMAT = new SimpleDateFormat("yyyyMMdd")
  private final val YMDHMFORMAT=new SimpleDateFormat("yyyyMMddHHmm")
  private val cur_time = System.currentTimeMillis()
  private val curDay=new Date(cur_time)
  private val dirName=YMDFORMAT.format(curDay)
  private val fileName=YMDHMFORMAT.format(curDay)
  private val filePrefix="xx-xxxxx-events-"
  private val fileSuffix=".txt"

  override def generateFileNameForKeyValue(key: Any, value: Any, name: String):String ={
    val line = key.toString
    val split=line.split(PREFLAG)
    val filePath=dirName+"/"+filePrefix+split(0)+"-"+fileName+name.substring(name.length()-2)+fileSuffix
    filePath
  }
}
```  
### 本机测试调用  
```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD
/**
  * Create by Liv on 2018/8/31.
  */
object TestXXOriFile {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local").setAppName("flatMap Demo")
    /*
    不设置此参数，会报错：
    Exception in thread "main" org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory file:/Users/yeziming/test/xxdata already exists
    此参数含义：
    若设置为true，saveAsHadoopFile会验证输出目录是否存在。
    虽然设为false可以忽略文件存在的异常，但建议使用 Hadoop文件系统的API手动删除输出目录。
    当通过Spark Streaming的StreamingContext时本参数会被忽略，因为当进行checkpoint恢复时会重写已经存在的文件。
    */
    sparkConf.set("spark.hadoop.validateOutputSpecs","false")

    val sc = new SparkContext(sparkConf)
    val fileName = "/Users/yeziming/IdeaProjects/SparkAclKafka/resource/table_data.properties"
    val rdd: RDD[String] =sc.textFile(fileName)
    val pairRdd: RDD[(String, Null)] = rdd.map(x=>(x,null))
    pairRdd.repartition(2).saveAsHadoopFile("/Users/yeziming/test/xxdata"+"/",classOf[String],classOf[String],classOf[RDDMultipleTextOutputFormat])
  }
}
```  
为了测试RDDMultipleTextOutputFormat的文件目录格式写的是否正确，所以在本机使用sparkcore写spark程序测试了一下，但是在测试的过程中，在最初的时候，没有设置spark.hadoop.validateOutputSpecs，因此每次执行一次之后，如果不删除原来的目录，则会报错如下：  
```
Exception in thread "main" org.apache.hadoop.mapred.FileAlreadyExistsException: Output directory file:/Users/yeziming/test/xxdata already exists
	at org.apache.hadoop.mapred.FileOutputFormat.checkOutputSpecs(FileOutputFormat.java:132)
```  
是因为在spark中也是用的hadoop那一套，所以原来mr中保存文件的时候，其中的org.apache.hadoop.mapred.FileOutputFormat.checkOutputSpecs方法会检查输出目录是否合法，如果没有指定，则抛出InvalidJobConfException，文件已经存在则抛出FileAlreadyExistsException。  
因此，因为再次执行，目录是存在的，所以会报出FileAlreadyExistsException的错误。  
可是之前直接通过spark streaming程序的时候，并没有出现这个错误。查询这个错误怎么解决的时候，大部分都说自己删除目录即可。后来通过查询spark的参数设置，发现了“spark.hadoop.validateOutputSpecs”这个参数。其说明如下：  
```
spark.hadoop.validateOutputSpecs
默认是true。若设置为true，saveAsHadoopFile会验证输出目录是否存在。虽然设为false可以忽略文件存在的异常，但建议使用 Hadoop文件系统的API手动删除输出目录。当通过Spark Streaming的StreamingContext时本参数会被忽略，因为当进行checkpoint恢复时会重写已经存在的文件。  
If set to true, validates the output specification (e.g. checking if the output directory already exists) used in saveAsHadoopFile and other variants. This can be disabled to silence exceptions due to pre-existing output directories. We recommend that users do not disable this except if trying to achieve compatibility with previous versions of Spark. Simply use Hadoop's FileSystem API to delete output directories by hand. This setting is ignored for jobs generated through Spark Streaming's StreamingContext, since data may need to be rewritten to pre-existing output directories during checkpoint recovery.
```  
所以，我们再用spark streaming程序的时候，不会出现这个问题。  
因此，如果后续使用spark core程序的时候，如果出现了此问题，想要覆盖目录，则可设置此参数。  
注意：文件名称不同，文件是不会覆盖的。  
通过设置了这个参数之后，再次运行我上面写的程序，目录下的文件如下：  
```
yezm:20180903 yeziming$ ls
xx-xxxxx-events-60-20180903132300.txt
xx-xxxxx-events-60-20180903132301.txt
xx-xxxxx-events-60-20180903132400.txt
xx-xxxxx-events-60-20180903132401.txt
xx-xxxxx-events-60-20180903134300.txt
xx-xxxxx-events-60-20180903134301.txt
xx-xxxxx-events-61-20180903132300.txt
xx-xxxxx-events-61-20180903132301.txt
xx-xxxxx-events-61-20180903132400.txt
xx-xxxxx-events-61-20180903132401.txt
xx-xxxxx-events-61-20180903134300.txt
xx-xxxxx-events-61-20180903134301.txt
...
```  
即，我的输出文件是以分钟命名的，所以在不同的分钟运行多次，会在那个目录里面生成多个文件，并不会将文件覆盖。  
但是，一般情况下，是建议不设置这个参数的，因为需要将hdfs中的旧文件删除的话。  
具体spark中删除HDFS目录，有个文章中写了但是没有测试，可以参考[spark删除hdfs文件](https://blog.csdn.net/zhouyan8603/article/details/51658950)。  
  
---

至此，本篇内容完成。  
本文由2018-07-02开始建立文档，于2018-07-28整理完成，几乎历时一个月，期间断断续续抽时间整理成文，很有收获。  
2018-09-03更新6补充需求2。
