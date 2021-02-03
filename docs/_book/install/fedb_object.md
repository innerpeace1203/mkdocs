# 部署RTIDB

* [部署集群版](#cluster)
* [部署单机版](#single)

## 部署集群版 {#cluster}

RTIDB集群由nameserver和tablet两种组件组成。tablet存储表的数据，nameserver负责管理tablet以及failover等。部署高可用的集群至少部署两个nameserver和三个tablet。    
** 注意：如果使用serverName和自动获取本地ip的功能，则集群中的nameserver、tablet和blobserer都需要使用新方式部署（修改port和use_name) **

### 部署nameserver

#### 1 下载RTIDB部署包

```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
$ mv rtidb-cluster-1.6.3.0 rtidb-ns-1.6.3.0
$ cd rtidb-ns-1.6.3.0
```

#### 2 修改配置文件conf/nameserver.flags

* 修改endpoint(如果使用注册本地ip功能，则修改port和use_name配置)
- 修改role 为nameserver
* 修改zk\_cluster为已经启动的zk集群地址. ip为zk所在机器的ip, port为zk配置文件中clientPort配置的端口号. 如果zk是集群模式用逗号分割, 格式为ip1:port1,ip2:port2,ip3:port3
* 如果和其他RTIDB共用zk需要修改zk\_root\_path

方式一、 指定endpoint
```asciidoc
#--port=9527
--endpoint=172.27.128.31:6527
--role=nameserver
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
#--use_name=true
#--data_dir=./data
```    
方式二、 修改port, use_name，注释endpoint（程序获取本地ip), 配置data_dir目录(默认是./data)，在data_dir目录创建name.txt并写入全局唯一的server_name(如果没有手动创建该目录和文件, 程序会自动创建目录和文件并生成唯一标识写入)
```asciidoc
--port=9527
#--endpoint=172.27.128.31:6527
--role=nameserver
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
--use_name=true
--data_dir=./data
```

**注1: endpoint不能用0.0.0.0和127.0.0.1, server_name 不能带有/**  

#### 3 启动服务

```
$ sh bin/start_ns.sh start
```

**注1: 服务启动后会在bin目录下产生ns.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**

### 部署tablet

#### 1 下载RTIDB部署包

```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
$ mv rtidb-cluster-1.6.3.0 rtidb-tablet-1.6.3.0
$ cd rtidb-tablet-1.6.3.0
```

*如果部署 fpga 版本的 rtidb, 需要将 bin 目录下的 rtidb_fpga rename 为 rtidb*

#### 2 修改配置文件conf/tablet.flags

* 修改endpoint(如果使用注册本地ip功能，则修改port和use_name配置)
- 修改role 为tablet
* 修改zk\_cluster为已经启动的zk集群地址
* 如果和其他RTIDB共用zk需要修改zk\_root\_path
    

方式一、 指定endpoint
```asciidoc
#--port=9527
--endpoint=172.27.128.33:9527
--role=tablet

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
#--use_name=true
#--data_dir=./data
```
方式二、 修改port, use_name，注释endpoint（程序获取本地ip), 配置data_dir目录(默认是./data)，在data_dir目录创建name.txt并写入全局唯一的server_name(如果没有手动创建该目录和文件, 程序会自动创建目录和文件并生成唯一标识写入)
```asciidoc
--port=9527
#--endpoint=172.27.128.33:9527
--role=tablet

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
--use_name=true
--data_dir=./data
```

**注1: endpoint不能用0.0.0.0和127.0.0.1, server_name 不能带有/**  
**注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到**  
**注3: zk\_cluster和zk\_root\_path配置和nameserver的保持一致**  
**注4: 如果修改了db的路径, 同时也需要将recycle修改为db相同目录下**
**注5: 如果要创建ssd/hdd表, 将ssd\_root\_path hdd\_root\_path recycle\_ssd\_bin\_root\_path和recycle\_hdd\_bin\_root\_path配置到对应的目录下. 其中每一项可以配置多个盘用英文逗号分割**  

#### 3 启动服务

```bash
$ sh bin/start.sh start
```

**注1: 服务启动后会在bin目录下产生tablet.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**

>>>>>>> origin/master:install/rtidb.md
### 部署blob server

#### 1 下载RTIDB部署包

```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
$ mv rtidb-cluster-1.6.3.0 rtidb-tablet-1.6.3.0
$ cd rtidb-tablet-1.6.3.0
```

#### 2 修改配置文件conf/tablet.flags

* 修改endpoint(如果使用注册本地ip功能，则修改port和use_name配置)
- 修改zk\_cluster为已经启动的zk集群地址
- 如果和其他RTIDB共用zk需要修改zk\_root\_path
- 修改role 为blob
- 修改hdd_root_path 和 recycle_hdd_root_path

方式一、 指定endpoint
```asciidoc
#--port=9528
--endpoint=172.27.128.33:9528
--role=blob

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
#--use_name=true
#--data_dir=./data
--hdd_root_path=./db
--recycle_hdd_root_path=./recycle_db
```
方式二、 修改port, use_name，注释endpoint（程序获取本地ip), 配置data_dir目录(默认是./data)，在data_dir目录创建name.txt并写入全局唯一的server_name(如果没有手动创建该目录和文件, 程序会自动创建目录和文件并生成唯一标识写入)

```
--port=9528
#--endpoint=172.27.128.33:9528
--role=blob

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
--use_name=true
--data_dir=./data
--hdd_root_path=./db
--recycle_hdd_root_path=./recycle_db
```

**注1: endpoint不能用0.0.0.0和127.0.0.1, server_name 不能带有/**  
**注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到**  
**注3: zk\_cluster和zk\_root\_path配置和nameserver的保持一致**  
**注4: 如果修改了hdd_root_path的路径, 同时也需要将recycle_hdd_root_path修改为db相同目录下**
**注5: hdd\_root\_path 配置多个盘用英文逗号分割**  

#### 3 启动服务

```bash
$ sh bin/start.sh start
```

**注1: 服务启动后会在bin目录下产生tablet.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**

### 部署blob proxy

#### 1 下载RTIDB部署包

```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
$ mv rtidb-cluster-1.6.3.0 rtidb-tablet-1.6.3.0
$ cd rtidb-tablet-1.6.3.0
```

#### 2 修改配置文件conf/tablet.flags

* 修改endpoint(如果使用注册本地ip功能，则修改port和use_name配置)
- 修改zk\_cluster为已经启动的zk集群地址
- 如果和其他RTIDB共用zk需要修改zk\_root\_path
- 修改role 为blob_proxy

方式一、 指定endpoint
```asciidoc
#--port=9529
--endpoint=172.27.128.33:9529
--role=blob_proxy

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
#--use_name=true
#--data_dir=./data
```
方式二、 修改port, use_name，注释endpoint（程序获取本地ip), 配置data_dir目录(默认是./data)，在data_dir目录创建name.txt并写入全局唯一的server_name(如果没有手动创建该目录和文件, 程序会自动创建目录和文件并生成唯一标识写入)
```
--port=9529
#--endpoint=172.27.128.33:9529
--role=blob_proxy

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
--use_name=true
--data_dir=./data
```


**注1: endpoint不能用0.0.0.0和127.0.0.1, server_name 不能带有/**  
**注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到**  
**注3: zk\_cluster和zk\_root\_path配置和nameserver的保持一致**  

#### 3 启动服务

```bash
$ sh bin/start.sh start
```

**注1: 服务启动后会在bin目录下产生tablet.pid文件, 里边保存启动时的进程号。如果该文件内的pid正在运行则会启动失败**

### 部署monitor(该模块是收集监控数据的, 如qps、cpu内存磁盘等)

#### 1 修改配置文件conf/monitor.conf

* 修改tablet_endpoint为当前机器部署tablet的endpoint
* 修改tablet_log_dir为当前机器部署tablet的日志目录
* 如果当前机器部署了nameserver修改ns_log_dir为当前机器nameserver的日志目录, 如果没有部署nameserver就把这项注释掉

#### 2 启动monitor
```
bash bin/start_monitor.sh start
```

## 部署单机版 {#single}

单机版可以只部署一个节点也可以部署两个节点的一主一从或者三个节点的一主两从模式。主从模式就是单独起两个或者三个tablet节点，通过建表和添加副本命令建立主从关系

#### 1 下载RTIDB部署包. 如果已经有部署包就跳过这步  

```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
$ mv rtidb-cluster-1.6.3.0 rtidb-tablet-1.6.3.0
$ cd rtidb-tablet-1.6.3.0
```

*如果部署 fpga 版本的 rtidb, 需要将 bin 目录下的 rtidb_fpga rename 为 rtidb*

#### 2 修改配置文件

修改conf/tablet.flags中的endpoint. endpoint包括服务启动的地址和端口号   

```
--endpoint=172.27.128.33:9527
```

**注1: 如果此处使用的域名, 所有使用rtidb的client所在的机器都需要配上对应的host. 不然会访问不到**  
**注2: 部署单机版时conf/tablet.flags配置中的zk\_cluster和zk\_root\_path需要注释掉**  

#### 3 启动tablet

```bash
$ sh bin/start.sh start
```

