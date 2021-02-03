## 目录介绍

FEDB部署目录结构如下

* bin  // 脚本和二进制存放目录
  * fedb   // RTIDB二进制文件
  * mon   // 守护进程文件
  * start.sh  // tablet启动脚本
  * start\_ns.sh    // nameserver启动脚本
* conf  
  * tablet.flags  // tablet配置文件
  * nameserver.flags  // nameserver配置文件
* db  // binlog和snapshot存放目录. nameserver没有此目录
  * tid\_pid    tid是表的id, pid是分片id. 有多少个\(tid, pid\)对, 就有多少个对应的目录
    * table\_meta.txt   // 记录表元数据文件, 创建表的一些信息可以从该文件中查到
    * binlog  // 该目录下放的是binlog文件
    * snapshot  // 该目录下放的是snapshot相关文件
      * MANIFEST  // 该文件是对snapshot文件的一个汇总, 包含offset, snapshot文件名, 记录条数和term
      * xxx.sdb   // snapshot文件
    * data  // 该目录下放的是ssd/hdd表的数据文件
* tools // 放一些工具脚本
* logs // 日志目录
  * rtidb\_mon.log  // 守护进程启动tablet的日志文件
  * rtidb\_ns\_mon.log // 守护进程启动nameserver的日志文件
  * nameserver.info.log  // nameserver日志文件
  * tablet.info.log  // tablet日志文件
* recycle  // 回收站目录. drop table之后就会把db目录下对应的tid\_pid目录mv到这个目录下.
