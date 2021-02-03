# DDL(Data Definition Language, 数据定义语言)

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

### create-database
```
CREATE DATABASE db_name
```


### show databases {#show-databases}
```
SHOW DATABASES
```

### use database {#use-database}
```
USE db_name
```


### drop database {#drop-database}
```
DROP DATABASE db_name
```


### create table{#create-table}

```
CREATE TABLE [IF NOT EXISTS] tbl_name
    (create_definition,...)
    [table_options]
  
table_options:
    table_option [[,] table_option] ...
table_option: {
    REPLICANUM=replica_num
    | DISTRIBUTION(ROLETYPE=endpoint [[,] ROLETYPE=endpoint]...)
}
ROLETYPE: {
    LEADER | FOLLOWER
}
```

### show tables {#show-tables}

```
SHOW TABLES
```


### desc table {#desc-table}

```
DESC table_name
```


### drop table {#drop-table}
```
DROP TABLE table_name
```


### create procedure{#create-procedure}
> - proc_parameter由主表对应的schema决定
> - CONST用来标识是不是公共列

```
CREATE PROCEDURE sp_name(proc_parameter[,...])
BEGIN
　　select_stmt;
END
 
proc_parameter:
    [CONST] param_name type
type:
    Any valid FESQL data type
select_stmt:
    Valid select sql statement
```


### show procedure status {#show-procedure-status}

```
SHOW PROCEDURE STATUS
```


### show create procedure {#show-create-procedure}

```
SHOW CREATE PROCEDURE [db_name.]sp_name
```


### drop procedure {#drop-procedure}
```
DROP procedure sp_name
```
