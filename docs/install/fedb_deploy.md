# 部署FEDB

* [部署通用集群(通用版本)](#cluster)
* [部署集群(fpga版本)](#fpga-cluster)
* [部署rdma集群(rdma版本)](#rdma-cluster)

## 部署集群(通用版本) {#cluster}

FEDB集群由nameserver和tablet两种组件组成。tablet存储表的数据，nameserver负责管理tablet以及failover等。部署高可用的集群至少部署两个nameserver和三个tablet。

### 部署nameserver

#### 1 下载RTIDB部署包

```bash
$ wget http://pkg.4paradigm.com/fedb/fedb-2.0.0.0.tar.gz
$ tar -zxvf fedb-2.0.0.0.tar.gz
$ mv fedb-2.0.0.0 fedb-ns-2.0.0.0
$ cd fedb-ns-2.0.0.0
```

#### 2 修改配置文件conf/nameserver.flags

* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址. ip为zk所在机器的ip, port为zk配置文件中clientPort配置的端口号. 如果zk是集群模式用逗号分割, 格式为ip1:port1,ip2:port2,ip3:port3
* 如果和其他FEDB共用zk需要修改zk\_root\_path

```
--endpoint=172.27.128.31:6527
--role=nameserver
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/fedb_cluster
```

**注1: endpoint不能用0.0.0.0和127.0.0.1**  

#### 3 启动服务

```
$ sh bin/start_ns.sh start
```

**注1: 服务启动后会在bin目录下产生ns.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**

### 部署tablet

#### 1 下载FEDB部署包


```bash
$ wget http://pkg.4paradigm.com/fedb/fedb-2.0.0.0.tar.gz
$ tar -zxvf fedb-2.0.0.0.tar.gz
$ mv fedb-2.0.0.0 fedb-tablet-2.0.0.0
$ cd fedb-tablet-2.0.0.0
```


#### 2 修改配置文件conf/tablet.flags

* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址
* 如果和其他FEDB共用zk需要修改zk\_root\_path

```
--endpoint=172.27.128.33:9527
--role=tablet

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/fedb_cluster
```

**注1: endpoint不能用0.0.0.0和127.0.0.1**  
**注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到**  
**注3: zk\_cluster和zk\_root\_path配置和nameserver的保持一致**  
**注4: 如果修改了db的路径, 同时也需要将recycle修改为db相同目录下**
**注5: 如果要创建ssd/hdd表, 将ssd\_root\_path hdd\_root\_path recycle\_ssd\_bin\_root\_path和recycle\_hdd\_bin\_root\_path配置到对应的目录下. 其中每一项可以配置多个盘用英文逗号分割**  

#### 3 启动服务

```bash
$ sh bin/start.sh start
```

**注1: 服务启动后会在bin目录下产生tablet.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**


## 部署集群(fpga版本) {#fpga-cluster}

fpga版本和通用版本的部署步骤基本相同，主要区别如下
- 要求部署tablet的机器上有fpga环境
- 使用的安装包不同, 需要下载fpga版本的安装包
- 新增了tablet的配置文件配置项


#### 修改配置文件conf/tablet.flags

除了通用版本中的修改外，还需要修改有关fpga压缩相关的配置。 如果在没有fpga环境的机器上，下面两个配置不是off的话，tablet启动失败  
    
> - pz采用fpga压缩
> - zlib和snappy采用cpu压缩，在没有fpga的环境中也可以使用  
  
```
-- snapshot_compression=off // 配置snapshot压缩算法, 此选项需要机器上安装好 fpga 和相应驱动, 默认为 off 即关闭，选项分别为 off, pz, zlib, snappy
-- file\_compression=off // 配置压缩算法, 此选项需要机器上安装好 fpga 和相应驱动, 默认为 off 即关闭，选项分别为 off, pz, lz4, zlib
```

## 部署集群(rdma版本) {#rdma-cluster}

rdma版本和通用版本的部署步骤基本相同，主要区别如下
- 要求部署tablet的机器上有rdma环境
- 使用的安装包不同, 需要下载rdma版本的安装包
- 新增了tablet的配置文件配置项


#### 修改配置文件conf/tablet.flags

除了通用版本中的修改外，还需要修改有关rdma的配置。
```
-- use_rdma=true
```
