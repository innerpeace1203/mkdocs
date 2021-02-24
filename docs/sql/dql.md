# DQL(Data Query Language, 数据查询语言)


## Select Statement

| 语句类型   |                                                              |
| :--------- | :----------------------------------------------------------- |
| 简单SELECT | 简单SELECT的边界等价于[#4 函数和运算能力](https://wiki.4paradigm.com/pages/viewpage.action?pageId=79040464)的边界 |
| GROUP BY   | **不支持请求模式**； 仅支持【**列】**作为Group By的表达式    |
| WHERE      | 不支持请求模式                                               |
| LIMIT      | 请求模式下，LIMIT没有实际意义                                |
| WINDOW     | 关于FESQL的WINDOW与传统WINDOW的一些差异，在wiki：[#2 WINDOW语句](https://wiki.4paradigm.com/pages/viewpage.action?pageId=80863617)[SQL语法设计](https://wiki.4paradigm.com/pages/viewpage.action?pageId=72973720)中有更详细的描述。 |
| LAST JOIN  | 关于FESQL的LAST JOIn的定义、描述以及与LEFT JOIN的差异，在wiki：[#3 JOIN语句](https://wiki.4paradigm.com/pages/viewpage.action?pageId=80863627)[SQL语法设计](https://wiki.4paradigm.com/pages/viewpage.action?pageId=72973720)中有更详细的描述。 |

### 1. 简单SELECT

```sql

-- desc: SELECT ALL
  SELECT * FROM t1;
 
-- desc: SELECT列
  SELECT COL1, -COL1 FROM t1;
 
-- desc: SELECT列重命名
  SELECT COL1 as c1 FROM t1;
 
-- desc: SELECT列重命名2
  SELECT COL1 c1 FROM t1;
 
-- desc: SELECT 表.列
  SELECT t1.COL1 FROM t1;
 
-- desc: SELECT 表.列 重命名
  SELECT t1.COL1 as c1 FROM t1;
 
-- desc: SELECT 表.列 重命名2
  SELECT t1.COL1 c1 FROM t1;
 
-- desc: SELECT所有列
  SELECT * FROM t1;
 
-- desc: SELECT 表达式+
  SELECT COL1 + COL2 as col12 FROM t1;
 
-- desc: SELECT 函数表达式
  SELECT substr(col1, 3, 6);
```

### 2. WINDOW语句

#### window_clause语法描述

```SQL
window_spec:
    WINDOW window_name AS window_clause
    
window_clause: 
[window_union_clause] PARTITION BY subject ORDER BY order frame_clause [EXCLUDE CURRENT_TIME] [INSTANCE_NOT_IN_WINDOW]
 
frame_clause:
    frame_units frame_extend opt_frame_size
 
frame_units:
    {ROWS|ROWS_RANGE}
 
opt_frame_size:
    MAXSIZE const_expr|/*EMPTY*/
 
frame_extent:
    frame_between|frame_start
frame_between:
    BETWEEN frame_start AND frame_end
 
frame_start, frame_end: {
    CURRENT ROW
  | UNBOUNDED PRECEDING
  | UNBOUNDED FOLLOWING
  | expr PRECEDING
  | expr FOLLOWING
  | expr OPEN PRECEDING
  | expr OPEN FOLLOWING
 
window_union_clause:
    UNION table_ref [, table_ref ...]
 
table_ref:
    {table_name | sub_query }

}
```



#### 窗口类型

FESQL有两类窗口：ROWS 和 ROWS_RANGE。SQL标准的RANGE类窗口FESQL系统目前暂不支持。他们直接的对比差异如下图所示

![image-20210203131721898](./img/window_frame_type.png)

#### 窗口边界Frame Bound

| 语法                          | 描述                                                         |
| :---------------------------- | :----------------------------------------------------------- |
| CURRENT ROW                   | For `ROWS`, the bound is the current row.For `ROWS_RANGE`, the bound is the current row. For `RANGE`, the bound is the peers of the current row.Peers of the current row： 表示当前行order值相同的行 |
| UNBOUNDED PRECEDING           | The bound is the first partition row.                        |
| `UNBOUNDED FOLLOWING`         | The bound is the last partition row.                         |
| *`expr`* PRECEDING闭区间      | For `ROWS`, the bound is *`expr`* rows before the current row.For `ROWS_RANGE`, the bound is the rows with values **equal** to the current row value minus *`expr.`*if the current row value is `NULL`, the bound is the current row.For `RANGE`, the bound is the rows with values **equal** to the current row value minus *`expr.`*if the current row value is `NULL`, the bound is the peers of the current row. |
| *`expr`* FOLLOWING闭区间      | For `ROWS`, the bound is *`expr`* rows before the current row.For `ROWS_RANGE`, the bound is the rows with values **equal** to the current row value plus *`expr.`*if the current row value is `NULL`, the bound is the current row.For `RANGE`, the bound is the rows with values **equal** to the current row value plus *`expr.`*if the current row value is `NULL`, the bound is the peers of the current row. |
| *`expr`* OPEN PRECEDING开区间 | For `ROWS`, the bound is *`expr-1`* rows before the current row.For `ROWS_RANGE`, the bound is the rows with values **less than** to the current row value minus *`expr.`*if the current row value is `NULL`, the bound is the current row.For `RANGE`, the bound is the rows with values **less than** to the current row value minus *`expr.`*if the current row value is `NULL`, the bound is the peers of the current row. |
| *`expr`* OPEN FOLLOWING开区间 | For `ROWS`, the bound is *`expr`* rows before the current row.For `ROWS_RANGE`, the bound is the rows with values **less than** to the current row value plus *`expr.`*if the current row value is `NULL`, the bound is the current row.For `RANGE`, the bound is the rows with values **less than** the current row value plus *`expr.`*if the current row value is `NULL`, the bound is the peers |



#### 窗口示例

**有名窗口（Named Window）**

```SQL
-- 
SELECT sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)
```

**匿名窗口**

```SQL
SELECT id, pk1, col1, std_ts,
sum(col1) OVER (PARTITION BY pk1 ORDER BY std_ts ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) as w1_col1_sum
from t1;
```

**ROWS窗口**

```SQL
-- ROWS example
-- desc: window ROWS, 前1000条到当前条
SELECT sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW);
```

**ROWS_RANGE窗口**

```SQL
-- ROWS example
-- desc: window ROWS, 前1000条到当前条
SELECT sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (PARTITION BY col1 ORDER BY col5 ROWS_RANGE BETWEEN 1000s PRECEDING AND CURRENT ROW);
```



####  FESQL语法特性：WINDOW UNION

**window with union 一张副表**

```SQL
SELECT col1, col5, sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (UNION t2 PARTITION BY col1 ORDER BY col5 ROWS_RANGE BETWEEN 10s PRECEDING AND CURRENT ROW) limit 10;
```

![image-20210204152105774](./img/window_union_1_table.png)

**window with union 多张副表**

```SQL
SELECT col1, col5, sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (UNION t2, t3 PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) limit 10;
```

![image-20210204152526828](./img/window_union_2_table.png)

**window with union 样本表不进入窗口**

```SQL
SELECT col1, col5, sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (UNION t2 PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 3 PRECEDING AND CURRENT ROW INSTANCE_NOT_IN_WINDOW) limit 10;
```

![image-20210204152439360](./img/window_union_1_table_instance_not_in_window.png)

**window with union subquery**

```SQL
SELECT col1, col5, sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS
(UNION (select c1 as col1, c2 as col2, 0.0 as col3, 0.0 as col4, c5 as col5, "NA" as col6 from t2),
(select c1 as col1, c2 as col2, 0.0 as col3, 0.0 as col4, c5 as col5, "NA" as col6 from t3)
PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) limit 10;
```



#### FESQL语法特性：Exclude CURRENT_TIME

**ROWS窗口EXCLUDE CURRENT_TIME**

```SQL
-- ROWS example
-- desc: window ROWS, 前1000条到当前条, 除了current row以外窗口内不包含当前时刻的其他数据
SELECT sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (PARTITION BY col1 ORDER BY col5 ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW EXCLUDE CURRENT_TIME);
```

**ROWS_RANGE窗口EXCLUDE CURRENT_TIME**

```SQL
-- ROWS example
-- desc: window ROWS, 前1000s到当前条，除了current row以外窗口内不包含当前时刻的其他数据
SELECT sum(col2) OVER w1 as w1_col2_sum FROM t1
WINDOW w1 AS (PARTITION BY col1 ORDER BY col5 ROWS_RANGE BETWEEN 1000s PRECEDING AND CURRENT ROW EXCLUDE CURRENT_TIME);
```

![image-20210204152624503](./img/window_exclude_current_time.png)

#### FESQL语法特性： MAXSIZE

window frame定义了窗口的范围，FESQL引入MAXSIZE，来限制window内允许的有效窗口内最大数据条数

![image-20210203131850391](./img/window_max_size.png)

**普通的ROWS_RANGE窗口的聚合操作**

```
SELECT``count``(col2) OVER w1 ``as` `w1_col2_cnt,``FROM` `t1``WINDOW w1 ``AS` `(PARTITION ``BY` `col1 ``ORDER` `BY` `col5 ROWS_RANGE ``BETWEEN` `10 PRECEDING ``AND` `CURRENT` `ROW) limit 10;
```

**普通的ROWS_RANGE窗口的聚合操作**

```
SELECT``count``(col2) OVER w2 ``as` `w1_col2_cnt,``FROM` `t1``WINDOW w2 ``AS` `(PARTITION ``BY` `col1 ``ORDER` `BY` `col5 ROWS_RANGE ``BETWEEN` `10 PRECEDING ``AND` `CURRENT` `ROW MAXSIZE 5) limit 10;
```



### 3. JOIN语句

关于FESQL的LAST JOIn的定义、描述以及与LEFT JOIN的差异，在wiki：[SQL语法设计#LastJoin设计](https://wiki.4paradigm.com/pages/viewpage.action?pageId=72973720#SQL语法设计-LastJoin设计) 中有更详细的描述。

#### 语法描述

```SQL
last_join_clause:
    table_ref last_join_item [ last_join_item ...]
  
  
last_join_item:
    LAST JOIN table_ref [ORDER BY order_col] ON expr
   
table_ref:
    {table_name | sub_query}
```

**LAST JOIN without ORDER BY**

```sql
-- desc: 简单拼表查询 without ORDER BY

SELECT t1.col1 as t1_col1, t2.col1 as t2_col2 from t1 LAST JOIN t2 ON t1.col1 = t2.col1
```

![image-20210203132611349](./img/last_join_without_order.png)

以左表第二行为例，符合条件的右表有2条，选择第一个匹配的拼接到左表。拼表结果如下：

![image-20210203132630089](./img/last_join_without_order2.png)

**LAST JOIN with ORDER BY**

```SQL
-- desc: 简单拼表查询 with ORDER BY
SELECT t1.col1 as t1_col1, t2.col1 as t2_col2 from t1 LAST JOIN t2 ORDER BY ts.std_ts ON t1.col1 = t2.col1
```

![image-20210203132659546](./img/last_join_with_order1.png)

以左表第二行为例，符合条件的右表有2条，按`std_ts`排序后，选择最后一条`2020-05-20 10:11:13`

最后的拼表结果如下：

![image-20210203132722447](./img/last_join_with_order2.png)



**LAST JOIN后作window查询**

```SQL
SELECT t1.col1 as id, t1.col2 as t1_col2, t1.col5 as t1_col5, t2.col5 as t2_col5, str1, sum(t1.col1) OVER w1 as w1_col1_sum
FROM t1 last join t2 order by t2.col5 on t1.col1=t2.col1 and t1.col5 >= t2.col5
WINDOW w1 AS (PARTITION BY t1.col2 ORDER BY t1.col5 ROWS_RANGE BETWEEN 3 PRECEDING AND CURRENT ROW) limit 10;

```

#### LastJoin边界说明

last join有两种用法(具体的功能边界可以查询功能边界文档[#3 JOIN语句](https://wiki.4paradigm.com/pages/viewpage.action?pageId=80863627)):

1. Last Join with ORDER BY
   1. 在性能敏感模式下（在RTDIB环境下）要求JOIN条件和ORDER BY列都能命中索引
2. Last Join without ORDER BY 在性能敏感模式下（如RTIDB环境下）：
   1. 如果是行拼接（右边部分是一个Row)，则没有边界限制
   2. 如果是表拼接（右边部分是Table), 要求JOIN条件命中右表索引。

### 4. Group语句

所有的group by目前仅仅批模式支持（也就是控制台的调试SQL支持，离线模式还是开发中）

```SQL
-- desc: 简单SELECT分组KEY
  SELECT COL1 FROM t1 group by COL1;
-- desc: 简单SELECT分组KEY表达式
  SELECT COL1+COL2 FROM t1 group by COL1+COL2;
-- desc: SELECT多KEY的分组
  SELECT COL1, COL2 FROM t1 group by COL1, COL2;
-- desc: 简单SELECT分组聚合
  SELECT COL1, SUM(COL2) FROM t1 group by COL1;
-- desc: SELECT多KEY的分组聚合
  SELECT COL1, COL2, SUM(COL3) FROM t1 group by COL1, COL2;
```

### 5 . Where语句

```SQL
-- desc: SELECT简单过滤
  sql: SELECT COL1 FROM t1 where COL1 > 10;
-- desc: SELECT过滤条件是复杂逻辑关系表达式
  sql: SELECT COL1 FROM t1 where COL1 > 10 and COL2 = 20 or COL1 =0;
```

### 6. Limit

请求模式下，LIMIT没有实际意义

```SQL
-- desc: SELECT Limit
  SELECT t1.COL1 c1 FROM t1 limit 10;
```



# FESQL基本运算

### 1. 比较函数与运算



| Name      | FeSQL | Description                                                  | case覆盖 |
| :-------- | :---- | :----------------------------------------------------------- | :------- |
| `>`       |       | Greater than operator操作类型边界：L Rnumberdatetimestampstringnumber    date    timestamp    string |          |
| `>=`      |       | Greater than or equal operator操作类型边界同上               |          |
| `<=`      |       | Less than operator操作类型边界同上                           |          |
| `!=`      |       | Not equal operator操作类型边界同上                           |          |
| `<=`      |       | Less than or equal operator操作类型边界同上                  |          |
| `<=>`     |       | NULL-safe equal to operator                                  |          |
| `=`, `==` | =,==  | Equal operator操作类型边界同上                               |          |
| `ISNULL`  |       | Test whether the argument is NULL                            |          |

### 2. 逻辑运算

| Name                                                         | MySQL | SparkSQL | FlinkSQL | FeSQL | Description                                                  | case覆盖 |
| :----------------------------------------------------------- | :---- | :------- | :------- | :---- | :----------------------------------------------------------- | :------- |
| [`AND`, `&&`](https://dev.mysql.com/doc/refman/8.0/en/logical-operators.html#operator_and) |       | AND      | AND      |       | Logical AND操作数类型边界L Rnumbertimestampdatestringnumber    timestamp    date    string |          |
| [`OR`, `||`](https://dev.mysql.com/doc/refman/8.0/en/logical-operators.html#operator_or) |       | OR       | OR       |       | Logical OR操作数边界同上                                     |          |
| [`XOR`](https://dev.mysql.com/doc/refman/8.0/en/logical-operators.html#operator_xor) |       | XOR      | XOR      |       | Logical XOR操作数边界同上                                    |          |
| [`NOT`, `!`](https://dev.mysql.com/doc/refman/8.0/en/logical-operators.html#operator_not) |       | NOT      | NOT      |       | Negates value支持所有基础类型numbertimestampdatestring       |          |

### 3. 算术运算

| Name                                                         | Description                                              | case覆盖 |
| :----------------------------------------------------------- | :------------------------------------------------------- | :------- |
| [`%`, `MOD`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_mod) | Modulo operator                                          |          |
| [`*`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_times) | Multiplication operator                                  |          |
| [`+`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_plus) | Addition operator                                        |          |
| [`-`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_minus) | Minus operator                                           |          |
| [`-`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_unary-minus) | Change the sign of the argument只支持数值型操作数-number |          |
| [`/`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_divide) | Division operator                                        |          |
| [`DIV`](https://dev.mysql.com/doc/refman/8.0/en/arithmetic-functions.html#operator_div) | Integer division                                         |          |



### 4. 类型运算

**Table 12.14 类型转换兼容情况**

| src\|dist     | bool   | smallint | int    | float  | int64  | double | timestamp | date   | string |
| :------------ | :----- | :------- | :----- | :----- | :----- | :----- | :-------- | :----- | :----- |
| **bool**      | Safe   | Safe     | Safe   | Safe   | Safe   | Safe   | UnSafe    | X      | Safe   |
| ·**smallint** | UnSafe | Safe     | Safe   | Safe   | Safe   | Safe   | UnSafe    | X      | Safe   |
| **int**       | UnSafe | UnSafe   | Safe   | Safe   | Safe   | Safe   | UnSafe    | X      | Safe   |
| **float**     | UnSafe | UnSafe   | UnSafe | Safe   | Safe   | Safe   | UnSafe    | X      | Safe   |
| **bigint**    | UnSafe | UnSafe   | UnSafe | UnSafe | Safe   | UnSafe | UnSafe    | X      | Safe   |
| **double**    | UnSafe | UnSafe   | UnSafe | UnSafe | UnSafe | Safe   | UnSafe    | X      | Safe   |
| **timestamp** | UnSafe | UnSafe   | UnSafe | UnSafe | Safe   | UnSafe | Safe      | UnSafe | Safe   |
| **date**      | UnSafe | X        | X      | X      | X      | X      | UnSafe    | Safe   | Safe   |
| **string**    | UnSafe | UnSafe   | UnSafe | UnSafe | UnSafe | UnSafe | UnSafe    | UnSafe | Safe   |



###  5. 位运算

**Table 12.16 Bit Functions and Operators**

| Name | Description |
| :--- | :---------- |
| `&`  | Bitwise AND |
| `>>` | Right shift |
| `<<` | Left shift  |

### 6. Case When运算

语法描述

```SQL
CASE value WHEN [compare_value] THEN result [WHEN [compare_value] THEN result ...] [ELSE result] END

CASE WHEN [condition] THEN result [WHEN [condition] THEN result ...] [ELSE result] END
```

