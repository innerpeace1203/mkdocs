* [资源评估](#resourse)
* [机器环境准备](#conf)

## 资源评估 {#resourse}

### 内存

fedb目前支持四种表: 
* 按条数过期的表即latest表
* 按时间过期的表即absolute表，
* absandlat表
* absorlat表  

#### latest和absorlat表

如果不同维度的数据落在同一节点和分片下, 会共享value.  

内存大小 = 维度1的主键数 \* (维度1主键的大小 + 156) + 维度2的主键数 \* (维度2主键的大小 + 156) + ... + 维度n的主键数 \* (维度n主键的大小 + 156) + n \* 数据条数 \* 56 + k \* 数据条数 \* value大小    
value大小的计算方式参考下面schema编码大小计算方式

说明:   
1 主键数是指有多少个不同的主键, 如主键为卡号就指有多少个不同的卡号  
2 k大于等于1小于等于n  

#### absolute和absandlat表

如果不同维度的数据落在同一节点和分片下, 会共享value  

内存大小 = 维度1的主键数 \* (维度1主键的大小 + 156) + 维度2的主键数 \* (维度2主键的大小 + 156) + ... + 维度n的主键数 \* (维度n主键的大小 + 156) + n \* 数据条数 \* 60 + k \* 数据条数 \* value大小    
value大小的计算方式参考下面schema编码大小计算方式

说明:   
1 主键数是指有多少个不同的主键, 如主键为卡号就指有多少个不同的卡号  
2 k大于等于1小于等于n

#### schema编码大小计算方式

如果schema字段数小于128, 数据编码大小为各个字段的累加和 + 1  
如果schema字段数大于等于128, 数据编码大小为各个字段的累加和 + 2  
各类型字段大小如下: 
  * string : 如果字符串长度小于128为 2 + 字符串长度, 否则是 3 + 字符串长度. (string长度不能超过32767)
  * int16 : 4
  * uint16 : 4
  * int32 : 6
  * int64 : 10
  * uint32 : 6
  * uint64 : 10
  * float : 6
  * double : 10
  * bool : 3
  * date : 10
  * timestamp : 10

```
示例: 单维度有schema的计算方式
一张交易表有5000万条数据, 卡号为主键共1000万个卡号. schema如下, 其中卡号长度是10, mcc长度是5
create table trx (
    card string,
    mcc string,
    amt double,
    ts timestamp,
    index(key=card, ts=ts)
)

数据的内存占用 = 10000000 * (10 + 156) + 50000000 * (12 + 7 + 6 + 1 + 60) = 5960000000Byte
```

## 机器环境准备 {#conf}

* [关闭swap](#swap)
* [关闭THP](#thp)
* [时间和时区](#time)

### 关闭操作系统swap {#swap}

查看当前系统swap是否关闭

```bash
$ free
              total        used        free      shared  buff/cache   available
Mem:      264011292    67445840     2230676     3269180   194334776   191204160
Swap:             0           0           0
```

如果swap一项全部为0表示已经关闭，否则运行下面命令关闭swap

```
$ swapoff -a
```

### 关闭THP(Transparent Huge Pages) {#thp}

查看THP是否关闭

```
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
$ cat /sys/kernel/mm/transparent_hugepage/defrag
[always] madvise never
```

如果上面两个配置中"never"没有被方括号圈住就需要设置一下

```bash
$ echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
$ echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag
```

查看是否设置成功，如果"never"被方括号圈住表明已经设置成功，如下所示：

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
$ cat /sys/kernel/mm/transparent_hugepage/defrag
always madvise [never]
```

### 时间和时区设置 {#time}

fedb数据过期删除机制依赖于系统时钟, 如果系统时钟不正确会导致过期数据没有删掉或者删掉了没有过期的数据

```bash
$ date
Wed Aug 22 16:33:50 CST 2018
```
请确保时间是正确的

