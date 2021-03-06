# fedb cli使用说明

## 支持的字段类型列表
```
bool
smallint
int
bigint
float
double
date
timestamp
string
``` 

## 进入sql client

```
# 可以将错误信息输出到其他地方，避免额外信息让命令行输出到命令行
./bin/fedb  --zk_cluster=xxxx --zk_root_path=xxx --role=sql_client 2>/dev/null
  ______   _____  ___
 |  ____|  |  __ \|  _ \
 | |__ ___ | |  | | |_) |
 |  __/ _  \ |  | |  _ <
 | | |  __ / |__| | |_) |
 |_|  \___||_____/|____/

v2.0.0.0
xxxx/>
```
**注： cli默认以分号判断输入结束，当语句中有分号时，无法分行输入**


## DDL示例

- database
    - [create database](#create-database) 创建database
    - [show databases](#show-databases) 显示所有database
    - [use database](#use-database) 切换database
    - [drop database](#drop-database) 删除database
- table 
    - [create table](#create-table) 创建表
    - [show tables](#show-tables) 显示所有表
    - [desc table](#desc-table) 显示某张表的详细信息
    - [drop table](#drop-table) 删除表
- procedure
    - [create procedure](#create-procedure) 创建存储过程
    - [show procedure status](#show-procedure-status) 显示所有存储过程
    - [show create procedure](#show-create-procedure) 显示某个存储过程的详细信息
    - [drop procedure](#drop-procedure) 删除存储过程


### create database
```
> create database mydb;
Create database success
```


### show databases
```
> show databases;
 ------------
  Databases
 ------------
  mydb
 ------------
1 rows in set
```


### use database
```
> use mydb;
Database changed
```


### drop database
```
> drop database mydb;
drop ok
```


### create table

- INDEX必选, INDEX列的值不能为null,
- 必须有KEY, 允许为bool, smallint, int, bigint, string, date, timestamp类型
- TS 允许为timestamp, bigint类型, TS的值不能为null
- KEY和TS不能为同一列

###### IF NOT EXIST

```SQL
create table IF NOT EXISTStest(
        column1 int NOT NULL,
        column2 smallint NOT NULL,
        column3 float NOT NULL,
        column4 double NOT NULL,
        column5 bigint NOT NULL,
        std_ts timestamp NOT NULL,
        std_date date NOT NULL,
        index(key=(column2), ts=std_ts)
);
```

###### 配置一个单key时序索引

```SQL
create table test(
        column1 int NOT NULL,
        column2 smallint NOT NULL,
        column3 float NOT NULL,
        column4 double NOT NULL,
        column5 bigint NOT NULL,
        std_ts timestamp NOT NULL,
        std_date date NOT NULL,
        index(key=(column2), ts=std_ts)
);
```

###### 配置一个多key时序索引

```SQL
-- 使用column4-column3作为联合key，column2作为时间索引
create table test(
          column1 int NOT NULL,
          column2 timestamp NOT NULL,
          column3 int NOT NULL,
          column4 string NOT NULL,
          column5 int NOT NULL,
          index(key=(column4, column3), ts=column2)
      );
```

###### 配置一个多key时序索with ttl

```SQL
-- 使用column2 作为时间索引，并且数据保留60天
create table test(
    column1 int NOT NULL,
    column2 timestamp NOT NULL,
    column3 int NOT NULL,
    column4 string NOT NULL,
    column5 int NOT NULL,
    index(key=(column4, column3), ts=column2, ttl=60d)
);
```

###### 配置时序索引with ttl和ttl type

```SQL

create table test(
  column1 int NOT NULL,
  column2 timestamp NOT NULL,
  column3 int NOT NULL,
  column4 string NOT NULL,
  column5 int NOT NULL,
  index(key=(column4, column3), ts=column2, ttl=60d, ttl_type=absolute)
);
create table test(
  column1 int NOT NULL,
  column2 timestamp NOT NULL,
  column3 int NOT NULL,
  column4 string NOT NULL,
  column5 int NOT NULL,
  index(key=column4, ts=column2, ttl=1000, ttl_type=latest)
);

 create table test(
   column1 int NOT NULL,
   column2 timestamp NOT NULL,
   column3 int NOT NULL,
   column4 string NOT NULL,
   column5 int NOT NULL,
   index(key=column1, ts=column2, ttl=(10d,100), ttl_type=absorlat)
 );
 
 create table test(
   column1 int NOT NULL,
   column2 timestamp NOT NULL,
   column3 int NOT NULL,
   column4 string NOT NULL,
   column5 int NOT NULL,
   index(key=column1, ts=column2, ttl=(10d,100), ttl_type=absandlat)
 );
```

**note:** ttl_type有四种类型: absolute、latest、absandlat、absorlat   

* absolute: 超过时间限制的数据将会被删除， 设置格式为ttl=10d（必须加单位，单位可以是d,h和m, 分别代表天，小时和分钟）  
* latest: 一个key下保留的最大记录条数, 设置格式为ttl=10  
* absandlat: 超过时间限制且条数限制的数据会被删除,  设置格式为ttl=(10d,100)  
* absorlat: 超过时间限制或条数限制的数据会被删除,  设置格式为ttl=(10d,100)  

###### 指定replica num  

```
> use mydb;
Database changed
> create table t1(
> col1 string,
> col2 timestamp,
> col3 double,
> index(key=col1, ts=col2)) replicanum=2;

```
* note: 副本数不能大于机器节点数。  

###### 指定replica num, 并且指定主从节点分布  

```
> use mydb;
Database changed
> create table t1(
> col1 string,
> col2 timestamp,
> col3 double,
> index(key=col1, ts=col2)) replicanum=2, distribution(leader='172.27.128.37:9542', follower='172.27.128.37:9543');

```
* note: 节点的endpoint需要使用引号；分布节点的endpoint 不能重复；replica_num和分布节点数须相等。 

###### 指定partition num  

```
> use mydb;
Database changed
> create table t1(
> col1 string,
> col2 timestamp,
> col3 double,
> index(key=col1, ts=col2)) partitionnum=8;
```

### show tables
```
> use mydb;
Database changed
> show tables;
 --------
  Tables
 --------
  t1
 --------
```


### desc table

```
> use mydb;
Database changed
> desc t1;
 --- ------- ------------ ------
  #   Field   Type         Null
 --- ------- ------------ ------
  1   col1    kVarchar     YES
  2   col2    kTimestamp   YES
  3   col3    kDouble      YES
 --- ------- ------------ ------
 --- -------------------- ------ ------ ----- -----------
  #   name                 keys   ts     ttl   ttl_type
 --- -------------------- ------ ------ ----- -----------
  1   INDEX_0_1611828127   col1   col2   0     kAbsolute
 --- -------------------- ------ ------ ----- -----------
```


### drop table
```
> use mydb;
Database changed
> drop table trans;
Drop table trans? yes/no
yes
drop ok
```


### create procedure

```
> use mydb;
Database changed
> create table trans(c1 string,
> c3 int,
> c4 bigint,
> c5 float,
> c6 double,
> c7 timestamp,
> c8 date,
> index(key=c1, ts=c7));

> CREATE PROCEDURE sp (CONST c1 string, CONST c3 int, c4 bigint, c5 float, c6 double, c7 timestamp, c8 date)
> BEGIN
> SELECT c1, c3, sum(c4) OVER w1 AS w1_c4_sum FROM trans WINDOW w1 AS (PARTITION BY trans.c1 ORDER BY trans.c7 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW); END;
```
**注： 命令行创建存储过程时，多个分号必须在同一行**

### show procedure status

```
> show procedure status;
 ------------ -------
  DB           SP
 ------------ -------
  mydb         sp
 ------------ -------
1 rows in set
```


### show create procedure

###### 示例1: 需要先use database
```
> use mydb;
Database changed
> show create procedure sp;
 ------- ----
  DB      SP
 ------- ----
  mydb1   sp
 ------- ----
1 row in set
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  SQL
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  CREATE PROCEDURE sp (CONST c1 string, CONST c3 int, c4 bigint, c5 float, c6 double, c7 timestamp, c8 date)
BEGIN
    SELECT c1, c3, sum(c4) OVER w1 AS w1_c4_sum FROM trans WINDOW w1 AS (PARTITION BY trans.c1 ORDER BY trans.c7 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW); END;
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1 row in set
 --- ------- ------------ ------------
  #   Field   Type         IsConstant
 --- ------- ------------ ------------
  1   c1      kVarchar     YES
  2   c3      kInt32       YES
  3   c4      kInt64       NO
  4   c5      kFloat       NO
  5   c6      kDouble      NO
  6   c7      kTimestamp   NO
  7   c8      kDate        NO
 --- ------- ------------ ------------

 --- ----------- ---------- ------------
  #   Field       Type       IsConstant
 --- ----------- ---------- ------------
  1   c1          kVarchar   NO
  2   c3          kInt32     NO
  3   w1_c4_sum   kInt64     NO
 --- ----------- ---------- ------------
```
**结果中4项分别表示如下含义：**

- db_name和sp_name的对应关系
- 存储过程对应的select sql
- 主表schema(即输入schema)
- 输出schema

###### 示例2: 不需要先use database
```
> show create procedure mydb.sp;
 ------- ----
  DB      SP
 ------- ----
  mydb1   sp
 ------- ----
1 row in set
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  SQL
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  CREATE PROCEDURE sp (CONST c1 string, CONST c3 int, c4 bigint, c5 float, c6 double, c7 timestamp, c8 date)
BEGIN
    SELECT c1, c3, sum(c4) OVER w1 AS w1_c4_sum FROM trans WINDOW w1 AS (PARTITION BY trans.c1 ORDER BY trans.c7 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW); END;
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1 row in set
 --- ------- ------------ ------------
  #   Field   Type         IsConstant
 --- ------- ------------ ------------
  1   c1      kVarchar     YES
  2   c3      kInt32       YES
  3   c4      kInt64       NO
  4   c5      kFloat       NO
  5   c6      kDouble      NO
  6   c7      kTimestamp   NO
  7   c8      kDate        NO
 --- ------- ------------ ------------

 --- ----------- ---------- ------------
  #   Field       Type       IsConstant
 --- ----------- ---------- ------------
  1   c1          kVarchar   NO
  2   c3          kInt32     NO
  3   w1_c4_sum   kInt64     NO
 --- ----------- ---------- ------------
```


### drop procedure
```
> use mydb;
Database changed
> drop procedure sp;
Drop store procedure sp? yes/no
yes
drop ok
```

## DML示例

### insert

```
>insert into trans (c1, c3) values('sss', 22);

>insert into trans values("aa",23,33,1.4,2.4,1590738993000, "2020-05-04");
```

## DQL示例

### select
```
> select * from trans;
 ----- ---- ---- ---------- ---------- --------------- ----------
  c1    c3   c4   c5         c6         c7              c8
 ----- ---- ---- ---------- ---------- --------------- ----------
  aa    23   33   1.400000   2.400000   1590738993000   2020-5-4
  aa    23   33   1.400000   2.400000   1590738993000   2020-5-4
```
