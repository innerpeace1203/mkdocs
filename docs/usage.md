# 快速入门

## 创建表

#### 1 命令行方式创建

```bash
$ ./bin/rtidb --zk_cluster=172.27.2.52:12200 --zk_root_path=/onebox --role=ns_client
> create test 144000 8 3 card:string:index mcc:string:index value:float
Create table ok
> showschema test
  #  name   type    index
---------------------------
  0  card   string  yes
  1  mcc    string  yes
  2  value  float   no
> showtable test
  name  tid  pid  endpoint             role      ttl        is_alive  compress_type  offset  record_cnt  memused  diskused
----------------------------------------------------------------------------------------------------------------------------
  test  33   0    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   0    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   0    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30859  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30859  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000 
```

#### 2 用建表文件创建

准备一个建表文件table\_meta.txt

```
name : "test"
ttl: 144000
ttl_type : "kAbsoluteTime"
partition_num: 8
replica_num: 3
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

连上ns client创建表

```bash
$ ./bin/rtidb --zk_cluster=172.27.2.52:12200 --zk_root_path=/onebox --role=ns_client
> create table_meta.txt
Create table ok
> showschema test
  #  name  type    index
--------------------------
  0  card  string  yes
  1  mcc   string  yes
  2  amt   float   no
> showtable test
  name  tid  pid  endpoint             role      ttl        is_alive  compress_type  offset  record_cnt  memused  diskused
----------------------------------------------------------------------------------------------------------------------------
  test  33   0    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   0    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   0    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   1    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30859  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   2    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   3    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   4    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30859  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   5    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30858  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   6    172.27.128.37:30860  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30858  leader    144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30859  follower  144000min  yes       kNoCompress    0       0           0.000    0.000   
  test  33   7    172.27.128.37:30860  follower  144000min  yes       kNoCompress    0       0           0.000    0.000 
```

## 插入和查询数据

```bash
$ ./bin/rtidb --zk_cluster=172.27.2.52:12200 --zk_root_path=/onebox --role=ns_client
> put test 1534992854000 card0 mcc0 1.5
Put ok
> put test 1534992888000 card0 mcc1 12.3
Put ok
> put test 1534992910000 card1 mcc1 10.7
Put ok
> scan test card0 card 1534992910000 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1534992888000  card0  mcc1  12.3000002
  2  1534992854000  card0  mcc0  1.5
> scan test card1 card 1534992910000 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1534992910000  card1  mcc1  10.6999998
> scan test mcc1 mcc 1534992910000 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1534992910000  card1  mcc1  10.6999998
  2  1534992888000  card0  mcc1  12.3000002
> get test card0 card 1534992854000
  #  ts             card   mcc   amt
--------------------------------------
  1  1534992854000  card0  mcc0  1.5
> get test card0 card 0
  #  ts             card   mcc   amt
---------------------------------------------
  1  1534992888000  card0  mcc1  12.3000002
```
