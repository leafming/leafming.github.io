---
layout: post
title: "Kerberos具体实践2-Kerberos与HDFS的整合操作"
date: 2018-06-28
description: 本文属于Kerberos具体实践整理的第二部分，主要涉及kerberos与HDFS的整合操作。
categories:
- BigData
tags:
- Kerberos
---
> 本文主要关于Kerberos的简单应用。因为工作测试需要，自己装了一套集群进行了Kerberos的部署，并且与HDFS、ZK进行整合，然后将操作过程进行了整理，以便后续再查看。本文主要涉及到其与HDFS的整合操作，此文为2018-05-05文章Kerberos具体实践1的后续，上文说明了Kerberos集群的安装以及基本命令的使用；由于篇幅有限，下文在2018-06-29文章Kerberos具体实践3中继续说明Kerberos与ZK整合操作。  

# HDFS整合Kerberos（部署）
## 创建认证规则
在 Kerberos 安全机制里，一个principal就是realm里的一个对象，一个principal总是和一个密钥（secret key）成对出现的。  
这个 principal 的对应物可以是service，可以是host，也可以是user，对于Kerberos来说，都没有区别。  
Kdc(Key distribute center) 知道所有principal的secret key，但每个principal对应的对象只知道自己的那个secret key。这也是“共享密钥“的由来。  
对于hadoop，principals的格式为username/fully.qualified.domain.name@YOUR-REALM.COM。  
本次测试中，直接解压安装2.6.0-cdh5.11.0，并使用hadoop用户来进行NameNode和DataNode的启动，因此为集群中每个服务器节点添加principals：hadoop、HTTP、host（后续使用）。  
> 补充：若通过yum源安装的cdh集群中，NameNode和DataNode是通过hdfs启动的，故为集群中每个服务器节点添加两个principals：hdfs、HTTP。-官网说法：The properties for each daemon (NameNode, Secondary NameNode, and DataNode) must specify both the HDFS and HTTP principals, as well as the path to the HDFS keytab file.

在 KCD server 上（这里是 node1）创建 hadoop principal：  

```
kadmin.local -q "addprinc -randkey hadoop/node1@HADOOP.COM"   
kadmin.local -q "addprinc -randkey hadoop/node2@HADOOP.COM"  
kadmin.local -q "addprinc -randkey hadoop/node3@HADOOP.COM"
```  

创建 HTTP principal：  
```
kadmin.local -q "addprinc -randkey HTTP/node1@HADOOP.COM" 
kadmin.local -q "addprinc -randkey HTTP/node2@HADOOP.COM"  
kadmin.local -q "addprinc -randkey HTTP/node3@HADOOP.COM"
```

>说明：randkey 标志没有为新principal 设置密码，而是指示kadmin生成一个随机密钥。之所以在这里使用这个标志，是因为此principal不需要用户交互。它是计算机的一个服务器帐户。  
  
创建完成后，查看：

```
$ kadmin.local -q "listprincs"
```

操作结果小例子：
```
[root@node1 /]# kadmin.local -q "addprinc -randkey hadoop/node1@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for hadoop/node1@HADOOP.COM; defaulting to no policy
Principal "hadoop/node1@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "addprinc -randkey hadoop/node2@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for hadoop/node2@HADOOP.COM; defaulting to no policy
Principal "hadoop/node2@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "addprinc -randkey hadoop/node3@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for hadoop/node3@HADOOP.COM; defaulting to no policy
Principal "hadoop/node3@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "addprinc -randkey HTTP/node1@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for HTTP/node1@HADOOP.COM; defaulting to no policy
Principal "HTTP/node1@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "addprinc -randkey HTTP/node2@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for HTTP/node2@HADOOP.COM; defaulting to no policy
Principal "HTTP/node2@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "addprinc -randkey HTTP/node3@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for HTTP/node3@HADOOP.COM; defaulting to no policy
Principal "HTTP/node3@HADOOP.COM" created. 
[root@node1 hadoop_logs]# kadmin.local -q "addprinc -randkey host/node1@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for host/node1@HADOOP.COM; defaulting to no policy
Principal "host/node1@HADOOP.COM" created.
[root@node1 hadoop_logs]# kadmin.local -q "addprinc -randkey host/node2@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for host/node2@HADOOP.COM; defaulting to no policy
Principal "host/node2@HADOOP.COM" created.
[root@node1 hadoop_logs]# kadmin.local -q "addprinc -randkey host/node3@HADOOP.COM"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for host/node3@HADOOP.COM; defaulting to no policy
Principal "host/node3@HADOOP.COM" created.
[root@node1 /]# kadmin.local -q "listprincs"
Authenticating as principal root/admin@HADOOP.COM with password.
HTTP/node1@HADOOP.COM
HTTP/node2@HADOOP.COM
HTTP/node3@HADOOP.COM
K/M@HADOOP.COM
hadoop/node1@HADOOP.COM
hadoop/node2@HADOOP.COM
hadoop/node3@HADOOP.COM
host/node1@HADOOP.COM
host/node2@HADOOP.COM
host/node3@HADOOP.COM
kadmin/admin@HADOOP.COM
kadmin/changepw@HADOOP.COM
kadmin/node1@HADOOP.COM
kiprop/node1@HADOOP.COM
krbtgt/HADOOP.COM@HADOOP.COM
root/admin@HADOOP.COM
```
## 生成keytab
keytab是包含principals和加密principal key的文件。keytab文件对于每个host是唯一的，因为key包含hostname。keytab文件用于不需要人工交互和保存纯文本密码，实现到kerberos上验证一个主机上的principal。因为服务器上可以访问keytab文件即可以以principal的身份通过 kerberos 的认证，所以，keytab文件应该被妥善保存，应该只有少数的用户可以访问。  
创建包含hadoop principal和HTTP principal的keytab。  
1. 方法一：  
在node1节点，即KDC server节点上执行下面命令，会在当前目录下创建keytab，名字为hdfs.keytab：  
```
1、kadmin.local进入kerberos shell中
2、kadmin.local:  xst -norandkey -k hdfs.keytab hadoop/node1@HADOOP.COM hadoop/node2@HADOOP.COM hadoop/node3@HADOOP.COM HTTP/node1@HADOOP.COM HTTP/node2@HADOOP.COM HTTP/node3@HADOOP.COM
```  
使用klist显示hdfs.keytab文件列表,查看执行结果：  
```
[root@node1 etc]# klist -ket  hdfs.keytab
Keytab name: FILE:hdfs.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 hadoop/node1@HADOOP.COM (des-cbc-md5)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 hadoop/node2@HADOOP.COM (des-cbc-md5)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 hadoop/node3@HADOOP.COM (des-cbc-md5)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 HTTP/node1@HADOOP.COM (des-cbc-md5)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 HTTP/node2@HADOOP.COM (des-cbc-md5)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (des3-cbc-sha1)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (arcfour-hmac)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (camellia256-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (camellia128-cts-cmac)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (des-hmac-sha1)
   1 09/06/2017 17:53:54 HTTP/node3@HADOOP.COM (des-cbc-md5)
```
2. 方法二（未使用）：  
在node1节点，即KDC server节点上执行下面命令，分别创建keytab，名字为hdfs.keytab：  
```
$ cd /var/kerberos/krb5kdc/  
$ kadmin.local -q "xst  -k hdfs-unmerged.keytab hadoop/node1@HADOOP.COM"  
$ kadmin.local -q "xst  -k hdfs-unmerged.keytab hadoop/node2@HADOOP.COM"  
$ kadmin.local -q "xst  -k hdfs-unmerged.keytab hadoop/node3@HADOOP.COM"  
$ kadmin.local -q "xst  -k HTTP.keytab  HTTP/node1@HADOOP.COM"  
$ kadmin.local -q "xst  -k HTTP.keytab  HTTP/node2@HADOOP.COM"  
$ kadmin.local -q "xst  -k HTTP.keytab  HTTP/node3@HADOOP.COM"  
```
这样，就会在/var/kerberos/krb5kdc/（当前）目录下生成hdfs-unmerged.keytab和HTTP.keytab两个文件，接下来使用ktutil合并者两个文件为hdfs.keytab。  
```
$ cd /var/kerberos/krb5kdc/   
$ ktutil  
ktutil: rkt hdfs-unmerged.keytab  
ktutil: rkt HTTP.keytab  
ktutil: wkt hdfs.keytab
```  
使用klist显示hdfs.keytab文件列表：  
```
$ klist -ket  hdfs.keytab
```  
3. 测试：  
验证是否正确合并了key，使用合并后的keytab，分别使用hadoop和HTTP principals来获取证书。  
```
$ kinit -k -t hdfs.keytab hadoop/node1@HADOOP.COM
$ kinit -k -t hdfs.keytab HTTP/node1@HADOOP.COM
```  
如果出现错误：kinit: Key table entry not found while getting initial credentials，则上面的（生成）合并有问题，重新执行前面的操作。  

## 部署Kerberos Keytab（hdfs）  
拷贝生成的hdfs.keytab文件到其他节点的/etc/hadoop/conf目录【可以自己定义目录，后续需要在hdfs配置文件中使用此目录】   
```
$ cd /var/kerberos/krb5kdc/  【刚刚生成keytab文件所在目录】
$ scp hdfs.keytab node1:/etc/hadoop/conf  
$ scp hdfs.keytab node2:/etc/hadoop/conf  
$ scp hdfs.keytab node3:/etc/hadoop/conf
```
并设置权限，分别在node1、node2、node3上执行更改文件属主和权限命令：  
```
$ ssh node1 "chown hadoop:hadoop /etc/hadoop/conf/hdfs.keytab ;chmod 400 /etc/hadoop/conf/hdfs.keytab"
$ ssh node2 "chown hadoop:hadoop /etc/hadoop/conf/hdfs.keytab ;chmod 400 /etc/hadoop/conf/hdfs.keytab"
$ ssh node3 "chown hadoop:hadoop /etc/hadoop/conf/hdfs.keytab ;chmod 400 /etc/hadoop/conf/hdfs.keytab"
```  
> 注意：由于keytab相当于有了永久凭证，不需要提供密码(如果修改kdc中的principal的密码，则该keytab就会失效)，所以其他用户如果对该文件有读权限，就可以冒充 keytab中指定的用户身份访问hadoop，所以keytab文件需要确保只对owner有读权限(0400)

## HDFS配置修改  
关于HDFS集群的部署配置就不在此文中说明，以下直接进行说明Kerberos整合HDFS之时所做的修改操作。  
### 修改core-site.xml配置文件
在集群中所有节点的core-site.xml文件中添加下面的配置:  
```xml
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>

<property>
  <name>hadoop.security.authorization</name>
  <value>true</value>
</property>
```
### 修改hdfs-site.xml配置文件
在集群中所有节点的hdfs-site.xml文件中添加下面的配置：
```xml
<!-- General HDFS security config -->
<property>
  <name>dfs.block.access.token.enable</name>
  <value>true</value>
</property>

<!-- NameNode security config -->
<property>
  <name>dfs.namenode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- HDFS keytab的目录 -->
</property>
<property>
  <name>dfs.namenode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.namenode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>

<!-- DataNode security config -->
<property>
  <name>dfs.datanode.data.dir.perm</name>
  <value>700</value>
</property>
<property>
  <name>dfs.datanode.address</name>
  <value>0.0.0.0:61004</value>
</property>
<property>
  <name>dfs.datanode.http.address</name>
  <value>0.0.0.0:61006</value>
</property>
<property>
  <name>dfs.datanode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- HDFS keytab的目录 -->
</property>
<property>
  <name>dfs.datanode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.datanode.kerberos.https.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>

<!-- 开启SSL(jsvc时可不配置；可选，详见下属总结以及启动datanode部分) -->
<property>
  <name>dfs.http.policy</name>
  <value>HTTPS_ONLY</value>
</property>
<!-- SASL 模式-->
<property>
  <name>dfs.data.transfer.protection</name>
  <value>integrity</value>
</property>

<!-- Web Authentication config -->
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
<property>
  <name>dfs.web.authentication.kerberos.keytab</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- HDFS keytab的目录 -->
</property>
<property>
  <name>dfs.web.authentication.kerberos.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
 </property>

<!--- HDFS QJM HA：如果HDFS配置了QJM HA，则需要添加；另外，你还要在zookeeper上配置kerberos（详见后续文章说明） -->
<property>
  <name>dfs.journalnode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value>
</property>
<property>
  <name>dfs.journalnode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.journalnode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>
```
### 修改hadoop-env.sh配置文件    
在hadoop-env.sh文件中，需要修改HADOOP_SECURE_DN_USER的值，其依据datanode启动方式不同，有不同的设置。  
注意的关键点如下：  
1. jsvc模式启动datanode：HADOOP_SECURE_DN_USER有值
2. SASL模式启动datanode：HADOOP_SECURE_DN_USER没值，要注释掉。

```
#export HADOOP_SECURE_DN_USER=${HADOOP_SECURE_DN_USER}
#export HADOOP_SECURE_DN_USER=hadoop
```
具体配置方式，见下述启动datanode的部分，以不同的启动方式来详细说明。

配置中有几点要注意的： 
1. 如果不进行SSL配置，则hadoop需要使用jsvc来启动（如下条解释）。
2. dfs.datanode.address、dfs.datanode.http.address表示data transceiver RPC server所绑定的hostname或IP地址，此部分配置与datanode的启动方式有关（本条内容目前是靠个人理解，由于目前还是菜鸟学习阶段，所以理解可能问题，请大家指出并告诉我，谢谢啦～我的邮箱leafming@foxmail.com），这部分的端口值有两种设置情况；  
    - 一定小于1024-jsvc:如果开启security，并且使用jsvc模式启动datanode（不配置SSL），dfs.datanode.address、dfs.datanode.http.address的端口一定要小于1024（privileged端口），否则的话启动datanode时候会报Cannot start secure cluster without privileged resources错误。CDH官网说明如下（详见[Configure Secure HDFS](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/cdh_sg_secure_hdfs_config.html#concept_nsy_21z_xn)）：  
    ```
    The dfs.datanode.address and dfs.datanode.http.address port numbers for the DataNode must be below 1024,because this provides part of the security mechanism to make it impossible for a user to run a map task which impersonates a DataNode. 
    The port numbers for the NameNode and Secondary NameNode can be anything you want, but the default port numbers are good ones to use.
    ```
    - 一定大于1024-SASL:如果使用在2.6之前的版本，安全模式的hadoop只能使用jsvc，即先以root用户来启动datanode，然后再切到普通用户。而在2.6.0之后的版本，SASL可以用来验证数据传输。在使用SASL的配置下，不再需要root用jsvc来启动安全模式的集群，并且也不在需要使用特权端口。如果启用SASL模式，需要在hdfs-site.xml中设置dfs.data.transfer.protection，并且dfs.datanode.address的端口要大于1024（non-privileged端口），并且设置dfs.http.policy为HTTPS_ONLY，而且保证HADOOP_SECURE_DN_USER环境变量值没有设置。一定要注意的是，dfs.datanode.address若设置成了特权端口，则SASL无法使用，这是向后兼容性原因所必需的。官网说明如下[Hadoop in Secure Mode-Secure DataNode](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SecureMode.html) ：  
    ```
    As of version 2.6.0, SASL can be used to authenticate the data transfer protocol. 
    In this configuration, it is no longer required for secured clusters to start the DataNode as root using jsvc and bind to privileged ports. To enable SASL on data transfer protocol, set dfs.data.transfer.
    protection in hdfs-site.xml, set a non-privileged port for dfs.datanode.address, set dfs.http.policy to HTTPS_ONLY and make sure the HADOOP_SECURE_DN_USER environment variable is not defined. 
    Note that it is not possible to use SASL on data transfer protocol if dfs.datanode.address is set to a privileged port. This is required for backwards-compatibility reasons.
    ```
3. principal中的instance部分可以使用_HOST标记，系统会自动替换它为全称域名。    
4. 如果开启了security, hadoop会对hdfs block data(由dfs.data.dir指定)做permission check，方式用户的代码不是调用hdfs api而是直接本地读block data，这样就绕过了kerberos和文件权限验证，管理员可以通过设置dfs.datanode.data.dir.perm 来修改datanode 文件权限，这里我们设置为700。  


本次集群配置示例：  

```xml
core-site.xml  
-----------------------------------------------------------------
<configuration>
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://ns1</value>
</property>
<property>
  <name>ha.zookeeper.quorum</name>
  <value>node1:2181,node2:2181,node3:2181</value>
</property>
<property>
  <name>io.compression.codecs</name>
  <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
<property>
  <name>hadoop.tmp.dir</name>
  <value>/opt/hadoop-dir</value>
</property>
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
<property>
  <name>hadoop.security.authorization</name>
  <value>true</value>
</property>
</configuration>
-----------------------------------------------------------------
hdfs-site.xml
-----------------------------------------------------------------
<configuration>
<!-- General HDFS security config -->
<property>
  <name>dfs.block.access.token.enable</name>
  <value>true</value>
</property>
<property>
  <name>dfs.nameservices</name>
  <value>ns1</value>
</property>
<property>
  <name>dfs.ha.namenodes.ns1</name>
  <value>node1,node2</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.ns1.node1</name>
  <value>node1:8020</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.ns1.node2</name>
  <value>node2:8020</value>
</property>
<property>
  <name>dfs.namenode.http-address.ns1.node1</name>
  <value>node1:8570</value>
</property>
<property>
  <name>dfs.namenode.http-address.ns1.node2</name>
  <value>node2:8570</value>
</property>
<!-- NameNode security config -->
<property>
  <name>dfs.namenode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- path to the HDFS keytab -->
</property>
<property>
  <name>dfs.namenode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.namenode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>
<!-- DataNode security config -->
<property>
  <name>dfs.datanode.data.dir.perm</name>
  <value>700</value>
</property>
<property>
  <name>dfs.datanode.address</name>
  <value>0.0.0.0:61004</value>
</property>
<property>
  <name>dfs.datanode.http.address</name>
  <value>0.0.0.0:61006</value>
</property>
<property>
  <name>dfs.datanode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- path to the HDFS keytab -->
</property>
<property>
  <name>dfs.datanode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.datanode.kerberos.https.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.http.policy</name>
  <value>HTTPS_ONLY</value>
</property>
<property>
  <name>dfs.data.transfer.protection</name>
  <value>integrity</value>
</property>
<!-- Web Authentication config -->
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
<property>
  <name>dfs.web.authentication.kerberos.keytab</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value> <!-- path to the HTTP keytab -->
</property>
<property>
  <name>dfs.web.authentication.kerberos.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>
<!-- HDFS QJM HA-->
<property>
  <name>dfs.journalnode.keytab.file</name>
  <value>/opt/hadoop_krb5/conf/hdfs.keytab</value>
</property>
<property>
  <name>dfs.journalnode.kerberos.principal</name>
  <value>hadoop/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.journalnode.kerberos.internal.spnego.principal</name>
  <value>HTTP/_HOST@HADOOP.COM</value>
</property>
<property>
  <name>dfs.namenode.shared.edits.dir</name>
  <value>qjournal://node1:8485;node2:8485;node3:8485/ns1</value>
</property>
<property>
  <name>dfs.journalnode.edits.dir</name>
  <value>/opt/hadoop-dir/journal</value>
</property>
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>sshfence</value>
</property>
<property>
  <name>dfs.ha.fencing.ssh.private-key-files</name>
  <value>/home/hadoop/.ssh/id_rsa</value>
</property>
<property>
  <name>dfs.permissions.enabled</name>
  <value>true</value>
</property>
<property>
  <name>dfs.namenode.acls.enabled</name>
  <value>true</value>
</property>
<property>
  <name>dfs.permissions.superusergroup</name>
  <value>cgroup</value>
</property>
<property>
  <name>dfs.replication</name>
  <value>3</value>
  <final>true</final>
</property>
<property>
  <name>dfs.namenode.name.dir</name>
  <value>file:///opt/hadoop-dir/hadoop_data/nn</value>
</property>
<property>
  <name>dfs.datanode.data.dir</name>
  <value>file:///opt/hadoop-dir/hadoop_data/dn</value>
</property>
<property>
  <name>ha.zookeeper.session-timeout.ms</name>
  <value>5000</value>
</property>
<property>
  <name>dfs.ha.automatic-failover.enabled</name>
  <value>true</value>
</property>
<property>
  <name>dfs.client.failover.proxy.provider.ns1</name>  
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
<property>
  <name>dfs.socket.timeout</name>
  <value>90000</value>
</property>
<property>
  <name>dfs.balance.bandwidthPerSec</name>
  <value>10485760</value>
</property>
</configuration>
-----------------------------------------------------------------
```
## 检查集群上的HDFS和本地文件的权限（未尝试，[HDFS配置Kerberos认证](https://yq.aliyun.com/articles/25636?spm=5176.100240.searchblog.8.wbY2wv) 文章中写需要此操作）
请参考[Verify User Accounts and Groups in CDH 5 Due to Security](https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_users_groups_verify.html?spm=5176.100239.blogcont25636.6.wkeCKj)或者[Hadoop in Secure Mode](http://hadoop.apache.org/docs/r2.5.2/hadoop-project-dist/hadoop-common/SecureMode.html?spm=5176.100239.blogcont25636.7.wkeCKj)。

# HDFS整合Kerberos（启动）
启动之前，请确认JCE jar已经替换，即若master_key_type和supported_enctypes使用aes256-cts（klist -e 命令查看采用了什么encryption），则需要替换JCE jar，请参考前面的说明。否则可能出现问题，可参考[The NameNode starts but clients cannot connect to it and error message contains enctype code 18](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/cm_sg_sec_troubleshooting.html#topic_17_1_5)。
而本次操作中，没有使用aes256-cts，所以不用进行替换。  

## KerberosKDC已经启动
## 启动NameNode（可不单独做这个操作，直接配置好之后启动，但可优先测试）
### 启动操作  
上述更改配置文件的时候，已经停掉了全部hdfs服务，现在进行启动操作。  
在每个节点上获取root用户的ticket，这里root为之前创建的root/admin的密码。  

```
$ ssh cdh1 "echo root@1234|kinit root/admin"
$ ssh cdh2 "echo root@1234|kinit root/admin"
$ ssh cdh3 "echo root@1234|kinit root/admin"
```  
关闭selinux  

```
setenforce 0
```

获取node1的启动namenode的ticket：   

```
kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node1@HADOOP.COM
```  
如果出现下面异常kinit: Password incorrect while getting initial credentials，则重新导出keytab再试试。  

然后启动服务，观察日志： 
在node1启动namenode  
hadoop-daemon.sh start namenode   

> 待全部配置好之后，hdfs ha启动可参看[配置HDFS HA(高可用)](http://www.cnblogs.com/raphael5200/p/5154325.html)  

### 测试启动
在本次操作中，未直接进行测试，是在namenode和datanode全部启动之后才进行测试，但是，验证NameNode是否启动可参照下属操作：  
1. 打开 web 界面查看启动状态  
2. 运行下面命令查看 hdfs（hadoop fs -ls /）：  
    ```
    $ hadoop fs -ls /
    Found 4 items
    drwxrwxrwx   - yarn hadoop          0 2014-06-26 15:24 /logroot
    drwxrwxrwt   - hdfs hadoop          0 2014-11-04 10:44 /tmp
    drwxr-xr-x   - hdfs hadoop          0 2014-08-10 10:53 /user
    drwxr-xr-x   - hdfs hadoop          0 2013-05-20 22:52 /var
    ```  
  
### 问题总结  
采用此种方式过程中，可能遇到以下问题运行问题：  
1. 一个用户如果在你的凭据缓存中没有有效的kerberos ticket，执行上面命令将会失败，将会出现下面的错误：  
```
        11/01/04 12:08:12 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException:
        GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        Bad connection to FS. command aborted. exception: Call to nn-host/10.0.0.2:8020 failed on local exception: java.io.IOException:
        javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
```  
解决方法：可通过klist来查看缓存状态，或重新kinit更新缓存。  
> 此问题官网说明参考[Running any Hadoop command fails after enabling security.](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/cm_sg_sec_troubleshooting.html#topic_17_1_0)   
2. 但是，如果使用的是MIT kerberos 1.8.1或更高版本、并且Oracle JDK 6 Update 26或更低版本，即使成功的使用kinit获取了ticket，java仍然无法读取kerberos票据缓存，可能出现如下错误：  
```
        11/01/04 12:08:12 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException:
        GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        Bad connection to FS. command aborted. exception: Call to nn-host/10.0.0.2:8020 failed on local exception: java.io.IOException:
        javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
```  
问题原因：这个问题是因为MIT kerberos 1.8.1或更高版本在写凭证缓存的时候做了一个更改（change参考[MIT Kerberos Change](http://krbdev.mit.edu/rt/Ticket/Display.html?id=6206)），如果使用是Oracle JDK 6 Update 26或更低版本，会遇到一个bug（bug参考[Report of bug in Oracle JDK 6 Update 26 and lower](http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=6979329)），这个bug导致了在MIT kerberos 1.8.1或更高版中，java无法去读取Kerberos的票据缓存。而需要额外说明的是，Kerberos 1.8.1在Ubuntu Lucid或更高版本、 Debian Squeeze或更高版本中是默认Kerberos，会出现这个问题；但是在RHEL或CentOS中,默认的Kerberos是较老的版本，所以目前不会出现此问题。  
解决方法：在使用kinit获取ticket之后使用***kinit -R***来renew ticket。这样，将重写票据缓存中的ticket为java可读的格式，如下：  
```
        $ klist
        klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_1000)
        $ hadoop fs -ls
        11/01/04 13:15:51 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException:
        GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        Bad connection to FS. command aborted. exception: Call to nn-host/10.0.0.2:8020 failed on local exception: java.io.IOException:
        javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        $ kinit
        Password for atm@YOUR-REALM.COM: 
        $ klist
        Ticket cache: FILE:/tmp/krb5cc_1000
        Default principal: atm@YOUR-REALM.COM
        
        Valid starting     Expires            Service principal
        01/04/11 13:19:31  01/04/11 23:19:31  krbtgt/YOUR-REALM.COM@YOUR-REALM.COM
        
        renew until 01/05/11 13:19:30
        $ hadoop fs -ls
        11/01/04 13:15:59 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException:
        GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        Bad connection to FS. command aborted. exception: Call to nn-host/10.0.0.2:8020 failed on local exception: java.io.IOException:
        javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
        $ kinit -R
        $ hadoop fs -ls
        Found 6 items
        drwx------   - atm atm          0 2011-01-02 16:16 /user/atm/.staging
```  
3. 在上述问题2中，也有kinit -R 问题：  
        但是，需要注意的是使用2这种方法，要求的是ticket是可renew的。而能否获取renewable tickets依赖于KDC的设置和每一个principal的设置（包括realm中有问题的principal和TGT service端的principal）。  
        如果，一个ticker不可renew，那么它的"valid starting"和"renew until"的值是相同的时间。这时，使用kinit -R就会遇到下面的问题：  
```
        kinit: Ticket expired while renewing credentials
```  
> 此问题官网说明参考[Java is unable to read the Kerberos credentials cache created by versions of MIT Kerberos 1.8.1 or higher.](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/cm_sg_sec_troubleshooting.html#topic_17_1_1)   
  
	所以为了获取可renew的ticket，可尝试使用以下方法-未彻底测试，若有问题，还需要大家自行解决下，并希望大家告诉我一下啦～（以下解决方法部分参考其他文章，[CDH的Kerberos认证配置](http://blog.csdn.net/wulantian/article/details/42173095)，时间过去挺久了，有些遗忘，待有时间再重新测试下）：  
a.在kdc.conf中添加默认flag  
```
        default_principal_flags = +forwardable,+renewable
```  
但是实际没有起作用，因为查看资料，默认的principal_flags就包含了renewable，所以问题不是出在这里。另外需要说明一点，default_principal_flags只对这个flags生效以后创建的principal生效，之前创建的不生效，需要使用modprinc来使之前的principal生效。  
b.在kdc.conf中添加，并加大该参数：  
```
        max_renewable_life = 10d
```  
修改之后重启kdc，重新kinit，再重新执行kinit -R则可以正常renew了。  
为了进行验证，再次将参数修改为“max_renewable_life = 0s”，再重新kinit后执行kinit -R则再次不能renew，则说明是否可以获取renew的ticket中，默认是可以获取renew的ticket的，但是，可以renw的最长时间是0s，所以造成无法renew，解决的办法是在kdc.conf中增大该参数。  
        
> 另外补充：  
> 1」关于krb5.conf中的renew_lifetime = 7d参数，该参数设置该服务器上的使用kinit -R时renew的时间。  
> 2」可以通过modprinc来修改max_renewable_life的值，使用modprinc修改的值比kdc.conf中的配置有更高的优先级，例如，使用modprinc设置了为7天，kdc.conf中设置了为10天，使用getprinc可以看出，实际生效的是7天。需要注意的是，既要修改krbtgt/for_hadoop@for_hadoop，也要修改类似于hdfs/hadoop.local@for_hadoop这样的prinicials。使用modprinc来修改max_renewable_life，即kadmin.local进入kerberos的shell之后，执行：  
> ```
> modprinc -maxrenewlife 7days krbtgt/for_hadoop@for_hadoop  
> getprinc krbtgt/for_hadoop@for_hadoop
> ```  
  
到这里，kinit -R的问题解决，可以成功的执行hadoop fs -ls了。  
其他问题参考官网[Troubleshooting Authentication Issues](https://www.cloudera.com/documentation/enterprise/5-11-x/topics/cm_sg_sec_troubleshooting.html#topic_17_1_0)。  

## 启动datanode  
根据hdfs配置的区别，有两种启动datanode的方式，一种是使用jscv，一种是开启SASL。
### 方式一：jsvc启动datanode-目前此种方式测试有些问题，能成功启动并正常使用，但是启动后通过jps查看，发现datanode处有进程号，但无进程名字“datanode”，原因未知，待解决。
如果没有配置TLS/SSL for HDFS（hdfs-site.xml里的dfs.http.policy），则datanode需要通过JSVC启动（只能root用户）。  

关于Datanode+jsvc：  
在2.6版本前，启用hadoop安全模式，因为DataNode的数据传输协议没有使用HadoopRPC框架，DataNodes必须使用被dfs.datanode.address和dfs.datanode.http.address指定的特权端口来认证他们自己，该认证使基于假设攻击者无法获取在DataNode主机上的root特权。启动datanode的时候，必须用root用户，而当你使用root执行hdfs datanode命令时，服务器进程首先绑定特权端口，随后销毁特权并使用被HADOOP_SECURE_DN_USER指定的用户账号运行。这个启动进程使用被安装在JSVC_HOME的jsvc program。因此必须在启动项中hadoop-env.sh指定HADOOP_SECURE_DN_USER和JSVC_HOME做为环境变量。   
  
jsvc模式启动datanode关键点先说明：  
- hdfs-site.xml中没有开启TLS/SSL HDFS配置；  
- hdfs-site.xml中dfs.datanode.address、dfs.datanode.http.address的端口一定小于1024（特权端口）；  
- hadoop-env.sh设置HADOOP_SECURE_DN_USER为实际启动用户；  
    
    ```
    export HADOOP_SECURE_DN_USER=hadoop
    ```  
- 安装jsvc，在hdfs-site.xml下加入安装目录JSVC_HOME，jsvc相关jar包替换；  
    
    ```
    export JSVC_HOME=/opt/hadoop/sbin/Linux/bigtop-jsvc-1.0.10-cdh5.11.0
    ```  
- 启动服务前，先获取ticket再运行相关命令，要使用root用户启动datanode。  

#### i.jsvc环境部署（hdfs集群的所有节点都要进行此操作）  
使用jsvc模式，hdfs-site.xml中去掉ssl配置，并且相关端口号使用特权端口，之后需要部署jsvc环境。
因为系统里未安装jsvc，则首先进行安装操作，安装参考[jscv安装](http://blog.csdn.net/shuchangwen/article/details/45242549)。  
1. 下载安装包
因为我的hdoop版本是2.6.0-cdh5.11.0，因此我在[CDH组件下载](http://archive.cloudera.com/cdh5/cdh/5/)处下载了bigtop-jsvc-1.0.10-cdh5.11.0.tar.gz。  
> 补充：有些文章说明，apache hadoop可去[apache网站](http://archive.apache.org/dist/commons/daemon/binaries/)下载源代码和bin包。并且，关于JSVC，默认指向hadoop安装目录的libexec下，但libexec下并没有jsvc文件（hadoop是直接下载的tar.gz包，不是rpm安装），如下载commons-daemon-1.0.15-src.tar.gz及commons-daemon-1.0.15-bin.tar.gz，先解压src包后进入src/native/unix目录依次(可能需要先yum install gcc make sdk autoconf，再执行)执行./configure命令，make命令，这样会在当前目录下生成一个叫jsvc的文件，之后把它拷贝到hadoop目录下的libexec下，或更改环境变量如3。  
2. 安装操作  
把安装包放到了HADOOP_HOME家目录"/sbin/Linux/"目录下，直接解压即可。  
解压成bigtop-jsvc-1.0.10-cdh5.11.0后，进入到$HADOOP_HOME/sbin/Linux/bigtop-jsvc-1.0.10-cdh5.11.0目录下，可以看到jsvc，使用“./jsvc -help”命令测试jsvc能否使用（注意修改整个文件的权限drwxr-xr-x. 5 hadoop root   4096 Sep  8 13:54 bigtop-jsvc-1.0.10-cdh5.11.0）。  
3. 配置环境变量  
在hadoop-env.sh文件中加入以下内容：  
```
export HADOOP_SECURE_DN_USER=hadoop
export JSVC_HOME=/opt/hadoop/sbin/Linux/bigtop-jsvc-1.0.10-cdh5.11.0
```  
> 说明：设置了HADOOP_SECURE_DN_USER的环境变量后，start-dfs.sh的启动脚本将会自动跳过DATANODE的启动。  
4. 替换jar包  
之前是没有进行替换jar包的操作，之后启动过程中出现错误，之后查阅资料得知，JSVC运行还需要一个commons-daemon-xxx.jar包，因此发现需要进行jar包替换，通过查阅资料得知，需要替换$HADOOP_HOME/share/hadoop/hdfs/lib下的commons-daemon-xxx.jar。  
因为我下载的是bigtop-jsvc-1.0.10-cdh5.11.0.tar.gz，而在HADOOP_HOME家目录的/share/hadoop/hdfs/lib下的是commons-daemon-1.0.13.jar，因此从[commons／daemon](http://archive.apache.org/dist/commons/daemon/binaries/)下下载了commons-daemon-1.0.10.jar，然后把得到的commons-daemon-1.0.10.jar文件拷贝到hadoop安装目录下share/hadoop/hdfs/lib下（有文章说明要同时删除自带版本的commons-daemon-xxx.jar包，但我没有删除也可以使用）并更改了权限和其它内容保持一致(如-rwxr-xr-x. 1 hadoop root    24242 Sep  8 14:13 commons-daemon-1.0.10.jar)。  

> 接1补充：如果下载的是apache commons，即可解压bin包，然后把得到的commons-daemon-**.jar文件拷贝到hadoop安装目录下share/hadoop/hdfs/lib下，同时删除自带版本的commons-daemon-xxx.jar包。  

将上述操作在集群中的所有节点重复一遍（node1、node2、node3）。  
至此，jsvc的安装结束，注意hdfs的所有节点都要进行以上安装操作。  
#### ii.启动（集群）datanode  
##### 启动操作  
设置了Security后，NameNode、QJM、ZKFC可以通过start-dfs.sh启动，DataNode需要使用（jsvc）root权限启动。  
并且因为设置了HADOOP_SECURE_DN_USER的环境变量后，执行start-dfs.sh的启动脚本将会自动跳过DATANODE的启动。  
所以使用jsvc模式，启动集群的时候需要两个步骤：  
1. 启动NameNode、QJM、ZKFC（详细参考上述启动namenode）（若不是第一次启动，第一次配置好了hadoop的启动需要参考[配置HDFS HA(高可用)](http://www.cnblogs.com/raphael5200/p/5154325.html) ）  
```
#每台机器上（kinit启动的用户）
kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node1@HADOOP.COM
#node1上
start-dfs.sh
```
使用start-dfs.sh之后查看QJM的日志和ZKFC的日志（QJM的报错不会有明显的提示）检查有无exception；如果启动不成功则仍需进行排查（keytab、zk的kerberos等）。   

2. 启动Datanode   
因为Datanode需要通过jsvc启动进程，当你使用root执行启动脚本（如hdfs datanode）命令时，服务器进程首先绑定特权端口，随后销毁特权并使用被HADOOP_SECURE_DN_USER指定的用户账号运行。  
所以先需要在集群中的每台机器上都获取原启动用户（hadoop）ticket（现在登陆的是root用户,时间有点长了，有些遗忘好像也需要kinitroot的），然后再启动服务。  
kinit并启动服务：  
    ```
    # ssh node1 "kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node1@HADOOP.COM; ./opt/hadoop/sbin/hadoop-daemon.sh start datanode"
    # ssh node2 "kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node2@HADOOP.COM; ./opt/hadoop/sbin/hadoop-daemon.sh start datanode"
    # ssh node3 "kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node3@HADOOP.COM; ./opt/hadoop/sbin/hadoop-daemon.sh start datanode"
    ```
    补充，或在kinit后每台机器上使用命令启动：  
    ```
    HADOOP_DATANODE_USER=hadoop sudo -E /opt/hadoop/bin/hdfs datanode
    ```
    或：  
    ```
    [root@node1 ~]# kinit -k -t /opt/hadoop_krb5/conf/hdfs.keytab hadoop/node1@HADOOP.COM
    [root@node1 ~]# start-secure-dns.sh
    node3: starting datanode, logging to /app/dcos/hadoop_logs//hadoop-hadoop-datanode-node3.out
    node1: starting datanode, logging to /app/dcos/hadoop_logs//hadoop-hadoop-datanode-node1.out
    node2: starting datanode, logging to /app/dcos/hadoop_logs//hadoop-hadoop-datanode-node2.out
    
    ```
3. 验证datanode启动情况：观看node1上NameNode日志，出现下面日志表示DataNode启动成功：  
```
2017-09-11 10:06:27,667 INFO org.apache.hadoop.security.UserGroupInformation: Login successful for user hadoop/node1@HADOOP.COM using keytab file /opt/hadoop_krb5/conf/hdfs.keytab
```  
   
##### 遇到问题  
通过此方法启动datanode之后，通过jps查看，发现进程名字无法显示，但是hadoop集群运行正常。
```
[root@node1 sbin]# jps
6912 QuorumPeerMain
8785 NameNode
9251 DFSZKFailoverController
27156 Jps
10456
19929 Main
9033 JournalNode
``` 
排查问题补充相关：  
jps(Java Virtual Machine Process Status Tool)是提供的一个显示当前所有java进程pid的命令，而jvm运行时会生成一个目录hsperfdata_$USER(其中user是启动java进程的用户)，默认的是生成在 java.io.tmpdir目录下，在linux中默认是/tmp，目录下会有些pid文件,存放jvm进程信息可以利用strings查看里面的文件内容，一般就是jvm的进程信息而已。  
以上问题是只有进程号，没有进程名字，就思考是不是pid文件问题，于是去/tmp/hsperfdata_hadoop/和/tmp/hsperfdata_root/下看看是否有相关进程信息，发现datanode虽然是用jsvc方式用root用户启动的，而“10456”进程在/tmp/hsperfdata_hadoop/下。  
```
[root@node1 tmp]# cd hsperfdata_
hsperfdata_apple1/  hsperfdata_apple2/  hsperfdata_hadoop/  hsperfdata_root/    hsperfdata_sitech1/ hsperfdata_sitech2/
[root@node1 tmp]# cd hsperfdata_hadoop/
[root@node1 hsperfdata_hadoop]# ls
10456  6912  8785  9033  9251
```
之后就去尝试使用SASL方式启动datanode了，这个问题就没有继续解决。怀疑是不是上面部署步骤出现问题了，有时间回来再看看是什么问题，再进行修改@TODO，希望大家有消息告诉我下，非常感谢！邮箱：leafming@foxmail.com。  
> 此文“[Hadoop的kerberos的实践部署](http://blog.csdn.net/xiao_jun_0820/article/details/39375819)”和“[CDH的Kerberos认证](http://blog.csdn.net/wulantian/article/details/42173095)”内有关于datanode使用jsvc启动内容，但未完全参照。  

> 补充其他问题（好像这个问题没啥关系，不是这么解决，就先放这放着吧）：有人说，有时候端口问题服务无法正常启动，如：“WARN org.apache.hadoop.hdfs.server.datanode.DataNode: Problem connecting to server: node2/10.211.55.11:8020”，使用“netstat -an\|grep 8020”命令查看端口状态，如果不正常，映射为127.0.0.1:8020，则可尝试修改/etc/hosts文件中，去掉“127.0.0.1 localhost”，添加自己ip和主机名的映射，重新启动。  
  
##### 小总结  
加强理解，再次重复总结：jsvc模式启动datanode关键点：  
- hdfs-site.xml中没有开启TLS/SSL HDFS配置；  
- hdfs-site.xml中dfs.datanode.address、dfs.datanode.http.address的端口一定小于1024（特权端口）；  
- hadoop-env.sh设置HADOOP_SECURE_DN_USER为实际启动用户；  
    
    ```
    export HADOOP_SECURE_DN_USER=hadoop
    ```  
- 安装jsvc，在hdfs-site.xml下加入安装目录JSVC_HOME，jsvc相关jar包替换；  
    
    ```
    export JSVC_HOME=/opt/hadoop/sbin/Linux/bigtop-jsvc-1.0.10-cdh5.11.0
    ```  
- 启动服务前，先获取ticket再运行相关命令，要使用root用户启动datanode，其它正常用hadoop用户即可。

### 方式二：Hfds中开启SASL整体配置  

> 过渡说明：关于迁移一个使用root验证的hdfs集群为SASL方式，官方网站（[Hadoop in Secure Mode-Secure DataNode](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SecureMode.html) ）上有如下说明：  
> 
> ```
> In order to migrate an existing cluster that used root authentication to start using SASL instead, first ensure that version 2.6.0 or later has been deployed to all cluster nodes as well as any external applications that need to connect to the cluster. Only versions 2.6.0 and later of the HDFS client can connect to a DataNode that uses SASL for authentication of data transfer protocol, so it is vital that all callers have the correct version before migrating. After version 2.6.0 or later has been deployed everywhere, update configuration of any external applications to enable SASL. If an HDFS client is enabled for SASL, then it can connect successfully to a DataNode running with either root authentication or SASL authentication. Changing configuration for all clients guarantees that subsequent configuration changes on DataNodes will not disrupt the applications. Finally, each individual DataNode can be migrated by changing its configuration and restarting. It is acceptable to have a mix of some DataNodes running with root authentication and some DataNodes running with SASL authentication temporarily during this migration period, because an HDFS client enabled for SASL can connect to both.
> ```

方式一介绍了使用jsvc启动datanode（稍有问题），下面开始介绍使用SASL的方式。  
> 有文章写了sasl in hdfs的配置方式相关操作，参考[HDFS使用Kerberos](http://blog.csdn.net/dxl342/article/details/55510659)。  

关于SASL模式的配置关键主要有以下几点，文章上面配置hadoop部分也有部分说明：  
- hdfs-site.xml中开启TLS/SSL HDFS配置；  
- hdfs-site.xml中dfs.datanode.address、dfs.datanode.http.address的端口一定大于1024（非特权端口）；  
- hadoop-env.sh中的HADOOP_SECURE_DN_USER内容一定为空，并且和jsvc模式不同，不需要进行jsvc的环境部署；  
- https配置；
- 正常启动hdfs集群。  

#### i.hdfs配置（1基础配置）  
虽然上面配置地方已经提过了，不过这次还是再重复写一遍吧，保持操作完整性嘛，大家配置过的可以去检查一次。  
1. hdfs-site.xml中开启TLS/SSL HDFS配置，即在hdfs-site.xml中定义dfs.http.policy，指定要在HDFS的守护进程启动HTTPS服务器。增加如下内容：  
```
	<!-- 开启SSL(jsvc时可不配置；可选，详见下属总结以及启动datanode部分) -->
	<property>
  	<name>dfs.http.policy</name>
  	<value>HTTPS_ONLY</value>
	</property>
	<!-- SASL 模式-->
	<property>
  	<name>dfs.data.transfer.protection</name>
 	 <value>integrity</value>
	</property>
```  
	说明，这里可以使用三个参数：    
HTTP_ONLY: Only HTTP server is started  
HTTPS_ONLY: Only HTTPS server is started  
HTTP_AND_HTTPS: Both HTTP and HTTPS server are started    
值得一提的是，如果dfs.http.policy设置了https_only，则普通的WebHDFS不再可用。您将需要使用WebHDFS基于HTTPS，在这种结构中，它能保护你通过WebHDFS的数据。  
2. 检查hdfs-site.xml中dfs.datanode.address、dfs.datanode.http.address的端口一定大于1024（非特权端口）；hadoop-env.sh中的HADOOP_SECURE_DN_USER内容一定为空。  

#### ii.https配置  
经过以上操作后，尝试启动hdfs，会发现namenode报错后退出，如下：  
```
2017-09-13 14:12:37,273 INFO org.apache.hadoop.http.HttpServer2: HttpServer.start() threw a non Bind IOException
java.io.FileNotFoundException: /home/dtdream/.keystore (No such file or directory)
```
发现没有keystore文件，这个问题跟hadoop、Kerberos没什么关系，纯粹是https的配置了。  
简单说明https相关原理：https要求集群中有一个CA，它会生成ca_key和ca_cert，想要加入这个集群的节点，需要拿到这2个文件，然后经过一连串的动作生成keystore，并在hadoop的ssl-server.xml和ssl-client.xml中指定这个keystore的路径和密码，这样各个节点间就可以使用https进行通信了。
因此若要继续使用此模式，后续需要进行hdfs中的https配置。详细hdfs的https配置，可参考[DEPLOYING HTTPS IN HDFS](https://zh.hortonworks.com/blog/deploying-https-hdfs/)，下面直接帖命令简单说明下过程。 
  
注：在每台（node1、node2、node3）机器上都创建了/opt/ca 这个目录，存放ca相关文件，以下命令均在集群中各个机器的此目录下执行。    
1. 生成每个机器的密钥和证书  
首先需要在集群里面的每个机器上创建keyhecertificate，可以使用keytool命令来创建，如：  
```
$ keytool -keystore {keystore} -alias localhost -validity {validity} -genkey
```  
即通过此命令定义两个关键参数：  
keystore：keystore文件负责存储certificate，里面包括certificate的私钥，因此需要好好保存；  
validity：定义certificate的有效时间。  
keytool命令会设置certificate的一些相关细节，如CN、OU、O、L、ST、C；其中CN要和节点全域名（FQDN）保持一致，通常情况下，在有部署DNS的情况下，客户端将cn与DNS域名进行比较，以确保它确实连接到所需服务器，而不是恶意服务器。  
此操作命令示例（注意密码）：  
```
#node1  ／opt／ca目录下
keytool -keystore keystore -alias node1 -validity 9999 -genkey -keyalg RSA -keysize 2048 -dname "CN=node1, OU=security, O=hadoop, L=beijing, ST=beijing, C=CN"  
#node2  ／opt／ca目录下
keytool -keystore keystore -alias node2 -validity 9999 -genkey -keyalg RSA -keysize 2048 -dname "CN=node1, OU=security, O=hadoop, L=beijing, ST=beijing, C=CN"  
#node3  ／opt／ca目录下
keytool -keystore keystore -alias node3 -validity 9999 -genkey -keyalg RSA -keysize 2048 -dname "CN=node1, OU=security, O=hadoop, L=beijing, ST=beijing, C=CN"
```  
执行完上述命令，会在当前目录下生成keystore文件。  
  
2. 创建自己的CA    
经过第一步之后，集群的每个机器中有个自己的公私密钥对以及一个标识机器的证书。但是，证书是没有注册的，这意味着攻击者可以创建这样的证书冒充任何机器。  
因此，为了防止伪造证书，要为注册签名集群中的每台机器的证书。证书颁发机构（CA）负责签署证书。CA的工作就像一个政府签发护照，政府印章（标志）每个护照，使护照变得难以伪造。其他政府核实这些邮票以确保护照是真实的。类似地，CA在证书上签名，加密保证了签名证书难以伪造。因此，只要CA是一个真正可信的权威，客户就保证他们连接到真正的机器。  
下面，使用openssl命令去生成一个新的CA证书，生成的CA仅仅是一个公私密钥对和证书，可以用它签署其他证书。
我们这里将node1作为CA，在node1上执行如下命令（注意密码）：
```
#node1  ／opt／ca目录下
openssl req -new -x509 -keyout test_ca_key -out test_ca_cert -days 9999 -subj '/C=CN/ST=beijing/L=beijing/O=hadoop/OU=security/CN=node1'
```
说明：这里cn是CA服务器的地址。  
执行完以上命令，会在当前目录下生成test_ca_cert、test_ca_key文件。  

3. 添加CA到客户机器的信任库  
在生成了CA之后，需要添加CA到每个客户端的信任库中，使客户可以信任这个CA，使用命令：  
```
$ keytool -keystore {truststore} -alias CARoot -import -file {ca-cert}
```  
此步骤操作命令示例： 
即，首先在node1上，将上面生成的test_ca_key和test_ca_cert拷贝到集群中的所有机器上。  
```
#node1  ／opt／ca目录下
scp test_ca_cert node2:/opt/ca/  
scp test_ca_cert node3:/opt/ca/  
scp test_ca_key node3:/opt/ca/  
scp test_ca_key node2:/opt/ca/
```
在集群中每台机器上执行如下命令（注意密码）：  
```
#node1  ／opt／ca目录下  
keytool -keystore truststore -alias CARoot -import -file test_ca_cert
#node2  ／opt／ca目录下  
keytool -keystore truststore -alias CARoot -import -file test_ca_cert
#node3  ／opt／ca目录下  
keytool -keystore truststore -alias CARoot -import -file test_ca_cert
```
执行完以上命令，会在当前目录下生成truststore文件。  
在步骤1中的ketstore存储的是每一台机器自己的身份，而客户机的truststore存储的是客户端信任的所有证书。导入证书到一个truststore中意味着信任那个证书签名的其它所有证书。如以上所述，信任政府（CA）也意味着信任它签发的所有护照（证书）。这个属性称为信任链，它在大型Hadoop集群上部署HTTPS时特别有用。你可以在集群中用一个CA签名所有认证，然后每个机器就能使用一个相同信任了CA的truststore，通过这种方式，所有的机器可以验证所有其他机器。  
  
4. 签名证书  
下一步就是要使用步骤2生成的CA来签名步骤1中生成的所有证书。  
    1. 首先，你需要从密钥库中导出的证书，使用如下命令：  
        ```
        $ keytool -keystore -alias localhost -certreq -file {cert-file}
        ```  
        即在node1、node2、node3上执行（注意密码）：  
        
        ```
        #node1  ／opt／ca目录下
        keytool -certreq -alias node1 -keystore keystore -file cert  
        #node2  ／opt／ca目录下
        keytool -certreq -alias node2 -keystore keystore -file cert
        #node3  ／opt／ca目录下
        keytool -certreq -alias node3 -keystore keystore -file cert
        ```  
        执行完以上命令，会在当前目录下生成cert文件。  
    2. 之后，使用CA进行签名，使用如下命令：  
        ```
        $ openssl x509 -req -CA {ca-cert} -CAkey {ca-key} -in {cert-file} -out {cert-signed} -days {validity} -CAcreateserial -passin pass:{ca-password}
        ```
        即在node1、node2、node3上执行（注意密码）：  
        ```
        #node1  ／opt／ca目录下
        openssl x509 -req -CA test_ca_cert -CAkey test_ca_key -in cert -out cert_signed -days 9999 -CAcreateserial -passin pass:hadoop@1234
        #这个pass是CA的密码，即Then sign it with the CA。
        #node2  ／opt／ca目录下
        openssl x509 -req -CA test_ca_cert -CAkey test_ca_key -in cert -out cert_signed -days 9999 -CAcreateserial -passin pass:hadoop@1234
        #node3  ／opt／ca目录下
        openssl x509 -req -CA test_ca_cert -CAkey test_ca_key -in cert -out cert_signed -days 9999 -CAcreateserial -passin pass:hadoop@1234
        ```  
        执行完以上命令，会在当前目录下生成cert_signed文件。
    3. 最后，将CA正式和被签名的证书都导入keystore中，使用如下命令：  
        ```
        $ keytool -keystore -alias CARoot -import -file {ca-cert}
        $ keytool -keystore -alias localhost -import -file {cert-signed}
        ```  
        即在node1、node2、node3上执行（注意密码）：  
        
        ```
        #node1  ／opt／ca目录下
        keytool -keystore keystore -alias CARoot -import -file test_ca_cert
        keytool -keystore keystore -alias node1 -import -file cert_signed
        #node2  ／opt／ca目录下
        keytool -keystore keystore -alias CARoot -import -file test_ca_cert
        keytool -keystore keystore -alias node2 -import -file cert_signed
        #node3  ／opt／ca目录下
        keytool -keystore keystore -alias CARoot -import -file test_ca_cert
        keytool -keystore keystore -alias node3 -import -file cert_signed
        ```  
    4. 补充：以上命令参数说明：  
        keystore:keystore的位置  
        ca-cert:CA证书  
        ca-key:CA私钥  
        ca-password:CA密码  
        cert-file:导出的服务器的未签名证书  
        cert-signed:服务器的签名的证书  
  
#### iii.hdfs配置（2整合https） 
最后需要去配置HDFS中的HTTPs。  
其次，需要配置在$HADOOP_HOME/etc/hadoop目录下的ssl-server.xml和ssl-client.xml，告诉HDFS的密钥库和信任存储区。  
在所有节点上均进行此配置，现在node1上配置好，之后拷贝到其他节点（因为文件设置的都是在一样的路径）。  
说明：这里没来得及深究，有些配置可以省略。以后又时间再补充。@TODO  
1. ssl-client.xml  
本次操作配置文件示例： 
```xml
	<configuration>
	<property>
	  <name>ssl.client.truststore.location</name>
	  <value>/opt/ca/truststore</value>
	  <description>Truststore to be used by clients like distcp. Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.client.truststore.password</name>
	  <value>hadoop@1234</value>
	  <description>Optional. Default value is "".
	  </description>
	</property>
	<property>
	  <name>ssl.client.truststore.type</name>
	  <value>jks</value>
	  <description>Optional. The keystore file format, default value is "jks".
	  </description>
	</property>
	<property>
	  <name>ssl.client.truststore.reload.interval</name>
	  <value>10000</value>
	  <description>Truststore reload check interval, in milliseconds.
	  Default value is 10000 (10 seconds).
	  </description>
	</property>
	<property>
	  <name>ssl.client.keystore.location</name>
	  <value>/opt/ca/keystore</value>
	  <description>Keystore to be used by clients like distcp. Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.client.keystore.password</name>
	  <value>hadoop@1234</value>
	  <description>Optional. Default value is "".
	  </description>
	</property>
	<property>
	  <name>ssl.client.keystore.keypassword</name>
	  <value>hadoop@1234</value>
	  <description>Optional. Default value is "".
	  </description>
	</property>
	<property>
	  <name>ssl.client.keystore.type</name>
	  <value>jks</value>
	  <description>Optional. The keystore file format, default value is "jks".
	  </description>
	</property>
	</configuration>
```  
   
2. ssl-server.xml  
本次操作配置文件示例：  
```xml
	<configuration>
	<property>
	  <name>ssl.server.truststore.location</name>
	  <value>/opt/ca/truststore</value>
	  <description>Truststore to be used by NN and DN. Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.server.truststore.password</name>
	  <value>hadoop@1234</value>
	  <description>Optional. Default value is "".
	  </description>
	</property>
	<property>
	  <name>ssl.server.truststore.type</name>
	  <value>jks</value>
	  <description>Optional. The keystore file format, default value is "jks".
	  </description>
	</property>
	<property>
	  <name>ssl.server.truststore.reload.interval</name>
	  <value>10000</value>
	  <description>Truststore reload check interval, in milliseconds.
	  Default value is 10000 (10 seconds).
	  </description>
	</property>
	<property>
	  <name>ssl.server.keystore.location</name>
	  <value>/opt/ca/keystore</value>
	  <description>Keystore to be used by NN and DN. Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.server.keystore.password</name>
	  <value>hadoop@1234</value>
	  <description>Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.server.keystore.keypassword</name>
	  <value>hadoop@1234</value>
	  <description>Must be specified.
	  </description>
	</property>
	<property>
	  <name>ssl.server.keystore.type</name>
	  <value>jks</value>
	  <description>Optional. The keystore file format, default value is "jks".
	  </description>
	</property>
	<property>
	  <name>ssl.server.exclude.cipher.list</name>
	  <value>TLS_ECDHE_RSA_WITH_RC4_128_SHA,SSL_DHE_RSA_EXPORT_WITH_DES40_CBC_SHA,SSL_RSA_WITH_DES_CBC_SHA,SSL_DHE_RSA_WITH_DES_CBC_SHA,SSL_RSA_EXPORT_WITH_RC4_40_MD5,SSL_RSA_EXPORT_WITH_DES40_CBC_SHA,SSL_RSA_WITH_RC4_128_MD5</value>
	  <description>Optional. The weak security cipher suites that you want excluded from SSL communication.</description>
	</property>
	</configuration>
```  
以上，关于Hfds中开启SASL整体配置说明完毕。  
  
下面总结一下都做了什么：  
- hdfs-site.xml中开启TLS/SSL HDFS配置；   
- hdfs-site.xml中dfs.datanode.address、dfs.datanode.http.address的端口一定大于1024（非特权端口）；   
- hadoop-env.sh中的HADOOP_SECURE_DN_USER内容一定为空，并且和jsvc模式不同，不需要进行jsvc的环境部署；   
- 配置https：生成每个机器的密钥和证书；创建自己的CA；添加CA到客户机器的信任库；签名证书；  
- hdfs中整合https：配置ssl-server.xml和ssl-client.xml。  

#### iv.测试启动（集群）  
以上，关于hdfs中开启SASL的整体配置完毕，下面即可直接进行集群的启动。  
补充：此处作为补充完整描述下全新集群的启动过程，通常情况下，直接启动即可。    
1. 全新HA集群启动过程：  
    1. kinit验证
    2. 分别启动zookeeper  
    
        ```
        zkServer.sh start
        ```

    3. 启动三台JournalNode：node1、node2、node3  
    
        ```
        hadoop-daemon.sh start journalnode
        ```
    4. 在其中一个NameNode上格式化hadoop.tmp.dir并初始化(格式化完成之后，将会在$$\$\{dfs.namenode.name.dir\}/current$$目录下生成元数据，$$\$\{dfs.namenode.name.dir\}$$是在hdfs-site.xml中配置的，默认值是$$file://\$\{hadoop.tmp.dir\}/dfs/name$$，dfs.namenode.name.dir属性可配多个目录，如 /data1/dfs/name，/data2/dfs/name，/data3/dfs/name等。各个目录存储的文件结构和内容都完全一样，相当于备份，这样做的好处是当其中一个目录损坏了，也不会影响到Hadoop的元数据，特别是当其中一个目录是NFS（网络文件系统 Network File System，NFS）之上，即使你这台机器损坏了，元数据也得到保存；$$\$\{hadoop.tmp.dir\}$$是在core-si te.xml文件内配置的） 
        ```
        #node1
        hdfs namenode -format
        ```
    5. 把格式化后的元数据拷备到另一台NameNode节点上    
        ```
        #node1
        $ scp -r ${hadoop.tmp.dir} root@node2:${hadoop.tmp.dir}
        ```  
    6. 启动NameNode  
    
        ```
        #node1
        hadoop-daemon.sh start namenode
        #node2
        hdfs namenode -bootstrapStandby（Y/N？）
        hadoop-daemon.sh start namenode
        ```  
    7. 初始化zkfc  
    
        ```
        #Node1:
        hdfs zkfc -formatZK
        ```  
    8. 全面停止并全面启动
    
        ```
        #Node1:
        stop-dfs.sh
        start-dfs.sh
        ```
2. 因为启动过程中用到了hadoopHA，因此，可测试namenode的HA切换。  
    1. 用命令查看namenode的状态（两台namenode，在hdfs-si te.xml中配置的名字和主机名保持了一直，也是node1，所以这里测试写node1）：  
        
        ```
        hdfs haadmin -getServiceState node1
        ```
    2. 通过使用上述命令，有时候出现两台namenode都是standby的情况，则可以使用如下命令进行强制切换：  
    
        ```
        hdfs haadmin -transitionToActive --forcemanual node1
        ```
    3. 再次使用上述命令查看切换状态。
  
---
至此，本篇内容完成。以上内容基本上是完全关于HDFS中的Kerberos部署，HDFS结合Kerberos的整体部署完毕，若配置中使用了HA，则需要进行下文ZK配置之后才能完成完整部署。  

如有问题，请发送邮件至leafming@foxmail.com联系我，谢谢～  
©商业转载请联系作者获得授权，非商业转载请注明出处。

