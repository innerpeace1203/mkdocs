# 简单部署fedb

## 获取本机ip

```
hostname -i
```
假设ip为ipa

## 安装jdk

安装jdk，并且将其加入到环境变量中

## 部署zk


```
wget https://downloads.apache.org/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
tar -zxvf zookeeper-3.4.14.tar.gz && cd zookeeper-3.4.14 
mv conf/zoo_sample.conf conf/zoo.conf
./bin/zkServer.sh start
```
说明此时的zk的地址为`ipa:2181`

## 部署fedb

### 下载压缩

```
wget https://storage.4paradigm.com/api/public/dl/MhpxvBkl/fedb-cluster-2020-08-11-b5b57be.tar.gz
tar -zxvf fedb-cluster-2020-08-11-b5b57be.tar.gz
cd fedb-cluster-2020-08-11-b5b57be
```

### 修改conf/tablet.flags配置


* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址. ip为zk所在机器的ip, port为zk配置文件中clientPort配置的端口号. 如果zk是集群模式用逗号分割, 格式为ip1:port1,ip2:port2,ip3:port3
* 如果和其他FEDB共用zk需要修改zk\_root\_path

```
--endpoint=ipa:9527
--role=tablet
--zk_cluster=ipa:2181
--zk_root_path=/fedb_cluster
```

### 修改conf/nameserver.flags

* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址
* 如果和其他FEDB共用zk需要修改zk\_root\_path

```
--endpoint=ipa:6527
--role=nameserver
--zk_cluster=ipa:2181
--zk_root_path=/fedb_cluster
```

注1: endpoint不能用0.0.0.0和127.0.0.1**  
注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到

### 启动

```
./bin/start_ns.sh start
./bin/start.sh start
```

