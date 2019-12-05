[TOC]



## 一、概念说明

ELK是三款开源软件的缩写，即：ElasticSearch + Logstash + Kibana。这三个工具组合形成了一套实用、易用的监控架构，可抓取系统日志、apache日志、nginx日志、mysql日志等多种日志类型，目前很多公司用它来搭建可视化的集中式日志分析平台。

- ElasticSearch：是一个分布式的RESTful风格的搜索和数据分析引擎，同时还提供了集中存储功能，它主要负责将logstash抓取来的日志数据进行检索、查询、分析等。

- Logstash：日志处理工具，负责日志收集、转换、解析等，并将解析后的日志推送给ElasticSearch进行检索。

- Kibana：Web前端，可以将ElasticSearch检索后的日志转化为各种图表，为用户提供数据可视化支持。

- Filebeat：一个轻量级开源日志文件数据搜集器，基于 Logstash-Forwarder 源代码开发，是对它的替代。在需要采集日志数据的 server 上安装 Filebeat，并指定日志目录或日志文件后，Filebeat 就能读取数据，迅速发送到 Logstash 进行解析，亦或直接发送到 Elasticsearch 进行集中式存储和分析

- Winlogbeat：轻量型windows事件日志采集器，负责采集wondows的事件日志，并将采集来的日志推送给logstash进行处理。



示意图：

![](http://ww1.sinaimg.cn/large/007Xg1efgy1g68lu023dnj30w00i03yu.jpg)



## 二、环境列表

OS:Centos7

均按单机架构来部署

所需版本均按当前最新版本6.8.2来部署

ELK三大组件版本要保持一致

redis：3.2.12 redis需要加密码



| 角色                | IP              | 功能                                |
| ------------------- | --------------- | ----------------------------------- |
| Elasticsearch+redis | 192.168.255.101 | ES:日志存放  redis：日志转发        |
| Logstash            | 192.168.255.102 | 日志收集                            |
| Kibana              | 192.168.255.103 | 展示                                |
| Filebeat+nginx      | 192.168.255.104 | 对nginx日志和其他类型的日志进行收集 |



## 三、部署Elasticsearch

3.1、安装java

```bash
# yum -y install java-openjdk-devel java-openjdk
```



3.2、配置yum文件

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```



3.3、导入

```bash
 rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
yum clean all
yum makecache
```



3.4 、安装

```bash
yum -y install elasticsearch
```

确认版本

```bas
[root@elasticsearch opt]# rpm -qi elasticsearch
Name        : elasticsearch
Epoch       : 0
Version     : 6.8.2
Release     : 1
Architecture: noarch
Install Date: Mon 12 Aug 2019 12:37:16 PM CST
Group       : Application/Internet
Size        : 238207418
License     : Elastic License
Signature   : RSA/SHA512, Thu 25 Jul 2019 12:39:01 AM CST, Key ID d27d666cd88e42b4
Source RPM  : elasticsearch-6.8.2-1-src.rpm
Build Date  : Wed 24 Jul 2019 11:33:35 PM CST
Build Host  : packer-virtualbox-iso-1559162487
Relocations : /usr 
Packager    : Elasticsearch
Vendor      : Elasticsearch
URL         : https://www.elastic.co/
Summary     : Elasticsearch is a distributed RESTful search engine built for the cloud. Reference documentation can be found at https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html and the 'Elasticsearch: The Definitive Guide' book can be found at https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html
Description :
Elasticsearch subproject :distribution:packages
```



3.5、配置

内存限制 //根据实际分配

```bash
[root@elasticsearch opt]# vim /etc/elasticsearch/jvm.options 
-Xms1g
-Xmx1g
```



```bash
[root@elasticsearch opt]# grep -v "^#" /etc/elasticsearch/elasticsearch.yml 
node.name: ELK-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0 
http.port: 9200
discovery.zen.ping.unicast.hosts: ["192.168.255.101"]
```



3.6、启动

```bash
[root@elasticsearch opt]# systemctl start elasticsearch
[root@elasticsearch opt]# systemctl enable elasticsearch
```



3.7、确认

查看端口使用情况

对外服务的HTTP端口是9200，节点间交互的TCP端口是9300

```bash
# ss -tlnp|grep -E '9200|9300'
LISTEN     0      128       ::ffff:127.0.0.1:9200                    :::*                   users:(("java",pid=4161,fd=210))
LISTEN     0      128        ::1:9200                    :::*                   users:(("java",pid=4161,fd=209))
LISTEN     0      128       ::ffff:127.0.0.1:9300                    :::*                   users:(("java",pid=4161,fd=196))
LISTEN     0      128        ::1:9300                    :::*                   users:(("java",pid=4161,fd=195))
```



3.8、测试

```bash
[root@elasticsearch opt]# curl -X GET http://localhost:9200
{
  "name" : "XyLpu15",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Nczrk1WGQy2XO92B0O466A",
  "version" : {
    "number" : "6.8.2",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "b506955",
    "build_date" : "2019-07-24T15:24:41.545295Z",
    "build_snapshot" : false,
    "lucene_version" : "7.7.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```



3.9、redis安装

redis可以单独部署，这里为了节约服务器资源，和ES放在一起

需要设置密码，在此不再累述





## 四、安装Logstash

4.1、配置yum

```bash
 #rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
 
 #cat <<EOF |tee /etc/yum.repos.d/logstash.repo
[logstash-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
 
#yum makecache && yum install logstash -y
```



4.2、配置logstash

```bash
[root@logstash logstash]# grep -v "^#" /etc/logstash/logstash.yml|grep -v "^$"
path.data: /var/lib/logstash
http.host: "127.0.0.1"
log.level: info
path.logs: /var/log/logstash
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://192.168.255.101:9200"]
xpack.monitoring.elasticsearch.sniffing: true
xpack.monitoring.collection.interval: 10s
xpack.monitoring.collection.pipeline.details.enabled: true



[root@logstash logstash]# vim jvm.options

-Xms1g
-Xmx1g


#创建配置文件，读取redis转发的filebeat收集的日志
[root@logstash conf.d]# grep -v "^#" /etc/logstash/conf.d/logstash.conf
input {
        redis {
        host => "192.168.255.101"       #redis ip         
        port => "6379"                  #redis 端口
        password => "passw"            #redis 密码
        db => "1"                       #db id 
        data_type => "list"             #类型
        key => "client-log"             #key
}

}

output {
        if "system" in [tags]{  #根据6.4章节filebeat中配的tag判断，如是system生成system前缀索引

        elasticsearch {
        hosts => ["192.168.255.101:9200"]
        index => "system-log-%{+YYYY.MM.dd}"   #生成system前缀索引
        }
      }
        else if "nginx" in [tags]{   #根据filebeat中配的tag判断，如是nginx生成nginx前缀索引
        elasticsearch {
        hosts => ["192.168.255.101:9200"]
        index => "nginx-log-%{+YYYY.MM.dd}"
      }
    }
}

```



4.3、生成SSL证书

```bash
[root@logstash logstash]# cd /etc/pki/tls/

[root@logstash tls]# openssl req -subj '/CN=ELK_server_fqdn/' -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
Generating a 2048 bit RSA private key
........................+++
.....+++
writing new private key to 'private/logstash-forwarder.key'
-----
```



4.4、启动

```bash
#后台执行
[root@logstash logstash]# nohup /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf  &

[root@logstash logstash]# nohup /usr/share/logstash/bin/logstash -f  /etc/logstash/conf.d/logstash.conf --path.settings /etc/logstash &  
```



## 五、安装kibana

5.1、配置yum

```bash
 #rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
 
 #cat <<EOF |tee /etc/yum.repos.d/kibana.repo
[kibana.x]
name=kibana repository for x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
 
#yum makecache && yum -y install kibana
```



5.2、配置

```bash
[root@kibana ~]# cat /etc/kibana/kibana.yml|grep -v "^#"|grep -v "^$"
server.port: 5601
server.host: 192.168.255.103 
elasticsearch.url: ["http://192.168.255.101:9200"]
logging.dest: /var/log/kibana.log 
```



5.3、启动

```bash
[root@kibana ~]# systemctl start kibana
[root@kibana ~]# systemctl enable kibana
```



5.4、查看

```html
http://192.168.255.103:5601
```





5.5、安装nginx和httpd-tools

```bash
[root@kibana ~]# yum -y install nginx httpd-tools 
```



5.6、使用htpasswd创建一个名为“kibanaadmin”的管理员用户（可以使用其他名称），该用户可以访问Kibana Web界面：

```bash
[root@linuxprobe ~]# htpasswd -c /etc/nginx/htpasswd.users kibanaadmin
New password:              # 自定义
Re-type new password: 
Adding password for user kibanaadmin
```





5.7、配置nginx

```bash
[root@kibana ~]# vim /etc/nginx/conf.d/kibana.conf

server {
    listen       80;
    server_name  kibana.aniu.co;
    access_log  /var/log/nginx/kibana.aniu.co.access.log main;
    error_log   /var/log/nginx/kibana.aniu.co.access.log;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;
    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;        
    }
}
```



5.8、启动nginx

```bash
[root@kibana ~]# systemctl start nginx
[root@kibana ~]# systemctl enable nginx
```



5.9、测试



## 六、设置日志客户端

客户端可以使用Filebeat收集日志

6.1、安装filebeat

```bash
 #rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
 
 #cat <<EOF |tee /etc/yum.repos.d/filebeat.repo
[filebeat.x]
name=filebeat repository for x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
 
#yum makecache && yum -y install filebeat
```



6.2、安装nginx

```bash
#yum -y install nginx
```



6.3、对nginx日志做json处理

```bash
# vim /etc/nginx/nginx.conf   #替换之前的log_format

    log_format access_json '{"@timestamp":"$time_iso8601",'
                 '"host":"$server_addr",'
                 '"request_method": "$request_method", '
                 '"clientip":"$remote_addr",'
                 '"size":$body_bytes_sent,'
                 '"responsetime":$request_time,'
                 '"upstreamtime":"$upstream_response_time",'
                 '"upstreamhost":"$upstream_addr",'
                 '"http_host":"$host",'
                 '"url":"$uri",'
                 '"xff":"$http_x_forwarded_for",'
                 '"referer":"$http_referer",'
                 '"agent":"$http_user_agent",'
                 '"status":"$status"}';


    access_log  /var/log/nginx/access.log  access_json;

#systemctl restart nginx

```



6.4、配置filebeat

```bash
#实现系统日志、nginx日志的收集，将其打上tag，写入到redis中

[root@ansiblemaster filebeat]# cat /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true 
  paths:
    - /var/log/nginx/access.log   #nginx日志
  tags: ["nginx"]                 #打上nginx tag

- type: log
  enabled: true
  paths:
    - /var/log/messages           #系统日志
  tags: ["system"]                #打上system tag
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 3
setup.kibana:
output.redis:                     #输出到redis中
   hosts: ["192.168.255.101"]     #redis ip 
   db: "1"                        #db id
   port: "6379"                   #端口
   password: "passw"             #redis 密码
   key: "client-log"              #redis key
   data_type: "list"              #类型

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
logging.level: info 
xpack.monitoring.enabled: false
```







## 七、整体测试

1、启动redis、ES、kibana、nginx 等 



2、启动filebeat收集日志

```bash
# filebeat -e -c  /etc/filebeat/filebeat.yml 
```



3、启动logstash

```bash
[root@logstash conf.d]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf
```



4、点击nginx主页界面，产生日志内容

​      另外对/var/log/messages系统日志新增内容，模拟系统日志的产生





5、redis接受到消息，立马转发给logstash

本身不做存储日志



6、logstash转发给ES



7、ES创建索引

分别创建nginx和system前缀索引

![](http://ww1.sinaimg.cn/large/007Xg1efgy1g68n1i2unbj31h90l60um.jpg)



8、发现索引



nginx日志：

![](http://ww1.sinaimg.cn/large/007Xg1efgy1g68n2dr7izj31h10q9n0w.jpg)



system日志：

![](http://ww1.sinaimg.cn/large/007Xg1efgy1g68qvcagvvj31gw0ogmzf.jpg)


