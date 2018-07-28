---
layout: post
title: "Kerberos具体实践4-Kerberos主从KDC部署"
date: 2018-06-29
updated: 2018-06-29
description: 本文属于Kerberos具体实践整理的第四部分，主要涉及kerberos主从KDC部署。
categories:
- BigData
tags:
- Kerberos
---
> 本文主要关于Kerberos的简单应用。因为工作测试需要，自己装了一套集群进行了Kerberos的部署，并且与HDFS、ZK进行整合，然后将操作过程进行了整理，以便后续再查看。本文主要涉及Kerberos主从KDC部署，此文为2018-06-29文章Kerberos具体实践3的后续；本文为本系列的最后一篇。  
  
# Kerberos主从KDC部署  
在进行生产环境中进行Kerberos部署时，最好配置1个master KDC以及多个slave KDC来保证Kerberos服务的可靠性。每个KDC都能够包括一个Kerberos数据库的副本。在master KDC中包含一个可写的realm数据库副本，并且他会定期同步到slave KDC中。所有数据库的更改（如更改密码）都是由master KDC负责。当主KDC不可用，slave KDCs提供Kerberos的票据授予服务，不负责数据库管理。  

规划：  
node1   - master KDC    
node2  - slave KDC  
-HADOOP.COM - realm name  
.k5.ATHENA.MIT.EDU  - stash file  
root/admin         - admin principal  

主从KDC部署可在远单节点KDC的基础上进行配置更改，配置内容基本与单节点情况相似，关键细节参考上述单节点部署。  
本次操作即，当配置好单节点KDC之后，再其基础上进行更改。  

## 准备工作-添加主体  
部署单节点KDC完毕，node1为主KDC，在主KDC服务器上启动kadmin。  
1. 在主KDC服务器上，向数据库中添加主机主体（如果尚未完成）：  
每个kdc都要有host key在kerberos数据库中。这些keys用于双向认证的时候，从主kdc向从属KDC传播数据库转储文件。  
如果需要将从KDC和主KDC交换的话，要使从KDC服务器正常工作，它必须有主机主体。  
注：当主体实例名为主机名时，无论名称服务中的域名是大写还是小写，都必须以小写字母指定FQDN。  
```
kadmin：addprinc -randkey host/node2
```  
2. 在主KDC服务器上，创建kiprop主体。  
kiprop主体用于授权来自主KDC服务器的更新。
```
kadmin：addprinc -randkey kiprop/node1
kadmin：addprinc -randkey kiprop/node2
```  
  
## 在node2上安装Kerberos-server    
node2:  
```
$ yum install krb5-server -y 
```  
  
## 修改配置文件
kdc 服务器涉及到三个配置文件：  
/etc/krb5.conf  
/var/kerberos/krb5kdc/kdc.conf  
/var/kerberos/krb5kdc/kadm5.acl  
### krb5.conf配置-node1  
在主KDC服务器（node1）上编辑krb5.conf文件，基本和单节点KDC配置类似，仅在[realms]部分加入slave kdc信息，并且设置admin_server为主kdc即可，如下：    
```
#node1上操作krb5.conf，之后再拷贝到其他机器
[realms]
HADOOP.COM = {
  kdc = node1
  kdc = node2
  admin_server = node1
  default_domain = HADOOP.COM
 }
```
配置文件整体示例：  
```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 # forwardable = true
 rdns = false
 default_realm = HADOOP.COM
 udp_preference_limit = 1

[realms]
HADOOP.COM = {
  kdc = node1
  kdc = node2
  admin_server = node1
  #default_domain = HADOOP.COM
 }

[kdc]
 profile=/var/kerberos/krb5kdc/kdc.conf

# [domain_realm]
# .hadoop.com = HADOOP.COM
# hadoop.com = HADOOP.COM
# # if the domain name and realm name are equivalent,
# # this entry is not needed
```  
  
### kdc.conf配置-node1  
在主KDC服务器（node1）上编辑kdc.conf文件，realms部分添加用于增量传播和选择主KDC服务器将在日志中保留的更新数的行。  

```
[realms]
HADOOP.COM={
    ...
  iprop_enable = true
  iprop_master_ulogsize = 1000
    ...
}
```

### kadm5.acl配置-node1   
在主KDC服务器上，向kadm5.acl添加kiprop项。  
通过此项，主KDC服务器可以接受对kdc2服务器的kerberos数据库增量传播请求。
```
[root@node1 ~]# cat /var/kerberos/krb5kdc/kadm5.acl
root/admin@HADOOP.COM    *
kiprop/node2@HADOOP.COM p
```

### 在KDC服务器上，重启kadmin以使用kadm5.acl文件中新的内容  
```
[root@node1 etc]# service kadmin restart
```
### 将主KDC服务器上以下文件复制到所有从KDC上  
主KDC和从KDC间的数据库扩展，传播的是master的数据库内容的拷贝，而不是配置文件、stash文件或kadm5 ACL文件。
需要在所有的从 KDC 服务器上执行该步骤，因为主 KDC 服务器已经更新了每个 KDC 服务器需要的信息。
因此以下文件必须手动从主KDC拷贝到从KDC的对应目录上。
- krb5.conf
- kdc.conf
- kadm5.acl（kadm5.acl is only needed to allow a slave to swap with the master KDC.）
- master key stash file  

```
$ scp /etc/krb5.conf node2:/etc/krb5.conf
$ scp /var/kerberos/krb5kdc/kdc.conf node2:/var/kerberos/krb5kdc/kdc.conf
$ scp /var/kerberos/krb5kdc/.k5.HADOOP.COM node2:/var/kerberos/krb5kdc/.k5.HADOOP.COM
#此步骤有人说可以在新的从 KDC 服务器上，使用kdb5_util创建一个存储文件。（需测试）
$ scp /var/kerberos/krb5kdc/kadm5.acl node2:/var/kerberos/krb5kdc/kadm5.acl
```
  
## 从、主服务器其他配置操作
### 在所有的从KDC服务器上，将主KDC服务器和每个从KDC服务器的项添加到数据库传播配置文件kpropd.acl中
在master KDC和slave KDCs之间传输数据库是通过kpropd daemon来实现，因此需要明确的定义允许在slave机器的新数据库上提供kerberos dump更新的主体。  
在所有从KDC服务器上，在KDC state文件夹中创建kpropd.acl文件，将主KDC服务器和每个从KDC服务器的主机主体项添加到其中。  
```
[root@node2 krb5kdc]# cat kpropd.acl
host/node1@HADOOP.COM
host/node2@HADOOP.COM
```  
说明：从KDC服务器上的kpropd.acl文件提供主机主体名称的列表(每个名称占一行)，用于指定该KDC可以通过传播从其接收更新数据库的系统。如果希望主KDC和从KDC能够交换，则需要在全部KDC的kpropd.acl列出全部KDC服务器的host主体。否则，如果仅使用主KDC服务器传播到所有从KDC服务器，则每个从KDC服务器上的kpropd.acl文件仅需包含主KDC服务器的主机主体名称。  

### 在所有从KDC服务器上，确保未填充kerberos访问控制列表文件kadm5.acl。
如果该文件中包含kiprop项，请将其删除。  

### 在新的从KDC服务器上，更改kdc.conf某个项  
使用定义“iprop_slave_poll = 2m”替换“iprop_master_ulogsize = 1000”。该项将轮询时间设为2分钟。
```
[realms]
HADOOP.COM={
    ...
  iprop_enable = true
  iprop_slave_poll = 2m
    ...
}
```
### 在新的从KDC服务器上，启动kadmin命，添加主体到密钥表。  
必须使用在配置主KDC服务器时创建的一个admin主体名称登录。  

```
echo root@1234|kinit root/admin
```   
1. 使用kadmin将从KDC服务器的主机主体添加到从KDC服务器的密钥表文件中。  
该项可使kprop及其他基于Kerberos的应用程序正常工作。请注意，当主体实例为主机名时，无论名称服务中的域名是大写还是小写，都必须以小写字母指定FQDN。  
注：添加自己的host主体到自己的密钥表文件中。  
```
#在node2上执行  
kadmin: ktadd host/node2  
```    
理想情况下，应该在自己本地导出自己的keytab。如果不可行，则应该创建一个加密的会话来在网络中传输。  

2. 将kiprop主体添加到从KDC服务器的密钥表文件中。  
通过将kiprop主体添加到krb5.keytab文件，允许kpropd命令在启动增量传播时对其自身进行验证。  
```
#在node2执行
kadmin: ktadd kiprop/node2
```
  
### 在新的从KDC服务器上，启动Kerberos传播守护进程。（不是增量传播是否需要此步骤@TODO记不清了）
如果需要启动增量传播，需要启动kpropd作为一个独立的进程。

```
kpropd
```

至此，slave KDC可以接受数据库传播，需要将数据库从master server中传过来。  
注意：目前不能够启动slave KDC，因为还没有复制master的database。  

## 将数据库复制到每一个slave KDC中
在master KDC（node1）上创建dump file：  

```
[root@node1 kerberos]# kdb5_util dump /var/kerberos/krb5kdc/slave_datatrans 
[root@node1 kerberos]# ls
krb5  krb5kdc  slave_datatrans  slave_datatrans.dump_ok
```  

### 手动传播（脚本传播）  
1. 手动传播操作  
手动传输数据库文件到每一个slave KDC上：  
```
[root@node1 kerberos]#  kprop -f /var/kerberos/krb5kdc/slave_datatrans  node2
Database propagation to node2: SUCCEEDED
```  
> 补充说明：Kerberos的数据库在master KDC上，因此需要定期的把这个数据库复制到slave KDCs上。如何设置数据库复制的频率，需要跟进传播时间以及用户等待密码生效的最大时间而设置。如果传播时间比这个最大限制时间长，（比如有一个很大的数据库，并且有很多slave，或者频繁的网络延迟），则可能希望能通过并行传播来减少传播延迟。因此，可以将数据库先从主节点传播给一部分从节点，然后从从节点再传播给其他节点。    
因为此时没有配置开启增量传播，因此，需要手动或脚本定时复制数据库，类似如下。  
  
	```
#脚本方式：
#!/bin/sh
kdclist = "kerberos-1.mit.edu kerberos-2.mit.edu"
kdb5_util dump /usr/local/var/krb5kdc/slave_datatrans
for kdc in $kdclist
do
    kprop -f /usr/local/var/krb5kdc/slave_datatrans $kdc
done
	```  
需要设置一个定时任务去运行这个脚本，或者在切换之前执行。  
至此，slave KDC中有了Kerberos database的复制, slave上/var/kerberos/krb5kdc/会多出一些文件。  
现在即可启动krb5kdc daemon:  
```
[root@node2 kerberos]# systemctl start krb5kdc.service
```  
  
2. 启动遇到问题处理   
    1. 手动传输数据库文件报错：  
    ```
    kprop: No route to host while connecting to server
    ```  
处理：Make sure that the hostname of the slave (as given to kprop) is correct, and that any firewalls between the master and the slave allow a connection on port 754.
    2. 从节点启动可以直接打“krb5kdc”来启动，但是直接krb5kdc和systemctl start krb5kdc.service只能使用一种。  
报错：  
```
    [root@node2 krb5kdc]# systemctl status krb5kdc.service -l
    ● krb5kdc.service - Kerberos 5 KDC
      Loaded: loaded (/usr/lib/systemd/system/krb5kdc.service; disabled; vendor preset: disabled)
       Active: failed (Result: exit-code) since Tue 2017-09-19 16:06:57 CST; 17s ago
      Process: 9097 ExecStart=/usr/sbin/krb5kdc -P /var/run/krb5kdc.pid $KRB5KDC_ARGS (code=exited, status=1/FAILURE)
    Sep 19 16:06:57 node2 systemd[1]: Starting Kerberos 5 KDC...
    Sep 19 16:06:57 node2 systemd[1]: krb5kdc.service: control process exited, code=exited status=1
    Sep 19 16:06:57 node2 systemd[1]: Failed to start Kerberos 5 KDC.
    Sep 19 16:06:57 node2 systemd[1]: Unit krb5kdc.service entered failed state.
    Sep 19 16:06:57 node2 systemd[1]: krb5kdc.service failed.
```  
处理：  
```
    [root@node2 krb5kdc]# ps -ef | grep krb5
    root      5080     1  0 15:47 ?        00:00:00 krb5kdc
    root      9330  2940  0 16:07 pts/0    00:00:00 grep --color=auto krb5
    [root@node2 krb5kdc]# kill 5080
    [root@node2 krb5kdc]# systemctl start krb5kdc.service
    [root@node2 krb5kdc]# systemctl status krb5kdc.service -l
    ● krb5kdc.service - Kerberos 5 KDC
       Loaded: loaded (/usr/lib/systemd/system/krb5kdc.service; disabled; vendor preset: disabled)
       Active: active (running) since Tue 2017-09-19 16:08:09 CST; 2s ago
      Process: 9345 ExecStart=/usr/sbin/krb5kdc -P /var/run/krb5kdc.pid $KRB5KDC_ARGS (code=exited, status=0/SUCCESS)
     Main PID: 9348 (krb5kdc)
       Memory: 2.7M
       CGroup: /system.slice/krb5kdc.service
               └─9348 /usr/sbin/krb5kdc -P /var/run/krb5kdc.pid    
    Sep 19 16:08:09 node2 systemd[1]: Starting Kerberos 5 KDC...
    Sep 19 16:08:09 node2 systemd[1]: PID file /var/run/krb5kdc.pid not readable (yet?) after start.
    Sep 19 16:08:09 node2 systemd[1]: Started Kerberos 5 KDC.
```  
  
### 增量传播（目前配置增量传播后，有些问题无法主从切换-因此此部分内容仅供参考，未证实可用@TODO，手动传播正常可以使用）  
当开启增量数据库传播，当master KDC更改了，则会将更改信息写到一个更新日志中，日志处于一个固定的缓存大小。
在每一个slave KDC 上会有一个进程链接master KDC上的一个服务，并且定期请求自上次检查以来所做的更改。默认情况下，每2分钟检查一次。如果数据库刚刚在前几秒进行了修改(currently the threshold is hard-coded at 10 seconds)，则slave不能检查到更新，而是暂停并在不久之后再次尝试。这减少了当一个管理员试图在同一时间进行很多数据库更改而导致增量更新查询造成延迟的可能性。  
每个realm的增量更新在kdc.conf中配置。  
1. 主、从配置文件如上面步骤已经配置  
2. 创建kiprop/hostname主体（master 和slave KDC）和keytab  （已经操作则不用再进行）  
master和slave KDC必须在Kerberos database中含有kiprop/hostname主体，并且创建默认keytab。  
在1.13版本后，kiprop/hostname principal会自动在master KDC中创建，但是在slave KDCs需手动创建。   
If incremental propagation is enabled, the principal kiprop/slavehostname@REALM (where slavehostname is the name of the slave KDC host, and REALM is the name of the Kerberos realm) must be present in the slave’s keytab file.  
3. slave KDC运行kpropd  
当增量传播开启之后，slave kdc 的kpropd进程会链接master kdc的kadmind，并且开始请求更新。  
通常情况下，kprop在增量传播支持中是禁用的。然而，如果很长时间内slave不能从master kdc中取到更新（比如因为网络原因），那么在master上的slave没来得急检索的日志可能被覆盖。在这种情况下，slave会通知master KDC转储当前的数据库到一文件中，并且调用一次kprop传播，并且通过特殊选项的设置，也能够更新日志中的关于slave应该获取增量更新的位置传输过去。因此，keytab和acl需要进行上述的kprop配置。  
如果环境中有大量的slave，则希望将他们安排在一个层次性结构中，而不是让master为每一个slave提供更新。这么做的情况下，在每一个中间的slave上使用kadmind -proponly命令，并且使用kpropd -A upstreamhostname 将下游slaves指向对应的的上游slave。  
当前实现中有几个已知的限制：  
    - The incremental update protocol does not transport changes to policy objects. Any policy changes on the master will result in full resyncs to all slaves.  
    - The slave’s KDB module must support locking; it cannot be using the LDAP KDB module.  
    - The master and slave must be able to initiate TCP connections in both directions, without an intervening NAT.    
    直接输入如下，启动kpropd作为一个独立的进程。  
    ```
    kpropd
    ```
    关闭：pkill kpropd  
4. 测试启动kdc  
  
## master和slave KDC切换  
通过手动同步数据库的方式，将数据库从主KDC复制到从KDC之后，可进行KDC主从的切换操作。  
仅当主 KDC 服务器由于某种原因出现故障，或者需要重新安装主 KDC 服务器(例如，由于安 装了新硬件)时，才应将主KDC服务器与从KDC 服务器进行交换。    
1. 切换步骤：  
- 如果master KDC正在运行，则需要在老的master KDC上执行：  
0、原主kdc上的kpropd.acl文件去掉[root@node1 krb5kdc]# mv kpropd.acl.bak kpropd.acl；注释掉kadm5.acl中的kiprop项。   
1、kill kadmind进程 systemctl stop kadmin.service  
2、关闭传播数据库的定时任务  
3、手动执行数据库传播脚本，保证从节点们都有最新的数据库复制  
- 在新的master KDC上执行：  
0、新主kdc上增加kpropd.acl文件[root@node2 krb5kdc]# mv kpropd.acl kpropd.acl.bak；增加kadm5.acl中的kiprop项，和原主kdc一致。  
1、开启kadmin进程 systemctl start kadmin.service  
2、运行传播数据库的定时任务  
3、切换 master KDC 的cname。如果不能够这么做，则需要去更改每一个client节点的krb5.conf文件里面配置的Kerberos realm。  
2. 测试主从是否生效(成功)    
    1. 从第三台服务器，使用kinit获取ticket，正常情况下会从master上获取  
    2. 关闭master上的kdc服务  
    3. 再次从第三台服务器上，使用kinit 获取ticket，如果成功，说明生效。也可以观察kdc的日志，在/var/log/krb5kdc.log。  
3. 错误处理   
    1. found kpropd.acl-报错如下：  
    ```
    [root@node1 krb5kdc]# systemctl status kadmin.service
    ● kadmin.service - Kerberos 5 Password-changing and Administration
       Loaded: loaded (/usr/lib/systemd/system/kadmin.service; enabled; vendor preset: disabled)
       Active: failed (Result: exit-code) since Tue 2017-09-19 15:29:26 CST; 8s ago
      Process: 32161 ExecStart=/usr/sbin/_kadmind -P /var/run/kadmind.pid $KADMIND_ARGS (code=exited, status=6)
     Main PID: 5242 (code=exited, status=2)
    Sep 19 15:29:26 node1 systemd[1]: Starting Kerberos 5 Password-changing and Administration...
    Sep 19 15:29:26 node1 _kadmind[32161]: Error. This appears to be a slave server, found kpropd.acl
    Sep 19 15:29:26 node1 systemd[1]: kadmin.service: control process exited, code=exited status=6
    Sep 19 15:29:26 node1 systemd[1]: Failed to start Kerberos 5 Password-changing and Administration.
    Sep 19 15:29:26 node1 systemd[1]: Unit kadmin.service entered failed state.
    Sep 19 15:29:26 node1 systemd[1]: kadmin.service failed.
    ```  
    问题解决：  
    成为主节点的KDC，不能有kpropd.acl文件，删除原主kpropd.acl。   
    解决参照：[报错解决](http://compgroups.net/comp.protocols.kerberos/kerberos-master-slave-setup-database/3021964)。
    2. 密钥版本不匹配-Decrypt integrity check failed  
    ```
    [root@node1 krb5kdc]#  kprop -f /var/kerberos/krb5kdc/slave_datatrans  node2
    kprop: Server rejected authentication (during sendauth exchange) while authenticating to server
    kprop: Key version is not available signalled from server
    Error text from server: Key version is not available
    或者报错：
    kprop: Decrypt integrity check failed while getting initial credentials
    ```  
    原因：您的票证可能无效。  
    解决方法分析：  
    请检验以下两种情况：  
    - 确保您的凭证有效。使用 kdestroy 销毁票证，然后使用 kinit 创建新的票证。  
    - 确保目标主机的密钥表文件的服务密钥版本正确。使用 kadmin 查看 Kerberos 数据库中服务主体（例如 host/FQDN-hostname）的密钥版本号。另外，在目标主机上使用 klist -k，以确保该主机具有相同的密钥版本号。可用“kvno  host/node2@HADOOP.COM”命令查看版本号。    
    实际解决：  
    我出错的原因是因为主KDC和从KDC中都有host/node2@HADOOP.COM和host/node1@HADOOP.COM的keytab内容，然后导致互相获取kvno的时候版本不匹配。  
    1」.主从同时删除keytab文件和kdestroy  
    ```
    #node1和node2上执行
    rm -rf krb5.keytab
    kadmin:kdestroy
    ```  
    2」.分别执行：  
    node1上：  
    ```
    [root@node1 krb5kdc]# kvno  host/node1@HADOOP.COM
    kvno: No credentials cache found (filename: /tmp/krb5cc_0) while getting client principal name
    kadmin: ktadd host/node1  
    [root@node1 krb5kdc]# kinit -k -t /etc/krb5.keytab host/node1@HADOOP.COM
    ```  
    node2上：  
    ```
    kadmin: ktadd host/node2  
    [root@node2 tmp]# kinit -k -t /etc/krb5.keytab   host/node2@HADOOP.COM
    ```
    3」.查看密钥版本是否互相匹配  
    node1上：  
    ```
    [root@node1 krb5kdc]# klist -k
    #此处此命令可查看到自己的kvno
    [root@node1 krb5kdc]# kvno  host/node1@HADOOP.COM
    host/node1@HADOOP.COM: kvno = 4
    [root@node2 tmp]# kvno host/node2@HADOOP.COM
    host/node2@HADOOP.COM: kvno = 6
    #看此是否与node2的对应kvno匹配
    ```
    node2上：  
    ```
    [root@node2 krb5kdc]# klist -k
    #此处此命令可查看到自己的kvno
    [root@node2 tmp]# kvno host/node2@HADOOP.COM
    host/node2@HADOOP.COM: kvno = 6
    [root@node2 krb5kdc]# kvno  host/node1@HADOOP.COM
    host/node1@HADOOP.COM: kvno = 4
    #看此是否与node1的对应kvno匹配
    ```  
    之后即可解决。  
    问题处理参考：[密钥版本不匹配](http://compgroups.net/comp.protocols.kerberos/decrypt-integrity-check-failed-3/1946334)。  
  
---
至此，本篇内容完成。  
以上，此文章以及0505、0628三篇内容基本上是完全关于Kerberos相关知识，HDFS、ZK结合Kerberos的部署等内容，由于时间有限以及测试不够充分，仍存在很多漏洞以及待处理的问题，以后有机会再深入学习一下。  
  

