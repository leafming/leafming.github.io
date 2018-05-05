---
layout: post
title: "Kerberos具体实践1-Kerberos环境准备及安装"
date: 2018-05-05
description: 本文主要关于Kerberos的简单应用。因为工作测试需要，自己装了一套集群进行了Kerberos的部署，并且与HDFS、ZK进行整合，然后将操作过程进行了整理，以便后续再查看。本文主要涉及到Kerberos集群的安装以及基本命令的使用，后续会继续说明其与HDFS、ZK的整合操作。
categories:
- BigData
tags:
- Kerberos
---
> 本文属于Kerberos具体实践整理的第一部分，主要涉及到Kerberos集群的安装以及基本命令的使用。

# 概述
## 为什么要将HDFS和Kerberos整合  
HDFS权限控制过于简单，直接使用hadoop的acl进行权限控制，可能会出现在程序中使用**System.setProperty("HADOOP_USER_NAME","root");**直接更改用户，伪装成任意用户使用。因此，需要加强HDFS权限控制，则采用进行kerberos认证的方式。  
hadoop的用户鉴权是基于JAAS的，其中hadoop.security.authentication属性有simple 和kerberos 等方式。如果hadoop.security.authentication等于”kerberos”,那么是“hadoop-user-kerberos”或者“hadoop-keytab-kerberos”，否则是“hadoop-simple”。在使用了kerberos的情况下，从javax.security.auth.kerberos.KerberosPrincipal的实例获取username。在没有使用kerberos时，首先读取hadoop的系统环境变量，如果没有的话，对于windows 从com.sun.security.auth.NTUserPrincipal获取用户名，对于类unix 从com.sun.security.auth.UnixPrincipal中获得用户名，然后再看该用户属于哪个group，从而完成登陆认证。 
## Kerberos原理  
### 术语说明  

| 术语 | 简述 |
| --- | --- | 
| KDC | 在启用Kerberos的环境中，KDC用于验证各个模块 |
| Kerberos KDC Server | KDC所在的机器 |
| Kerberos Client | 任何一个需要通过KDC认证的机器（或模块）|
| Principal | 用于验证一个用户或者一个Service的唯一的标识，相当于一个账号，需要为其设置密码（这个密码也被称之为Key）|
| Keytab | 包含有一个或多个Principal以及其密码的文件 |
| Relam | 由KDC以及多个Kerberos Client组成的网络 |
| KDC Admin Account | KDC中拥有管理权限的账户（例如添加、修改、删除Principal）|
| Authentication Server (AS) | 用于初始化认证，并生成Ticket Granting Ticket (TGT) |
| Ticket Granting Server (TGS) | 在TGT的基础上生成Service Ticket。一般情况下AS和TGS都在KDC的Server上 |  

Kerberos原理则不在此处进行说明，以后再做补充@TODO。本文直接先说明测试部署过程。  
# 部署操作
本文中部署操作首先先进行Kerberos单节点KDC部署，之后将单节点KDC的基础上结合HDFS进行配置，最后对整体集群进行优化，部署主从KDC防止Kerberos单点故障。  
全部部署过程均在RHEL 7.1系统上进行。  
> 参考链接：  
> [Kerberos从入门到放弃（一）：HDFS使用kerberos](https://ieevee.com/tech/2016/06/07/kerberos-1.html)  
> [HDFS配置Kerberos认证](https://yq.aliyun.com/articles/25636?spm=5176.100240.searchblog.8.wbY2wv)   
  
## 环境说明
系统环境：  
操作系统：RHEL 7.1
Hadoop版本：2.6.0-cdh5.11.0
JDK版本：jdk1.8.0  
运行用户：hadoop
## Kerberos单节点KDC部署
节点规划:  
  
| hosts | 角色 |
| --- | --- |
| node1 | kerberos server、AS、TGS、agent、namenode、datanode、zk |
| node2 | kerberos client、SecondaryNameNode 、datanode、zk |
| node3 | kerberos client、datanode、zk |  

### (1)添加主机名解析到/etc/hosts文件或配置DNS  
在Kerberos中，Principal中需各主机的FQDN名，所以集群中需要有DNS。  
但在本次测试中，未进行DNS配置，直接使用配置hosts进行测试。  
```
[root@node1 ~]# cat /etc/hosts
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.211.55.10 node1
10.211.55.11 node2
10.211.55.12 node3
```  
**注意：hostname 请使用小写，要不然在集成 kerberos 时会出现一些错误。**  
### (2)配置NTP
Kerberos集群对时间同步敏感度较高，默认时间相差超过5分钟就会出现问题，所以最好是在集群中增加NTP服务器。不过我的集群机器比较少，先跳过NTP吧。  
### (3)安装Kerberos  
Kerberos有不同的实现，如MIT KDC、Microsoft Active Directory等。我这里采用的是MIT KDC。  
在 KDC (这里是 node1 ) 上安装包 krb5、krb5-server 和 krb5-client：   
```
$ yum install krb5-server krb5-libs krb5-workstation  -y 
```
在其他节点（node1、node2、node3）安装 krb5-devel、krb5-workstation：     
```
$ yum install krb5-libs krb5-workstation  -y
```  
### (4).修改配置文件
kdc 服务器涉及到三个配置文件：  
/etc/krb5.conf  
/var/kerberos/krb5kdc/kdc.conf  
/var/kerberos/krb5kdc/kadm5.acl    
#### a).krb5.conf配置
/etc/krb5.conf:包含Kerberos的配置信息。例如，KDC的位置，Kerberos的admin的realms 等。需要所有使用的Kerberos的机器上的配置文件都同步。这里仅列举需要的基本配置。详细参考[krb5.conf](http://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)  

```
[root@node1 etc]# cat krb5.conf
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/
[logging] #[logging]：表示 server 端的日志的打印位置
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults] #[libdefaults]：每种连接的默认配置，需要注意以下几个关键的小配置
 default_realm = HADOOP.COM #设置Kerberos应用程序的默认领域。如果您有多个领域，只需向[realms]节添加其他的语句
 dns_lookup_realm = false
 #clockskew = 120 #时钟偏差是不完全符合主机系统时钟的票据时戳的容差，超过此容差将不接受此票据。通常，将时钟扭斜设置为 300 秒（5 分钟）。这意味着从服务器的角度看，票证的时间戳与它的偏差可以是在前后 5 分钟内。~~
 ticket_lifetime = 24h #表明凭证生效的时限，一般为24小时
 renew_lifetime = 7d #表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败
 forwardable = true #允许转发解析请求  
 rdns = false
 udp_preference_limit = 1 #禁止使用udp可以防止一个Hadoop中的错误

[realms] #列举使用的realm
HADOOP.COM = {
  kdc = node1:88 #代表要kdc的位置。格式是机器:端口。测试过程中也可不加端口。
  admin_server = node1:749 #代表admin的位置。格式是机器:端口。测试过程中也可不加端口。
  default_domain = HADOOP.COM #代表默认的域名。
 }

[kdc]
 profile=/var/kerberos/krb5kdc/kdc.conf
 
#[domain_realm] 
#.hadoop.com = HADOOP.COM
#hadoop.com = HADOOP.COM
```  
总结：  
- 这个文件可以用include和includedir引入其他文件或文件夹，但必须是绝对路径  
- realms:包含kerberos realm的名字（部分键控），显示关于realm的特殊信息，包括该realm内的主机要从哪里寻找kerberos的server。  
- domain realm：将domain名和子domain名映射到realm名上    
- 必须填的有以下几项：    
- default-realm：在libdefault部分  
- admin_server：在realm部分  
- domain_realm：当domain名和realm名不同的时候要设置  
- logging：当该机器作为KDC时要设置  
- [appdefaults]：可以设定一些针对特定应用的配置，覆盖默认配置。

#### b).kdc.conf配置
/var/kerberos/krb5kdc/kdc.conf:包括KDC的配置信息。默认放在 /usr/local/var/krb5kdc。或者通过覆盖KRB5_KDC_PROFILE环境变量修改配置文件位置。详细参考[kdc.conf](http://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/kdc_conf.html)。  
```
[root@node1 ~]# cat /var/kerberos/krb5kdc/kdc.conf
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 HADOOP.COM = { #是设定的 realms。名字随意。Kerberos 可以支持多个 realms，会增加复杂度。大小写敏感，一般为了识别使用全部大写。这个realms跟机器的host没有大关系。
  #master_key_type = aes256-cts
  #和supported_enctypes默认使用aes256-cts。由于，JAVA使用aes256-cts验证方式需要安装额外的jar包（后面再做说明）。推荐不使用，并且删除aes256-cts。
  kadmind_port = 749
  acl_file = /var/kerberos/krb5kdc/kadm5.acl #标注了admin的用户权限，需要用户自己创建。文件格式是：Kerberos_principal permissions [target_principal] [restrictions] 支持通配符等。最简单的写法是*/admin@HADOOP.COM *,代表名称匹配*/admin@HADOOP.COM 都认为是admin，权限是 *。代表全部权限。
  dict_file = /usr/share/dict/words
  database_name = /var/kerberos/krb5kdc/principal
  key_stash_file =  /var/kerberos/krb5kdc/.k5.HADOOP.COM
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab #KDC 进行校验的 keytab
  max_life = 24h
  max_renewable_life = 10d #涉及到是否能进行ticket的renwe必须配置
  default_principal_flags = +renewable, +forwardable
  supported_enctypes = des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal #支持的校验方式.注意把aes256-cts去掉
 }
```  
总结：  
- HADOOP.COM： 是设定的 realms。名字随意。Kerberos可以支持多个。  
- realms，会增加复杂度。大小写敏感，一般为了识别使用全部大写。这个 realms 跟机器的 host 没有大关系。  
- master_key_type：和 supported_enctypes 默认使用 aes256-cts。JAVA 使用 aes256-cts 验证方式需要安装 JCE 包，见下面的说明。为了简便，你可以不使用aes256-cts 算法，这样就不需要安装 JCE 。  
- acl_file：标注了 admin 的用户权限，需要用户自己创建。文件格式是：Kerberos_principal permissions [target_principal] [restrictions]  
- supported_enctypes：支持的校验方式。  
- admin_keytab：KDC 进行校验的 keytab。  

> 补充说明-关于AES-256加密（未测试，参考[HDFS配置Kerberos认证](https://yq.aliyun.com/articles/25636?spm=5176.100240.searchblog.8.wbY2wv)）：   
> 对于使用 centos5. 6 及以上的系统，默认使用 AES-256 来加密的。这就需要集群中的所有节点上安装 JCE，如果你使用的是 JDK1.6 ，则到 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 6 页面下载，如果是 JDK1.7，则到 Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 7 下载。下载的文件是一个 zip 包，解开后，将里面的两个文件放到下面的目录中：$JAVA_HOME/jre/lib/security  

#### c).kadm5.acl配置  
为了能够不直接访问KDC控制台而从Kerberos数据库添加和删除主体，请对Kerberos管理服务器指示允许哪些主体执行哪些操作。通过编辑文件 /var/kerberos/krb5kdc/kadm5.acl完成此操作。ACL（访问控制列表）允许您精确指定特权。
```
[root@node1 ~]# cat /var/kerberos/krb5kdc/kadm5.acl
*/admin@HADOOP.COM	*
```

### (5)同步配置文件  
将 kdc 中的 /etc/krb5.conf 拷贝到集群中其他服务器即可。
```
$ scp /etc/krb5.conf node2:/etc/krb5.conf
$ scp /etc/krb5.conf node3:/etc/krb5.conf
```
请确认集群如果关闭了 selinux。

### (6)创建数据库  
在node1上运行初始化数据库命令。其中 -r 指定对应 realm。  
该命令会在 /var/kerberos/krb5kdc/ 目录下创建 principal 数据库。  
如果遇到数据库已经存在的提示，可以把/var/kerberos/krb5kdc/目录下的principal的相关文件都删除掉。默认的数据库名字都是 principal。可以使用 -d 指定数据库名字。  
```
$ kdb5_util create -r JAVACHEN.COM -s
```
> 补充小技巧-（未测试，其他帖子看到的）：出现Loading random data的时候另开个终端执行点消耗CPU的命令如cat /dev/sda > /dev/urandom 可以加快随机数采集。

操作结果小例子：
```
[root@node1 etc]# kdb5_util create -r HADOOP.COM -s
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'HADOOP.COM',
master key name 'K/M@HADOOP.COM'
You will be prompted for the database Master Password.    #在此处输入的是root@1234
It is important that you NOT FORGET this password.
Enter KDC database master key:
Re-enter KDC database master key to verify:
```  

### (7)启动服务
在node1节点上运行：
```
[root@node1 etc]# service krb5kdc start
Redirecting to /bin/systemctl start krb5kdc.service
[root@node1 etc]# service kadmin start
Redirecting to /bin/systemctl start kadmin.service
```
至此kerberos，搭建完毕。

## Kerberos基本环境创建及简单命令测试  
### (1)创建Kerberos管理员
关于 kerberos 的管理，可以使用 kadmin.local 或 kadmin，至于使用哪个，取决于账户和访问权限：  
- 如果有访问 kdc 服务器的 root 权限，但是没有 kerberos admin 账户，使用 kadmin.local  
- 如果没有访问 kdc 服务器的 root 权限，但是用 kerberos admin 账户，使用 kadmin

在node1上创建远程管理的管理员，系统会提示输入密码，密码不能为空，且需妥善保存。：
```
#手动输入两次密码，这里密码为 root
$ kadmin.local -q "addprinc root/admin"
# 也可以不用手动输入密码，通过echo将密码引入
$ echo -e "root\nroot" | kadmin.local -q "addprinc root/admin"
```
建议：  
最好把username设为root，会提示输入密码，可以输入一个密码。  
创建第一个principal，必须在KDC自身的终端上进行，而且需要以root登录，这样才可以执行kadmin.local命令。  

操作结果小例子：  
```
[root@node1 etc]# kadmin.local -q "addprinc root/admin"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for root/admin@HADOOP.COM; defaulting to no policy
Enter password for principal "root/admin@HADOOP.COM":
Re-enter password for principal "root/admin@HADOOP.COM":
Principal "root/admin@HADOOP.COM" created.
```

### (2)添加更多主体-创建host主体并将自己的host主题添加到自己密钥表文件  
#### a).基于Kerberos 的应用程序(例如 kprop)将使用主机主体将变更传播到从KDC服务器。也可以通过该主体提供对使用应用程序(如ssh)的KDC服务器的安全远程访问。  
请注意，当主体实例为主机名时，无论名称服务中的域名是大写还是小写，都必须以小写字母指定FQDN。  
  
```
kadmin: addprinc -randkey host/node1
Principal "host/node1@HADOOP.COM" created. 
kadmin:
```  
或用命令（创建了node1、node2、node3的host主体）：  
```  
kadmin.local -q "addprinc -randkey host/node1@HADOOP.COM"
kadmin.local -q "addprinc -randkey host/node2@HADOOP.COM"
kadmin.local -q "addprinc -randkey host/node3@HADOOP.COM"
```    
#### b).将KDC服务器的host主体添加到KDC服务器的密钥表文件。  
通过将主机主体添加到密钥表文件，允许应用服务器(如sshd)自动使用该主体。  
**注意：将自己的host主体添加到自己的密钥表文件。**  
  
```
#在node1上执行
kadmin: ktadd host/node1
```  

### (3)Kerberos简单命令测试  
输入kadmin或者kadmin.local命令进入kerberos的shell(登录到管理员账户:如果在本机上，可以通过kadmin.local直接登录。其它机器的，先使用kinit进行验证)，如下：  
```
1、其他机器上先使用kinit进行验证：
echo root@1234|kinit root/admin  
2、之后再使用kadmin或者kadmin.local命令进入kerberos的shell
```  
#### a).principals操作  
查看principals  
```
$ kadmin: list_principals
```  
添加一个新的 principal  
```  
$ kadmin:  addprinc user1  
```  
删除 principal  
```
$ kadmin:  delprinc user1  
$ kadmin: exit
```  
也可以直接通过下面的命令来执行：    
```
#提示需要输入密码
$ kadmin -p root/admin -q "list_principals"
$ kadmin -p root/admin -q "addprinc user2"
$ kadmin -p root/admin -q "delprinc user2"

#不用输入密码
$ kadmin.local -q "list_principals"
$ kadmin.local -q "addprinc user2"
$ kadmin.local -q "delprinc user2"
```  
  
#### b).ticket操作:创建一个测试用户test，密码设置为test  
  
```
$ echo -e "test\ntest" | kadmin.local -q "addprinc test"
```
获取 test 用户的 ticket：
```
#通过用户名和密码进行登录
[root@node2 ~]# kinit test
Password for test@HADOOP.COM:
[root@node2 ~]# klist  -e
Ticket cache: KEYRING:persistent:0:krb_ccache_CGo77JQ
Default principal: test@HADOOP.COM
Valid starting       Expires              Service principal
09/06/2017 16:06:57  09/07/2017 16:06:57  krbtgt/HADOOP.COM@HADOOP.COM
	renew until 09/13/2017 16:06:57, Etype (skey, tkt): des3-cbc-sha1, des3-cbc-sha1
```  
销毁该 test 用户的 ticket:  
```
[root@node2 ~]# kdestroy
Other credential caches present, use -A to destroy all
[root@node2 ~]# kdestroy -A
[root@node2 ~]# klist  -e
klist: Credentials cache keyring 'persistent:0:krb_ccache_CGo77JQ' not found
[root@node2 ~]# klist
klist: Credentials cache keyring 'persistent:0:krb_ccache_CGo77JQ' not found
```  
更新 ticket:  
```
$ kinit root/admin
  Password for root/admin@HADOOP.COM:
$  klist
  Ticket cache: FILE:/tmp/krb5cc_0
  Default principal: root/admin@JAVACHEN.COM
  Valid starting     Expires            Service principal
  11/07/14 15:33:57  11/08/14 15:33:57  krbtgt/JAVACHEN.COM@JAVACHEN.COM
    renew until 11/17/14 15:33:57
  Kerberos 4 ticket cache: /tmp/tkt0
  klist: You have no tickets cached
$ kinit -R
$ klist
  Ticket cache: FILE:/tmp/krb5cc_0
  Default principal: root/admin@JAVACHEN.COM
  Valid starting     Expires            Service principal
  11/07/14 15:34:05  11/08/14 15:34:05  krbtgt/JAVACHEN.COM@JAVACHEN.COM
    renew until 11/17/14 15:33:57
  Kerberos 4 ticket cache: /tmp/tkt0
  klist: You have no tickets cached
```  
#### c). 抽取密钥到在本地keytab文件  
抽取密钥并将其储存在本地keytab文件/etc/krb5.keytab中。这个文件由超级用户拥有，所以您必须是root用户才能在kadmin shell 中执行以下命令:  
```
$ kadmin.local -q "ktadd kadmin/admin"
$ klist -k /etc/krb5.keytab
  Keytab name: FILE:/etc/krb5.keytab
  KVNO Principal
  ---- ----------------------------------------------------------------------
     3 kadmin/admin@LASHOU-INC.COM
     3 kadmin/admin@LASHOU-INC.COM
     3 kadmin/admin@LASHOU-INC.COM
     3 kadmin/admin@LASHOU-INC.COM
     3 kadmin/admin@LASHOU-INC.COM
   -----------------
```

至此，本篇内容完成。以上为Kerberos单KDC版搭建过程，下篇文章进行HDFS和Kerberos整合操作的整理。  
如有问题，请发送邮件至leafming@foxmail.com联系我，谢谢～  
©商业转载请联系作者获得授权，非商业转载请注明出处。

