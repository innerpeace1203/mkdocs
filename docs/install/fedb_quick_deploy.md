# 一键部署RTIDB集群

### 准备包
* 下载jdk包 wget http://pkg.4paradigm.com/rtidb/dev/jdk-8u121-linux-x64.tar.gz
* 下载zookeeper包 wget http://pkg.4paradigm.com/rtidb/dev/zookeeper-3.4.10.tar.gz
* 下载rtidb包 wget http://pkg.4paradigm.com/rtidb/rtidb-cluster-1.6.3.0.tar.gz
* 解压rtidb包

### 修改配置文件tools/conf.json

```
{
    "zookeeper":{
            "path":"/home/rtidb/env/test", // zk的部署路径
            "package":"./package/zookeeper-3.4.10.tar.gz", // 放zk包地址的路径
            "java_package":"./package/jdk-8u121-linux-x64.tar.gz",  //java jdk路径
            "inner_port1":2881,  // zk内部通信端口
            "inner_port2":3881,  // zk选主端口
            "need_deploy": true,  // 是否要部署zk, 如果已经部署有zk集群此处填false
            "address_arr":[
                {
                    "address":"172.27.129.198:12201", // 配置zk的部署节点及端口
                    "path":"/home/rtidb/env/zk", // 单独配置节点的zk部署路径
                    "inner_port1":5881, // 单独配置zk内部通信端口
                    "inner_port2":6881 // 单独配置zk选主端口
                },
                {
                    "address":"172.27.128.33:12202" // 配置zk的部署节点及端口
                }
            ]
    },
    "zk_root_path":"/rtidb_cluster",
    "nameserver": {
        "path":"/home/rtidb/env/test/nameserver",  // nameserver的部署路径
        "package":"./package/rtidb-cluster-1.6.3.0.tar.gz",  // rtidb包路径
        "address_arr": [
            {
                "address":"172.27.129.198:6927"  // nameserver的ip和port
            },
            {
                "address":"172.27.129.33:6927",  // nameserver的ip和port
                "path":"/home/rtidb/env/test/ns" // nameserver的部署路径, 如果单独配了path就用此path
            }
        ]
    },
    "tablet": {
        "path":"/home/rtidb/env/test/tablet",  // tablet的部署路径
        "package":"./package/rtidb-cluster-1.6.3.0.tar.gz",  // rtidb包路径
        "address_arr": [
            {
                "address":"172.27.129.198:9027",  // tablet的ip和port
                "path":"/home/rtidb/env/test/tb" // tablet的部署路径, 如果单独配了path就用此path
            },
            {
                "address":"172.27.128.33:9027"
            }
        ]
    }
}    
```
**注: 配置文件是一个json, 修改后必须还是一个合法的json**  

### 建立互信机制
建立互信的方法可以参考https://blog.csdn.net/chenghuikai/article/details/52807074

### 设定系统环境参数(关闭swap和THP)
* 切换到root账户
* 执行python tools/deploy.py tools/conf.json init_env

**注: 此处可以自行修改系统参数, 如果已经修改好就跳过此步**

### 开始部署
* 切换到工作账户
* 执行python tools/deploy.py tools/conf.json
