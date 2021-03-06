

# 内置类型

## 数值类型

| 类型     | 大小    | 范围（有符号）                                               | 范围（无符号）                                               | 用途            |
| :------- | :------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :-------------- |
| BOOL     | 1 byte  | (-128，127)                                                  | (0，255)                                                     | 小整数值        |
| SMALLINT | 2 bytes | (-32 768，32 767)                                            | (0，65 535)                                                  | 大整数值        |
| INT      | 4 bytes | (-2 147 483 648，2 147 483 647)                              | (0，4 294 967 295)                                           | 大整数值        |
| BIGINT   | 8 bytes | (-9,223,372,036,854,775,808，9 223 372 036 854 775 807)      | (0，18 446 744 073 709 551 615)                              | 极大整数值      |
| FLOAT    | 4 bytes | (-3.402 823 466 E+38，-1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38) | 0，(1.175 494 351 E-38，3.402 823 466 E+38)                  | 单精度 浮点数值 |
| DOUBLE   | 8 bytes | (-1.797 693 134 862 315 7 E+308，-2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308) | 0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308) | 双精度 浮点数值 |

------

## 日期和时间类型

表示时间值的日期和时间类型为DATE、TIMESTAMP

每个时间类型有一个有效值范围和一个NULL值，当指定不合法FESQL不能表示的值时使用NULL值。

| 类型      | 大小 (bytes) | 范围                                                         | 格式            | 用途                     |
| :-------- | :----------- | :----------------------------------------------------------- | :-------------- | :----------------------- |
| DATE      | 4            | 1900-01-01 ~                                                 | YYYY-MM-DD      | 日期值                   |
| TIMESTAMP | 8            | 1970-01-01 00:00:00/2038结束时间是第 **2147483647** 秒，北京时间 **2038-1-19 11:14:07**，格林尼治时间 2038年1月19日 凌晨 03:14:07 | YYYYMMDD HHMMSS | 混合日期和时间值，时间戳 |

------

## 字符串类型

| 类型   | 大小 | 用途       |
| :----- | :--- | :--------- |
| STRING | 4096 | 变长字符串 |


