---
layout: post
title: "Docker+Mesos+Marathon监控方法使用总结"
date: 2018-06-15
updated: 2018-06-15
description: 本文主要关于Docker、Mesos、Marathon的监控方法的总结，主要使用java方法和相关restful方式获取监控数据。
categories:
- Cloud
tags:
- Docker
- Mesos
- Marathon
---
> 本文主要关于Docker、Mesos、Marathon的监控方法的总结。  
> **注意: 本文中使用的版本是Docker1.12.6、Mesos1.3.0、Marathon1.4.5。**  
  
Docker+Mesos+Marathon所搭建的平台监控所用到的后台方法已经基本上写完了，现在用一些时间总结下相关监控方法，满满用心总结的，不过用到的不止这些，还差很多以后再另开文章补充。在总结过程中也重新梳理了很多东西，现在分享出来，希望对大家有用啦，也省的自己以后忘了～哈哈。  
  
# Docker管理-容器相关  
## 前期准备：开启Docker远程访问  
Docker daemon可以有3种远程访问方式的socket：unix、tcp和fd。  
其中Docker支持使用restful api远程获取信息，但是需要提前开启一个Tcp Socket，这样才能实现远程通信。  
默认情况下，Docker守护进程Unix socket（/var/run/docker.sock）来进行本地进程通信，而不会监听任何端口，因此只能在本地使用docker客户端或者使用Docker API进行操作,并且需要root权限或者是docker group成员。如果想在其他主机上操作Docker主机，就需要让Docker守护进程打开一个Tcp Socket，这样才能实现远程通信。  
补充说明：可以使用HTTPs方式，不过目前没有进行配置，可以后续更改。但是，如果你用和Https的socket，需要注意的是只支持TLS1.0及以上的协议，SSLv3或者更低的是不支持的。  
官方网站描述如下[Daemon socket option](https://docs.docker.com/v1.12/engine/reference/commandline/dockerd/)。  
我们来使用开启TCP Socket的方式。  
需要更改docker配置文件，修改OPTIONS，具体方式如下：  
1. 如果找不到docker配置文件，可以以下述方式查找：  
找到docker.service文件：  
systemctl show --property=FragmentPath docker  
```  
---@---[~]$systemctl show --property=FragmentPath docker 
FragmentPath=/usr/lib/systemd/system/docker.service
```   
找到docker.service文件并查看，发现docker文件位置：/etc/sysconfig/docker  
![DockerService内容](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-11-dockerservice内容.png?raw=true)  
2. 找到文件后编辑文件docker  
在OPTIONS=后加上 -H=unix:///var/run/docker.sock -H=tcp://0.0.0.0:端口  
说明：  
1、确定sock文件位置：/var/run/docker.sock  
2、可用指定ip：指定端口  
3、重启docker  
![DockerService内容](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-11-dockertcpSocket配置.png?raw=true)  
说明：  
有资料表述，一般情况下执行以下操作即可：  
1、vim /etc/sysconfig/docker（更改docker文件）  
2、在OPTIONS=后加上 -H=unix:///var/run/docker.sock -H=tcp://0.0.0.0:端口即可  
如若不成功，可尝试下述操作：  
1、vim /usr/lib/systemd/system/docker.service （更改docker.service文件）  
2、直接在ExecStart=/user/bin/dockerd后添加-H=tcp://0.0.0.0:端口 -H=unix:///var/run/docker.sock(注意顺序)    
3、执行命令：systemctl daemon-reload 、systemctl restart docker  
3. 测试使用：  
配置之后，尝试使用docker restful api来获取信息。  
![2018-06-11-dockerrestful获取尝试](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-11-dockerrestful获取尝试.png?raw=true)  
或者直接在机器上尝试查询：“sudo docker -H IP:端口 命令”来测试是否能够取到内容。  
    
## Docker-Client的选择  
Docker-client的前期准备已经完毕，下面进行DockerAPI的使用。  
### Docker版本与Docker API的对应关系 和 相关Library  
Docker提供了Docker Engine API 去连接Docker daemon 来进行Docker的管理。Docker Engine API是Restful API，可以通过http客户端，比如说wget或者curl等来使用。  
Docker官方提供Python和Go的SDK，可以直接使用。而在我们这里使用的是java语言，所以可以直接使用restful API的方式来获取相关信息或者直接使用他人封装好的Library。  
#### Docker版本和API版本对应  
Docker的不同版本对用不同版本的API。具体以一个表格的形式列了出来，具体可以参考官方网站所写的内容:[Docker版本和API对应表-API version matrix](https://docs.docker.com/develop/sdk/#docker-ee-and-ce-api-mismatch)。  
通过查看这个表，我们知道，我现在用的docker版本是1.12.6，所以来说，所对应的API版本为1.24。  
关于Docker Engine API 1.24的使用，可以参考官网[Docker Engine API 1.24](https://docs.docker.com/engine/api/v1.24/)。  
#### Docker unofficial libraries  
Docker中出了官方提供的Python和Go的SDK之外，仍然有一些非官方的libraries可以使用，并且可使用的语言类型更加的广泛。关于可以使用的Library，官方在官网上也有所推荐，见[官网Unofficial libraries](https://docs.docker.com/develop/sdk/#unofficial-libraries)所示。  
目前，项目里面并没有直接使用Docker Engine API，而是使用Java的第三方Library,第三方Library也是对Docker Engine API的一个封装。关于Java的Library，Docker官方推荐有三种，docker-client、docker-java、docker-java-api，而这次在项目中，使用的是[spotify/docker-client](https://github.com/spotify/docker-client)，在下面会将部分所用Library中的方法和Docker Engine API对应着介绍一下。  
  
## Docker-Client的使用  
我们使用的是[spotify/docker-client](https://github.com/spotify/docker-client/blob/master/docs/user_manual.md)，在github上有详细的使用说明，直接点击链接即可。所以就不详细介绍了，下面就直接列出来使用的几个方法记录一下。
### 引入jar包  
我们使用的是spotify/docker-client，可以直接通过maven或者jar包的方式引入，maven如下：  
```  
<dependency>
  <groupId>com.spotify</groupId>
  <artifactId>docker-client</artifactId>
  <version>LATEST-VERSION</version>
</dependency>
```  
针对Docker1.12.6，选择的是8.9.1版本。  
  
### 开始使用  
因为项目中主要用到的是监控，所以基本上都是从服务器上查询的方法，关于对于创建容器、删除容器等操作，因为容器直接归于marathon管理，所以不在Docker这里进行容器创建、删除等操作了。  
  
#### 创建和关闭Docker连接  
##### 创建连接
在之前的操作，开启了Docker RPC端口之后，可以在其他机器上创建远程Docker连接。  
而且我们也没有开启https，所以可以直接使用如下方式：  
传入容器宿主机IP+端口构成的uri，建立和宿主机的连接。  
```  
// use the builder
String dockerUri ="http://"+hostIp+":"+port;
final DockerClient docker = DefaultDockerClient.builder().uri(URI.create(dockerUri)).build();
```  
除了这种方式，还有其它几种方式创建docker连接，详细参考[Creating a docker-client](https://github.com/spotify/docker-client/blob/master/docs/user_manual.md#creating-a-docker-client)。  
##### 关闭Docker连接  
关闭连接直接通过创建的DockerClient docker连接对象，调用close方法即可。  
```java  
// Close the docker client
docker.close();
```  
#### Docker操作  
##### 容器列表  
获取宿主机上的容器列表，方法如下：  
这个方法能够列出宿主机上全部**正在运行**的容器的列表。  
```java  
List<Container> containers = docker.listContainers(DockerClient.ListContainersParam.allContainers());
```  
通过这个方法，可以返回List<Container>，其中的Container对象，可以取到的容器的id、名称、镜像信息、网络信息、状态等内容。  
  
![container属性](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-container属性.png?raw=true)  
  
并且，其中包括网络信息可以通过networkSettings取到，包括内容如下：  
  
![networkSettings](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-networkSettings属性.png?raw=true)  
  
##### 容器详情  
上述获取容器列表的过程中取到的容器信息并不详细，如果想看容器的详细信息可使用inspect方法,需要传入的内容是容器的id，不过通过测试，不需要传入全部的完整id，部分。  
```java  
ContainerInfo info = docker.inspectContainer("containerID");
```  
通过这个方法可以获取到ContainerInfo对象，其中包括容器的详细信息，如下：  
![containerInfo](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-containerInfo属性.png?raw=true)  
###### 容器状态对象  
通过 ContainerState state = info.state();可以取到容器的状态对象ContainerState，其中属性如下。  
![ContainerState](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-ContainerState.png?raw=true)  
取对象的state.status()，可以取到当前容器的状态，容器状态总共包括：created、restarting、running、paused、exited、dead共5种。  
###### 容器配置信息  
通过 ContainerConfig config = containerInfo.config();取到容器的配置信息，其中属性包括如下。  
![ContainerConfig](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-ContainerConfig属性.png?raw=true)  
通过ContainerConfig可以获取到容器的hostname、env配置、启动容器的命令、所用镜像、工作目录、labels等信息。  
###### 容器HostConfig  
通过 HostConfig hostConfig = containerInfo.hostConfig();可以取到容器的host配置信息，其中属性如下。  
![HostConfig](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-HostConfig.png?raw=true)  
通过HostConfig对象，可以获取到容器和宿主机之间映射的目录、端口等信息，以及所给容器分配的cpu core数量（hostConfig.cpuShares()取到的数值除以1024）、memory（hostConfig.memory()）、swap的大小（hostConfig.memorySwap()-hostConfig.memory()）等信息。  
###### 容器网络信息  
通过NetworkSettings networkSettings = containerInfo.networkSettings();可以取到容器的网络信息，此对象和容器列表中取到的内容一致。通过它可以取到容器的网络模式、网关、ip、mac等信息。  
##### 容器内进程  
上述操作均是获取容器的基本信息，如果获取容器内当时的进程信息，可以使用如下方法：  
传入的ps_args为aux，和在容器内执行“ps aux”命令取到的数据一致，即“[USER, PID, %CPU, %MEM, VSZ, RSS, TTY, STAT, START, TIME, COMMAND]”等内容。  
通过如下方法能取到TopResults对象，此对象内包括属性为process和titles，通过获取其中的process，可得到容器内的进程情况。  
```java  
//例子
//final TopResults topResults = docker.topContainer("containerID", "ps_args");
TopResults topResults = docker.topContainer("containerID", "aux");
ImmutableList<ImmutableList<String>> processes = topResults.processes();
```  
##### 容器资源使用情况  
在命令行中，我们可以通过docker stats来查看docker容器资源使用情况的信息，通过命令行能查询到如下内容：  
```  
CONTAINER    CPU %    MEM USAGE / LIMIT    MEM %    NET I/O    BLOCK I/O
```    
docker-client也提供了方法，来通过如下方式获取docker stats信息：  
```java
ContainerStats stats = docker.stats("containerID");
```  
通过此方法，能够取到ContainerStats对象，其中包含内容如下：  
![ContainerStats](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-13-ContainerStats.png?raw=true)  
但是，我们取到的数据并不能直接使用，而需要通过计算，来得到更加直观的数据，下面进行一下详细说明。  
而计算的过程，主要参考[“dockerstats命令源码分析结果”](https://www.2cto.com/net/201701/582048.html)文章中，对于docker stats源码的分析。  
###### CPU利用率计算  
1. 计算分析  
在上述参考文章中提到:  
> docker daemon会记录这次读取/sys/fs/cgroup/cpuacct/docker/[containerId]/cpuacct.usage的值，作为cpu_total_usage；并记录了上一次读取的该值为pre_cpu_total_usage；读取/proc/stat中cpu field value，并进行累加，得到system_usage;并记录上一次的值为pre_system_usage；读取/sys/fs/cgroup/cpuacct/docker/[containerId]/cpuacct.usage_percpu中的记录，组成数组per_cpu_usage_array。  
  
	因此，可以得到docker stats计算Cpu Percent的算法如下： 
	```  
cpu_delta = cpu_total_usage - pre_cpu_total_usage;
system_delta = system_usage - pre_system_usage;
CPU % = ((cpu_delta / system_delta) * length(per_cpu_usage_array) ) * 100.0
```  
2. 代码实现  
所以说，将上述内容中得到的公式所需数据与我们通过取到的数据相对应，通过已经得到的stats对象，用方法可以取到两个CpuStats对象：  
```java  
CpuStats cpuStats = stats.cpuStats();
CpuStats precpuStats = stats.precpuStats();
```  
然后我们计算cpu利用率，就可以这样来计算：  
```java  
String cpuPercent="---";
if (null!=cpuStats&&null!=precpuStats){
	/**
 	* 计算cpu利用率 %
    	* cpuDelta = cpuTotalUsage - preCpuTotalUsage;
      	* systemDelta = systemUsage - preSystemUsage;
      	* CPU % = ((cpuDelta / systemDelta) * length(per_cpu_usage_array) ) * 100.0
    	*/
    	Long cpuTotalUsage=cpuStats.cpuUsage().totalUsage();
    	Long preCpuTotalUsage =precpuStats.cpuUsage().totalUsage();
    	Long cpuDelta =cpuTotalUsage - preCpuTotalUsage;
    	Long systemUsage =cpuStats.systemCpuUsage();
    	Long preSystemUsage =precpuStats.systemCpuUsage();
    	Long systemDelta =systemUsage-preSystemUsage;
    	Integer percpuUsageListSize = cpuStats.cpuUsage().percpuUsage().size();
    	cpuPercent = ((cpuDelta.floatValue() / systemDelta.floatValue()) * percpuUsageListSize.floatValue()) * 100+"%";
    }
```  
  
###### memory利用率计算  
1. 计算分析  
在上述参考文章中提到:
> 读取/sys/fs/cgroup/memory/docker/[containerId]/memory.usage_in_bytes的值，作为mem_usage；如果容器限制了内存，则读取/sys/fs/cgroup/memory/docker/[id]/memory.limit_in_bytes作为mem_limit，否则mem_limit = machine_mem。  
  
	因此可以得到docker stats计算Memory数据的算法如下：  
	```  
MEM USAGE = mem_usage
MEM LIMIT = mem_limit
MEM % = (mem_usage / mem_limit) * 100.0
```  
2. 代码实现  
通过已经得到的stats对象，用方法可以取到MemoryStats对象。  
```java  
MemoryStats memoryStats = stats.memoryStats();
```  
然后根据公式来计算memory利用率：  
```java  
 String memUsageFormat="---";
 String memLimitFormat="---";
 String memPercent="---";
 if (null!=memoryStats){
 	/**
         * 计算memory利用率 %
         * MEM USAGE = mem_usage
           MEM LIMIT = mem_limit
           MEM % = (mem_usage / mem_limit) * 100.0
         */
           Long memUsage= memoryStats.usage();
           Long memLimit=memoryStats.limit();
           memUsageFormat= Utils.getDynamicSizeUnit(memUsage);
           memLimitFormat=Utils.getDynamicSizeUnit(memLimit);
           memPercent=(memUsage.floatValue()/memLimit.floatValue())*100+"%";
 }
```  
  
###### Network IO计算  
1. 计算分析  
在上述参考文章中提到:  
> network stats的计算可以根据“获取属于该容器network namespace veth pairs在主机中对应的veth*虚拟网卡EthInterface数组，然后循环数组中每个网卡设备，读取/sys/class/net//statistics/rx_bytes得到rx_bytes, 读取/sys/class/net//statistics/tx_bytes得到对应的tx_bytes。将所有这些虚拟网卡对应的rx_bytes累加得到该容器的rx_bytes。将所有这些虚拟网卡对应的tx_bytes累加得到该容器的tx_bytes。   
  
	因此我们可以得到network stats的计算算法如下：  
	```  
NET I = rx_bytes
NET O = tx_bytes
```  
2. 代码实现  
通过已经得到的stats对象，可以取到 ImmutableMap<String, NetworkStats>。  
```java  
ImmutableMap<String, NetworkStats> networks = stats.networks();
```  
然后根据公式来计算network io：  
```java  
 String rxBytesSumFormat="---";
 String txBytesSumFormat="---";
 if (null!=networks){
 	/**
         * docker stats计算Network IO数据的算法：
           NET I = rx_bytes(所有网卡累加)
           NET O = tx_bytes（所有网卡累加）
         */
           Long rxBytesSum=0L;
           Long txBytesSum=0L;
           for (Map.Entry<String, NetworkStats> entry:networks.entrySet()){
     	      rxBytesSum+=entry.getValue().rxBytes();
              txBytesSum+=entry.getValue().txBytes();
           }
	  //Utils.getDynamicSizeUnit()是一个动态单位转换的方法
           rxBytesSumFormat=Utils.getDynamicSizeUnit(rxBytesSum);
           txBytesSumFormat=Utils.getDynamicSizeUnit(txBytesSum);
 }
```  
  
###### Blk IO计算  
1. 计算分析  
在上述参考文章中提到:  
> Blk IO计算可以根据“获取每个块设备的IoServiceBytesRecursive数据：先去读取/sys/fs/cgroup/blkio/docker/[containerId]/blkio.io_serviced_recursive中是否有有效值，如果有，则读取/sys/fs/cgroup/blkio/docker/[containerId]/blkio.io_service_bytes_recursive的值返回； 如果没有，就去读取/sys/fs/cgroup/blkio/docker/[containerId]/blkio.throttle.io_service_bytes中的值返回；将每个块设备的IoServiceBytesRecursive数据中所有read field对应value进行累加，得到该容器的blk_read值；将每个块设备的IoServiceBytesRecursive数据中所有write field对应value进行累加，得到该容器的blk_write值。  
  
	因此，我们可以得到Blk IO计算算法如下：  
	```  
BLOCK I = blk_read
BLOCK O = blk_write
```  
2. 代码实现  
```java  
 String blkReadFormat="---";
 String blkWriteFormat="---";
 if (null!=blockIoStats){
 	/**
         * docker stats计算Block IO数据的算法：
           BLOCK I = blkRead
           BLOCK O = blkWrite
         */
           ImmutableList<Object> objects = blockIoStats.ioServiceBytesRecursive();
           Long blkRead =0L;
           Long blkWrite =0L;
           //因为是Object，不好遍历，所以转变成了jsonobj遍历
           for (Object obj: objects) {
           	String jsonString= JSON.toJSONString(obj);
                JSONObject jsonObj = JSON.parseObject(jsonString);
                if ("Read".equals(jsonObj.getString("op"))){
            	    blkRead+=jsonObj.getLong("value");
                }else if ("Write".equals(jsonObj.getString("op"))){
		    blkWrite+=jsonObj.getLong("value");
                }
           }
           blkReadFormat=Utils.getDynamicSizeUnit(blkRead);
           blkWriteFormat=Utils.getDynamicSizeUnit(blkWrite);
 }
```  
  
###### 获取时间  
至此，以上将docker stats所需要的数据均计算出来了，之后我们只需获取下当前的采集时间即可。  
```java  
 Date read = stats.read();
```  
  
##### 容器其他  
上面说的基本上是关于docker容器监控所用的方法，剩下的都是直接操作容器的工具，先不写出来了，有需要的可以直接参考官方说明[spotify/docker-client](https://github.com/spotify/docker-client/blob/master/docs/user_manual.md#creating-a-docker-client)。  
  
#### 容器宿主机监控  
以上说明了容器的基础监控，我们在应用中，如果需要监控容器所在宿主机的情况，也可以使用下述方法：  
```java  
Info info = docker.info();
```  
获取到了Info对象，通过此对象可以取到宿主机上运行的容器状态、资源情况等，具体可获取到的如下。  
![Info](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-14-dockerhostInfo.png?raw=true)  
  
#### 其他  
除了以上说的那些，还可以获取、操作Networks、Images等。因为我们这里自己创建的一个镜像库，所以不直接使用docker-client的方法获取镜像信息，下面会统一再说明镜像库的搭建和列表的获取。  
至此，docker-client所用到的方法都说明完毕了，其他没提到的还是直接去看官方说明吧[spotify/docker-client](https://github.com/spotify/docker-client/blob/master/docs/user_manual.md#creating-a-docker-client)。  
  
## Engine API的使用  
上述进行说明了Docker-client的使用，我们也可以直接通过Engine API v1.24来进行Docker的操作。  
也是开启TCP端口之后，直接发送request到宿主机地址：端口，然后会收到json格式的response。  
这里直接使用工具测试，获取到容器的列表信息，如下图。  
![2018-06-14-restful获取容器列表](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-14-restful获取容器列表.png?raw=true)  
此后，如果需要，可以直接参考官方说明[Engine API v1.24](https://docs.docker.com/engine/api/v1.24/)，不过需要注意的是，我这里使用的docker对应的是v1.24的，其他版本的要选择不同对应版本的api哦，在本文最开始也选择docker-client的“Docker版本和API版本对应”处也说明了，忘记了的话回过头去看哦。  
  
# Docker管理-Docker镜像仓库  
我们上述说明了使用Docker-Client获取Docker容器的一些基础信息，而镜像我们是单独部署了一个镜像服务器，因此，不直接使用docker-client来获取镜像列表。下面会先说明镜像服务器的部署，之后说下如何获取镜像服务器中的镜像列表。  
## 简单说明镜像服务器的部署  
Docke官方提供了Docker Hub网站来作为一个公开的集中仓库。然而，本地访问Docker Hub速度往往很慢，并且很多时候我们需要一个本地的私有仓库只供网内使用。  
Docker仓库实际上提供两方面的功能，一个是镜像管理，一个是认证。前者主要由docker-registry项目来实现，通过http服务来上传下载；后者可以通过docker-index（闭源）项目或者利用现成认证方案（如nginx）实现http请求管理。  
目前来说，我们只需要在内网建立一个存储镜像的地方，因此只使用docker registry来搭建一个私有镜像库。  
下面简单说明下仓库搭建过程，也可以直接参考官方部署["Deploy a registry server"](https://docs.docker.com/registry/deploying/)。  
1. 官方提供了registry镜像，我们直接下载：  
```  
docker pull registry
```  
2. 默认情况下，会将仓库存放于容器内,这样的话如果容器被删除，则存放于容器中的镜像也会丢失，所以我们需要指定一个本地目录挂载到容器内的存放镜像的目录下。所以，我们以如下命令来启动一个docker容器：  
```  
docker run --privileged=true -d -v /datacenter02/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry
```  
3. 查看启动的容器  
```  
---@---[~]#docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
458a9e2301a6        registry            "/entrypoint.sh /etc/"   7 months ago        Up 7 months         0.0.0.0:5000->5000/tcp   registry
```  
4. 测试使用  
创建一个小的镜像，然后测试push  
```  
docker push 132.-.-.-:5000/busybox（这里的ip没写全，用-代替了，自己写的时候写全）
```  
可以看到push失败，报关于https的错误类似如下。  
```  
Error response from daemon: invalid registry endpoint https://132.-.-.-:5000/v1/: Get https://132.-.-.-:5000/v1/_ping: http: server gave HTTP response to HTTPS client. If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add `--insecure-registry 132.-.-.-:5000` to the daemon's arguments. In the case of HTTPS, if you have access to the registry's CA certificate, no need for the flag; simply place the CA certificate at /etc/docker/certs.d/132.-.-.-:5000/ca.crt
```  
5. 配置http访问  
因为Docker从1.3.X之后，与docker registry交互默认使用的是https，然而此处搭建的私有仓库只提供http服务，所以当与私有仓库交互时就会报上面的错误。为了解决这个问题需要在启动docker server时增加启动参数为默认使用http访问。  
我们需要修改docker配置文件，在docker配置文件中增加“--insecure-registry 132.-.-.-:5000”，其中IP为了不泄露用-代替了，自己写的时候需要写全：  
```  
# If you have a registry secured with https but do not have proper certs
# distributed, you can tell docker to not look for full authorization by
# adding the registry to the INSECURE_REGISTRY line and uncommenting it.
# INSECURE_REGISTRY='--insecure-registry'
INSECURE_REGISTRY='--insecure-registry 132.-.-.-:5000'
```  
之后重启docker服务。  
之后再次测试，可以发现没有报错啦，push成功。  
6. 命令行仓库管理  
有文章说，查询私有仓库中的所有镜像，使用docker search命令，但是我这里经过尝试，发现会报错：“Error response from daemon: Unexpected status code 404”。  
后来发现，docker search是v1版本的API，我们现在的是v2版本，所以不能使用。  
如果需要查询，则也是直接curl使用v2版本的API：  
```  
---@---[~]#curl 132.90.130.13:5000/v2/_catalog
{"repositories":["alluxio","basic","go","go1.10","go_zhzj","hdfs_datanode","hello-mine","hiveonspark","hiveonspark-gx","hivespark","jdk","jtwspark1.6.3","kafka","ninecon","nm","rm1","rm2","spark-2.3.0","spark2","spark2.2.1","spark2.2.1-gx","spark2.2.1-jtw","spark2.2.1-rx-demo","sparkdemo","sparkrx1.6.3","sparkrx2.2.1","tensorflow"]}
```  
详细的将在下面说明。  
  
## 镜像仓库器管理  
对于Docker Registry，Docker官方提供了HTTP API V2用于与其交互，用来管理docker images。对于此，在官方网站上也有说明：[HTTP API V2](https://docs.docker.com/registry/spec/api/)。  
因为我们只用到了镜像管理功能，所以下面只是用到的说明获取仓库内的镜像列表以及tags的方法，其他的需要自己参考官网啦。  
### 镜像列表  
我们使用下述方式来获取仓库里面的镜像列表，获取也是请求之后来获取一个包含信息（仓库、镜像名称）的json串，官方给出的request和response例子如下：  
request:  
```  
GET /v2/_catalog
```  
response:  
```  
200 OK
Content-Type: application/json

{
  "repositories": [
    <name>,
    ...
  ]
}
```  
我这里直接使用json工具，获取到我们仓库中的镜像列表：  
![镜像列表](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-14-仓库镜像列表.png?raw=true)  
### 镜像tags  
但是，对于一个镜像来说，有不同的tags来标明它的版本，所以单单靠上述方法获取镜像的名称还是不够的，还需获取到tags。官方也给出了相关的例子：  
request:  
```  
GET /v2/<name>/tags/list
```  
response:  
```  
200 OK
Content-Type: application/json

{
    "name": <name>,
    "tags": [
        <tag>,
        ...
    ]
}
```  
之后，我也使用json工具，获取了仓库中“spark2.2.1”的tags信息如下：  
![镜像tags获取](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-14-镜像tags获取.png?raw=true)  
  
至此，我们就可以得到仓库里面镜像对我们来说的有用的信息了。  
  
### 代码实现  
而在后台程序里，直接使用了最原始的httpclient写了restful工具类，直接获取镜像名称和相关tags信息，后来封装成一个对象放到list中返回直接使用，也可以直接使用restful的相关框架就比较方便了，就不详细说明了。  
```java  
  /**
     * 获取镜像服务器内的镜像列表
     * http://..../v2/_catalog
     * @throws IOException
     */
    public  static List<String>   getImagesList() throws IOException {
        String url = "http://"+ ConstantOp.IMAGES_REPOSITORIES_IP+":"+ConstantOp.IMAGES_REPOSITORIES_PORT+"/v2/_catalog";
        List<String> images=new ArrayList<>();
        RestfulHttpCli.HttpResponse response = RestfulHttpCli.getClient(url)
                .get()              //设置post请求
                .addHeader("Content-Type","application/json")
                .addHeader("Accept", "application/json")
                .addHeader("charset","utf-8")
                .request();         //发起请求
        //响应状态码
        //  System.out.println(response.getCode());
        //最终发起请求的地址
        //  System.out.println(response.getRequestUrl());
        if(response.getCode() == 200){
            //请求成功
            String content = response.getContent();
            //格式化json
            JSONObject jsonObject = JSON.parseObject(content);
            JSONArray repositories = jsonObject.getJSONArray("repositories");
            images = repositories.toJavaList(String.class);
        }
        return images;
    }

    /**
     * 获取镜像服务器内的镜像tags
     * 如http://..../v2/_catalog
     * @throws IOException
     */
    public  static String  getTagsList(String image_name) throws IOException {
        String url = "http://"+ ConstantOp.IMAGES_REPOSITORIES_IP+":"+ConstantOp.IMAGES_REPOSITORIES_PORT+"/v2/{image_name}/tags/list";
        String tags_str="---";
        RestfulHttpCli.HttpResponse response = RestfulHttpCli.getClient(url)
                .get()              //设置post请求
                .addHeader("Content-Type","application/json")
                .addHeader("Accept", "application/json")
                .addHeader("charset","utf-8")
                .addPathParam("image_name", image_name)
                .request();         //发起请求
        if(response.getCode() == 200){
            //请求成功
            String content = response.getContent();
            //格式化json
            JSONObject jsonObject = JSON.parseObject(content);
            JSONArray tags = jsonObject.getJSONArray("tags");
            tags_str = StringUtils.join(tags, ",");
        }
        return tags_str;
    }
    
    /**
     * 获取镜像服务器中的image列表，返回 List<ImagesDetails>
     * @return
     * @throws IOException
     */
    public static List<ImagesDetails>   getImagesDetailsList() throws IOException {
        List<ImagesDetails>  imagesDetailsList=new ArrayList<>();
        List<String> imagesList = getImagesList();
        if (null!=imagesList){
            for (String image_name:imagesList) {
                ImagesDetails imagesDetails=new ImagesDetails();
                imagesDetails.setName(image_name);
                imagesDetails.setTags(getTagsList(image_name));
                imagesDetailsList.add(imagesDetails);
            }
        }
        return imagesDetailsList;
    }
```
至此，完成了镜像仓库列表获取的全部工具类。  
  
以上也将Docker监控的总结暂时告一段落，下面进行Mesos的说明。  
  
# Mesos管理  
对于Mesos管理，官方提供2中方式：HTTP Endpoints和Operator HTTP API。  
## HTTP Endpoints  
对于Mesos master和Mesos agent均可以直接使用HTTP Endpoints，而且使用方式很简单，如下：  
```  
http://ip:port/endpoint
如：
http://masterIp:5050/metrics/snapshot
http://slaveIp:5050/metrics/snapshot
```  
具体使用可以参考官方网站说明：[HTTP Endpoints](http://mesos.apache.org/documentation/latest/endpoints/#http-endpoints)。  
通过观察，发现mesos原生监控页面中，是使用的这种方式，直接在前台进行调用的此api，所以，如果端口没开，当终端访问页面的时候，会无法获取数据。  
下面就是从前台控制台看到的获取的内容，会调用定时获取snapshot和state2个内容。  
![2018-06-15-mesos监控页面截图](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-15-mesos监控页面截图.png?raw=true)   
不过需要注意的是，如果使用的是Mesos1.1或之后的版本，不建议使用此种方式，可以使用下面说的新的[v1 Operator HTTP API](http://mesos.apache.org/documentation/latest/operator-http-api/)。  
  
## Operator HTTP API  
对于Mesos master和agents均支持/api/v1 endpoint。不过这个api仅支持post请求。并且请求内容需要是json格式（Content-Type: application/json）或者protobuf格式（Content-Type: application/x-protobuf）。  
对于[Master](http://mesos.apache.org/documentation/latest/operator-http-api/#master-api)和[Agent](http://mesos.apache.org/documentation/latest/operator-http-api/#agent-api)均提供了丰富的api，直接点击连接即可查看官方的说明，我就不在这里一一列出来了，直接说一下用到的一个方法。  
### 使用mesos api  
对于mesos的api，我这里只使用了master的GET_METRICS方法，目的是想自己做一个监控，能监控到mesos中的全部的资源使用情况，如下图mesos原生监控页面里面所示：  
![2018-06-15-mesos原生监控页面资源情况](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-15-mesos原生监控页面资源情况.png?raw=true)  
所以，为了获取这些内容，可以直接选择官方的master的[GET_METRICS](http://mesos.apache.org/documentation/latest/operator-http-api/#get_metrics)方法，官方给出的请求例子如下：  
```  
GET_METRICS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_METRICS",
  "get_metrics": {
    "timeout": {
      "nanoseconds": 5000000000
    }
  }
}
```  
然后我们可以直接使用工具进行获取测试：  
![2018-06-15-mesosRestfulPost工具截图](https://github.com/leafming/bak/blob/master/images/cloud/2018-06-15-mesosRestfulPost工具截图.png?raw=true)  
通过post方式发送对应json到“http://masterIP:5050/api/v1”即可取到了相应json格式的监控数据。  
而在程序中，也是用了最原始的方法直接使用的httpclient写的restful工具类，通过提交post请求后，获取到对应的响应内容，也可以直接使用restful的框架会比较方便，其他的具体就不详细说了，直接贴下代码：  
不过这里因为取到的json不是全部使用了，所以json解析是对应取到的数据提取自己有用的东西，使用了最笨的方法，求别嫌弃(,,• . •̀,) 。  
```java  
/**
     * 生成mesos的访问连接
     * @param hostIp
     * @return
     */
    public String generateUrl(String hostIp){
        //格式如：http://-.-.-.-:5050/api/v1
        return "http://"+hostIp+":"+ ConstantOp.MESOS_PORT+"/api/v1";
    }

    public MasterMesosMetrics getMasterMetrics(String mesosMasterIp) throws IOException {
        String url = generateUrl(mesosMasterIp);
        MasterMesosMetrics masterMesosMetrics = new MasterMesosMetrics();
        RestfulHttpCli.HttpResponse request = null;
        request = RestfulHttpCli.getClient(url)
                .post()   
                .addHeader("Content-Type", "application/json")
                .addHeader("Accept", "application/json")
                .addHeader("charset", "utf-8")
                .body("{\n" +
                        "  \"type\": \"GET_METRICS\",\n" +
                        "  \"get_metrics\": {\n" +
                        "    \"timeout\": {\n" +
                        "      \"nanoseconds\": 5\n" +
                        "    }\n" +
                        "  }\n" +
                        "}").request();
//        System.out.println(request.getCode());
//        System.out.println(request.getRequestUrl());
        if (200 == request.getCode()) {
            String content = request.getContent();
            //格式化json
            JSONObject contentObject = JSON.parseObject(content);
            JSONObject get_metrics = contentObject.getJSONObject("get_metrics");
            JSONArray metrics = get_metrics.getJSONArray("metrics");
            if (metrics.size() > 0) {
                for (int j =0; j < metrics.size(); j++) {
                    // 遍历 jsonarray 数组，把每一个对象转成 json 对象
                    JSONObject job = metrics.getJSONObject(j);
                    Object name = job.get("name");
                    //agents
                    if ("master/slaves_active".equals(name)){
                        masterMesosMetrics.setSlavesActive(job.getInteger("value"));
                    }else if ("master/slaves_inactive".equals(name)){
                        masterMesosMetrics.setSlavesInactive(job.getInteger("value"));
                    }else if ("master/slaves_disconnected".equals(name)){
                        masterMesosMetrics.setSlavesDisconnected(job.getInteger("value"));
                        //tasks
                    }else if ("master/tasks_staging".equals(name)){
                        masterMesosMetrics.setTasksStaging(job.getInteger("value"));
                    }else if ("master/tasks_starting".equals(name)){
                        masterMesosMetrics.setTasksStarting(job.getInteger("value"));
                    }else if ("master/tasks_running".equals(name)){
                        masterMesosMetrics.setTasksRunning(job.getInteger("value"));
                    }else if ("master/tasks_unreachable".equals(name)){
                        masterMesosMetrics.setTasksUnreachable(job.getInteger("value"));
                    }else if ("master/tasks_killing".equals(name)){
                        masterMesosMetrics.setTasksKilling(job.getInteger("value"));
                    }else if ("master/tasks_finished".equals(name)){
                        masterMesosMetrics.setTasksFinished(job.getInteger("value"));
                    }else if ("master/tasks_killed".equals(name)){
                        masterMesosMetrics.setTasksKilled(job.getInteger("value"));
                    }else if ("master/tasks_failed".equals(name)){
                        masterMesosMetrics.setTasksFailed(job.getInteger("value"));
                    }else if ("master/tasks_lost".equals(name)){
                        masterMesosMetrics.setTasksLost(job.getInteger("value"));
                        //resources
                        //cpus
                    }else if ("master/cpus_total".equals(name)){
                        masterMesosMetrics.setCpusTotal(job.getInteger("value"));
                    }else if ("master/cpus_used".equals(name)){
                        masterMesosMetrics.setCpusUsed(job.getInteger("value"));
                    }else if ("master/cpus_percent".equals(name)){
                        //cpuused/cputotal
                        masterMesosMetrics.setCpusPercent(job.getDouble("value"));
                        //gpu
                    }else if ("master/gpus_total".equals(name)){
                        masterMesosMetrics.setGpusTotal(job.getInteger("value"));
                    }else if ("master/gpus_used".equals(name)){
                        masterMesosMetrics.setGpusUsed(job.getInteger("value"));
                    }else if ("master/gpus_percent".equals(name)){
                        masterMesosMetrics.setGpusPercent(job.getDouble("value"));
                        //mem
                    }else if ("master/mem_total".equals(name)){
                       masterMesosMetrics.setMemTotal(job.getInteger("value"));
                    }else if ("master/mem_used".equals(name)){
                      masterMesosMetrics.setMemUsed(job.getInteger("value"));
                    }else if ("master/mem_percent".equals(name)){
                        masterMesosMetrics.setMemPercent(job.getDouble("value"));
                        //disk
                    }else if ("master/disk_total".equals(name)){
                        masterMesosMetrics.setDiskTotal(job.getInteger("value"));
                    }else if ("master/disk_used".equals(name)){
                        masterMesosMetrics.setDiskUsed(job.getInteger("value"));
                    }else if ("master/disk_percent".equals(name)){
                        masterMesosMetrics.setDiskPercent(job.getDouble("value"));
                    }else if ("master/uptime_secs".equals(name)){
                        masterMesosMetrics.setUptimeSecs(job.getInteger("value"));
                    }
                }
            }
        }
        return masterMesosMetrics;
    }
```  
至此，关于mesos的监控就这么多了，都是直接使用的httpApi官网都列明白了就不复制过来了，再贴一遍官网连接，有需要的直接去查吧：[HTTP Endpoints](http://mesos.apache.org/documentation/latest/endpoints/#http-endpoints)和[Operator HTTP API](http://mesos.apache.org/documentation/latest/operator-http-api/#operator-http-api)。  
  
# Marathon管理  
对于Marathon的管理，官方也提供了2种方式，1为直接使用marathon的restful api；2是官方也提供了封装好的方法，直接使用就可以啦。  
## Marathon-client  
对于marathon-client，也是对于restful api的一种封装，官方也给出的详细的使用说明:[marathon-client](https://github.com/mesosphere/marathon-client)。  
我们使用的marathon版本是Marathon1.4.5，所以来说，可以使用下列依赖：  
```  
<dependency>
  <groupId>com.mesosphere</groupId>
  <artifactId>marathon-client</artifactId>
  <version>0.6.0</version>
</dependency>
```  
导入依赖之后，我们就可以直接使用了。  
首先，使用**MarathonClient.getInstance()**创建Marathon实例对象，之后就可以直接使用其对应的各种方法了，在这里就不和docker-client似的详细说明了，因为这里没有参与过多就不贴详细使用代码啦，官方使用说明挺好的（[marathon-client](https://github.com/mesosphere/marathon-client)），所以直接放上来一个小例子来参考吧(IP全用-.-.-.-代替了)。  
```java  
public static void main(String args[]) throws Exception{
        String endpoint = "http://-.-.-.-:8080";
        Marathon marathon = MarathonClient.getInstance(endpoint);

        App app = new App();
        app.setId("spark-with-api");
        app.setCmd("/bin/bash  /usr/tools/demo.sh user1 user1234");
        app.setCpus(new Double(4));
        app.setMem(new Double(4096));
        app.setInstances(new Integer(1));
        Container container = new Container();

        //设置docker镜像配置
        Docker docker = new Docker();
        docker.setImage("-.-.-.-:5000/sparkwithhadoop_1.6.3:2.2");
        docker.setNetwork("BRIDGE");
        docker.setPrivileged(true);

        //设置容器的参数
        Collection<Parameter> parameters = new ArrayList<Parameter>();
        parameters.add(new Parameter("net","pub_net"));
        parameters.add(new Parameter("ip","-.-.-.-"));
        parameters.add(new Parameter("hostname","master1"));
        docker.setParameters(parameters);

        //设置端口映射
        Collection<Port> portMappings = new ArrayList<Port>();
        Port port22 = new Port();
        port22.setContainerPort(22);
        port22.setProtocol("tcp");
        portMappings.add(port22);
        
        Port port7077 = new Port();
        port7077.setContainerPort(7077);
        port7077.setProtocol("tcp");
        portMappings.add(port7077);
        
        Port port7070 = new Port();
        port7070.setContainerPort(7070);
        port7070.setProtocol("tcp");
        portMappings.add(port7070);
        docker.setPortMappings(portMappings);
        
        container.setDocker(docker);
        container.setType("DOCKER");
        
        //设置容器的挂载目录
        Collection<Volume> volumes = new ArrayList<Volume>(); 
        LocalVolume v1 = new LocalVolume();
        v1.setContainerPath("/usr/tools/hd");
        v1.setHostPath("/app/dcos/");
        v1.setMode("RW");
        volumes.add(v1);
        container.setVolumes(volumes);
        
        app.setContainer(container);
        
        //设置健康检查
        List<HealthCheck> healthChecks = new ArrayList<HealthCheck>();
        HealthCheck h1 = new HealthCheck();
        h1.setProtocol("MESOS_HTTP");
        h1.setGracePeriodSeconds(300);
        h1.setIntervalSeconds(60);
        h1.setTimeoutSeconds(60);
        h1.setMaxConsecutiveFailures(3);
        h1.setPortIndex(2);
        healthChecks.add(h1);
        app.setHealthChecks(healthChecks);
        marathon.createApp(app);
    }
```  
  
## Marathon restful api  
另外一种管理marathon的方式就是直接使用其[Marathon REST API](http://mesosphere.github.io/marathon/api-console/index.html)。  
不过它和docker和mesos的有所不同，他们是只有get或post请求，而在marathon中，是正常的restful api使用方式，不同的请求操作有不同的含义。  
拿/v2/apps来举例：  
如果是GET，则可以获取到对应的正在运行的applications的列表。  
如果是POST的话，则是创建或者开启一个新的application，提交的请求将创建的application所需要的参数带着就可以啦。  
具体的这里的restful api的使用方式就不细说了，直接看官方说明之后按照restful api正常使用方式使用就可以啦。  
再贴一遍，官方的详细说明如下：[Marathon REST API](http://mesosphere.github.io/marathon/api-console/index.html)。  
  
好了，marathon的管理方式也说明到这里。  
  
  
  
  
    
---
至此，本篇内容完成。  
本文由2018-06-11开始整理，于2018-06-15整理完成。  
如有问题，请发送邮件至leafming@foxmail.com联系我，谢谢～  
©商业转载请联系作者获得授权，非商业转载请注明出处。  
