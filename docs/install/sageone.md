# 一体机部署RTIDB

## 准备机器环境
#### 1 修改系统参数

[参考这里](./env.md)
#### 2 创建部署目录

* df -h 找到8T多的HDD盘挂载目录, 假如是/mnt/disk0
* cd到对应目录
* mkdir rtidb 创建rtidb部署目录
* cd rtidb

#### 3 下载部署包
```bash
$ wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
$ tar -zxvf rtidb-cluster-1.6.3.0.tar.gz
```

## 部署zookeeper节点
如果需要在一体机上部署zk节点，那么部署在另外一块HDD盘上  
部署方式[参考这里](./zk.md)

## 部署nameserver
#### 1 拷贝部署目录
```bash
$ cp -r rtidb-cluster-1.6.3.0 rtidb-ns-1.6.3.0
$ cd rtidb-ns-1.6.3.0
```
#### 2 修改配置文件 conf/nameserver.flags

* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址. ip为zk所在机器的ip, port为zk配置文件中clientPort配置的端口号. 如果zk是集群模式用逗号分割, 格式为ip1:port1,ip2:port2,ip3:port3
* 如果和其他RTIDB共用zk需要修改zk\_root\_path

```asciidoc
--endpoint=172.27.128.31:6527
--role=nameserver
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
```

**注1: endpoint不能用0.0.0.0和127.0.0.1** 
#### 3 启动服务  

```
$ sh bin/start_ns.sh start
```

## 部署tablet
#### 1 拷贝部署目录

```bash
$ cp -r rtidb-cluster-1.6.3.0 rtidb-tablet-1.6.3.0
$ cd rtidb-tablet-1.6.3.0
```
#### 2 修改配置文件 conf/tablet.flag

* 修改endpoint
* 修改zk\_cluster为已经启动的zk集群地址
* 如果和其他RTIDB共用zk需要修改zk\_root\_path
* 修改ssd\_root\_path
     * df -h 查看ssd的挂载目录(关键字nvme), 如挂载目录为 /mnt/ssd0和/mnt/ssd1
     * 在对应ssd目录创建rtidb/db目录 
     * 配置ssd\_root\_path, 多个目录用逗号分割
```
     文件系统                 容量  已用  可用 已用% 挂载点
     /dev/mapper/zstack-root  887G  6.3G  881G    1% /
     devtmpfs                 496G     0  496G    0% /dev
     tmpfs                    496G     0  496G    0% /dev/shm
     tmpfs                    496G   35M  496G    1% /run
     tmpfs                    496G     0  496G    0% /sys/fs/cgroup
     /dev/sda1               1016M  174M  842M   18% /boot
     /dev/sdb1                8.7T  573G  7.7T    7% /mnt/disk0
     tmpfs                    100G     0  100G    0% /run/user/0
     /dev/nvme1n1p1           1.8T   77M  1.7T    1% /mnt/ssd1
     /dev/nvme0n1             1.8T  542G  1.2T   32% /mnt/ssd0
```

* 修改recycle\_ssd\_bin\_root\_path
    - 在对应ssd目录创建rtidb/recycle目录
    - 配置recycle\_ssd\_bin\_root\_path, 多个目录用逗号分割

```asciidoc
--endpoint=172.27.128.33:9527
--role=tablet

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/rtidb_cluster
--ssd_root_path=/mnt/ssd0/rtidb/db,/mnt/ssd1/rtidb/db
--recycle_ssd_root_path=/mnt/ssd0/rtidb/recycle,/mnt/ssd1/rtidb/recycle
```

**注1: endpoint不能用0.0.0.0和127.0.0.1**  
**注2: 如果此处使用的域名, 所有使用rtidb的client所在的机器都得配上对应的host. 不然会访问不到**  
**注3: zk\_cluster和zk\_root\_path配置和nameserver的保持一致**  
**注4: 如果修改了db的路径, 同时也需要将recycle修改为db相同目录下**
**注5: 将ssd\_root\_path和recycle\_ssd\_bin\_root\_path要对应, 既以逗号分隔的每个项都是对应的同一个盘, 如第一项都是mnt/ssd0目录**

#### 3 启动服务

```bash
$ sh bin/start.sh start
```
