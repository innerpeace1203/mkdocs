# 常见问题

#### 1 插入的数据scan不出来

* 查看put的返回值是否put成功
* start\_time和end\_time是不是写反了, start\_time必须大于end\_time
* 如果能get出来却scan不出来，查看创建的表是不是latest表，latest表不支持scan。\(showtable 命令可以查看表类型，ttl后面是min则为absolute表否则为latest表\)
* 排查数据是不是过期了。创建表是absolute指定的**ttl是以分钟为单位的**，插入时指定的**ts精确到毫秒**

#### 2 连上client后输入的命令不支持或者提示参数不对

client分为tablet client和ns client，由role指定。如果指定--role=ns\_client启动的是ns client, 如果--role=client启动的是tablet client

ns client连接的是nameserver，集群版大部分交互命令都是基于ns client的

```bash
$ ./bin/rtidb --zk_cluster=172.27.2.52:12200 --zk_root_path=/onebox --role=ns_client
```

而tablet client连接的是tablet

```bash
$ ./rtidb --endpoint=172.27.2.52:9520 --role=client
```

#### 3 集群版创建表失败

* 执行showtable看下是不是表已经存在了，如果已经存在并且确认已有的表没用了drop后再重新创建下或者再换个表名
* 查看磁盘是不是满了
* ttl超过了最大值。默认配置中latest最大为1000条, absolute表最长为30年
* 设置的副本数大于了节点数
* 查看建表schema是否正确, 引号冒号等是否是英文的
* 到nameserver leader的机器日志路径下查看失败原因 ./logs/nameserver.info.log。连接nsclient的时候会显示ns leader

```bash
$ ./bin/rtidb --zk_cluster=172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090 --zk_root_path=/rtidb_cluster --role=ns_client
Welcome to rtidb with version 1.3.10
xxxxxxxxxxx 
ns leader: 172.27.128.33:6312
```

#### 4 启动失败

* 查看端口是否被占用
* 查看配置的endpoint中ip是否为本机ip
* 查看zk是否正常

#### 5 drop后数据没有放到recycle目录下

检查是否修改了db的路径, 将recycle路径必须和db配到相同的目录下

#### 6 节点重启后表alive的状态没有变为no

查看auto_failover是否开启. auto_failover关闭情况下不会做任何操作

#### 7 java client遍历全表时陷入死循环  

遍历全表时如果同一个pk下相同的记录条数超过一次请求的条数(默认为200条)就会导致java client陷入死循环. 可以将RemoveDuplicateByTime设置为true  
```
config.setRemoveDuplicateByTime(true);
```

#### 8 java client遍历全表时抛出BufferUnderflowException异常

查看迭代器在一次迭代中getDecodedValue是否调用了多次. 在一次迭代中getDecodedValue只能调用一次

#### 9 删除pk后内存没有释放掉, 记录数没有变  

删除pk后会将pk加入待删除链表, 默认经过两轮gc后会删除

#### 10  tablet配的域名, client访问不到集群(不能建表插入查询等)  

检查client所在机器是否配置了对应的host
