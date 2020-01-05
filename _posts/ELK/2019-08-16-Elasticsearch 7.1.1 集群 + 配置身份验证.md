---
layout: post
title:  Elasticsearch 7.1.1 集群 + 配置身份验证
categories: ELK
description: Elasticsearch 7.1.1 集群 + 配置身份验证
keywords: ELK
---
# [Elasticsearch 7.1.1 集群 + 配置身份验证](https://www.cnblogs.com/weavepub/p/11045025.html)

## 一、安装Elasticsearch

### 1.1 环境说明

Centos7.6

Elasticsearch7.1.1

 

\#挂载数据盘

```
fdisk /dev/vdb
n,p,1,回车，回车，wq
fdisk -l
mkfs.ext4 /dev/vdb1 
echo '/dev/vdb1 /opt ext4 defaults 0 0' >>/etc/fstab
mount -a
df -h
```

 

\#时间同步

```
yum install -y ntp 
systemctl enable ntpd && systemctl start ntpd
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp yes
ntpq -p
```

 

### 1.2 操作系统调优

```
cat >> /etc/sysctl.conf <<EOF
fs.file-max=655360
vm.max_map_count = 262144
EOF
```

sysctl -p

vim /etc/security/limits.conf

```
* soft nproc 20480
* hard nproc 20480
* soft nofile 65536
* hard nofile 65536
* soft memlock unlimited
* hard memlock unlimited
```

 

vim /etc/security/limits.d/20-nproc.conf

```
* soft nproc 20480
```

 

 

### 1.3 安装JDK

yum install -y java-1.8.0-openjdk*

vim /etc/profile

```
# set java environment 
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

source /etc/profile

echo "source /etc/profile" >> /etc/bashrc

 

### 1.4 安装es

1）新建用户

groupadd elsearch
useradd elsearch -g elsearch -p elasticsearch

 

2）下载
cd /opt
wget https://img.yiyao.cc/elasticsearch-7.1.1-linux-x86_64.tar.gz
tar -zxvf elasticsearch-7.1.1-linux-x86_64.tar.gz
mv elasticsearch-7.1.1 elasticsearch
chown -R elsearch.elsearch ./elasticsearch

3）JVM调优
物理内存一半
vim /opt/elasticsearch/config/jvm.options

```
-Xms8g
-Xmx8g
```

 

4）配置es，三个节点同时作为 master 和 data

```
vim /opt/elasticsearch/config/elasticsearch.yml
```

\#节点1

```
cluster.name: wmqees
node.name: es-node1
node.master: true
node.data: true
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 172.16.2.141
http.port: 9200
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping.unicast.hosts: ["172.16.2.141:9300","172.16.2.142:9300","172.16.2.143:9300"]
cluster.initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```

cluster.initial_master_nodes参数说明：es7 引用了 [Bootstrapping a cluster](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-discovery-bootstrap-cluster.html) 后，首次启动Elasticsearch集群需要在集群中的一个或多个符合主节点的节点上显式定义初始的符合主节点的节点集。这称为*群集自举，*这仅在群集首次启动时才需要。

\#节点2

```
cluster.name: wmqees
node.name: es-node2
node.master: true
node.data: true
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 172.16.2.142
http.port: 9200
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping.unicast.hosts: ["172.16.2.141:9300","172.16.2.142:9300","172.16.2.143:9300"]
cluster.initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```

\#节点3

```
cluster.name: wmqees
node.name: es-node3
node.master: true
node.data: true
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 172.16.2.143
http.port: 9200
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping.unicast.hosts: ["172.16.2.141:9300","172.16.2.142:9300","172.16.2.143:9300"]
cluster.initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```

 

5）启动

su - elsearch -c "/opt/elasticsearch/bin/elasticsearch -d"

6）验证
 curl "172.16.2.143:9200/_xpack"

```
{"build":{"hash":"7a013de","date":"2019-05-23T14:05:50.009976Z"},"license":{"uid":"344f983f-9d20-4476-851a-4172fd669f12","type":"basic","mode":"basic","status":"active"},"features":{"ccr":{"description":"Cross Cluster Replication","available":false,"enabled":true},"graph":{"description":"Graph Data Exploration for the Elastic Stack","available":false,"enabled":true},"ilm":{"description":"Index lifecycle management for the Elastic Stack","available":true,"enabled":true},"logstash":{"description":"Logstash management component for X-Pack","available":false,"enabled":true},"ml":{"description":"Machine Learning for the Elastic Stack","available":false,"enabled":true,"native_code_info":{"version":"7.1.1","build_hash":"fd619a36eb77df"}},"monitoring":{"description":"Monitoring for the Elastic Stack","available":true,"enabled":true},"rollup":{"description":"Time series pre-aggregation and rollup","available":true,"enabled":true},"security":{"description":"Security for the Elastic Stack","available":true,"enabled":false},"sql":{"description":"SQL access to Elasticsearch","available":true,"enabled":true},"watcher":{"description":"Alerting, Notification and Automation for the Elastic Stack","available":false,"enabled":true}},"tagline":"You know, for X"}
```

说明：显示 license 不为空则安装成功。es7版本默认已经包含xpack认证，无需注册。

 

### 1.5 开机自启

vim /etc/init.d/elasticsearch

```
#!/bin/sh
#chkconfig: 2345 80 05
#description: elasticsearch
#processname: elasticsearch-7.1.1

export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64
export JAVA_BIN=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/bin
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export ES_HOME=/opt/elasticsearch

case $1 in
    start)
        su elsearch<<!
        cd $ES_HOME
        ./bin/elasticsearch -d -p pid
        exit
!
        echo "elasticsearch is started"
        ;;
    stop)
        pid=`cat $ES_HOME/pid`
        kill -9 $pid
        echo "elasticsearch is stopped"
        ;;
    restart)
        pid=`cat $ES_HOME/pid`
        kill -9 $pid
        echo "elasticsearch is stopped"
        sleep 1
        su elsearch<<!
        cd $ES_HOME
        ./bin/elasticsearch -d -p pid
        exit
!
        echo "elasticsearch is started"
        ;;
    *)
        echo "start|stop|restart"
        ;;  
esac
exit 0
```

说明：需指定JDK环境，要不然会默认使用es自带的JDK，自带的版本太新，去除了GC。

 

\#添加到开机启动任务

chmod +x /etc/init.d/elasticsearch
chkconfig --add elasticsearch

\#启动

service elasticsearch start

 

## 二、配置 TLS 和身份验证

2.1 创建fu

在一个master上执行即可

```
cd /opt/elasticsearch
./bin/elasticsearch-certutil ca
两次回车
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12 
三次回车
```

赋予权限

```
mkdir config/certs
mv elastic-*.p12 config/certs/
chown -R elsearch:elsearch config/certs/
```

**再把证书文件 elastic-certificates.p12 复制到其他master节点并赋予权限。** 

 

### 2.2 修改配置

\#所有主机配置文件添加ssl

```
cat >> config/elasticsearch.yml <<EOF
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
EOF
```

重启 elasticsearch 

service elasticsearch restart

 

### 2.3 生成客户端证书

```
cd /opt/elasticsearch
bin/elasticsearch-certutil cert --ca \
config/certs/elastic-stack-ca.p12 \
-name "CN=esuser,OU=dev,DC=weqhealth,DC=com"

回车
client.p12
回车
```

拆分证书

```
mv client.p12 config/certs/
cd config/certs/

openssl pkcs12 -in client.p12 -nocerts -nodes > client-key.pem
openssl pkcs12 -in client.p12 -clcerts -nokeys  > client.crt
openssl pkcs12 -in client.p12 -cacerts -nokeys -chain > client-ca.crt

chown elsearch:elsearch client*
```

 

### 2.4 设置默认密码

```
bin/elasticsearch-setup-passwords interactive
```

y，分别设置 elastic、apm_system、kibana、logstash_system、beats_system、remote_monitoring_user账号的密码。

 

### 2.5 配置kibana

修改 kibana.yml 文件

```
elasticsearch.username: "kibana"
elasticsearch.password: "elasticxxxxxxx"
```

然后用超级管理员账号 elastic 登入到 kibana。在kibana中设置角色和账号，也可以修改账号密码。

 

### 2.6 验证集群状态

因为开启了xpack验证，需要指定账号密码

```
curl --user elastic:elasticxxxxxx -XGET '172.16.2.143:9200/_cat/health?v&pretty'

epoch      timestamp cluster   status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1564645243 07:40:43  wmqees green           3         3     14   7    0    0        0             0                  -                100.0%
```

 

参考：<https://www.elastic.co/cn/blog/getting-started-with-elasticsearch-security>