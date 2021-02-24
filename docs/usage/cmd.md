## 命令行

RTIDB命令行有3种client: tablet client，ns client和bs client, 分别连到不同的server. 由启动时的--role来指定

* [ns client](#ns-client)
* [tablet client](#tablet-client)
* [bs client](#bs-client)

### NS Client

连接到ns client 需要指定zk\_cluster、zk\_root\_path和role。其中zk\_cluster是zk地址，zk\_root\_path是集群在zk的根路径

```bash
$ ./bin/rtidb --zk_cluster=172.27.2.52:12200 --zk_root_path=/onebox --role=ns_client
```

#### create

创建表有命令行方式和表元数据文件两种方式

##### 1 命令行方式

命令格式: create table\_name ttl partition\_num replica\_num \[colum\_name1:type:index colum\_name2:type ...\]

* table\_name 表名
* ttl 设置ttl。如果absolute表，直接设置为ttl大小，单位是分钟，如果设置为0表示不过期。如果创建latest表，格式为latest:num，其中冒号后面的数字为一个key下保留的最大记录条数，如果创建absandlat表，格式为absandlat:abs_ttl:lat_ttl，abs_ttl同absolute表的ttl，lat_ttl同latest表的num，只有既超过时间限制又超过条数限制的数据才会被删除。如果创建absorlat表，格式为absorlat:abs_ttl:lat_ttl，abs_ttl和lat_ttl的解释同absandlat表，超过时间限制或是条数限制的数据会被删除。absandlat以及absorlat这两种ttl类型当前版本只支持内存表
* partition\_num 分片数目
* replica\_num 副本数
* schema\(可选\)  格式为 colum\_name:type\[:index\]，多列之间以空格分割。支持的type有int16, uint16, int32, uint32, int64, uint64, float, double, string, bool, date, timestamp。如果某一列是index，则注明为index
  * card:string:index    列名为card，类型是string，这一列是index. float和double列不能指定为index
  * money:float    列名是money, 类型是float

示例1: 创建名字为test1的kv表，ttl类型是absolute, ttl大小是144000分钟，分片数设置为8，三副本

```
> create test1 144000 8 3
Create table ok
```

示例2: 创建test2的kv表，ttl类型为latest, ttl大小为1保留最近一条记录，分片数设置为1，副本数设置为2

```
> create test2 latest:1 1 2
Create table ok
```

示例3: 创建表名为test3的有schema的表，ttl类型为absolute, ttl大小是100分钟，分片数设置为8，副本数设置为1，有三列分别为card、mcc和money，card和mcc是index类型是string，money的类型为float

```bash
> create test3 100 8 1 card:string:index mcc:string:index money:float
```

示例4：创建表名为test4的kv表，ttl类型为absandlat，abs_ttl设置为60分钟，lat_ttl设置为10条，分片数设置为8，副本数设置为1，

```
> create test4 absandlat:60:10 8 1
```

示例5：创建表名为test5的kv表，ttl类型为absorlat，abs_ttl设置为1440分钟，lat_ttl设置为100条，分片数设置为4，副本数设置为3

```
> create test5 absorlat:1440:100 8 1
```

创建表可以通过纯粹命令行方式和表元数据文件方式

##### 2 指定文件方式

命令格式: create table\_meta\_file\_path

* table\_meta\_file\_path 指定表文件路径

表文件格式如下:

```
name : "test"               #表名 注意加双引号
ttl_type : "kAbsoluteTime"  #指定ttl类型, 这个字段是可选的, 如果没有这个字段默认为absolute。如果要创建latest表需指定为latest或者kLatestTime
ttl: 144000                 #ttl大小, 只对absolute和latest两种类型生效, 如果ttl_type是absolute表示过期时间单位是分钟, 如果ttl_type是latest表示保留的最大记录条数, 如果设置了ttl_desc字段则忽略此字段
ttl_desc {                  #ttl大小, 对absolute、latest、absandlat、absorlat四种ttl类型都有效，当ttl_desc和ttl同时出现时，ttl_desc会覆盖ttl
  abs_ttl: 30               #过期时间，单位为分钟，如果ttl_type为latest会忽略该字段
  lat_ttl: 0               #过期条数，如果ttl_type为absolute会忽略该字段
}
partition_num: 8            #指定分片数, 这个字段是可选的, 如果没有这个字段默认分片数是8
replica_num: 3              #指定副本数, 这个字段是可选的, 如果没有这个字段默认副本数是3。副本数表示一份数据保存多少份，如果设置为3表示数据保存三份, 一主两从模式
compress_type: "snappy"     #指定压缩类型, 这个字段是可选的, 如果没有这个字段默认不压缩数据。取值为[nocompress, snappy]
storage_mode: "kMemory"     #表的存储模式, 这个字段是可选的, 默认是kMemory. 取值为[kMemory, kSSD, kHDD]. 当前版本只有KMemory支持absandlat和absorlat两种ttl类型
key_entry_max_height: 8     #指定第二层skiplist的最大高度, 这个字段是可选的. absolute表默认是4, latest表默认为1
#partition_key: "mcc"        #指定partition key, 如果有多列一起组成一个partition key就再添加这么一行
column_desc {               #column_desc指定schema信息，如果是kv表不用设置column_desc
  name : "card"             #指定列名为card
  type : "string"           #指定这一列的类型为string。目前支持的类型有int32, uint32, int64, uint64, float, double, string, bool, date, timestamp
  add_ts_idx : true         #设定是否为索引列。如果指定为索引列就可以按这一列来查询数据。float和double列不能指定为index
}
column_desc {               #有多少列就设置多少个column_desc结构
  name : "mcc"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "amt"
  type : "float"
  add_ts_idx : false
}
column_desc {
  name : "ts"
  type : "int64"
  is_ts_col : true          #设定是否为时间戳列。如果指定为时间戳列，还可以指定该时间戳的abs_ttl以及lat_ttl，如果不设置abs_ttl以及lat_ttl则默认使用ttl_desc或是ttl指定的值。
  abs_ttl : 10              #abs_ttl，指定该时间戳上的过期时间，单位为分钟，对所有ttl_type都有效
  lat_ttl : 5               #lat_ttl，指定该时间戳上最少保留数据的条数，对所有ttl_type都有效
  ttl : 3                   #ttl大小，指定该时间戳的ttl，当且仅当ttl_type为absolute或latest且该时间戳没有设置abs_ttl或是lat_ttl时生效
}
```

设置好表文件后运行创建命令即可创建表

```
> create table_file_path.txt
```

示例1:  创建名字为test1的kv表，ttl类型是absolute, 过期时间为144000分钟，分片数设置为16，三副本。保存文件名为test1.txt

```
name : "test1"
ttl: 144000
```

```
> create test1.txt
Create table ok
```

示例2: 创建名字为test2的kv表，ttl类型是latest, 保留最近一条记录，分片数设置为8，三副本。后面示例省略了创建步骤，只给出表文件

```
name : "test2"
ttl_type: "latest"
ttl: 1
partition_num: 8
```

示例3: 创建名字为test3的kv表，ttl类型是absolute, 设置数据永不过期, 数据用snappy压缩，分片数设置为8，两副本

```
name : "test3"
ttl: 0
partition_num: 8
replica_num: 2
compress_type: "snappy"
```

示例4: 创建名字为test4的有schema的表，ttl类型是absolute, 过期时间为120分钟, 分片数为16，三副本，总共三列分别为card、mcc、money，其中card和mcc是索引维度列类型是string, money列类型是float

```
name : "test4"
ttl: 120
column_desc {
  name : "card"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "mcc"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "amt"
  type : "float"
  add_ts_idx : false
}
```

示例5: 创建名字为test5的有schema的表, 指定时间戳列为ts1(只有type为int64,uint64和timestamp的列才能指定为时间戳列)
```
name : "test5"
ttl: 120
column_desc {
  name : "card"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "mcc"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "amt"
  type : "float"
  add_ts_idx : false
}
column_desc {
  name : "ts1"
  type : "int64"
  is_ts_col : true
}
```

示例6: 创建名字为test6表, 指定时间戳列为ts1和ts2, 其中ts2的ttl为100. card和mcc为组合key对应的时间戳为ts1, card对应的时间戳列为ts1和ts2
```
name : "test5"
ttl: 120
column_desc {
  name : "card"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "mcc"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "amt"
  type : "float"
}
column_desc {
  name : "ts1"
  type : "int64"
  is_ts_col : true
}
column_desc {
  name : "ts2"
  type : "timestamp"
  is_ts_col : true
  ttl : 100
}
column_key {
  index_name : "card_mcc"
  col_name : "card"
  col_name : "mcc"
  ts_name : "ts1"
}
column_key {
  index_name : "card"
  ts_name : "ts1"
  ts_name : "ts2"
}
```

示例7: 创建名字为test7表, 指定时间戳列为ts1和ts2, 其中ts2的abs_ttl为30, lat_ttl为6. card和mcc为组合key对应的时间戳为ts1, card对应的时间戳列为ts1和ts2
```
name : "test5"
ttl_type : "kAbsOrLat"
ttl_desc : {
  abs_ttl : 10
  lat_ttl : 10
}
column_desc {
  name : "card"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "mcc"
  type : "string"
  add_ts_idx : true
}
column_desc {
  name : "amt"
  type : "float"
}
column_desc {
  name : "ts1"
  type : "int64"
  is_ts_col : true
}
column_desc {
  name : "ts2"
  type : "timestamp"
  is_ts_col : true
  abs_ttl : 30
  lat_ttl : 6
}
column_key {
  index_name : "card_mcc"
  col_name : "card"
  col_name : "mcc"
  ts_name : "ts1"
}
column_key {
  index_name : "card"
  ts_name : "ts1"
  ts_name : "ts2"
}
```

**注: latest表的ttl以及任意类型表的lat_ttl最大为1000, absolute表的ttl以及任意类型表的abs_ttl最大值为30年(15768000)**

示例8: 创建relation表

```
name : "test1"
table_type : "Relational"
column_desc {
  name : "id"
  type : "bigint"
  not_null : true
}
column_desc {
  name : "attribute"
  type : "varchar"
  not_null : true
}
column_desc {
  name : "image"
  type : "varchar"
  not_null : false
}
index {
  index_name : "id"
  col_name : "id"
  index_type : "PrimaryKey"
}
```

#### showtable

showtable可以查看所有表，也可以指定看某一个表

命令格式：showtable \[table\_name\]

```
72.27.128.37:30857> showtable
  name      tid  pid  endpoint             role      ttl        is_alive  compress_type  offset   record_cnt  memused   diskused 
-----------------------------------------------------------------------------------------------------------------------------------
  test-s-1  29   0    172.27.128.37:30858  leader    30min&&10  yes       kNoCompress    1442994  343266      76.319 M  237.694 M
  test-s-1  29   0    172.27.128.37:30859  follower  30min&&10  yes       kNoCompress    1442895  343352      76.338 M  237.694 M
  test-s-1  29   0    172.27.128.37:30860  follower  30min&&10  yes       kNoCompress    1442877  343418      76.352 M  237.694 M
  test-s-1  29   1    172.27.128.37:30858  follower  30min&&10  yes       kNoCompress    1440001  343284      76.308 M  237.182 M
  test-s-1  29   1    172.27.128.37:30859  leader    30min&&10  yes       kNoCompress    1440001  342908      76.228 M  237.182 M
  test-s-1  29   1    172.27.128.37:30860  follower  30min&&10  yes       kNoCompress    1440001  343301      76.311 M  237.182 M
  test-s-1  29   2    172.27.128.37:30858  follower  30min&&10  yes       kNoCompress    1460078  347186      77.178 M  240.487 M
  test-s-1  29   2    172.27.128.37:30859  follower  30min&&10  yes       kNoCompress    1460035  347026      77.144 M  240.488 M
  test-s-1  29   2    172.27.128.37:30860  leader    30min&&10  yes       kNoCompress    1460158  347068      77.152 M  240.488 M
  test-s-1  29   3    172.27.128.37:30858  leader    30min&&10  yes       kNoCompress    1449782  343961      76.477 M  238.842 M
  test-s-1  29   3    172.27.128.37:30859  follower  30min&&10  yes       kNoCompress    1449631  343846      76.453 M  238.842 M
  test-s-1  29   3    172.27.128.37:30860  follower  30min&&10  yes       kNoCompress    1449631  344032      76.492 M  238.841 M
  test-s-2  30   0    172.27.128.37:30858  follower  30min||10  yes       kNoCompress    1444553  141663      31.969 M  237.939 M
  test-s-2  30   0    172.27.128.37:30859  follower  30min||10  yes       kNoCompress    1444553  141333      31.901 M  237.939 M
  test-s-2  30   0    172.27.128.37:30860  leader    30min||10  yes       kNoCompress    1444553  141331      31.901 M  237.941 M
  test-s-2  30   1    172.27.128.37:30858  leader    30min||10  yes       kNoCompress    1440543  141203      31.868 M  237.238 M
  test-s-2  30   1    172.27.128.37:30859  follower  30min||10  yes       kNoCompress    1440424  141084      31.844 M  237.237 M
  test-s-2  30   1    172.27.128.37:30860  follower  30min||10  yes       kNoCompress    1440543  141620      31.954 M  237.238 M
  test-s-2  30   2    172.27.128.37:30858  follower  30min||10  yes       kNoCompress    1460379  142778      32.227 M  240.617 M
  test-s-2  30   2    172.27.128.37:30859  leader    30min||10  yes       kNoCompress    1460504  142472      32.164 M  240.617 M
  test-s-2  30   2    172.27.128.37:30860  follower  30min||10  yes       kNoCompress    1460504  142965      32.266 M  240.617 M
  test-s-2  30   3    172.27.128.37:30858  follower  30min||10  yes       kNoCompress    1447794  142055      32.060 M  238.426 M
  test-s-2  30   3    172.27.128.37:30859  follower  30min||10  yes       kNoCompress    1447860  141881      32.024 M  238.427 M
  test-s-2  30   3    172.27.128.37:30860  leader    30min||10  yes       kNoCompress    1447865  141892      32.026 M  238.427 M
  test-s-3  31   0    172.27.128.37:30858  follower  30min      yes       kNoCompress    1445594  343640      76.399 M  238.098 M
  test-s-3  31   0    172.27.128.37:30859  leader    30min      yes       kNoCompress    1445635  343013      76.265 M  238.098 M
  test-s-3  31   0    172.27.128.37:30860  follower  30min      yes       kNoCompress    1445581  343710      76.414 M  238.097 M
  test-s-3  31   1    172.27.128.37:30858  follower  30min      yes       kNoCompress    1441900  341722      75.980 M  237.566 M
  test-s-3  31   1    172.27.128.37:30859  follower  30min      yes       kNoCompress    1441900  341297      75.890 M  237.566 M
  test-s-3  31   1    172.27.128.37:30860  leader    30min      yes       kNoCompress    1442035  341790      75.995 M  237.566 M
  test-s-3  31   2    172.27.128.37:30858  leader    30min      yes       kNoCompress    1459556  346687      77.076 M  240.404 M
  test-s-3  31   2    172.27.128.37:30859  follower  30min      yes       kNoCompress    1459556  346409      77.017 M  240.403 M
  test-s-3  31   2    172.27.128.37:30860  follower  30min      yes       kNoCompress    1459556  346953      77.132 M  240.404 M
  test-s-3  31   3    172.27.128.37:30858  follower  30min      yes       kNoCompress    1447200  342764      76.230 M  238.391 M
  test-s-3  31   3    172.27.128.37:30859  leader    30min      yes       kNoCompress    1447325  342233      76.117 M  238.392 M
  test-s-3  31   3    172.27.128.37:30860  follower  30min      yes       kNoCompress    1447200  342877      76.254 M  238.391 M
  test-s-4  32   0    172.27.128.37:30858  leader    10         yes       kNoCompress    1442523  140715      31.774 M  237.615 M
  test-s-4  32   0    172.27.128.37:30859  follower  10         yes       kNoCompress    1442395  140354      31.699 M  237.615 M
  test-s-4  32   0    172.27.128.37:30860  follower  10         yes       kNoCompress    1442471  141130      31.859 M  237.615 M
  test-s-4  32   1    172.27.128.37:30858  follower  10         yes       kNoCompress    1440743  140737      31.772 M  237.341 M
  test-s-4  32   1    172.27.128.37:30859  leader    10         yes       kNoCompress    1440896  139885      31.597 M  237.342 M
  test-s-4  32   1    172.27.128.37:30860  follower  10         yes       kNoCompress    1440813  140779      31.781 M  237.342 M
  test-s-4  32   2    172.27.128.37:30858  follower  10         yes       kNoCompress    1460389  142334      32.136 M  240.603 M
  test-s-4  32   2    172.27.128.37:30859  follower  10         yes       kNoCompress    1460389  141706      32.006 M  240.604 M
  test-s-4  32   2    172.27.128.37:30860  leader    10         yes       kNoCompress    1460549  142081      32.084 M  240.604 M
  test-s-4  32   3    172.27.128.37:30858  leader    10         yes       kNoCompress    1449127  141192      31.882 M  238.692 M
  test-s-4  32   3    172.27.128.37:30859  follower  10         yes       kNoCompress    1449127  140952      31.833 M  238.688 M
  test-s-4  32   3    172.27.128.37:30860  follower  10         yes       kNoCompress    1449127  141576      31.961 M  238.688 M
172.27.128.37:30857> showtable test-s-3
  name      tid  pid  endpoint             role      ttl    is_alive  compress_type  offset   record_cnt  memused   diskused 
-------------------------------------------------------------------------------------------------------------------------------
  test-s-3  31   0    172.27.128.37:30858  follower  30min  yes       kNoCompress    1460960  359006      79.631 M  238.098 M
  test-s-3  31   0    172.27.128.37:30859  leader    30min  yes       kNoCompress    1461000  358378      79.497 M  238.098 M
  test-s-3  31   0    172.27.128.37:30860  follower  30min  yes       kNoCompress    1460958  359087      79.648 M  238.097 M
  test-s-3  31   1    172.27.128.37:30858  follower  30min  yes       kNoCompress    1457180  357002      79.193 M  237.566 M
  test-s-3  31   1    172.27.128.37:30859  follower  30min  yes       kNoCompress    1457180  356577      79.102 M  237.566 M
  test-s-3  31   1    172.27.128.37:30860  leader    30min  yes       kNoCompress    1457327  357082      79.209 M  237.566 M
  test-s-3  31   2    172.27.128.37:30858  leader    30min  yes       kNoCompress    1475040  362171      80.329 M  240.404 M
  test-s-3  31   2    172.27.128.37:30859  follower  30min  yes       kNoCompress    1474884  361737      80.237 M  240.403 M
  test-s-3  31   2    172.27.128.37:30860  follower  30min  yes       kNoCompress    1474887  362284      80.354 M  240.404 M
  test-s-3  31   3    172.27.128.37:30858  follower  30min  yes       kNoCompress    1462940  358504      79.539 M  238.391 M
  test-s-3  31   3    172.27.128.37:30859  leader    30min  yes       kNoCompress    1462955  357863      79.403 M  238.392 M
  test-s-3  31   3    172.27.128.37:30860  follower  30min  yes       kNoCompress    1462942  358619      79.564 M  238.391 M
```
* **注意：** ssd/hdd表中的recored_cnt是不精确统计，仅供参考

#### addindex

添加索引

命令格式: addindex table\_name index\_name \[col\_name\] \[ts\_name\]

多个col 使用英文, 隔开
如果索引列在schema中不存在的时候，会在建立索引的时候自动创建指定的列，但是命令行需要指定列类型，索引列类型不能为float 或者 double, 列名和类型使用英文冒号:隔开

类型: `int32 int64 uint32 uint64 float double string timestamp int16 uint16 bool date`

```
> addindex test1 mcc
addindex ok
> addindex test1 id_ck id ts1
addindex ok
> addindex test1 id2_ck id1,id2,id3 ts1
addindex ok
> addindex test1 new_ck col5:string,col6:int32 ts1
addindex ok
```

#### deleteindex

删除索引

命令格式: deleteindex table\_name index\_name

```
> deleteindex test1 mcc
delete index ok
```

#### showschema

查看表的schema信息

命令格式: showschema table\_name

```
> showschema flow
table flow has not schema
> showschema test
  #  name  type    index
--------------------------
  0  card  string  yes
  1  mcc   string  yes
  2  amt   float   no
> showschema test5
  #  name  type
----------------------
  0  card  string
  1  mcc   string
  2  amt   float
  3  ts1   int64
  4  ts2   timestamp

#ColumnKey
  #  index_name  col_name  ts_col  ttl
-------------------------------------------
  0  card_mcc    card|mcc  ts1     120min
  1  card        card      ts1     120min
  2  card        card      ts2     100min
```

#### info

查看表的详细信息

命令格式: info table\_name

```
172.27.128.37:30857> info test-s-1
  attribute      value      
------------------------------
  name           test-s-1   
  replica_num    3          
  partition_num  8          
  ttl            30min&&10  
  ttl_type       kAbsAndLat 
  compress_type  kNoCompress
  storage_mode   kMemory    
  record_cnt     3120914    
  memused        689.273 M  
  diskused       1.861 G  
```
* **注意：** info展示的memused是所有alive状态的leader节点的memused之和，diskused同理

#### drop

删除表

命令格式: drop table\_name

```
> drop test
Drop table test? yes/no
yes
drop ok
```

#### put

插入数据

##### 1 kv表插入

命令格式: put table\_name pk ts value

* table\_name 表名
* pk  主键
* ts 时间戳，**精确到毫秒，ts不能为0**
* value 主键对应的值

```
> put flow key1 1535371622000 value1
Put ok
```

##### 2 schema表插入

命令格式: put table\_name ts value1 value2 value3 ...

* table\_name 表名
* ts 时间戳，**精确到毫秒，ts不能为0**
* schema各字段对应的value, 字段先后顺序可以执行showschema查看

```
> showschema test
  #  name  type    index
--------------------------
  0  card  string  yes
  1  mcc   string  yes
  2  amt   float   no
> put test 1535371622000 card0 mcc0 1.5
Put ok
```

##### 3 有指定时间列的表插入
命令格式: put table\_name value1 value2 value3 ...

* table\_name 表名
* schema各字段对应的value, 字段先后顺序可以执行showschema查看

```
172.27.128.37:6627> showschema test5
  #  name  type
----------------------
  0  card  string
  1  mcc   string
  2  amt   float
  3  ts1   int64
  4  ts2   timestamp

#ColumnKey
  #  index_name  col_name  ts_col  ttl
-------------------------------------------
  0  card_mcc    card|mcc  ts1     120min
  1  card        card      ts1     120min
  2  card        card      ts2     100min
> put test card0 mcc0 1.5 1560859166000 1560859166001
Put ok
```
##### 3 relation表插入

命令格式：put table_name=xxx col1=xxx col2=xxx col3=xxx

* table_name 指定表名
* col 指定各列的值

```
172.27.128.37:9529 > showschema test1
  #  name       type
-------------------------
  0  id         bigInt
  1  attribute  varchar
  2  image      varchar

#ColumnKey
  #  index_name  col_name  ts_col  ttl
-----------------------------------------
  0  id          id        -       0min
172.27.128.37:9529 > put table_name=test1 id=11 attribute=a1 image=i1
put ok
```

#### scan

查询一定范围内的数据

##### 1 查询kv表数据

命令格式: scan table\_name pk start\_time end\_time \[limit\]

* table\_name 表名
* pk 主键
* start\_time 查询范围的起始时间
* end\_time 查询范围的结束时间，如果end\_time设置为0查询从起始时刻的所有数据
* limit 限制最多返回条数，该字段是可选的，如果不设置就返回查询范围内所有数据

```
> scan flow key1 1535372567000 153537100000
#    Time    Data
1    1535372567000    value3
2    1535372566000    value2
> scan flow key1 1535372567000 0
#    Time    Data
1    1535372567000    value3
2    1535372566000    value2
3    9527    value1
> scan flow key1 1535372567000 0 2
#    Time    Data
1    1535372567000    value3
2    1535372566000    value2
```

##### 2 查询schema表数据

命令格式: scan table\_name key col\_name start\_time end\_time \[limit\]

* table\_name 表名
* key 主键名
* col\_name 要查询的主键所在的列名，**只有创建表时设置为index列才能查询**
* start\_time 查询范围的起始时间
* end\_time 查询范围的结束时间，如果end\_time设置为0查询从起始时刻的所有数据
* limit 限制最多返回条数，该字段是可选的，如果不设置就返回查询范围内所有数据

```
> scan test card0 card 1535373299010 1535373271000
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
  2  1535373272000  card0  mcc1  10.3999996
> scan test card0 card 1535373299010 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
  2  1535373272000  card0  mcc1  10.3999996
  3  1535371622000  card0  mcc0  1.5
>scan test card0 card 1535373299010 0 1
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
```
##### 3 查询指定时间列的表数据
  
命令格式：scan table_name=xxx key=xxx index_name=xxx st=xxx et=xxx ts_name=xxx [limit=xxx] [atleast=xxx] 

* table\_name 表名
* key 主键名
* index\_name 要查询的主键所在的列名，**只有创建表时设置为index列才能查询**
* st 查询范围的起始时间
* et 查询范围的结束时间，如果et设置为0查询从起始时刻的所有数据
* ts_name 指定时间列名
* limit 限制最多返回条数，该字段是可选的，如果不设置就返回查询范围内所有数据
* atleast 限制最少返回条数，该字段是可选的，如果设置了该字段，如果查询时间范围内的数据条数少于该值会忽略查询范围结束时间的限制，直到查询到的数据数量达到atleast或有效数据不足为止，如果同时设置了atleast和limit则必须满足 atleast <= limit
* **注意：**如果以scan table\_name key col\_name start\_time end\_time \[limit\]的格式查询，则默认指定第一个时间列  

```
> scan table_name=test key=card0 index_name=card st=1535373299010 et=1535373271000 ts_name=ts
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
  2  1535373272000  card0  mcc1  10.3999996
> scan table_name=test key=card0 index_name=card st=1535373299010 et=0 ts_name=ts
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
  2  1535373272000  card0  mcc1  10.3999996
  3  1535371622000  card0  mcc0  1.5
>scan table_name=test key=card0 index_name=card st=1535373299010 et=0 ts_name=ts limit=1
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
>scan table_name=test key=card0 index_name=card st=1535373299010 et=1535373299001 ts_name=ts limit=4 atleast=3
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
  2  1535373299005  card0  mcc2  5.80000000
  3  1535373299000  card0  mcc2  5.70000000
```

#### delete

1、删除pk

命令格式: delete table\_name key [col\_name]

* table\_name 表名
* key 主键值
* col\_name 列名. 该字段是可选的, 在多维度下如果不指定列名默认是第一个索引列

```
> delete test key1
Delete ok
> delete test1 card0 card
Delete ok
```
2、删除relation表数据

命令格式： delete table_name=xxx where col1=xxx

* table_name表名
* col 删除的条件

```
> delete table_name=test1 where id=11 
delete ok
```

#### count

##### 1 统计一个pk下的记录条数

命令格式: count table\_name key [col\_name] [filter\_expired\_data]

* table\_name 表名
* key 主键值
* col\_name 列名. 该字段时可选的
* filter\_expired\_data 是否过滤过期数据[true/false]. 默认值为false. 如果置为true, 会遍历pk下的链表性能较差
  
```
> count test card1077 card
count: 1041
```

##### 2 统计指定时间列的记录条数

命令格式：count table_name=xxx key=xxx index_name=card ts_name=xxx [filter_expired_data=true|false]

* table\_name 表名
* key 主键值
* index\_name 列名
* ts_name 指定时间列名
* filter\_expired\_data 是否过滤过期数据[true/false]. 默认值为false. 如果置为true, 会遍历pk下的链表性能较差  
  

```
> count table_name=test key=card1077 index_name=card ts_name=ts
count: 1041
```
#### preview

数据预览

命令格式: preview table\_name [limit]

* table\_name 表名
* limit 限制条数. 该字段是可选的, 默认为100条

```
> preview test 10
  #   ts             card      mcc      amt
------------------------------------------------------------
  1   1551167860977  card1077  mcc1342  9.6655599454491412
  2   1551167860879  card1077  mcc72    9.4099887366740891
  3   1551167860815  card1077  mcc1208  9.870822430478027
  4   1551167860715  card1077  mcc277   9.2822292386566794
  5   1551167860629  card1077  mcc958   9.263361336226696
  6   1551167860615  card1077  mcc547   9.2201535836470647
  7   1551167860254  card1077  mcc769   9.9815119658220137
  8   1551167860162  card1077  mcc592   9.5524055792939109
  9   1551167860069  card1077  mcc87    9.2436255334913984
  10  1551167860020  card1077  mcc637   9.0073365887995038
```

#### get

查询单条数据

##### 1 查询kv表数据

命令格式: get table\_name key ts

* table\_name 表名
* key 主键值
* ts 查询的ts，如果是0则返回最新的一条

```
> get flow key1 1535372567000
value :value3
> get flow key1 1535372566000
value :value2
> get flow key1 0
value :value3
```

##### 2 查询schema表数据

命令格式: get table\_name key col\_name ts

* table\_name 表名
* key 要get的键值
* col\_name 要查询的key所在的列名
* ts 查询的ts，如果是0则返回最新的一条

```
> get test card0 card 1535373272000
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373272000  card0  mcc1  10.3999996
> get test card0 card 1535371622000
  #  ts             card   mcc   amt
--------------------------------------
  1  1535371622000  card0  mcc0  1.5
> get test card0 card 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
```

##### 3 查询指定时间列的表数据

命令格式: get table\_name=xxx key=xxx index\_name=xxx ts=xxx ts_name=xxx 

* table\_name 表名
* key 要get的键值
* index\_name 要查询的key所在的列名
* ts 查询的ts，如果是0则返回最新的一条
* ts_name 指定时间列名  
* **注意：**若以get table_name key col_name ts 的格式查询，默认指定第一列

```
> get table_name=test key=card0 index_name=card ts=1535373272000 ts_name=ts
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373272000  card0  mcc1  10.3999996
> get table_name=test key=card0 index_name=card ts=1535371622000 ts_name=ts
  #  ts             card   mcc   amt
--------------------------------------
  1  1535371622000  card0  mcc0  1.5
> get table_name=test key=card0 index_name=card ts=0 ts_name=ts
  #  ts             card   mcc   amt
---------------------------------------------
  1  1535373299010  card0  mcc2  5.80000019
```


#### showtablet

查看tablet信息(如果使用的serverName和自动获取本地ip功能，则endpoint为serverName, real_endpoint为“-”)

```
> showtablet
  endpoint             real_endpoint     state           age
--------------------------------------------------------------
  6534708411798331392  172.17.0.12:9531  kTabletHealthy  1d
  6534708415948603392  172.17.0.13:9532  kTabletHealthy  1d
  6534708420092481536  172.17.0.14:9533  kTabletHealthy  14h
```
#### showblobserver

查看blobserver信息(如果使用的serverName和自动获取本地ip功能，endpoint为serverName, 则real_endpoint为“-”)

```
> showblobserver
  endpoint             real_endpoint     state           age
--------------------------------------------------------------
  6536558764777627648  172.17.0.15:9534  kTabletHealthy  1d
```

#### setsdkendpoint
命令格式： setsdkendpoint server sdkendpoint    

* server 如果配置endpoint, 则是endpoint; 如果配置port和use_name，则是server_name    
* sdkendpoint sdk访问的endpoint, 如果是null, 则清除对应sdkendpoint信息

查看sdkendpoint信息

```
> setsdkendpoint 6534708403616096256 172.27.128.37:9529
setsdkendpoint ok
```
```
> setsdkendpoint 6534708403616096256 null 
setsdkendpoint ok
```
#### showsdkendpoint

查看sdkendpoint信息(会显示历史配置过的所有数据，包括下线的server对应的sdkendpoint，可以手动执行setsdkednpoint server null来删除过期的配置)

```
> showsdkendpoint
  endpoint             sdk_endpoint
-------------------------------------------
  6534708403616096256  172.27.128.37:9529
  6534708407651229696  172.27.128.37:9530
  6534708411798331392  172.27.128.37:9531
```

#### addreplica

添加副本

命令格式: addreplica table\_name pid\_group endpoint

* table\_name 表名
* pid\_group 分片id集合. 可以有以下几种情况
    * 单个分片
    * 多个分片, 分片间用逗号分割. 如1,3,5
    * 分片区间, 区间为闭区间. 如1-5表示分片1,2,3,4,5
* endpoint 要添加为副本的节点endpoint

```
> showtable test1
  name  tid  pid  endpoint            role      ttl   is_alive  compress_type  offset  record_cnt  memused
------------------------------------------------------------------------------------------------------------
  test1  13   0    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
> addreplica test1 0 172.27.128.33:8541
AddReplica ok
> showtable test1
  name  tid  pid  endpoint            role      ttl   is_alive  compress_type  offset  record_cnt  memused
------------------------------------------------------------------------------------------------------------
  test1  13   0    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   0    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
> addreplica test1 1,2,3 172.27.128.33:8541
AddReplica ok
> addreplica test1 1-3 172.27.128.33:8541
AddReplica ok
```

#### delreplica

删除副本

命令格式: delreplica table\_name pid\_group endpoint

* table\_name 表名
* pid\_group 分片id集合. 可以有以下几种情况
    * 单个分片
    * 多个分片, 分片间用逗号分割. 如1,3,5
    * 分片区间, 区间为闭区间. 如1-5表示分片1,2,3,4,5
* endpoint 要删除副本的endpoint

```
> showtable test1
  name  tid  pid  endpoint            role      ttl   is_alive  compress_type  offset  record_cnt  memused
------------------------------------------------------------------------------------------------------------
  test1  13   0    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   0    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
> delreplica test1 0 172.27.128.33:8541
DelReplica ok  
> showtable test1
  name  tid  pid  endpoint            role      ttl   is_alive  compress_type  offset  record_cnt  memused
------------------------------------------------------------------------------------------------------------
  test1  13   0    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   1    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   2    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.31:8541  leader    0min  yes       kNoCompress    0       0           0.000
  test1  13   3    172.27.128.33:8541  follower  0min  yes       kNoCompress    0       0           0.000
> delreplica test1 1,2,3 172.27.128.33:8541
DelReplica ok  
> delreplica test1 1-3 172.27.128.33:8541
DelReplica ok  
```

#### makesnapshot

产生snapshot

命令格式: makesnapshot table\_name pid

* table\_name 表名
* pid 分片id

```bash
> makesnapshot flow 0
MakeSnapshot ok
```

#### migrate 

副本迁移

命令格式: migrate src\_endpoint table\_name pid\_group des\_endpoint

* src\_endpoint 需要迁出的节点
* table\_name 表名
* pid\_group 分片id集合. 可以有以下几种情况
    * 单个分片
    * 多个分片, 分片间用逗号分割. 如1,3,5
    * 分片区间, 区间为闭区间. 如1-5表示分片1,2,3,4,5
* des\_endpoint 迁移的目的节点

```
> migrate 172.27.2.52:9991 table1 1 172.27.2.52:9992
partition migrate ok
> migrate 172.27.2.52:9991 table1 1-5 172.27.2.52:9992
partition migrate ok
> migrate 172.27.2.52:9991 table1 1,2,3 172.27.2.52:9992
partition migrate ok
```


#### confget

获取配置信息，目前只支持auto\_failover

命令格式: confget \[conf\_name\]

* conf\_name 配置项名字，可选的

```
> confget
  key                 value
-----------------------------
  auto_failover       false
> confget auto_failover
  key            value
------------------------
  auto_failover  false
```

#### confset

修改配置信息，目前只支持auto\_failover

命令格式: confset conf\_name value

* conf\_name 配置项名字
* value 配置项设置的值

```
> confset auto_failover true
set auto_failover ok
```

#### offlineendpoint

下线节点。此命令是异步的返回成功后可通过showopstatus查看运行状态

命令格式: offlineendpoint endpoint [concurrency]

* endpoint是发生故障节点的endpoint。该命令会对该节点下所有分片执行如下操作:
  * 如果是主, 执行重新选主
  * 如果是从, 找到主节点然后从主节点中删除当前endpoint副本
  * 修改is_alive状态为no
* concurrency 控制任务执行的并发数. 此配置是可选的, 默认为2(name_server_task_concurrency配置可配), 最大值为name_server_task_max_concurrency配置的值

```bash
> offlineendpoint 172.27.128.32:8541
offline endpoint ok
>showtable
  name    tid  pid  endpoint            role      ttl       is_alive  compress_type  offset   record_cnt  memused
----------------------------------------------------------------------------------------------------------------------
  flow    4   0    172.27.128.32:8541  leader    0min       no        kNoCompress    0        0           0.000
  flow    4   0    172.27.128.33:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   0    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.33:8541  leader    0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.31:8541  follower  0min       yes       kNoCompress    0        0           0.000
  flow    4   1    172.27.128.32:8541  follower  0min       no        kNoCompress    0        0           0.000
```

该命令执行成功后所有分片都会有yes状态的leader

#### recoverendpoint

恢复节点数据。此命令是异步的返回成功后可通过showopstatus查看运行状态

命令格式: recoverendpoint endpoint [need_restore] [concurrency]

* endpoint是要恢复节点的endpoint
* need_restore 表拓扑是否要恢复到最初的状态, 此配置是可选的, 默认为false. 如果设置为true, 一个分片在该节点下为leader, 执行完recoverendpoint恢复数据后依然是leader
* concurrency 控制任务执行的并发数. 此配置是可选的, 默认为2(name_server_task_concurrency配置可配), 最大值为name_server_task_max_concurrency配置的值

```
> recoverendpoint 172.27.128.32:8541
recover endpoint ok
> recoverendpoint 172.27.128.32:8541 true
recover endpoint ok
> recoverendpoint 172.27.128.32:8541 true 3
recover endpoint ok
```

**注:**

1. **执行此命令前确保节点已经上线\(showtablet命令查看\)**

#### changeleader

对某个指定的分片执行主从切换。此命令是异步的返回成功后可通过showopstatus查看运行状态

命令格式: changeleader table\_name pid [candidate\_leader]

* table\_name 表名
* pid 分片id
* candidate\_leader 候选leader. 该参数是可选的, 如果设置成auto即使其他节点alive状态是yes也能切换

```
> changeleader flow 0
change leader ok
> changeleader flow 0 172.27.128.33:8541
change leader ok
> changeleader flow 0 auto
change leader ok
```

#### recovertable

恢复某个分片数据。此命令是异步的返回成功后可通过showopstatus查看运行状态

命令格式: recovertable table\_name pid endpoint

* table\_name 表名
* pid 分片id
* endpoint 要恢复分片所在的节点endpoint

```
> recovertable flow 1 172.27.128.31:8541
recover table ok
```

#### cancelop

取消一个正在执行或者待执行的操作. 取消后任务就状态变成kCanceled

命令格式: cancelop op\_id

* op\_id 需要取消的操作id

```
> cancelop 5
Cancel op ok!
```

#### showopstatus

显示操作执行信息

命令格式: showopstatus \[table\_name pid\]

* table\_name 表名 
* pid 分片id

```
> showopstatus
  op_id  op_type                name   pid  status  start_time      execute_time  end_time        cur_task
------------------------------------------------------------------------------------------------------------
  51     kMigrateOP             flow   0    kDone   20180824163316  12s           20180824163328  -
  52     kRecoverTableOP        flow   0    kDone   20180824195252  1s            20180824195253  -
  53     kRecoverTableOP        flow   1    kDone   20180824195252  1s            20180824195253  -
  54     kUpdateTableAliveOP    flow   0    kDone   20180824195838  2s            20180824195840  -
  55     kChangeLeaderOP        flow   0    kDone   20180824200135  0s            20180824200135  -
  56     kOfflineReplicaOP      flow   1    kDone   20180824200135  1s            20180824200136  -
  57     kReAddReplicaOP        flow   0    kDone   20180827114331  12s           20180827114343  -
  58     kAddReplicaOP          test1  0    kDone   20180827205907  8s            20180827205915  -
  59     kDelReplicaOP          test1  0    kDone   20180827210248  4s            20180827210252  -
> showopstatus flow
  op_id  op_type                name  pid  status  start_time      execute_time  end_time        cur_task
-----------------------------------------------------------------------------------------------------------
  51     kMigrateOP             flow  0    kDone   20180824163316  12s           20180824163328  -
  52     kRecoverTableOP        flow  0    kDone   20180824195252  1s            20180824195253  -
  53     kRecoverTableOP        flow  1    kDone   20180824195252  1s            20180824195253  -
  54     kUpdateTableAliveOP    flow  0    kDone   20180824195838  2s            20180824195840  -
  55     kChangeLeaderOP        flow  0    kDone   20180824200135  0s            20180824200135  -
  56     kOfflineReplicaOP      flow  1    kDone   20180824200135  1s            20180824200136  -
  57     kUpdateTableAliveOP    flow  0    kDone   20180824200212  0s            20180824200212  -
>showopstatus flow 1
  op_id  op_type                name  pid  status  start_time      execute_time  end_time        cur_task
-----------------------------------------------------------------------------------------------------------
  53     kRecoverTableOP        flow  1    kDone   20180824195252  1s            20180824195253  -
  56     kOfflineReplicaOP      flow  1    kDone   20180824200135  1s            20180824200136  -
```

#### gettablepartition

导出分片数据信息到当前目录，文件名为table\_name\_pid.txt

命令格式: gettablepartition table\_name pid

* table\_name 表名 
* pid 分片id

```
> gettablepartition flow 0
get table partition ok
> quit
bye
$ cat flow_0.txt 
pid: 0
partition_meta {
  endpoint: "172.27.128.32:8541"
  is_leader: true
  is_alive: false
}
partition_meta {
  endpoint: "172.27.128.33:8541"
  is_leader: true
  is_alive: false
}
partition_meta {
  endpoint: "172.27.128.31:8541"
  is_leader: true
  is_alive: true
}
term_offset {
  term: 15
  offset: 0
}
term_offset {
  term: 17
  offset: 1
}
term_offset {
  term: 19
  offset: 7154917
}
term_offset {
  term: 21
  offset: 7154917
}
term_offset {
  term: 23
  offset: 7154917
}
```

#### settablepartition

修改分片数据信息

命令格式: settablepartition table\_name partition\_file\_path

* table\_name 表名 
* partition\_file\_path 分片数据文件

用gettablepartiton导出相应文件后相应的修改

```
> settablepartition flow flow_0.txt
set table partition ok
```

**注: 此命令会改编nameserver中分片的拓扑信息，谨慎使用**

#### updatetablealive

修改分片alive状态

命令格式: updatetablealive table\_name pid endpoint is\_alive

* table\_name 表名
* pid 分片id, 如果要修改某个表的所有分片将pid指定为*
* endpoint 节点endpoint
* is\_alive 节点状态, 只能填yes或者no

```
> updatetablealive test * 172.27.128.31:8541 no
update ok
> updatetablealive test 1 172.27.128.31:8542 no
update ok
```
**注: 此命令不可当做故障恢复使用. 一般用于切流量，操作方式为将某个节点表的alive状态改为no读请求就不会落在该节点上**

#### update

更新relation表数据

命令格式: update table_name=xxx col1=xxx col2=xxx where col=xxx

* table_name 表名
* where前为需要更新的列和值
* where之后为条件

```
> update table_name=test1 attribute=a2 where id=11
update ok
```

#### query

查询relation表数据

命令格式：query table_name=xxx col1 col2 where col3=xxx
* table_name 表名
* col1col2 需要查询的列名，*表示要查询所有列 
* where后面表示查询的条件

```
172.27.128.37:9529 > query table_name=test1 * where id=11
  #  id  attribute  image
---------------------------
  1  11  a1         i1
172.27.128.37:9529 > query table_name=test1 id image where id=11
  #  id  image
----------------
  1  11  i1
```


#### showns

显示nameserver节点及其角色(如果使用的serverName和自动获取本地ip功能，则endpoint为serverName, real_endpoint为“-”)

命令格式: showns

```
>showns
  endpoint             real_endpoint     role
--------------------------------------------------
  6534708403616096256  172.17.0.10:9529  leader
  6534708407651229696  172.17.0.11:9530  standby
```


#### setttl

修改ttl

命令格式: setttl table\_name ttl\_type ttl [ts_name]

* table\_name 表名
* ttl\_type 过期类型[absolute/latest]
* ttl ttl的值
* ts_name 指定ts列的name

```
> setttl test absolute 10000
Set ttl ok !
> setttl test_latest latest 5
Set ttl ok !
> setttl ck absolute 100 ts1
Set ttl ok !
```

**注: 修改后在下一轮gc执行后才会生效**

#### addtablefield

**注:  
1.只能增加非索引列和非时间列的普通列。  
2.最多可以添加63列**  

命令格式：addtablefield table_name column_name column_type

* table_name 表名
* column_name 新增列名
* column_type 新增列类型

```
> showschema test
  #  name   type    index
---------------------------
  0  card   string  yes
  1  mcc    string  yes
  2  money  float   noi
> put test 111 1 2 3
Put ok
> preview test
  #  ts   card  mcc  money
----------------------------
  1  111  1     2    3
> addtablefield test aa string
add table field ok
> showschema test
  #  name   type    index
---------------------------
  0  card   string  yes
  1  mcc    string  yes
  2  money  float   no
  3  aa     string  no
> put test 111 2 3 4
Put ok
> put test 111 1 2 3 4
Put ok
> preivew test
  #  ts   card  mcc  money  aa
--------------------------------
  1  111  1     2    3      4
  2  111  1     2    3      -
  3  111  2     3    4      -
```

#### switchmode

切换集群状态 

tipls: 集群状态分三种 `normal` `leader` `follower`

命令格式: switchmode leader|normal

```bash
>switchmode leader // #切换到leader模式, 只有leader模式的集群才可以添加主集群
>switchmode normal // #切换到normal模式，主集群主动切换到normal,需要确保没有从集群和他关联，从集群切换到normal模式，会主动脱离集群
```



#### addrepcluster

添加从集群

tips: 条件限制: 添加从集群的表不能有表，或者从集群有表的情况下，表的schema 要和主集群的相同( 表数据的offset 不小于主集群分片leader snapshot的offset)

命令格式：addrepcluster zk_cluster zk_path alias

- zk_cluster: zk集群地址
- zk_path: zk 路径
- alias: 集群别名


```bash
> addrepcluster 172.27.128.37:6181 /rtidb-cluster1 rtidb-cluster1
adrepcluster ok
> addrepcluster 172.27.128.31:6181,172.27.128.32:6181 /rtidb-cluster2 rtidb-cluster2
adrepcluster ok

```



#### showrepcluster

查看从集群信息

```bash
172.27.128.37:6529> showrepcluster
  zk_endpoints        zk_path   alias  state            age
-------------------------------------------------------------
  172.27.128.37:6181  /issue-1  is1    kClusterHealthy  55s

```



#### removerepcluster

移除从集群

tips: 移除从集群不会删除从集群的表

命令格式：removerepcluster alias

- alias: 集群别名

```bash
>removerepcluster is1
```

#### synctable
手动同步主从集群表  

**注意：**  
1、若从集群没有有与主集群对应的表，则不要指定pid, 否则返回错误。  
2、若从集群有与主集群对应的表：如果所有分片都没有同步，则不指定pid；如果只是部分分片没有同步，则指定pid。

命令格式：synctable table_name cluster_alias [pid]  
- table_name: 表名  
- cluster_alias: 集群别名  
- pid: 表分片id，可选  
  
```
> synctable test remote 
synctable  ok
  
> synctable test remote 1 
synctable  ok
```

#### exit

退出客户端

```
> exit
bye
```

#### quit 

退出客户端

```
> quit
bye
```

#### help

获取帮助信息

命令格式: help \[cmd\]

* cmd 命令，这个参数是可选的，如果不写就列出所有命令

```
> help
addreplica - add replica to leader
cancelop - cancel the op
create - create table
confset - update conf
confget - get conf
count - count the num of data in specified key
changeleader - select leader again when the endpoint of leader offline
delete - delete pk
delreplica - delete replica from leader
drop - drop table
exit - exit client
get - get only one record
gettablepartition - get partition info
help - get cmd info
makesnapshot - make snapshot
migrate - migrate partition form one endpoint to another
man - get cmd info
offlineendpoint - select leader and delete replica when endpoint offline
preview - preview data
put -  insert data into table
quit - exit client
recovertable - recover only one table partition
recoverendpoint - recover all tables in endpoint when online
scan - get records for a period of time
showtable - show table info
showtablet - show tablet info
showns - show nameserver info
showschema - show schema info
showopstatus - show op info
settablepartition - update partition info
setttl - set table ttl
updatetablealive - update table alive status
addtablefield - add field to the schema table 
info - show information of the table
> help create
desc: create table
usage: create table_meta_file_path
usage: create table_name ttl partition_num replica_num [colum_name1:type:index colum_name2:type ...]
ex: create ./table_meta.txt
ex: create table1 144000 8 3
ex: create table2 latest:10 8 3
ex: create table3 latest:10 8 3 card:string:index mcc:string:index value:float
```

#### man

获取帮组信息

命令格式: man \[cmd\]

* cmd 命令，这个参数是可选的，如果不写就列出所有命令

```
> man
addreplica - add replica to leader
cancelop - cancel the op
create - create table
confset - update conf
confget - get conf
count - count the num of data in specified key
changeleader - select leader again when the endpoint of leader offline
delete - delete pk
delreplica - delete replica from leader
drop - drop table
exit - exit client
get - get only one record
gettablepartition - get partition info
help - get cmd info
makesnapshot - make snapshot
migrate - migrate partition form one endpoint to another
man - get cmd info
offlineendpoint - select leader and delete replica when endpoint offline
preview - preview data
put -  insert data into table
quit - exit client
recovertable - recover only one table partition
recoverendpoint - recover all tables in endpoint when online
scan - get records for a period of time
showtable - show table info
showtablet - show tablet info
showns - show nameserver info
showschema - show schema info
showopstatus - show op info
settablepartition - update partition info
setttl - set table ttl
updatetablealive - update table alive status
addtablefield - add field to the schema table 
info - show information of the table
> man drop
desc: drop table
usage: drop table_name
ex: drop table1
```

### Tablet Client

连接到tablet client需要指定endpoint和role

```bash
$ ./rtidb --endpoint=172.27.2.52:9520 --role=client
```

#### create

创建表

命名格式: create table\_name tid pid ttl segment\_cnt \[is\_leader compress\_type\]

* table\_name 为要创建表名称
* tid 指定table的id
* pid 指定table的分片id
* ttl 指定过期时间,默认过期类型为absolute. absolute表ttl单位为分钟，如果设置为0表示不过期。如果过期类型设置latest，格式为latest:保留条数\(ex: latest:10, 保留最近10条\)
* segment\_cnt 为表的segement 个数，一般设置为8
* is\_leader 指定是否为leader. 默认为leader
* compress\_type 指定字段压缩类型. 取值为\[nocompress, snappy\]，默认不压缩

示例1 创建表名为test1的leader表，ttl类型为absolute过期时间为144000分钟。tid设置为1，pid为0

```
> create test1 1 0 144000 8
Create table ok
```

示例2 创建表名为test2的follower表，ttl类型为latest保留最近两条记录。tid设置为2，pid为0

```
> create test2 2 0 latest:2 8 false
Create table ok
```

示例3 创建表名为test3的leader表，ttl类型为absolute设置不过期，数据用snappy压缩。tid设置为3，pid为0

```
> create test3 3 0 0 8 true snappy
Create table ok
```

**注: tid和pid唯一确定一张表，创建时不能和已有表重复**

#### screate

创建多维表

命名格式: screate table\_name tid pid ttl segment\_cnt is\_leader schema

* table\_name 为要创建表名称
* tid 指定table的id
* pid 指定table的分片id
* ttl 指定过期时间,默认过期类型为absolute. absolute表ttl单位为分钟，如果设置为0表示不过期。如果过期类型设置latest，格式为latest:保留条数\(ex: latest:10, 保留最近10条\)
* segment\_cnt 为表的segement 个数，一般设置为8
* is\_leader 指定是否为leader. 默认为leader
* schema  格式为 colum\_name:type\[:index\]，多列之间以空格分割。支持的type有int32, uint32, int64, uint64, float, double, string。如果某一列是index，则注明为index
  * card:string:index    列名为card，类型是string，这一列是index
  * money:float    列名是money, 类型是float

示例1 创建表名为test1的leader表, ttl类型为absolute过期时间为144000分钟。有card, mcc和money三列, 其中card和mcc为index列类型为string, money列的类型为float。tid设置为1，pid为0

```
> screate test1 1 0 144000 8 true card:string:index mcc:string:index money:float
Create table ok
```

示例2 创建表名为test2的follower表, ttl类型为latest最多保留最近两条记录。有card, mcc和money三列, 其中card是index列类型为uint64, mcc列类型为string, money列的类型为float。tid设置为2, pid为0

```
> screate test2 2 0 latest:2 8 false card:uint64:index mcc:string money:float
Create table ok
```

#### drop

删除表

命令格式: drop tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> drop 1 0
Drop table ok
```

#### put

插入数据

命令格式: put tid pid pk ts value

* tid 指定table的id
* pid 指定table的分片id
* pk 主键
* ts 时间戳, **精确到毫秒, ts不能为0** 
* value 主键对应的值

```
> put 2 0 key1 1535371622000 value1
Put ok
```

#### sput

给有schema的表插入数据

命令格式: sput tid pid ts value1 value2 value3 ...

* tid 指定table的id
* pid 指定table的分片id
* ts 时间戳, **精确到毫秒, ts不能为0** 
* schema 各字段对应的value, 字段先后顺序可以执行showschema查看

```
> sput 1 0 1535371622000 card0 mcc0 1.5
Put ok
```

#### scan

查询一定范围内的数据

命令格式: scan tid pid pk start\_time end\_time [limit]

* tid 指定table的id
* pid 指定table的分片id
* pk 主键
* start\_time 查询范围的起始时间
* end\_time 查询范围的结束时间 如果end\_time设置为0查询从起始时刻的所有数据
* limit 限制最多返回条数, 该字段是可选的, 如果不设置就返回查询范围内所有数据

```
> scan 2 0 test1 1535446708000 1535446686000
#	Time	Data
1	1535446708000	value3
2	1535446688000	value2
> scan 2 0 test1 1535446708000 0
#	Time	Data
1	1535446708000	value3
2	1535446688000	value2
3	1528858466000	value0
> scan 2 0 test1 1535446708000 0 1
#	Time	Data
1	1535446708000	value3
```

#### sscan

##### 1 查询有schema表一定范围内的数据

命令格式: sscan tid pid key col\_name start\_time end\_time [limit]

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* col\_name 要查询的主键所在的列名, 只有创建表时设置为index列才能查询
* start\_time 查询范围的起始时间
* end\_time 查询范围的结束时间 如果end\_time设置为0查询从起始时刻的所有数据
* limit 限制最多返回条数, 该字段是可选的, 如果不设置就返回查询范围内所有数据

```
> sscan 1 0 card0 card 1535446708000 1535446687000
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
  2  1535446688000  card0  mcc1  1.5
>sscan 1 0 card0 card 1535446708000 0
  #  ts             card   mcc   money
----------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
  2  1535446688000  card0  mcc1  1.5
  3  1528858466000  card0  mcc3  0.400000006
>sscan 1 0 card0 card 1535446708000 0 1
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
```

##### 2 查询有schema表一定范围内的数据

命令格式: sscan tid=xxx pid=xxx key=xxx index\_name=xxx st=xxx et=xxx ts_name=xxx [limit=xxx]

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* index\_name 要查询的主键所在的列名, 只有创建表时设置为index列才能查询
* st 查询范围的起始时间
* et 查询范围的结束时间 如果end\_time设置为0查询从起始时刻的所有数据
* ts_name 指定时间列名
* limit 限制最多返回条数, 该字段是可选的, 如果不设置就返回查询范围内所有数据

```
> sscan tid=1 pid=0 key=card0 index_name=card st=1535446708000 et=1535446687000 ts_name=ts
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
  2  1535446688000  card0  mcc1  1.5
>sscan tid=1 pid=0 key=card0 index_name=card st=1535446708000 et=0 ts_name=ts
  #  ts             card   mcc   money
----------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
  2  1535446688000  card0  mcc1  1.5
  3  1528858466000  card0  mcc3  0.400000006
>sscan tid=1 pid=0 key=card0 index_name=card st=1535446708000 et=0 ts_name=ts limit=1
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
```

#### get

查询单条数据

命令格式: get tid pid key ts

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* ts 时间戳, 如果是0则返回最新的一条

```
> get 2 0 test1 1535446688000
value :value2
> get 2 0 test1 0
value :value3
```

#### sget

##### 1 查询有schema表的单条数据

命令格式: sget tid pid key col\_name ts

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* col\_name 要查询的key所在的列名
* ts 时间戳, 如果是0则返回最新的一条

```
> sget 1 0 card0 card 1535446688000
  #  ts             card   mcc   money
----------------------------------------
  1  1535446688000  card0  mcc1  1.5
> sget 1 0 card0 card 0
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
```

##### 2 查询指定时间列的单条数据

命令格式: sget tid=xxx pid=xxx key=xxx index\_name=xxx ts=xxx ts_name=xxx

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* index\_name 要查询的key所在的列名
* ts 时间戳, 如果是0则返回最新的一条
* ts_name, 指定时间列名

```
> sget tid=1 pid=0 key=card0 index_name=card ts=1535446688000 ts_name=ts
  #  ts             card   mcc   money
----------------------------------------
  1  1535446688000  card0  mcc1  1.5
> sget tid=1 pid=0 key=card0 idex_name=card ts=0 ts_name=ts
  #  ts             card   mcc   money
---------------------------------------------
  1  1535446708000  card0  mcc0  1.29999995
```

#### delete

删除pk

命令格式: delete tid pid key [col\_name]

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* col\_name 要删除key所在的列名. 该字段是可选的

```
> delete 1 0 key1
Delete ok
```

#### preview

预览数据

命令格式: preview tid pid [limit]

* tid 指定table的id
* pid 指定table的分片id
* limit 限定输出的条数. 该字段是可选的. 默认为100条

```
>preview 9 0 10
  #   ts             card      mcc     amt
-----------------------------------------------------------
  1   1551168689441  card1077  mcc889  9.2206023558557888
  2   1551168689431  card1077  mcc632  9.3409316056017353
  3   1551168689285  card1077  mcc723  9.001503218597831
  4   1551168689276  card1077  mcc951  9.4250239440250034
  5   1551168689074  card1077  mcc101  9.6827419040776679
  6   1551168688788  card1077  mcc993  9.0653753412878455
  7   1551168688632  card1077  mcc659  9.8579586148719489
  8   1551168688617  card1077  mcc249  9.3870601038906241
  9   1551168688439  card1077  mcc448  9.1521059211468661
  10  1551168688216  card1077  mcc112  9.7253728651383557
```

#### count

##### 1 统计某一pk下的记录条数

命令格式: count tid pid key [col\_name] [filter\_expired\_data]

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* col\_name key所在的列名. 该字段是可选的
* filter\_expired\_data 是否过滤过期数据[true/false]. 默认值为false. 如果置为true, 会遍历pk下的链表性能较差

```
> count 9 0 card1077 card true
count: 7612
```

##### 2 统计指定时间列的记录条数

命令格式: count tid=xxx pid=xxx key=xxx index\_name=xxx ts_name=xxx [filter\_expired\_data=xxx]

* tid 指定table的id
* pid 指定table的分片id
* key 主键
* index\_name key所在的列名. 该字段是可选的
* ts_name=xxx 指定时间列名
* filter\_expired\_data 是否过滤过期数据[true/false]. 默认值为false. 如果置为true, 会遍历pk下的链表性能较差

```
> count tid=9 pid=0 key=card1077 index_name=card filter_expired_data=true
count: 7612
```


#### addreplica

添加副本

命令格式: addreplica tid pid endpoint

* tid 指定table的id
* pid 指定table的分片id
* endpoint 要添加为副本的节点endpoint, **此endpoint上得有对应tid和pid的follower表**

```
> addreplica 2 0 172.27.128.32:8541
AddReplica ok
```

#### delreplica

删除副本

命令格式: delreplica tid pid endpoint

* tid 指定table的id
* pid 指定table的分片id
* endpoint 要删除副本的节点endpoint

```
> delreplica 2 0 172.27.128.32:8541
DelReplica ok
```

**注: 删除副本操作并不会实际删除副本节点上的表, 只是断开了主从关系**

#### makesnapshot

产生snapshot, 会在db/tid\_pid/snapshot目录下产生snapshot文件

命令格式: makesnapshot tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> makesnapshot 1 0
MakeSnapshot ok
```

#### pausesnapshot

暂停snapshot, 执行完之后运行gettablestatus状态变为kSnapshotPaused

命令格式: pausesnapshot tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> pausesnapshot 1 0
PauseSnapshot ok
> gettablestatus 1 0
  tid  pid  offset  mode          state            enable_expire  ttl        ttl_offset  memused  compress_type
-----------------------------------------------------------------------------------------------------------------
  1    0    4       kTableLeader  kSnapshotPaused  true           144000min  0s          1.313 K  kNoCompress
```

#### recoversnapshot

恢复snapshot

命令格式: recoversnapshot tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> recoversnapshot 1 0
RecoverSnapshot ok
```

#### sendsnapshot

发送snapshot

命令格式: sensnapshot tid pid endpoint

* tid 指定table的id
* pid 指定table的分片id
* endpoint 将sendsnapshot发送到endpoint指定的节点上

```
> sendsnapshot 1 0 172.27.128.32:8541
SendSnapshot ok
```

**注: 运行sendsnapshot前必须执行下pausesnapshot, 发送完再执行recoversnapshot**

#### loadtable

1、创建表并加载数据

命令格式: loadtable table\_name tid pid ttl segment\_cnt

* table\_name 表名
* tid 指定table的id
* pid 指定table的分片id
* ttl 指定ttl
* segment\_cnt 指定segment\_cnt, 一般设置为8

```
> loadtable table1 1 0 144000 8
```

2、创建relation表并加载数据

命令格式: loadtable tid pid storage_mode

* tid 指定table的id
* pid 指定table的分片id
* storage_mode 指定存储模式

```
> loadtable 1 0 ssd
> loadTable ok
```


**注: loadtable创建的表是follower表**

#### changerole

改变表的leader角色

命令格式: changerole tid pid role \[term\]

* tid 指定table的id
* pid 指定table的分片id
* role 要修改的角色, 取值为\[leader, follower\]
* term 设定term, 该项是可选的, 默认为0

```
> changerole 1 0 follower
ChangeRole ok
> changerole 1 0 leader
ChangeRole ok
> changerole 1 0 leader 1002
ChangeRole ok
```

#### setexpire

设置是否过期

命令格式: setexpire tid pid is_expire

* tid 指定table的id
* pid 指定table的分片id
* is_expire 是否过期, 取值为\[true, false\]

```
>setexpire 1 0 false
setexpire ok
>setexpire 1 0 true
setexpire ok
```

#### showschema

查看表的schema

命令格式: showschema tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> showschema 1 0
  #  name   type    index
---------------------------
  0  card   string  yes
  1  mcc    string  yes
  2  money  float   no
> showschema 2 0
No schema for table
```

#### gettablestatus

获取表信息

命令格式: gettablestatus \[tid pid\]

* tid 指定table的id
* pid 指定table的分片id

```
> gettablestatus
  tid   pid  offset   mode            state            enable_expire  ttl               ttl_offset  memused  compress_type
----------------------------------------------------------------------------------------------------------------------------
  1     0    4        kTableLeader    kSnapshotPaused  true           144000min         0s          1.313 K  kNoCompress
  2     0    4        kTableLeader    kTableNormal     false          0min              0s          689.000  kNoCompress
> gettablestatus 2 0
  tid  pid  offset  mode          state         enable_expire  ttl   ttl_offset  memused  compress_type
---------------------------------------------------------------------------------------------------------
  2    0    4       kTableLeader  kTableNormal  false          0min  0s          689.000  kNoCompress
```

#### getfollower

查看从节点信息

命令格式: getfollower tid pid

* tid 指定table的id
* pid 指定table的分片id

```
> getfollower 4 1
  #  tid  pid  leader_offset  follower            offset
-----------------------------------------------------------
  0  4    1    5923724        172.27.128.31:8541  5920714
  1  4    1    5923724        172.27.128.32:8541  5921707
```

#### setttl

命令格式: setttl tid pid ttl\_type ttl

* tid 指定table的id
* pid 指定table的分片id
* ttl\_type ttl类型
* ttl 设置ttl的值

```
> setttl 1 0 absolute 1000
Set ttl ok !
> setttl 2 0 latest 5
Set ttl ok !
```

**注: 修改后在下一轮gc执行后才会生效**

#### setlimit

命令格式: setlimit method limit

* method 方法名(区分大小写). 如果是Server表示整体最大并发数
* limit 最大并发数. 如果limit为0表示不限制

```
> setlimit Server 5
Set Limit ok
> setlimit Server 0
Set Limit ok
> setlimit Put 10
Set Limit ok
```

#### exit

退出客户端

```
> exit
bye
```

#### quit

退出客户端

```
> quit
bye
```

#### help

获取帮助信息

命令格式: help \[cmd\]

```
> help
create - create table
screate - create multi dimension table
drop - drop table
put - insert data into table
sput - insert data into table of multi dimension
scan - get records for a period of time
sscan - get records for a period of time from multi dimension table
get - get only one record
sget - get only one record from multi dimension table
addreplica - add replica to leader
delreplica - delete replica from leader
makesnapshot - make snapshot
pausesnapshot - pause snapshot
recoversnapshot - recover snapshot
sendsnapshot - send snapshot to another endpoint
loadtable - create table and load data
changerole - change role
setexpire - enable or disable ttl
showschema - show schema
gettablestatus - get table status
getfollower - get follower
setttl - set ttl for partition
setlimit - set tablet max concurrency limit
exit - exit client
quit - exit client
help - get cmd info
man - get cmd info
> help create 
desc: create table
usage: create name tid pid ttl segment_cnt [is_leader compress_type]
ex: create table1 1 0 144000 8
ex: create table1 1 0 144000 8 true snappy
ex: create table1 1 0 144000 8 false
```

#### man 

获取帮组信息

命令格式: man \[cmd\]

```
> man
create - create table
screate - create multi dimension table
drop - drop table
put - insert data into table
sput - insert data into table of multi dimension
scan - get records for a period of time
sscan - get records for a period of time from multi dimension table
get - get only one record
sget - get only one record from multi dimension table
addreplica - add replica to leader
delreplica - delete replica from leader
makesnapshot - make snapshot
pausesnapshot - pause snapshot
recoversnapshot - recover snapshot
sendsnapshot - send snapshot to another endpoint
loadtable - create table and load data
changerole - change role
setexpire - enable or disable ttl
showschema - show schema
gettablestatus - get table status
getfollower - get follower
setttl - set ttl for partition
setlimit - set tablet max concurrency limit
exit - exit client
quit - exit client
help - get cmd info
man - get cmd info
> man create 
desc: create table
usage: create name tid pid ttl segment_cnt [is_leader compress_type]
ex: create table1 1 0 144000 8
ex: create table1 1 0 144000 8 true snappy
ex: create table1 1 0 144000 8 false
```


### Bs Client

连接到bs client需要指定endpoint和role

```bash
$ ./rtidb --endpoint=172.27.2.52:9531 --role=bs_client
```

#### loadtable

创建表并加载数据

命令格式: loadtable tid pid

* tid relation table的id
* pid relation table的分片id

```
> loadtable 1 0 
> loadTable ok
```
