# 进程内存磁盘监控

监控rtidb的进程、内存和磁盘使用情况

# RTIDB日志监控

| 文件                     | 监控关键词                            | 事件                   | 重要等级 |
| ------------------------ | ------------------------------------- | ---------------------- | -------- |
| logs/rtidb_mon.log       | mon : kill                            | tablet进程被kill       | 非常重要 |
| logs/rtidb_mon.log       | mon : exit                            | tablet进程退出         | 非常重要 |
| logs/tablet.info.log     | reconnect zk                          | tablet和zk断开重连     | 重要     |
| logs/rtidb_ns_mon.log    | mon : kill                            | nameserver进程被kill   | 重要     |
| logs/rtidb_ns_mon.log    | mon : exit                            | nameserver进程退出     | 重要     |
| logs/nameserver.info.log | offline tablet with endpoint          | 节点下线               | 重要     |
| logs/nameserver.info.log | Run OfflineEndpoint                   | 执行节点下线           | 非常重要 |
| logs/nameserver.info.log | Run RecoverEndpoint                   | 执行恢复节点           | 非常重要 |
| logs/nameserver.info.log | reconnect zk                          | nameserver和zk断开重连 | 重要     |
| logs/nameserver.info.log | kFailed                               | 任务执行失败           | 重要     |
| logs/nameserver.info.log | The execution time of op is too long  | 任务执行超时           | 重要     |


# RTIDB数据和状态监控

1. 分片alive状态监控
   ns client执行showtable 查看分片alive状态是否为yes 
2. 分片主从同步监控
   ns client执行showtable 对比主从的offset之差，如果超过一定值就报警(建议值为100000)
