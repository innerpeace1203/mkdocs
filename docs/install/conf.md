# 配置文件详解

## nameserver配置文件

用\#注释的配置一般不需要改动

* **endpoint**  // 启动时指定的endpoint
* **port**  // 如果使用自动获取本地ip功能，则注释endpoint, 配置此选项 
* role   // 启动时指定的角色, 如果启动的是nameserver就指定为nameserver
* **zk\_cluster**  // 指定zk cluster的地址
* **zk\_root\_path** // 指定root path
* **use_name** // 配合port配置，设置server_name, 不能带有 /
* data_dir  // name.txt文件的目录路径 
* log\_dir  // 配置日志路径
* log\_file\_count   // 保留文件数量, 一般不需要改动
* log\_file\_size   // 配置单个日志文件大小, 一般不需要改动
* log\_level   // 配置日志级别, 建议在生产环境中为info级别
* **auto\_failover**  // 配置是否开启auto failover, 默认是开启的. 如果设置开启就会自动做故障转移和恢复
* **thread\_pool\_size**  // 配置brpc内部占用线程数
* request\_max\_retry  // 请求失败时的最大重试次数
* request\_timeout\_ms   // 请求的超时时间, 单位是ms
* request\_sleep\_time  // 请求失败时的等待时间, 单位是ms
* **zk\_session\_timeout   **// 设置zk session的超时时间, 单位是ms
* zk\_keep\_alive\_check\_interval  // 检查和zk连接的时间间隔, 如果断开会自动重连. 单位是ms
* **tablet\_heartbeat\_timeout   // **配置心跳超时时间, 如果超过了配置的时间还没有连上zk就将相应节点置为下线状态
* tablet\_offline\_check\_interval  // 如果nameserver检测到tablet下线, 以配置的时间间隔去检查是否上线直到超时
* name\_server\_task\_pool\_size  // 配置nameserver任务线程池的大小
* name\_server\_task\_wait\_time  // 任务执行框架中任务队列为空时的休眠时间, 单位是ms
* get\_task\_status\_interval  // 配置获取tablet任务状态的时间间隔, 单位是ms
* max\_op\_num  // 配置nameserver内存中保存op的最大数
* **replica\_num**  // 创建表时默认副本数\(包含主节点\)
* **partition\_num** // 创建表时默认分片数 
* name_server_task_concurrency  // 配置执行op的默认并发数
* name_server_task_max_concurrency  // 配置最大的op并发数
* name_server_op_execute_timeout  // 配置任务执行超时时间. 如果超过这个时间就会打印warning日志
* get_table_status_interval // 更新表状态的时间间隔
* get_table_diskused_interval // 更新表占用磁盘空间的时间间隔
* check_binlog_sync_progress_delta  // 配置CheckBinlogSynsProgress任务的offset阈值. 如果主从之间的offset之差小于该值就认为数据已追平
* make_snapshot_time  // 配置make snapshot的时间, 如配置为23表示每天23点make snapshot
* make\_snapshot\_check\_interval  // 检查make snapshot的时间间隔, 单位是ms

配置文件demo如下

```asciidoc
# nameserver.conf
#endpoint不能用0.0.0.0和127.0.0.1
--endpoint=172.27.128.31:6527
#--port=6527
--role=nameserver
--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
--zk_root_path=/fedb_cluster
#--use_name=true
#--data_dir=./data

--log_dir=./logs
--log_file_count=24
--log_file_size=1024
--log_level=info

--auto_failover=true

#--thread_pool_size=16
#--request_max_retry=3
--request_timeout_ms=5000
#--request_sleep_time=1000

--zk_session_timeout=10000
#--zk_keep_alive_check_interval=15000
--tablet_heartbeat_timeout=300000
#--tablet_offline_check_interval=1000

#--name_server_task_pool_size=8
#--name_server_task_concurrency=2
#--name_server_task_max_concurrency=8
#--name_server_task_wait_time=1000
#--name_server_op_execute_timeout=7200000
#--get_task_status_interval=2000
#--get_table_status_interval=2000
#--check_binlog_sync_progress_delta=100000
#--max_op_num=10000

#--replica_num=3
#--partition_num=16
```

## tablet配置文件

* **endpoint ** // 指定节点的ip和port, 示例172.27.128.31:9991\(注:endpoint不能用0.0.0.0和127.0.0.1\)
* **port**  // 如果使用自动获取本地ip功能，则注释endpoint, 配置此选项 
* role  // 启动时指定的角色, 如果启动的是tablet就指定为tablet
* **zk\_cluster **// 集群模式时指定连接zookeeper集群的地址, 如果zookeeper是多节点用逗号分开, 示例 172.27.2.51:6330,172.27.2.52:6331
* **zk\_root\_path** // 指定在zk写入数据的根节点
* **use_name** // 配合port配置，设置server_name, 不能带有 /
* data_dir  // name.txt文件的目录路径 
* **thread\_pool\_size **// 配置brpc内部占用线程数
* **scan\_concurrency\_limit  **// 配置scan的最大并发数
* **put\_concurrency\_limit  **// 配置put的最大并发数
* **get\_concurrency\_limit  **//  配置get的最大并发数
* **zk\_session\_timeout ** //  设置zk session的超时时间, 单位是ms
* zk\_keep\_alive\_check\_interval  // 检查和zk连接的时间间隔, 如果断开会自动重连. 单位是ms 
* **log\_dir ** // 配置日志文件目录
* log\_file\_count  // 保留日志文件最大个数
* log\_file\_size  // 单个日志文件的最大值, 单位是MB
* log\_level  // 配置日志级别
* binlog\_coffee\_time  // 没有读出最新数据的等待待时间, 单位是ms
* binlog\_match\_logoffset\_interval  // 同步从节点offset任务的时间间隔, 单位是ms
* binlog\_notify\_on\_put  // 配置为true时, 有写操作会立即同步到从节点
* binlog\_single\_file\_max\_size  // 配置binlog文件的大小, 单位是MB
* binlog\_sync\_batch\_size  // 主从同步时一次同步的最大记录条数
* binlog\_sync\_to\_disk\_interval  // 数据刷磁盘的时间间隔, 单位是ms
* binlog\_sync\_wait\_time  // 主从同步时没有数据同步的等待时间, 单位是ms
* binlog\_name\_length  // 配置binlog名字的长度
* binlog\_delete\_interval  //  删除binlog任务的时间间隔, 单位是ms
* binlog\_enable\_crc  // 配置binlog是否要开启crc校验
* io\_pool\_size  // 执行io任务的线程池大小
* task\_pool\_size  //  tablet线程池的大小
* **db\_root\_path ** // 配置binlog和snapshot的存放目录, 可以配置多个以英文逗号分割
* **ssd\_root\_path ** // 配置ssd表数据、binlog和snapshot的存放目录, 可以配置多个以英文逗号分割，如果不指定则无法创建SSD表
* **hdd\_root\_path ** // 配置ssd表数据、binlog和snapshot的存放目录, 可以配置多个以英文逗号分割，如果不指定则无法创建HDD表
* **recycle\_bin\_root\_path ** // 配置droptable回收站的目录, 需要和db\_root\_path再同一目录下。可以配置多个以英文逗号分割
* **recycle\_ssd\_bin\_root\_path ** // 配置drop ssd table回收站的目录, 需要和ssd\_root\_path在同一个盘下。可以配置多个以英文逗号分割
* **recycle\_hdd\_bin\_root\_path ** // 配置drop hdd table回收站的目录，需要和hdd\_root\_path在同一个盘下。可以配置多个以英文逗号分割
* make\_snapshot\_time  // 配置make snapshot的时间, 如配置为23表示每天23点make snapshot
* make\_disktable\_snapshot\_interval  // SSD以及HDD类型表 make snapshot 的时间间隔，单位为分钟
* make\_snapshot\_check\_interval  // 检查make snapshot的时间间隔, 单位是ms
* make_snapshot_threshold_offset  // 执行makesnapshot的offset阈值, 当前offset和上次makesnapshot的offset之差大于这个值才会做snapshot
* make_snapshot_offline_interval //  用来控制tablet 多长时间未收到ns 的定义makesnaphot，自主执行snapshot，单位s
* snapshot_compression // 配置snapshot压缩算法, 此选项需要机器上安装好 fpga 和相应驱动, 默认为 off 即关闭，选项分别为 off, pz, zlib, snappy
* snapshot_pool_size // 配置make snapshot时线程池大小，默认是1
* **gc\_interval ** // 执行过期键删除任务的时间间隔, **单位是分钟**
* **disk\_gc\_interval ** // 执行hdd/ssd表过期键删除任务的时间间隔, **单位是分钟**
* gc\_pool\_size   // 执行过期删除任务的线程池大小
* gc\_safe\_offset  // ttl时间的偏移, 单位是分钟. 一般不用配置该项
* send\_file\_max\_try  // 发送文件失败时的最大重试次数
* retry\_send\_file\_wait\_time\_ms  // 发送文件失败时重试等待时间.
* stream\_close\_wait\_time\_ms  // 发送完一个文件的等待时间
* stream\_block\_size  // streaming一次发送数据块的最大值, 单位是byte
* stream\_bandwidth\_limit  // streaming方式发送数据时的最大速度, 单位是byte/s
* request\_max\_retry  // 请求失败时的最大重试次数
* request\_timeout\_ms  // 请求失败时的最大重试次数
* request\_sleep\_time  // 请求失败时的等待时间
* disable\_wal // 配置是否关掉rocksdb wal文件
* file\_compression // 配置压缩算法, 此选项需要机器上安装好 fpga 和相应驱动, 默认为 off 即关闭，选项分别为 off, pz, lz4, zlib

配置文件demo如下

```asciidoc
# tablet.conf
#endpoint不能用0.0.0.0和127.0.0.1
--endpoint=172.27.128.33:9527
#--port=6527
--role=tablet

# if tablet run as cluster mode zk_cluster and zk_root_path should be set
#--zk_cluster=172.27.128.33:7181,172.27.128.32:7181,172.27.128.31:7181
#--zk_root_path=/fedb_cluster
#--use_name=true
#--data_dir=./data

# concurrency conf
# thread_pool_size建议和cpu核数一致
--thread_pool_size=24
--scan_concurrency_limit=16
--put_concurrency_limit=8
--get_concurrency_limit=16

--zk_session_timeout=10000
#--zk_keep_alive_check_interval=15000

# log conf
--log_dir=./logs
--log_file_count=24
--log_file_size=1024
--log_level=info

# binlog conf
#--binlog_coffee_time=1000
#--binlog_match_logoffset_interval=1000
--binlog_notify_on_put=true
--binlog_single_file_max_size=2048
#--binlog_sync_batch_size=32
--binlog_sync_to_disk_interval=5000
#--binlog_sync_wait_time=100
#--binlog_name_length=8
#--binlog_delete_interval=60000
#--binlog_enable_crc=false

#--io_pool_size=2
#--task_pool_size=8

# path可以配置多个目录
--db_root_path=./db
--ssd_root_path=./db
--hdd_root_path=./db
--recycle_bin_root_path=./recycle
--recycle_ssd_bin_root_path=./recycle
--recycle_hdd_bin_root_path=./recycle

# snapshot conf
# 每天23点做snapshot
--make_snapshot_time=23
#--make_snapshot_check_interval=600000
#--make_snapshot_threshold_offset=100000

# garbage collection conf
# 60m
--gc_interval=60
#--disk_gc_interval=120
#--gc_pool_size=2
# 1m
#--gc_safe_offset=1

# send file conf
#--send_file_max_try=3
#--stream_close_wait_time_ms=1000
#--stream_block_size=1048576
--stream_bandwidth_limit=20971520
#--request_max_retry=3
#--request_timeout_ms=5000
#--request_sleep_time=1000
#--retry_send_file_wait_time_ms=3000

# rocksdb
#--disable_wal=true


# loadtable
#--load_table_batch=30
#--load_table_thread_num=3
#--load_table_queue_size=1000

```

## monitor配置文件
```
# port不用改  
port=8000  
# 数据采样频率, 默认为1秒  
interval=1  
# monitor log目录  
log_dir=./logs  

tablet_endpoint=172.27.128.37:9827  
# 要监控tablet的log目录  
tablet_log_dir=./logs  

# 要监控nameserver的log目录  
ns_log_dir=/home/denglong/env/rtidb_test/ns/logs  
```

## 配置优化

- 针对高并发的场景, tablet将以下配置加大些
    - thread\_pool\_size
    - scan\_concurrency\_limit
    - put\_concurrency\_limit
    - get\_concurrency\_limit 
- 如果部署目录空间较小, 可以将数据放到其他目录上
    - db\_root\_path
    - recycle\_bin\_root\_path


## blob_proxy mime db 配置文件

* config/mime.conf 描述了content-type 对应的后缀名列表, content-type 和后缀名列表使用分号分隔, 后缀名列表使用空格分隔, mime db 预先放置了一些配置, 如果有,没有记录到的 content-type , 用户可以自行添加, 添加完成后重启 blob_proxy 即可
