# 导入工具

* [使用说明](#使用说明)
* [配置文件](#配置文件)
* [性能](#性能)

### 使用说明 {#使用说明}

#### 功能
   实现将文件中数据导入rtidb的功能，支持parquet、csv和orc三种存储格式文件的导入，可以导入单一指定文件，也可以导入指定目录下的多个文件，如把多个parquet文件放在同一目录下进行操作。  

  程序运行时会根据文件后缀名自动进入相应解析程序。对于parquet文件和orc文件，程序可以读取文件得到schema，继而将数据插入rtidb;对于csv文件，用户需要在csv_schema.conf配置文件中定义schema.  
    
  **注意：**  
  1. 因为兼容问题，如果用户rtidb服务端的版本低于1.4.1，则暂不支持自动建表功能，须手动创建好表格之后再进行数据的导入.  
  2. rtidb服务端版本为1.4.1以上，才支持columnkey功能.
#### 执行步骤
1、在http://pkg.4paradigm.com/rtidb/tools/dataimporter-util.jar 下载dataimporter.jar；  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;config.properties,columnKey.conf和csv_schema.conf自己创建，可以复制本文档中模版进行修改。   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**注意：**  配置文件和jar包需要放在同一目录下  
2、准备本地的数据文件（ xxx.parquet、xxx.orc或者xxx.csv文件）  
3、配置config.properties，指定数据文件的绝对路径（若导入一个目录下的多个文件，配置目录的绝对路径,且目录文件中只能包含parquet、csv或者orc一种格式的文件，不能包含其它类型文件）和表名；如果支持组合主键和指定时间列功能，则配置columnKey.conf，并在config.properties中配置columnKeyPath；如果导入csv文件，配置csv_schema.conf文件，自定义schema。   
4、运行本地jar包，执行命令：java -cp jar包名 com._4paradigm.dataimporter.Main  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如 java -cp dataimporter.jar com._4paradigm.dataimporter.Main
 
####支持数据类型
1、parquet  
支持8种基本类型：int32,int64,int96,float,double,boolean,binary,fixed_len_byte_array  

|parquet类型|对应rtidb类型|
|-----|-----|
|int32|int32|
|int64|int64|
|int96|timestamp|
|float|float|
|double|double|
|boolean|bool|
|binary|string|
|fixed_len_byte_array|string|

**注意:**指定的ts类型必须为int64或者int96类型。
2、csv  
支持9种基本类型：string,int16,int32,int64,double,float,timestamp,date,bool     
csv文件中timestamp的格式为yyyy-mm-dd hh:mm:ss.[fff...],date的格式为yyyy-[m]m-[d]d
csv类型和rtidb类型一一对应  
**注意:**指定的ts类型必须为int64或者timestamp类型。

3、orc  
支持struct中包含14种基本类型：binaray,boolean,byte,date,double,float,int,long,short,string,timestamp,char,varchar,decimal  

|orc类型|对应rtidb类型|  
|-----|-----|
|binary|string|
|boolean|bool|
|byte|int16|
|date|date|
|double|double|
|float|float|
|int|int32|
|long|int64|
|short|int16|
|string|string|
|timestamp|timestamp|
|char|string|
|varchar|string|
|decimal|double|

**注意:**指定的ts类型必须为long或者timestamp类型。
### 配置文件{#配置文件}
#### config.properties


```
#文件或者目录的绝对路径
filePath=/Users/innerpeace/Desktop/part-r-00000.gz.parquet
#表的存储模式三种：memory ssd hdd
storageMode=memory
#表名
tableName=trans
#用户配置ritdb中是否已经提前创建成功表，若没有，程序会先创建表，后导入数据
tableExist=false
#指定索引列的列名,多个之间分号分割 (自动建表时才需要指定)
index=SD_PAN
#指定文件中时间戳列名,多个之间分号分割
timeStamp=trx_time_std
#指定parquet文件和orc文件需要导入的列下标,若没有配置，导入全部列，多个之间逗号分隔
inputColumnIndex=0,1,2,3,4,5,6,7,8,9,11
#组合主键配置文件路径,指定时间戳列必须指定组合键，如果没有组合键，则不指定路径 (自动建表时才需要指定)
columnKeyPath=columnKey.conf
#是否可以存储key为null或者emptystring
handleNull=true
#csv文件schema配置文件的相对路径，即文件名
csv.schemaPath=csv_schema.conf
#csv文件的分隔符，通常都为,
csv.separator=,
#csv文件的编码格式
csv.encodingFormat=UTF-8
#csv文件是否有表头
csv.hasHeader=true

#配置zk地址, 和集群启动配置中的zk_cluster保持一致
zkEndpoints=172.27.128.37:7181,172.27.128.37:7182,172.27.128.37:7183
#配置集群的zk根路径, 和集群启动配置中的zk_root_path保持一致
zkRootPath=/rtidb_cluster
#副本数 (自动建表时才需要指定)
replicaNum=1
#分片数 (自动建表时才需要指定)
partitionNum=1
#kAbsoluteTime或者kLatestTime (自动建表时才需要指定)
ttlType=kAbsoluteTime
# 0对应kNoCompress ； 1对应kSnappy (自动建表时才需要指定)
compressType=0
#生存时间，设置ttl. 如果ttl类型是kAbsoluteTime, 那么ttl的单位是分钟，ttl为0时，表示不过期；
#如果ttl类型是kLatestTime，ttl表示保留的最大记录条数 (自动建表时才需要指定)
ttl=0
# 配置编码格式，1代表新格式 (自动建表时才需要指定)
formatVersion=0

#线程池相关，一般不需要重新配置
#线程池的核心线程数和最大线程数，执行put操作的最大线程数为maximumPoolSize+1
maximumPoolSize=6
#阻塞队列大小
blockingQueueSize=10000
#成功put一定数量数据输出一条日志
log.interval=1000

```

#### csv_schema.conf
方式一

```
#rtidb中schema的字段名，字段类型和字段下标(标识该列在原始schema中的位置,用于导入部分列时使用 )。
columnName=col_1;columnType=int32;columnIndex=0
columnName=col_2;columnType=int64;columnIndex=1
columnName=col_3;columnType=string;columnIndex=2
columnName=col_4;columnType=float;columnIndex=3
columnName=col_5;columnType=double;columnIndex=4
columnName=col_6;columnType=boolean;columnIndex=5
```
  
假若原始schema共有6个column，想导入其中第1、3、5个column,则可以使用如下方式：  
```
columnName=col_1;columnType=int32;columnIndex=0
columnName=col_2;columnType=string;columnIndex=2
columnName=col_3;columnType=double;columnIndex=4
```
方式二
```
#如果配置文件中没有columnIndex这一列，则导入文件中全部列
columnName=col_1;columnType=int32
columnName=col_2;columnType=int64
columnName=col_3;columnType=string
columnName=col_4;columnType=float
columnName=col_5;columnType=double
columnName=col_6;columnType=boolean
```
注意：如果想导入全部列，且存在columnIndex配置，则columnIndex必须从0开始

#### columKey.conf
```
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
### 性能{#性能}

#### 测试环境

| OS| Linux m7-pce-dev01 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux |
| ------------- | ------------- |
| CPU | processor : 40   &nbsp;&nbsp;&nbsp;&nbsp;  Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz 
| MEM | 503G |

#### 测试过程

```
  数据量：1000w   
  put线程数：7  
  rtidb副本数：1  
  rtidb分片数：1  
  rtidb表类型：absolute表  
  ```
rtidb表schema如下：
  ```
  #   name       type    index
--------------------------------
  0   mdcardno   string  yes
  1   CARDSTAT   string  no
  2   CARDKIND   string  no
  3   SYNFLAG    string  no
  4   GOLDFLAG   string  no
  5   OPENDATE   string  no
  6   CDSQUOTA   string  no
  7   CDTQUOTA   string  no
  8   ACTCHANEL  string  no
  9   ACTDATE    string  no
  10  SALECOD    string  no
  ```
  其中mdcardno长度为8，其它字段长度均为2。


在单台机器环境下导入10,000,000数据用时3min5s,平均qps达到50,000以上。
