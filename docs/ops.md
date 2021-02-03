# FEDB集群运维

* [宕机与恢复](#failover)
* [一键恢复](#recoverdata)
* [扩容](#expansion)
* [升级](#upgrade)

### 宕机与恢复 {#failover}

FEDB高可用可配置为自动模式和手动模式. 在自动模式下如果节点宕机和恢复时系统会自动做故障转移和数据恢复, 否则需要用手动处理  

通过修改auto\_failover配置可以切换模式, 默认是开启自动模式。通过以下方式可以获取配置状态和修改配置

```bash
$ ./bin/fedb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/fedb_cluster --role=ns_client
> confget
  key                 value
-----------------------------
  auto_failover       true
> confset auto_failover false
set auto_failover ok
>confget
  key                 value
-----------------------------
  auto_failover       false
```

**auto_failover开启的话如果一个节点下线了, showtable的is\_alive状态就会变成no**

```
$ ./bin/fedb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/fedb_cluster --role=ns_client
> showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  leader    0min       no        kNoCompress    0        0           0.000
  flow    4   0    172.27.128.33:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       no        kNoCompress    0        0           0.000
```

在手动模式下节点下线和恢复时需要手动操作下。有两个命令offlineendpoint和recoverendpoint

命令格式: offlineendpoint endpoint

endpoint是发生故障节点的endpoint。该命令会对该节点下所有分片执行如下操作:

* 如果是主, 执行重新选主
* 如果是从, 找到主节点然后从主节点中删除当前endpoint副本

```bash
$ ./bin/fedb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/fedb_cluster --role=ns_client
> offlineendpoint 172.27.128.32:8541
offline endpoint ok
>showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  leader    0min       no        kNoCompress    0        0           0.000
  flow    4   0    172.27.128.33:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       no        kNoCompress    0        0           0.000
```

执行完offlineendpoint后每一个分片都会分配新的leader。如果某个分片执行失败可以单独对这个分片执行changleader，命令格式为：changeleader table\_name pid

如果节点已经恢复，就可以执行recoverendpoint来恢复数据

命令格式: recoverendpoint endpoint

endpoint是状态已经变为healthy节点的endpoint

```
$ ./bin/fedb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/fedb_cluster --role=ns_client
> showtablet
  endpoint            state           age
-------------------------------------------
  172.27.128.31:8541  kTabletHealthy  3h
  172.27.128.32:8541  kTabletHealthy  7m
  172.27.128.33:8541  kTabletHealthy  3h
> recoverendpoint 172.27.128.32:8541
recover endpoint ok
> showopstatus
  op_id  op_type              name  pid  status  start_time      execute_time  end_time        cur_task
-----------------------------------------------------------------------------------------------------------
  54     kUpdateTableAliveOP  flow  0    kDone   20180824195838  2s            20180824195840  -
  55     kChangeLeaderOP      flow  0    kDone   20180824200135  0s            20180824200135  -
  56     kOfflineReplicaOP    flow  1    kDone   20180824200135  1s            20180824200136  -
  57     kUpdateTableAliveOP  flow  0    kDone   20180824200212  0s            20180824200212  -
  58     kRecoverTableOP      flow  0    kDone   20180824200623  1s            20180824200624  -
  59     kRecoverTableOP      flow  1    kDone   20180824200623  1s            20180824200624  -
  60     kReAddReplicaOP      flow  0    kDoing  20180824200624  4s            -               kLoadTable
  61     kReAddReplicaOP      flow  1    kDoing  20180824200624  4s            -               kLoadTable
```

执行showopstatus查看任务运行进度，如果status是doing状态说明任务还没有运行完

```
$ ./bin/rtidb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/rtidb_cluster --role=ns_client
> showopstatus
  op_id  op_type              name  pid  status  start_time      execute_time  end_time        cur_task
---------------------------------------------------------------------------------------------------------
  54     kUpdateTableAliveOP  flow  0    kDone   20180824195838  2s            20180824195840  -
  55     kChangeLeaderOP      flow  0    kDone   20180824200135  0s            20180824200135  -
  56     kOfflineReplicaOP    flow  1    kDone   20180824200135  1s            20180824200136  -
  57     kUpdateTableAliveOP  flow  0    kDone   20180824200212  0s            20180824200212  -
  58     kRecoverTableOP      flow  0    kDone   20180824200623  1s            20180824200624  -
  59     kRecoverTableOP      flow  1    kDone   20180824200623  1s            20180824200624  -
  60     kReAddReplicaOP      flow  0    kDone   20180824200624  9s            20180824200633  -
  61     kReAddReplicaOP      flow  1    kDone   20180824200624  9s            20180824200633  -
> showtable
>showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.33:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       yes       kNoCompress    0        0           0.000
```

showtable如果都变成yes表示已经恢复成功。如果某些分片恢复失败可以单独执行recovertable，命令格式为：recovertable table\_name pid endpoint

**注：执行recoverendpoint前必须执行过一次offlineendpoint**

### 一键恢复 {#recoverdata}
如果集群是同时下线后重启的，autofailover无法自动恢复数据，需要执行一键恢复脚本。
####执行步骤：
1、修改ritdb包中tools目录中的env.sh。
配置文件里面有3个参数

* rtidb_bin_path 指定rtidb bin路径
* zk_cluster 指定zk cluster的地址
* zk_root_path 指定zk root path的地址  
  
2、执行./recoverdata.sh。
3、登陆ns_client，showopstatus查看相关op的执行进度。

### 扩容 {#expansion}

随着业务的发展, 当前集群的拓扑不能满足要求就需要动态的扩容。扩容就是把一部分分片从现有tablet节点迁移到新的tablet节点上从而减小内存占用

##### 1 启动一个新的tablet节点

启动方式参见前面章节部署RTIDB部分

启动后查看新增节点是否加入集群。如果执行showtablet命令列出了新节点endpoint说明已经加入到集群中

```bash
$ ./bin/rtidb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/rtidb_cluster --role=ns_client
> showtablet
  endpoint            state           age
-------------------------------------------
  172.27.128.31:8541  kTabletHealthy  15d
  172.27.128.32:8541  kTabletHealthy  15d
  172.27.128.33:8541  kTabletHealthy  15d
  172.27.128.37:8541  kTabletHealthy  1min
```

##### 2 迁移副本

副本迁移用到的命令是migrate。命令格式: migrate src\_endpoint table\_name partition des\_endpoint, 详细用法参考命令使用部分  

**迁移的时候只能迁移从分片, 不能迁移主分片**

```bash
$ ./bin/rtidb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/rtidb_cluster --role=ns_client
> showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.33:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       yes       kNoCompress    0        0           0.000
> migrate 172.27.128.33:8541 flow 0 172.27.128.37:8541
> showopstatus flow
  op_id  op_type     name  pid  status  start_time      execute_time  end_time        cur_task
------------------------------------------------------------------------------------------------
  51     kMigrateOP  flow  0    kDone   20180824163316  12s           20180824163328  -
> showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.37:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       yes       kNoCompress    0        0           0.000
```

### 升级 {#upgrade}

#### 1 升级nameserver

* 执行sh bin/start\_ns.sh stop (如果启动时版本小于1.3.11, 就直接kill进程)
* 备份旧版本bin和conf目录
* 下载新版本bin和conf
* 对比配置文件diff并修改必要的配置，如endpoint、zk\_cluster等
* 执行sh bin/start\_ns.sh start 启动nameserver
* 对剩余nameserver重复以上步骤

#### 2 升级tablet

* 执行sh bin/start.sh stop (如果启动时版本小于1.3.11, 就直接kill进程)
* 备份旧版本bin和conf目录
* 下载新版本bin和conf
* 对比配置文件diff并修改必要的配置，如endpoint、zk\_cluster等
* 执行sh bin/start.sh start启动tablet
* 如果auto\_failover关闭时得连上ns client执行如下操作恢复数据。其中**命令后面的endpoint为重启节点的endpoint**
  * offlineendpoint endpoint 
  * recoverendpoint endpoint

```
$ ./bin/rtidb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/rtidb_cluster --role=ns_client
> offlineendpoint 172.27.128.32:8541
offline endpoint ok
> recoverendpoint 172.27.128.32:8541
recover endpoint ok
```

##### 升级结果确认
* showopstatus命令查看所有操作是否为kDone, 如果有kFailed的任务查看日志排查原因
* showtable查看所有分片状态是否为yes

一个tablet节点升级完成后，对其他tablet重复上述步骤。\(**必须等到数据同步完才能升级下一个节点**\)

所有节点升级完成后恢复写操作, 执行showtable命令查看主从offset是否增加

#### 3 升级java client

* 更新pom文件中java client版本号
* 更新依赖包

