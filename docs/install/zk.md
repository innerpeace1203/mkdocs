# 部署zookeeper

部署集群版fedb需要部署zookeeper集群. 如果有已经部署好的zookeeper集群可以复用现有的zookeeper集群  

部署三个节点或者三个以上节点的zookeeper集群才能保证高可用, 建议在生产环境中至少部署三个节点  

**注:部署zk需要java, 执行java --version查看是否部了java

* [部署单机版](#single)
* [部署集群版](#cluster)

## 部署单机版zk {#single}

#### 下载zookeeper安装包

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
$ tar -zxvf zookeeper-3.4.14.tar.gz
$ cd zookeeper-3.4.14
$ cp conf/zoo_sample.cfg conf/zoo.cfg
```

#### 修改配置文件

打开文件conf/zoo.cfg 修改dataDir和clientPort

```
dataDir=./data
clientPort=6181
```

#### 启动zookeeper

```bash
$ sh bin/zkServer.sh start
```

## 部署集群版zk {#cluster}

#### 下载zookeeper安装包

```bash
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
$ tar -zxvf zookeeper-3.4.14.tar.gz
$ cd zookeeper-3.4.14
$ cp conf/zoo_sample.cfg conf/zoo.cfg
```

#### 修改配置文件 conf/zoo.cfg

* 修改dataDir和clientPort。dataDir是zk保存数据的目录，clientPort是zk对外提供的端口号
* 添加server配置 格式为server.id=ip:port1:port2, 假如部署三个节点就在最后添加三行

```
dataDir=./data
clientPort=6181
server.1=172.27.128.31:2881:3881
server.2=172.27.128.32:2882:3882
server.3=172.27.128.33:2883:3883
```

**注: port1和port2不能和对应ip机器已占用端口冲突**

参考的配置文件如下

```asciidoc
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=./data
# the port at which the clients will connect
clientPort=6181
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=60
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=172.27.128.31:2881:3881
server.2=172.27.128.32:2882:3882
server.3=172.27.128.33:2883:3883
```

#### 添加myid文件

分别在各个server的dataDir目录下创建myid文件并设置id，如果dataDir配置的是"dataDir=./data"，执行如下命令：

在server1机器上执行下面命令

```bash
$ echo 1 > ./data/myid
```

在server2机器上执行

```bash
$ echo 2 > ./data/myid
```

在server3机器上执行

```bash
$ echo 3 > ./data/myid
```

#### 启动zk

在三台机器上分别执行启动命令

```
$ sh bin/zkServer.sh start
```

#### 测试zk集群运行状态

用zkCli连上集群执行 "ls / " 能输出zookeeper说明集群运行正常

```
$ ./bin/zkCli.sh -server 172.27.128.31:6181,172.27.128.32:6181,172.27.128.33:6181
[zk: 172.27.128.31:6181,172.27.128.32:6181,172.27.128.33:6181(CONNECTED) 0] ls /
[zookeeper]
```


