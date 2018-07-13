---
layout: post
title: "Kerberos具体实践3-Kerberos与ZK的整合操作"
date: 2018-06-29
updated: 2018-06-29
description: 本文属于Kerberos具体实践整理的第三部分，主要涉及kerberos与ZK的整合操作。
categories:
- BigData
tags:
- Kerberos
---
> 本文主要关于Kerberos的简单应用。因为工作测试需要，自己装了一套集群进行了Kerberos的部署，并且与HDFS、ZK进行整合，然后将操作过程进行了整理，以便后续再查看。本文主要涉及到其与ZK的整合操做，此文为2018-06-28文章Kerberos具体实践2的后续。  
  
# Zookeeper整合Kerberos  
关于zookeeper上配置Kerberos，参考[zookeeper配置Kerberos认证](https://yq.aliyun.com/articles/25626)。
## Zookeeper Server配置  
若hdfs配置过程中需要配置zookeeper与Kerberos结合，则目前先需要配置server即可。
### 创建认证规则并生成keytab（zookeeper-server）    
zk整合Kerberos需要先创建启动zk用户的认证规则以及keytab，由于本次操作中，zk和hadoop均使用hadoop用户启动，则在本次操作中，不必再需要去创建新的认证规则和keytab，直接使用原来的hdfs.keytab即可。  
  
补充：  
若有创建需要（如需要使用zookeeper用户启动zk服务），可再继续再次参考下述操作（其实同hdfs的操作）。  
在node1节点，即KDC server节点上执行下面命令，在KCD server上（这里是 node1）创建zookeeper principal：  
```
kadmin.local -q "addprinc -randkey zookeeper/node1@HADOOP.COM"   
kadmin.local -q "addprinc -randkey zookeeper/node2@HADOOP.COM"  
kadmin.local -q "addprinc -randkey zookeeper/node3@HADOOP.COM"
```  
执行如下命令，在当前目录中创建keytab，名为zookeeper.keytab：
```
kadmin.local -q "xst  -k zookeeper.keytab  zookeeper/node1@HADOOP.COM "
kadmin.local -q "xst  -k zookeeper.keytab  zookeeper/node2@HADOOP.COM "
kadmin.local -q "xst  -k zookeeper.keytab  zookeeper/node3@HADOOP.COM "
```  
### 部署Keytab（zookeeper-server）   
拷贝node1上生成的keytab(新创建的就是zookeeper.keytab)文件到其他节点的某一目录上（如/opt/zookeeper/conf，为了在zk配置文件种找到keytab），并为其设置属主和权限400（如chown zookeeper:hadoop zookeeper.keytab ;chmod 400 *.keytab），同HDFS配置过程中的操作。  
此次测试中，启动zk和启动hdfs的用户都是hadoop，并且在进行hdfs配置kerberos的过程中，已经将hdfs.keytab复制到其他节点的/opt/hadoop_krb5/conf/hdfs.keytab目录下了，并且也设置了权限所以此次我不再进行操作。  
### 修改zookeeper配置文件（zookeeper-server）    
在node1节点上修改/opt/zookeeper-3.4.8/conf/（zookeeper安装目录下的conf）zoo.cfg文件，添加下面内容：  
```
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
jaasLoginRenew=3600000
```  
将修改的上面文件同步到其他节点：node2、node3：  
```
$ scp /opt/zookeeper-3.4.8/conf/zoo.cfg node2:/opt/zookeeper-3.4.8/conf/zoo.cfg
$ scp /opt/zookeeper-3.4.8/conf/zoo.cfg node3:/opt/zookeeper-3.4.8/conf/zoo.cfg
```
### 创建jaas文件（zookeeper-server）    
在node1的配置文件目录/opt/zookeeper-3.4.8/conf/（zookeeper安装目录下的conf）创建 jaas.conf文件，内容如下：
```
Server {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/opt/hadoop_krb5/conf/hdfs.keytab"【上述操作创建出的keytab，因为本次操作中zk用hadoop用户启动的，和hdfs一样，则没用重新生产keytab，和hdfs的用一个】
  storeKey=true
  useTicketCache=false【这里有踩过的一个小坑：如果krb5.conf 去掉了default_ccache_name = KEYRING:persistent:%{uid}，则一定用false，要不然zk起不来】
  principal="hadoop/node1@HADOOP.COM";
};
```
其中，keytab填写真实的keytab的绝对路径，principal填写对应的认证的用户和机器名称。需要在 node2和node3节点也创建该文件，注意每个节点的principal有所不同。  
### 创建java.env文件（zookeeper-server）    
在node1的配置文件目录/opt/zookeeper-3.4.8/conf/（zookeeper安装目录下的conf）创建 java.env文件，内容如下：  
```
export JVMFLAGS="-Djava.security.auth.login.config=/opt/zookeeper-3.4.8/conf/jaas.conf"
```
并将该文件同步到其他节点：  
```
$ scp /opt/zookeeper-3.4.8/conf/java.env node2:/opt/zookeeper-3.4.8/conf/java.env
$ scp /opt/zookeeper-3.4.8/conf/java.env node3:/opt/zookeeper-3.4.8/conf/java.env
```
以上Zookeeper-server配置完毕。  
### 重新启动zookeeper  
启动方式和平常无异，如成功使用安全方式启动，日志中看到如下日志：  
2017-09-18 10:23:30,067 ... - successfully logged in.  

## Zookeeper Client配置（未测试）
### 创建认证规则生成keytab
在node1节点，即KDC server节点上执行下面命令：
```
$ cd /var/kerberos/krb5kdc/
kadmin.local -q "addprinc -randkey zkcli/node1@HADOOP.COM "
kadmin.local -q "addprinc -randkey zkcli/node2@HADOOP.COM"
kadmin.local -q "addprinc -randkey zkcli/node3@HADOOP.COM"

kadmin.local -q "xst  -k zkcli.keytab  zkcli/node1@HADOOP.COM"
kadmin.local -q "xst  -k zkcli.keytab  zkcli/node2@HADOOP.COM "
kadmin.local -q "xst  -k zkcli.keytab  zkcli/node3@HADOOP.COM "
```
拷贝 zkcli.keytab文件到其他节点的zk配置目录：
```
$ scp zkcli.keytab node1:/etc/zookeeper/conf
$ scp zkcli.keytab node2:/etc/zookeeper/conf
$ scp zkcli.keytab node3:/etc/zookeeper/conf
```
并设置属主和权限，分别在node1、node2、node3 上执行：
```
$ ssh node1 "cd /etc/zookeeper/conf/;chown zookeeper:hadoop zkcli.keytab ;chmod 400 *.keytab"
$ ssh node2 "cd /etc/zookeeper/conf/;chown zookeeper:hadoop zkcli.keytab ;chmod 400 *.keytab"
$ ssh node3 "cd /etc/zookeeper/conf/;chown zookeeper:hadoop zkcli.keytab ;chmod 400 *.keytab"
```
### 创建jaas配置文件  
在node1的zk配置文件目录创建 client-jaas.conf 文件，内容如下：
```
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/etc/zookeeper/conf/zkcli.keytab"
  storeKey=true
  useTicketCache=false
  principal="zkcli@HADOOP.COM";
};
```
同步到其他client节点：
```
$ scp client-jaas.conf node2:/etc/zookeeper/conf
$ scp client-jaas.conf node3:/etc/zookeeper/conf
```
### 创建java.env文件
然后，在zk安装目录conf目录创建或者修改java.env，内容如下：
```
export CLIENT_JVMFLAGS="-Djava.security.auth.login.config=/etc/zookeeper/conf/client-jaas.conf"
```
> 如果，zookeeper-client 和 zookeeper-server 安装在同一个节点上，则 java.env 中的 java.security.auth.login.config 参数会被覆盖，这一点从 zookeeper-client 命令启动日志可以看出来。  

并将该文件同步到其他节点：
```
$ scp /etc/zookeeper/conf/java.env node2:/etc/zookeeper/conf/java.env
$ scp /etc/zookeeper/conf/java.env node3:/etc/zookeeper/conf/java.env
```
### 启动并验证
启动客户端：  
```
$ zookeeper-client -server node1:2181
```
创建一个znode 节点：  
```
k: node1:2181(CONNECTED) 0] create /znode1 sasl:zkcli@HADOOP.COM:cdwra
    Created /znode1
```  
验证该节点是否创建以及其 ACL：  
```
[zk: node1:2181(CONNECTED) 1] getAcl /znode1
    'world,'anyone
    : cdrwa
```  
以上为ZK整合Kerberos操作部分。  
  
  
---
至此，本篇内容完成。  
以上，此文章是关于ZK结合Kerberos的部署等内容，下篇将补充说明主从KDC的搭建。  
  
如有问题，请发送邮件至leafming@foxmail.com联系我，谢谢～  
©商业转载请联系作者获得授权，非商业转载请注明出处。

