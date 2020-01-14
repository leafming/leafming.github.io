---
layout: post
title: "AthenaX基于SQL的大规模流分析平台-开源项目简要分析"
date: 2019-06-19
updated: 2019-06-19
description: 本文主要关于Uber开源的AthenaX基于SQL的大规模流分析平台的项目简要分析。
categories:
- BigData
tags:
- Flink
---
> 文主要关于Uber开源的AthenaX基于SQL的大规模流分析平台的简单整理。想要学习一下基于SQL流处理平台的思路，将看到的博客和文章简单整理了一下。  
  
# 主要参考  
这次把主要参考文章列在了最前面，主要是基本上此文就是在前人的基础上整理在了一起，此文原创很少，参考链接如下:  
前人博客-Uber Athenax项目核心技术点剖析：[https://blog.csdn.net/yanghua_kobe/article/details/78573578](https://blog.csdn.net/yanghua_kobe/article/details/78573578)  
介绍文章-Introducing AthenaX, Uber Engineering’s Open Source Streaming Analytics Platform：[https://eng.uber.com/athenax/](https://eng.uber.com/athenax/)  
官网说明：[https://athenax.readthedocs.io/en/latest/](https://athenax.readthedocs.io/en/latest/)  
博客-如何构建一个flink sql平台:[https://my.oschina.net/qiangzigege/blog/2252975](https://my.oschina.net/qiangzigege/blog/2252975)  
  
# AthenaX特点  
- Streaming SQL  
production-ready流处理分析，支持Filtering, projecting和combining操作。  
支持基于processing time和event time的[GroupWindow](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/table/sql.html#group-windows)聚合。  
支持定制化SQL：用户定义函数（UDF）、用户定义聚合函数（UDAF）和用户定义表函数（UDTF-测试）。  
- 高效的执行过程  
为了满足Uber业务的规模化要求，AthenaX 对 SQL 查询进行了编译与优化，并将其交付至分布式流应用程序当中以确保仅利用 8 套 YARN 容器即可实现每秒数百万次的消息处理操作。  
- 监控和自动故障恢复  
AthenaX 以端到端方式管理各应用程序，具体包括持续监控其运行状态、根据输入数据大小自动进行规模伸缩，且可顺利在节点或者整体数据中心发生故障时通过故障转移保证业务的正常恢复。  
- 资源估算和自动缩放-AthenaX jobs自动扩展等  
AthenaX根据查询和输入数据的吞吐量估算vcores和内存的数量。AthenaX master会持续监视每个作业的watermarks和垃圾回收统计信息，并在必要时重新启动它们。Flink的容错性机制能够保障job运行正确的结果。  
  
# 依赖组件  
- AthenaX构建在Apache Calcite以及Apache Flink之上  
Apache Calcite 是一款开源SQL解析工具, 可以将各种SQL语句解析成抽象语法术AST(Abstract Syntax Tree), 之后通过操作AST就可以把SQL中所要表达的算法与关系体现在具体代码之中。  
Apache Flink是一个面向分布式数据流处理和批量数据处理的开源计算平台,它能够基于同一个Flink运行时(FlinkRuntime),提供支持流处理和批处理两种类型应用的功能。  
- 采用YARN集群来管理Job  
- LevelDB作为持久化存储  
LevelDB是Google开源的持久化KV单机数据库,具有很高的随机写,顺序读/写性能,但是随机读的性能很一般,也就是说,LevelDB很适合应用在查询较少,而写很多的场景。  
  
# 架构说明  
![2019-06-19-AthenaX架构](https://github.com/leafming/bak/blob/master/images/flink/2019-06-19-AthenaX架构.png?raw=true)  
AthenaX平台一般架构如上图。  
Data Sources  
Output:AthenaX支持多种数据源和输出。   
AthenaX platform  
- AthenaX master：管理job的全生命周期。  
- catalogs 和connectors：描述 AthenaX jobs如何与外部组件联系。
1. Catalog和connectors是可插拔的。Catalog指定了如何将SQL中的表映射到source或sink。Connectors定义AthenaX如何与外部系统交互。（如发布到Kafka主题或进行RPC调用）  
2. 用户自定义的Catalogs必须实现AthenaXTableCatalogProvider。Connectors必须实现DataSinkProvider或者TableSourceConverter。例子：https://github.com/uber/AthenaX/tree/master/athenax-vm-connectors/athenax-vm-connector-kafka/src/main/java/com/uber/athenax/vm/connectors/kafka  
  
# AthenaX中的一些概念  
Job：流式分析应用程序。包括SQL、运行集群、所需资源。  
Catalog：描述了如何将SQL中的table转换为data source或data sink。  
Cluster：运行AthenaX job 的YARN cluster。  
Instance：通过AthenaX job 转换而来的运行在特定集群上的Flink application。  
  
# AthenaX的工作流程  
1. 用户使用SQL开发作业，通过 REST APIs并将其提交给AthenaX master。  
2. AthenaX master验证查询并将其编译为Flink作业。  
	1. 为了有效地执行Athenax jobs，Athenax使用Flink's Table and SQL APIs将AthenaX jobs编译成本地的Flink应用程序。Athenax将catalogs和parameters（例如并行性）以及job的SQL编译为一个Flink application，形成Flink里的JobGraph。  
	2. 【未来】Athenax支持用户自定义函数的SQL。用户可以指定附加的JAR随SQL一起加载。为了安全地编译它们，Athenax在专用的过程中编译SQL。但是，在当前版本中，该功能（尤其是对UDF JAR的本地化）尚未完全实现。  
3. AthenaX master会将其打包、部署到YARN集群中执行。部署完成后，Athenax主机会持续监控作业的状态，并在出现故障时将其恢复。  
	1. Athenax使用两种状态描述作业的运行状态。第一个状态是desired state，它描述了集群和作业所需的启动资源。第二个是actual state，每个job的已创建instance的资源。  
	2. watchdog周期性的计算actual state并将其与desired state进行比较，AthenaX master可以根据差异启动或终止相应的Yarn application，这种情况下启动的作业只是对failed job的恢复。Auto scaling的处理也类似。需要注意，所有的actual state都是“soft states”，也就是说，它们可以通过Yarn cluster来获取。这样的设计允许AthenaX控件在多个区域中运行，前提是（1）底层持久数据存储在部署的区域中可用，并且（2）watchdog知道多个可用区域中的活动主机。  
4. 作业开始处理数据并将结果输出到外部系统（例如，Kafka）。  
  
# AthenaX核心技术点  
Athenax它提供了一个对Flink进行扩展以利用其运行时的一种机制。  
  
## 准备工作  
下载源码，编译：  
问题：Class "ExtendedJobDefinition, JobDefinition, JobDefinitionDesiredstate and JobDefinitionResource" are not in package   
解决：The "missing" classes are generated during the compilation process. You can find them once you run mvn clean compile install。  
  
## AthenaX主要分为以下几个模块  
athenax-backend：项目的后端服务实现。  
athenax-vm-compiler：主要负责将SQL编译为Flink作业。  
athenax-vm-api：Athenax提供给用户的去实现的一些API接口。  
athennax-vm-connectors：开放给用户 扩展的连接器。  
  
## athenax-backend  
项目的后端服务实现，提供了一个运行时实例。  
其主要启动步骤分为两步：  
1. 启动一个web server(com.uber.athenax.backend.server.WebServer)，用来接收restful的各种服务请求；  
这里的web server，事实上是一个Glashfish中的Grizzly所提供的一个轻量级的http server（Glashfish:Java EE应用服务器的实现;Grizzly:基于Java NIO实现的服务器），它也具备处理动态请求（web container，Servlet）的能力。  
RESTful API这块，AthenaX使用了swagger这一API开发框架来提供部分代码（实体类/服务接口类）的生成。  
web server接收用户的RESTful API请求，这些API可以分成三类：  
（1）Cluster: 集群相关的信息  
（2）Instance: Job运行时相关的信息  
（3）Job: 作业本身的信息；  
2. 启动了一个ServerContext（上下文-com.uber.athenax.backend.server.ServerContext），它封装了一些核心对象，是服务的具体提供者:  
job store：一个基于LevelDB的job元数据存储机制；  
job manager：注意这与Flink的JobManager没有关系，这是AthenaX封装出来的一个对象，用于对SQL Job进行管理；  
instance manager：一个instance manager管理着部署在YARN集群上所有正在被执行的job；  
watch dog：提供了对job的状态、心跳的检测，以适时进行failover；  
...  
  
## athenax-vm-compiler  
主要实现将SQL编译为Flink作业。分为3部分：  
planer：计划器，该模块的入口，它会顺序调用parser、validator、executor，最终得到一个称之为作业编译结果的JobCompilationResult对象；  
parser：编译器，这里主要是针对其对SQL的扩展提供相应的解析实现，主要是对Calcite api的实现，最终得到SqlNode集合SqlNodeList；  
validator：校验；  
executor：真正完成所谓的”编译“工作，这里编译之所以加引号，其实只是借助于Flink的API得到对应的JobGraph；  
这里，值得一提的是其”编译“的实现机制。AthenaX最终是要将其SQL Job提交给Flink运行时去执行，而对Flink而言JobGraph是其唯一识别的Job描述的对象，所以它最关键的一点就是需要得到其job的JobGraph。  
整体调用过程：  
1. Athenax Jobmanger编译  
```
//com.uber.athenax.backend.server.jobs.JobManager#compile Athenax Jobmanager编译
public JobCompilationResult compile(JobDefinition job, JobDefinitionDesiredstate spec) throws Throwable {
  Map<String, AthenaXTableCatalog> inputs = catalogProvider.getInputCatalog(spec.getClusterId());
 AthenaXTableCatalog output = catalogProvider.getOutputCatalog(spec.getClusterId(), job.getOutputs());
 Planner planner = new Planner(inputs, output);
 return planner.sql(job.getQuery(), Math.toIntExact(spec.getResource().getVCores()));
}
```  
2. Planner sql方法，返回JobCompilationResult，其中JobDescriptor是其Job业务相关的信息。  
```
public JobCompilationResult sql(String sql, int parallelism) throws Throwable {
//解析
 SqlNodeList stmts = parse(sql);
 //校验
 Validator validator = new Validator();
 validator.validateQuery(stmts);
 //Job相关信息
 JobDescriptor job = new JobDescriptor(
//输入源Catalog-将SQL中的table转换为data source
 inputs,
 //自定义fun
 validator.userDefinedFunctions(),
 //Catalog输出源-将SQL中的table转换为data sink
 outputs,
 //并行度
 parallelism,
 //SQL
 validator.statement().toString());
 // 调用excutor，使用Flink的API得到对应的JobGraph,res.jobGraph()
 // 说明：AthenX采用了运行时执行构造命令行执行JobCompiler的方法，然后利用socket+标准输出重定向的方式，来模拟UNIX PIPELINE，可能没必要这么绕弯路，直接调用就行了。
 // uses contained executor instead of direct compile for: JobCompiler.compileJob(job); 实际上即调用JobCompiler.compileJob(job)
 CompilationResult res = new ContainedExecutor().run(job);
 if (res.remoteThrowable() != null) {
throw res.remoteThrowable();
 }
return new JobCompilationResult(res.jobGraph(),
 validator.userDefinedFunctions().values().stream().map(Path::new).collect(Collectors.toList()));
}
```  
说明：  
（1）以上具体的触发机制，采用了运行时（com.uber.athenax.vm.compiler.executor.ContainedExecutor）执行构造命令行执行JobCompiler的方法，然后利用套接字+标准输出重定向的方式，来模拟UNIX PIPELINE，可以理解成直接调用JobCompiler.compileJob(job)。  
（2）parse的代码生成  
值得一提的是，parser涉及到具体的语法，这一块为了体现灵活性。AthenaX将解析器的实现类跟SQL语法绑定在一起通过fmpp(文本模板预处理器)的形式进行代码生成。  
fmpp是一个支持freemark语法的文本预处理器。  
  
3. JobCompiler（com.uber.athenax.vm.compiler.executor.JobCompiler）的compileJob方法，构造ComplicationResult。  
```
//构造CompilationResult（核心：构造JobGraph）
public static CompilationResult compileJob(JobDescriptor job) {
  //flink env
 StreamExecutionEnvironment execEnv = StreamExecutionEnvironment.createLocalEnvironment();
 StreamTableEnvironment env = StreamTableEnvironment.getTableEnvironment(execEnv);
 execEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
 CompilationResult res = new CompilationResult();
 try {
    //getJobGraph
 res.jobGraph(new JobCompiler(env, job).getJobGraph());
 } catch (IOException e) {
    res.remoteThrowable(e);
 }
  return res;
}
```  
  
4. 核心在于上面的getJobGraph（com.uber.athenax.vm.compiler.executor.JobCompiler#getJobGraph）方法。之前JobDescriptor是其Job业务相关的信息，这里动态设置非固定部分。  
```
//构造JobGraph 利用Flink的Table&SQL API编写的Table&SQL 程序模板
JobGraph getJobGraph() throws IOException {
  StreamExecutionEnvironment exeEnv = env.execEnv();
 exeEnv.setParallelism(job.parallelism());
 //AthenaX动态设置注册registerUdfs和registerInputCatalogs
 this
 .registerUdfs()
      .registerInputCatalogs();
 //flink table api
 Table table = env.sqlQuery(job.sql());
 for (String t : job.outputs().listTables()) {
    table.writeToSink(getOutputTable(job.outputs().getTable(t)));
 }
  //直接调用StreamExecutionEnvironment#getStreamGraph就可以自动获得JobGraph对象
 StreamGraph streamGraph = exeEnv.getStreamGraph();
 //JobGraph的生成还是由Flink 自己提供的，而AthenaX只需要拼凑并触发该对象的生成
 return streamGraph.getJobGraph();
}
```  
其中调用env.sql()这个方法说明它本质没能真正脱离Flink Table&SQL。  
设置完成之后，通过调用StreamExecutionEnvironment#getStreamGraph就可以自动获得JobGraph对象，因此JobGraph的生成还是由Flink 自己提供的，而AthenaX只需要拼凑并触发该对象的生成。  生成后会通过flink的yarn client实现（com.uber.athenax.backend.server.yarn.JobDeployer），将JobGraph提交给YARN集群，并启动Flink运行时执行Job。  
  
## athenax-vm-api  
这个模块就是Athenax提供给用户的去实现的一些API接口，它们是：  
function：各种函数的rich化(open/close方法对)扩展；  
catalog：table / source、sink的映射；  
sink provider：sink的扩展接口。  
  
## athennax-vm-connectors  
开放给用户去扩展的连接器，目前只提供了kafka这一个连接器的实现。  
  
  
---    
至此，本篇内容完成。  
