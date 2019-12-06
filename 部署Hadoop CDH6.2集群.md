



[TOC]





## 一、概述

CDH，全称Cloudera's Distribution, including Apache Hadoop。是Hadoop众多分支中对应中的一种，由Cloudera维护，基于稳定版本的Apache Hadoop构建，提供了Hadoop的核心（可扩展存储、分布式计算），最为重要的是提供基于web的用户界面。

CDH的优点：版本划分清晰，更新速度快，支持Kerberos安全认证，支持多种安装方式（如Yum、rpm等）。

CDH分为Cloudera Manager管理平台和CDH parcel（parcel包含各种组件的安装包）。这里采用CDH6.2.0。



## 二、环境准备

| 角色                     | 主机名 | IP          | 操作系统 |
| ------------------------ | ------ | ----------- | -------- |
| clunder-manager 管理节点 | node1  | 10.20.20.14 | CentOS7  |
| 节点                     | node2  | 10.20.20.16 | CentOS7  |
| 节点                     | node3  | 10.20.20.18 | CentOS7  |
| 节点                     | node4  | 10.10.20.72 | CentOS7  |

说明：以下操作，如果备注是管理节点，就是在node1管理节点上操作；如果备注是所有节点，就是在node1-4所有节点上操作

## 三、操作系统准备（所有节点）

1、安装系统的时候，不需要swap分区

2、关闭防火墙

```bash
systemctl stop firewalld 关闭防火墙
systemctl disable firewalld 禁止防火墙开机自启
```



3、关闭selinux

```bash
vim /etc/selinux/config —> SELINUX=disabled 
```



4、配置hosts文件

```bash
#vi /etc/hosts
10.20.20.14 node1 bigdata-14
10.20.20.16 node2 bigdata-16
10.20.20.18 node3 bigdata-18
10.10.20.72 node4 bigdata-72
```



5、配置yum文件

```bash
mkdir /etc/yum.repos.d/default

mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/default/

curl http://mirrors.ubtrobot.com/repos/centos.repo -o /etc/yum.repos.d/ubtrobot.repo

yum clean all

yum makecache  
```



## 四、配置ssh无密码登录（管理节点）

manager节点执行ssh-keygen -t rsa 一路回车到结束，在/root/.ssh/下面会生成一个公钥文件id_rsa.pub

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 将公钥追加到authorized_keys
chmod 600 ~/.ssh/authorized_keys 修改权限
将~/.ssh从当前节点分发到其他各个节点。如：scp -r ~/.ssh/ root@node2:~/.ssh/

```

ssh 各个节点互相登陆



## 五、配置NTP服务（所有节点）

修改时区（改为中国标准时区）

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

安装ntp 

```bash
yum -y install ntp
```


ntp主机配置 

vim /etc/ntp.conf

manager节点

```bash
[root@bigdata-14 ntp]# egrep  -v "^#" /etc/ntp.conf |grep -v "^$"
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
server ntp.aliyun.com
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```



其他节点以管理节点为准：

```bash
[root@bigdata-16 ~]# egrep -v "^#" /etc/ntp.conf |egrep -v "^$"
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
server node1
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```



```bash
重新启动 ntp 服务：service ntpd restart
设置开机自启：systemctl enable ntpd.service
ntpdc -c loopinfo #查看与时间同步服务器的时间偏差
ntpq -p #查看当前同步的时间服务器
ntpstat #查看状态

# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*203.107.6.88    100.107.25.114   2 u  352 1024  377   47.164    2.188   2.741
```



## 六、操作系统其他配置（所有节点）

6.1	修改Linux swappiness参数(所有节点）
为了避免服务器使用swap功能而影响服务器性能，一般都会把vm.swappiness修改为0（cloudera建议10以下）

```bash
#echo 0>/proc/sys/vm/swappiness 
# vim /etc/sysctl.conf 
vm.swappiness=0

# cd /usr/lib/tuned/

#都修改成以下值
# grep "vm.swappiness" * -R
latency-performance/tuned.conf:vm.swappiness=0
throughput-performance/tuned.conf:vm.swappiness=0
virtual-guest/tuned.conf:vm.swappiness = 0
```



6.2	禁用透明页

```bash
#echo never > /sys/kernel/mm/transparent_hugepage/defrag
#echo never > /sys/kernel/mm/transparent_hugepage/enabled

#永久生效
# cat /etc/rc.local 
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
#chmod +x /etc/rc.d/rc.local
```



6.3	安装jdk

```bash
#rpm -qa | grep java # 查询已安装的java
#yum remove java* # 卸载
#rpm -ivh oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
#vi /etc/profile 末尾添加
#java
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin

#source /etc/profile
#java -version 
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
```

6.4 	创建/usr/share/java目录，将mysql-jdbc包放过去

```bash
#mkdir -p /usr/share/java
#mv /opt/mysql-j/mysql-connector-java-5.1.34.jar /usr/share/java/mysql-connector-java.jar

//mysql-connector-java-5.1.34.jar 一定要命名为mysql-connector-java.jar
```



## 七、安装mysql（管理节点）

```bash
#卸载mariadb
#rpm -qa|grep mariadb
#rpm -e --nodeps mariadb-libs-5.5.60-1.el7_5.x86_64

#安装mysql
#cd /opt/mysql/
#tar -xvf ./mysql-5.7.19-1.el7.x86_64.rpm-bundle.tar
#rpm -ivh mysql-community-common-5.7.19-1.el7.x86_64.rpm
#rpm -ivh mysql-community-libs-5.7.19-1.el7.x86_64.rpm
#rpm -ivh mysql-community-client-5.7.19-1.el7.x86_64.rpm
#rpm -ivh mysql-community-server-5.7.19-1.el7.x86_64.rpm
#rpm -ivh mysql-community-libs-compat-5.7.19-1.el7.x86_64.rpm


#MYSQL配置:
#mysqld --initialize --user=mysql # 初始化mysql使mysql目录的拥有者为mysql用户
#cat /var/log/mysqld.log #最后一行将会有随机生成的密码
#systemctl start mysqld.service # 设置mysql服务自启
# systemctl enable mysqld.service
#mysql -uroot –p 

#设置root密码
$>mysql -u root
mysql>use mysql;
mysql>update user set authentication_string = password(‘123456’), password_expired = ‘N’, password_last_changed = now() where user = ‘root’;
mysql>exit;

#创建数据库
mysql>create database cmserver default charset utf8 collate utf8_general_ci;
mysql>grant all on cmserver.* to 'cmserveruser'@'%' identified by 'password';
mysql>create database metastore Default charset utf8  collate utf8_general_ci;
mysql>grant all on metastore.* to 'hiveuser'@'%' identified by 'password';
mysql>create database amon default charset utf8 collate utf8_general_ci;
mysql>grant all on amon.* to 'amonuser'@'%' identified by 'password';
mysql>create database rman default charset utf8 collate utf8_general_ci;
mysql>grant all on rman.* to 'rmanuser'@'%' identified by 'password';
mysql>create database oozie default charset utf8 collate utf8_general_ci;
mysql>grant all on oozie.* to 'oozieuser'@'%' identified by 'password';
mysql>create database hue default charset utf8 collate utf8_general_ci;
mysql>grant all on hue.* to 'hueuser'@'%' identified by 'password';
```



## 八、安装http服务（管理节点）

```bash
#yum install httpd
#service httpd start
#systemctl enable httpd.service #设置httpd服务开机自启
```



## 九、配置yum和安装插件（所有节点）

```bash
#mkdir /etc/yum.repos.d/default
#mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/default/
#curl http://mirrors.ubtrobot.com/repos/centos.repo -o /etc/yum.repos.d/ubtrobot.repo

# cat /etc/yum.repos.d/cloudera-manager.repo 
[cloudera-manager]
name = Cloudera Manager, Version 6.2.0
baseurl = https://archive.cloudera.com/cm6/6.2.0/redhat7/yum
gpgcheck = 1

#yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server --skip-broken --nogpgcheck

#yum clean all
#yum makecache  
```







## 十、安装配置Cloudera Manager包（管理节点）

```bash
#rpm --import https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPM-GPG-KEY-cloudera
#yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
#安装完CM后/opt/ 下会出现cloudera目录
#mv /opt/parcels/* /opt/cloudera/parcel-repo # 将parcel包移动到指定位置
#在/opt/cloudera/parcel-repo执行以下命令：
#sha1sum CDH-6.2.0-1.cdh6.2.0.p0.967373-el7.parcel | awk ‘{ print $1 }’ > CDH-6.2.0-1.cdh6.2.0.p0.967373-el7.parcel.sha


#执行初始化脚本:
#/opt/cloudera/cm/schema/scm_prepare_database.sh mysql cmserver cmserveruser password
#打开server服务:
#service cloudera-scm-server start
#静候几分钟，打开http://manager ip:7180

#设置开机自启动
# systemctl enable cloudera-scm-server  
```

安装CM可参考https://www.cnblogs.com/swordfall/p/10816797.html，不再累述







参考文档：

https://www.cnblogs.com/swordfall/p/10816797.html



