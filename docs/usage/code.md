###  返回码说明

|code|message                             |说明                   |
|----|------------------------------------|-----------------------|
|0   |ok                                  |成功                   |
|100 |table is not exist                  |表不存在               |
|101 |table already exists                |表已经存在             |
|102 |table is leader                     |表是leader             |
|103 |table is follower                   |表是follower           |
|104 |table is loading                    |表正在loading          |
|105 |table status is not kNormal         |表不是kNormal状态      |
|106 |table status is kMakingSnapshot     |表状态是kMakingSnapshot|
|107 |table status is not kSnapshotPaused |表状态不是kSnapshotPaused|
|108 |idx name not found                  |找不到索引列列名       |
|109 |key not found                       |找不到key              |
|110 |replicator is not exist             |replicator不存在       |
|111 |snapshot is not exist               |snapshot不存在         |
|112 |ttl type mismatch                   |ttl类型不匹配          |
|114 |ts must be greater than zero        |插入时ts必须大于0      |
|115 |invalid dimension parameter         |维度参数无效           |
|116 |put failed                          |put失败                |
|117 |st less than et                     |起始时间小于结束时间   |
|118 |reache the scan max bytes size      |scan的大小超过了最大值 |
|119 |replica endpoint already exists     |副本节点已存在         |
|120 |fail to add replica endpoint        |添加副本失败           |
|121 |replicator role is not leader       |replicator不是leader   |
|122 |fail to append entries to replicator|添加entries失败        |
|123 |file receiver init failed           |file receiver初始化失败|
|124 |cannot find receiver                |找不到receiver         |
|125 |block_id mismatch                   |block_id 不匹配        |
|126 |receive data error                  |数据接收错误           |
|127 |write data failed                   |数据写入错误           |
|128 |snapshot is sending                 |正在发送snapshot       |
|129 |table meta is illegal               |table meta不合法       |
|130 |table db path is not exist          |表的db路径不存在       |
|131 |create table failed                 |表创建失败             |
|132 |ttl is greater than conf value      |ttl超过了最大值        |
|133 |cannot update ttl between zero and nonzero|ttl不能在0和非0值之间转换|
|134 |no follower                         |没有follower           |
|135 |invalid concurrency                 |无效的concurrency      |
|136 |delete failed                       |删除失败               |
|137 |ts name not found                   |找不到时间戳列名       |
|138 |fail to get db root path            |找不到db根路径         |
|139 |fail to get recycle root path       |找不到recycle根路径     |
|    |                                    |                       |
|    |                                    |                       |
|300 |nameserver is not leader            |当前nameserver不是leader|
|301 |auto_failover is enabled            |auto_failover是开启的  |
|302 |endpoint is not exist               |endpoint不存在         |
|303 |tablet is not healthy               |tablet已经下线         |
|304 |set zk failed                       |写zk失败               |
|305 |create op failed                    |创建op失败             |
|306 |add op data failed                  |添加op失败             |
|307 |invalid parameter                   |参数错误               |
|308 |pid is not exist                    |分片id不存在           |
|309 |leader is alive                     |leader是健康的         |
|310 |no alive follower                   |没有alive的follower    |
|311 |partition is alive                  |分片是健康的           |
|312 |op status is not kDoing or kInited  |任务status不是kDoing或者kInited|
|313 |drop table error                    |删除表错误             |
|314 |set partition info failed           |设置partition结构失败  |
|315 |convert column desc failed          |转换schema失败         |
|316 |create table failed on tablet       |在tablet上创建表失败   |
|317 |pid already exists                  |分片已经存在           |
|318 |src_endpoint is not exist or not healthy|源节点不存在或者已经下线|
|319 |des_endpoint is not exist or not healthy|目的节点不存在或者已经下线|
|320 |migrate failed                      |副本迁移失败           |
|321 |no pid has update                   |分片信息没更新         |
|322 |fail to update ttl from tablet      |tablet上ttl更新失败    |
|323 |field name repeated in table_info   |增加的列名重复         |
|324 |the count of adding field is more than 63|增加的列数超过63  |
|325 |fail to update tableMeta for adding field |更新table_meta信息失败|
|326 |connect zk failed                 |连接zk失败           |
| | ||
| | ||
|400 |replica cluster alias duplicate |replica alias 重复|
|401 |connect relica cluster zk failed |连接从集群zk 失败|
|402 |not same replica name |replica alias 不一致|
|403 |connect ns failed |连接 nameserver 失败|
|404 |replica name not found |replica 未找到|
|405 |this is not follower |不是follower 集群|
|406 |term le cur term |请求携带的term 小于follower 当前的term|
|407 |zone name not equal |zone 名字 不一样|
|408 |already join zone |已经是一个从集群|
|409 |unkown server mode |未知集群状态|
|410 |zone not empty |从集群不为空|
|450 |create zk failed |创建zk 失败|
|451 |get zk failed |获取zk 失败|
|452 |del zk failed |删除zk 数据失败|
|453 |is follower cluster |从集群不可以主动添加和删除数据|
|454 |cur nameserver is not leader mdoe |当前的nameserver 不是 leader 模式|
|455 |showtable error when add replica cluster |添加从集群时showtable 发生错误|
|501 |nameserver is follower, and request has no zone info |当前集群是从集群, 且请求中缺少zone info|
|502 |zone_info mismathch |zone info 不匹配|
|503 |create CreateTableRemoteOP for replica cluster failed|创建从集群建表OP失败|
|504 |add task in replica cluster ns failed |在从集群ns 添加任务失败|
|505 |create DropTableRemoteOP for replica cluster failed|创建从集群删表OP失败|
|506 |nameserver is not replica cluster |nameserver不是从集群|
|507 |replica cluster not healthy |从集群状态不是healthy|
|508 |replica cluster has no table, do not need pid |从集群没有表，不需要指定pid|
|509 |table has a no alive leader partition |表有非alive状态的分片|
|510 |create remote table info failed |创建从集群table_info失败|
|511 |create AddReplicaRemoteOP failed |创建AddReplicaRemoteOP失败|
|512 |table has no pid xxx |表没有指定的pid|
|513 |create AddReplicasSimplyRemoteOP failed |创建AddReplicaSimplyRemoteOP失败|
|514 |remote table has a no alive leader partition |从集群表有非alive状态的分片|
|515 |request has no zone_info or task_info |请求中缺少zone info或者task_info信息|
|516 |cur nameserver is leader cluster |当前nameserver是主集群|
|601 |index delete failed | 索引删除失败|
|701 |operator not support| 不支持该操作|
