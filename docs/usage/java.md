java sdk文档

* [配置maven](#maven)
* [集群模式使用说明](#cluster)
* [单机模式使用说明](#single)

# 配置maven pom {#maven}

添加如下依赖配置, 其中version配置java sdk版本. 最新版本可以从[这里](http://nexus.4paradigm.com/nexus/content/repositories/releases/com/_4paradigm/rtidb-client/)获取

```
<dependency>
    <groupId>com._4paradigm</groupId>
    <artifactId>rtidb-client</artifactId>
    <version>1.4.1-RELEASE</version>
</dependency>
```

# 集群模式 {#cluster}

```java
package example;

import com._4paradigm.rtidb.client.GetFuture;
import com._4paradigm.rtidb.client.PutFuture;
import com._4paradigm.rtidb.client.ScanFuture;
import com._4paradigm.rtidb.client.KvIterator;
import com._4paradigm.rtidb.client.TableAsyncClient;
import com._4paradigm.rtidb.client.TableSyncClient;
import com._4paradigm.rtidb.client.TabletException;
import com._4paradigm.rtidb.client.ha.RTIDBClientConfig;
import com._4paradigm.rtidb.client.ha.impl.NameServerClientImpl;
import com._4paradigm.rtidb.client.ha.impl.RTIDBClusterClient;
import com._4paradigm.rtidb.client.ha.TableHandler.ReadStrategy;
import com._4paradigm.rtidb.client.impl.TableSyncClientImpl;
import com._4paradigm.rtidb.client.impl.TableAsyncClientImpl;
import com._4paradigm.rtidb.client.schema.ColumnDesc;
import com._4paradigm.rtidb.ns.NS;
import com.google.protobuf.ByteString;
import com._4paradigm.rtidb.common.Common.ColumnDesc;
import com._4paradigm.rtidb.common.Common.ColumnKey;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeoutException;

public class ClusterExample {
    private static String zkEndpoints = "172.27.128.31:8090,172.27.128.32:8090,172.27.128.33:8090";  // 配置zk地址, 和集群启动配置中的zk_cluster保持一致
    private static String zkRootPath = "/rtidb_cluster";   // 配置集群的zk根路径, 和集群启动配置中的zk_root_path保持一致
    // 下面这几行变量定义不需要改
    private static String leaderPath = zkRootPath + "/leader";
    // NameServerClientImpl要么做成单例, 要么用完之后就调用close, 否则会导致fd泄露
    private static NameServerClientImpl nsc = new NameServerClientImpl(zkEndpoints, leaderPath);
    private static RTIDBClientConfig config = new RTIDBClientConfig();
    private static RTIDBClusterClient clusterClient = null;
    // 发送同步请求的client
    private static TableSyncClient tableSyncClient = null;
    // 发送异步请求的client
    private static TableAsyncClient tableAsyncClient = null;

    static {
        try {
            nsc.init();
            config.setZkEndpoints(zkEndpoints);
            config.setZkRootPath(zkRootPath);
            // 设置读策略. 默认是读local
            // 可以单独设置全局值和表级别的值, 表级别优先于全局值
            // config.setGlobalReadStrategies(TableHandler.ReadStrategy.kReadLeader);
            // Map<String, ReadStrategy> strategy = new HashMap<String, ReadStrategy>();
            // strategy put传入的参数是表名和读策略. 读策略可以设置[kReadLeader, kReadFollower, kReadLocal, kReadRandom]
            // kReadLeader: 只读leader
            // kReadFollower: 优先读follower. 如果follower挂了会读leader. follower恢复后会自动切到follower上
            // kReadLocal: 优先读本地. 优先选择读取部署在当前机器的节点, 如果当前机器没有部署tablet则随机选择一个follower读取, 如果没有从节点就读leader
            // kReadRandom: 随机读. 会随机的选择一个节点读取
            // strategy.put("test1", ReadStrategy.kReadLocal);
            // config.setReadStrategies(strategy);
            // 如果要过滤掉同一个pk下面相同ts的值添加如下设置
            // config.setRemoveDuplicateByTime(true);
            // 设置最大重试次数
            // config.setMaxRetryCnt(3);
            clusterClient = new RTIDBClusterClient(config);
            clusterClient.init();
            tableSyncClient = new TableSyncClientImpl(clusterClient);
            tableAsyncClient = new TableAsyncClientImpl(clusterClient);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 创建kv表
    public void createKVTable() {
        String name = "test1";
        NS.TableInfo.Builder builder = NS.TableInfo.newBuilder();
        Tablet.TTLDesc ttlDesc = Tablet.TTLDesc.newBuilder()
                .setTtlType(Tablet.TTLType.kAbsoluteTime) // 设置ttl类型，共有四种 kAbsoluteTime, kLatestTime, kAbsAndLat, kAbsOrLat, 如不设置此域默认为kAbsoluteTime
                .setAbsTtl(30) // 设置absolute ttl的值,单位为分钟，不设置默认为0，即不限制
                .setLatTtl(0) // 设置latest ttl的值，单位为条，不设置默认为0，即不限制
                .build();
        builder = NS.TableInfo.newBuilder()
                .setName(name)  // 设置表名
                //.setReplicaNum(3)    // 设置副本数, 此设置是可选的, 默认为3
                //.setPartitionNum(8)  // 设置分片数, 此设置是可选的, 默认为16
                //.setCompressType(NS.CompressType.kSnappy) // 设置数据压缩类型, 此设置是可选的默认为不压缩
                //.setStorageMode(Common.StorageMode.kMemory) // 设置表的存储模式, 默认为kMemory. 还可以设置为Common.StorageMode.kSSD和Common.StorageMode.kHDD
                .setTtlDesc(ttlDesc);      // 设置ttl
        NS.TableInfo table = builder.build();
        // 可以通过返回值判断是否创建成功
        boolean ok = nsc.createTable(table);
        clusterClient.refreshRouteTable();
    }

    // 创建有schema的表
    public void createSchemaTable() {
        String name = "test2";
        NS.TableInfo.Builder builder = NS.TableInfo.newBuilder();
        Tablet.TTLDesc ttlDesc = Tablet.TTLDesc.newBuilder()
                .setTtlType(Tablet.TTLType.kAbsAndLat) // 设置ttl类型，共有四种kAbsoluteTime, kLatestTime, kAbsAndLat, kAbsOrLat, 如不设置此域默认为kAbsoluteTime
                .setAbsTtl(30) // 设置absolute ttl的值,单位为分钟，不设置默认为0，即不限制
                .setLatTtl(1) // 设置latest ttl的值，单位为条，不设置默认为0，即不限制
                .build();
        builder = NS.TableInfo.newBuilder()
                .setName(name)  // 设置表名
                //.setReplicaNum(3)    // 设置副本数. 此设置是可选的, 默认为3
                //.setPartitionNum(8)  // 设置分片数. 此设置是可选的, 默认为8
                //.setCompressType(NS.CompressType.kSnappy) // 设置数据压缩类型. 此设置是可选的默认为不压缩
                //.setStorageMode(Common.StorageMode.kMemory) // 设置表的存储模式, 默认为kMemory. 还可以设置为Common.StorageMode.kSSD和Common.StorageMode.kHDD
                .setTtlDesc(ttlDesc);      // 设置ttl. 如果ttl类型是kAbsoluteTime, 那么ttl的单位是分钟.
        // 设置schema信息
        ColumnDesc col0 = ColumnDesc.newBuilder()
                .setName("card")    // 设置字段名
                .setAddTsIdx(true)  // 设置是否为index, 如果设置为true表示该字段为维度列, 查询的时候可以通过此列来查询, 否则设置为false
                .setType("string")  // 设置字段类型, 支持的字段类型有[int32, uint32, int64, uint64, float, double, string]
                //.setIsTsCol(true)  // 设置是否为时间戳列
                .build();
        ColumnDesc col1 = ColumnDesc.newBuilder().setName("mcc").setAddTsIdx(true).setType("string").build();
        ColumnDesc col2 = ColumnDesc.newBuilder().setName("money").setAddTsIdx(false).setType("float").build();
        // 将schema添加到builder中
        builder.addColumnDescV1(col0).addColumnDescV1(col1).addColumnDescV1(col2);
        NS.TableInfo table = builder.build();
        // 可以通过返回值判断是否创建成功
        boolean ok = nsc.createTable(table);
        clusterClient.refreshRouteTable();
    }

    // 创建带有组合key的表
    public void createSchemaTable() {
        String name = "test3";
        NS.TableInfo.Builder builder = NS.TableInfo.newBuilder();
        Tablet.TTLDesc ttlDesc = Tablet.TTLDesc.newBuilder()
                .setTtlType(Tablet.TTLType.kAbsOrLat) // 设置ttl类型，共有四种kAbsoluteTime, kLatestTime, kAbsAndLat, kAbsOrLat, 如不设置此域默认为kAbsoluteTime
                .setAbsTtl(30) // 设置absolute ttl的值,单位为分钟，不设置默认为0，即不限制
                .setLatTtl(1) // 设置latest ttl的值，单位为条，不设置默认为0，即不限制
                .build();
        builder = NS.TableInfo.newBuilder()
                .setName(name)  // 设置表名
                //.setReplicaNum(3)    // 设置副本数. 此设置是可选的, 默认为3
                //.setPartitionNum(8)  // 设置分片数. 此设置是可选的, 默认为16
                //.setCompressType(NS.CompressType.kSnappy) // 设置数据压缩类型. 此设置是可选的默认为不压缩
                .setTtlDesc(ttlDesc);      // 设置ttldesc. 此设置是必须的
        // 设置schema信息
        ColumnDesc col0 = ColumnDesc.newBuilder().setName("card").setAddTsIdx(false).setType("string").build();
        ColumnDesc col1 = ColumnDesc.newBuilder().setName("mcc").setAddTsIdx(false).setType("string").build();
        ColumnDesc col2 = ColumnDesc.newBuilder().setName("amt").setAddTsIdx(false).setType("double").build();
        ColumnDesc col3 = ColumnDesc.newBuilder().setName("ts").setAddTsIdx(false).setType("int64").setIsTsCol(true).build();
        ColumnKey colKey1 = ColumnKey.newBuilder().setIndexName("card_mcc").addColName("card").addColName("mcc").addTsName("ts").build();
        // 将schema添加到builder中
        builder.addColumnDescV1(col0).addColumnDescV1(col1).addColumnDescV1(col2).addColumnDescV1(col3).addColumnKey(colKey1);
        NS.TableInfo table = builder.build();
        // 可以通过返回值判断是否创建成功
        boolean ok = nsc.createTable(table);
        clusterClient.refreshRouteTable();
    }

    // kv表put, scan, get
    public void syncKVTable() {
        String name = "test1";
        try {
            // 插入时指定的ts精确到毫秒
            long ts = System.currentTimeMillis();
            // 通过返回值可以判断是否插入成功
            boolean ret = tableSyncClient.put(name, "key1", ts, "value0");
            ret = tableSyncClient.put(name, "key1", ts + 1, "value1");
            ret = tableSyncClient.put(name, "key2", ts + 2, "value2");

            // scan数据, 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            KvIterator it = tableSyncClient.scan(name, "key1", ts + 1, 0);
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()]];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }

            // 可以通过limit限制最多返回的条数。如果不设置或设置为0，则不限制
            // 可以通过atleast忽略et的限制，限制至少返回的条数。如果不设置或设置为0，则不限制
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            int atleast = 1;
            ScanOption option = new ScanOption();
            option.setLimit(limit);
            option.setAtLeast(atleast);
            it = tableSyncClient.scan(name, "key1", ts + 1, 0, option);
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()]];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }

            // get数据, 查询指定ts的值. 如果ts设置为0, 返回最新插入的一条数据
            ByteString bs = tableSyncClient.get(name, "key1", ts);
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
            bs = tableSyncClient.get(name, "key1");
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (TabletException e) {
            e.printStackTrace();
        }
    }

    // schema表put, scan, get
    public void syncSchemaTable() {
        String name = "test2";
        try {
            // 插入数据. 插入时指定的ts精确到毫秒
            long ts = System.currentTimeMillis();
            // 通过map传入需要插入的数据, key是字段名, value是该字段对应的值
            Map<String, Object> row = new HashMap<String, Object>();
            row.put("card", "card0");
            row.put("mcc", "mcc0");
            row.put("money", 1.3f);
            // 通过返回值可以判断是否插入成功
            boolean ret = tableSyncClient.put(name, ts, row);
            row.clear();
            row.put("card", "card0");
            row.put("mcc", "mcc1");
            row.put("money", 15.8f);
            ret = tableSyncClient.put(name, ts + 1, row);

            // 也可以通过object数组方式插入
            // 数组顺序和创建表时schema顺序对应
            // 通过RTIDBClusterClient可以获取表schema信息
            // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
            Object[] arr = new Object[]{"card1", "mcc1", 9.15f};
            ret = tableSyncClient.put(name, ts + 2, arr);

            // scan数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            KvIterator it = tableSyncClient.scan(name, "card0", "card", ts + 1, 0);
            while (it.valid()) {
                // 解码出来是一个object数组, 顺序和创建表时的schema对应
                // 通过RTIDBClusterClient可以获取表schema信息
                // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }
            // 可以通过limit限制最多返回的条数。如果不设置或设置为0，则不限制
            // 可以通过atleast忽略et的限制，限制至少返回的条数。如果不设置或设置为0，则不限制
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            int atleast = 1;
            ScanOption option = new ScanOption();
            option.setLimit(limit);
            option.setAtLeast(atleast);
            option.setIdxName("card");
            it = tableSyncClient.scan(name, "card0", ts + 1, 0, option);
            while (it.valid()) {
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }

            // get数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询指定ts的值. 如果ts设置为0, 返回最新插入的一条数据

            // 解码出来是一个object数组, 顺序和创建表时的schema对应
            // 通过RTIDBClusterClient可以获取表schema信息
            // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
            Object[] result = tableSyncClient.getRow(name, "card0", "card", ts);
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
            result = tableSyncClient.getRow(name, "card0","card", 0);
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (TabletException e) {
            e.printStackTrace();
        }

    }

    // kv表异步put, 异步scan, 异步get
    public void AsyncKVTable() {
        String name = "test2";
        try {
            // 插入时指定的ts精确到毫秒
            long ts = System.currentTimeMillis();
            PutFuture pf1  = tableAsyncClient.put(name, "akey1", ts, "value4");
            PutFuture pf2 = tableAsyncClient.put(name, "akey1", ts + 1, "value5");
            PutFuture pf3 = tableAsyncClient.put(name, "akey2", ts + 2, "value6");
            // 调用get会阻塞到返回
            // 通过返回值可以判断是否插入成功
            boolean ret = pf1.get();
            ret = pf2.get();
            ret = pf3.get();

            // scan数据, 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            ScanFuture sf = tableAsyncClient.scan(name, "akey1", ts + 1, 0);
            KvIterator it = sf.get();
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()]];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }

            // 可以通过limit限制最多返回的条数。如果不设置或设置为0，则不限制
            // 可以通过atleast忽略et的限制，限制至少返回的条数。如果不设置或设置为0，则不限制
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            int atleast = 1;
            ScanOption option = new ScanOption();
            option.setLimit(limit);
            option.setAtLeast(atleast);
            sf = tableAsyncClient.scan(name, "akey1", ts + 1, 0, option);
            it = sf.get();
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()]];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }

            // get数据, 查询指定ts的值. 如果ts设置为0, 返回最新插入的一条数据
            GetFuture gf = tableAsyncClient.get(name, "akey1", ts);
            ByteString bs = gf.get();
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
            gf = tableAsyncClient.get(name, "akey1");
            bs = gf.get();
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
        } catch (TabletException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // schema表异步put, 异步scan, 异步get
    public void AsyncSchemaTable() {
        String name = "test2";
        try {
            // 插入数据. 插入时指定的ts精确到毫秒
            long ts = System.currentTimeMillis();
            // 通过map传入需要插入的数据, key是字段名, value是该字段对应的值
            Map<String, Object> row = new HashMap<String, Object>();
            row.put("card", "acard0");
            row.put("mcc", "amcc0");
            row.put("money", 1.3f);
            PutFuture pf1 = tableAsyncClient.put(name, ts, row);
            row.clear();
            row.put("card", "acard0");
            row.put("mcc", "amcc1");
            row.put("money", 15.8f);
            PutFuture pf2 = tableAsyncClient.put(name, ts + 1, row);

            // 也可以通过object数组方式插入
            // 数组顺序和创建表时schema顺序对应
            // 通过RTIDBClusterClient可以获取表schema信息
            // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
            Object[] arr = new Object[]{"acard1", "amcc1", 9.15f};
            PutFuture pf3 = tableAsyncClient.put(name, ts + 2, arr);
            // 调用get会阻塞到返回
            // 通过返回值可以判断是否插入成功
            boolean ret = pf1.get();
            ret = pf2.get();
            ret = pf3.get();

            // scan数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            ScanFuture sf = tableAsyncClient.scan(name, "acard0", "card", ts + 1, 0);
            KvIterator it = sf.get();
            while (it.valid()) {
                // 解码出来是一个object数组, 顺序和创建表时的schema对应
                // 通过RTIDBClusterClient可以获取表schema信息
                // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }
            // 可以通过limit限制最多返回的条数。如果不设置或设置为0，则不限制
            // 可以通过atleast忽略et的限制，限制至少返回的条数。如果不设置或设置为0，则不限制
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            int atleast = 1;
            ScanOption option = new ScanOption();
            option.setLimit(limit);
            option.setAtLeast(atleast);
            option.setIdxName("card");
            sf = tableAsyncClient.scan(name, "acard0", ts + 1, 0, limit);
            it = sf.get();
            while (it.valid()) {
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }

            // get数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询指定ts的值. 如果ts设置为0, 返回最新插入的一条数据

            // 解码出来是一个object数组, 顺序和创建表时的schema对应
            // 通过RTIDBClusterClient可以获取表schema信息
            // List<ColumnDesc> schema = clusterClient.getHandler(name).getSchema();
            GetFuture gf = tableAsyncClient.get(name, "acard0", "card", ts);
            Object[] result = gf.getRow();
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
            gf = tableAsyncClient.get(name, "acard0","card", 0);
            result = gf.getRow();
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
        } catch (TabletException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    // 组合key和指定ts列的put, scan, get
    try {
            String name = "test2";
            Map<String, Object> data = new HashMap<String, Object>();
            data.put("card", "card0");
            data.put("mcc", "mcc0");
            data.put("amt", 1.5);
            data.put("ts", 1234l);
            tableSyncClient.put(name, data);
            data.clear();
            data.put("card", "card0");
            data.put("mcc", "mcc1");
            data.put("amt", 1.6);
            data.put("ts", 1235l);
            tableSyncClient.put(name, data);
            Map<String, Object> scan_key = new HashMap<String, Object>();
            scan_key.put("card", "card0");
            scan_key.put("mcc", "mcc0");
            KvIterator it = tableSyncClient.scan(name, scan_key, "card_mcc", 1235l, 0l, "ts", 0);
            Object[] row = it.getDecodedValue();
            scan_key.put("mcc", "mcc1");
            it = tableSyncClient.scan(name, scan_key, "card_mcc", 1235l, 0l, "ts", 0);
            scan_key.put("mcc", "mcc2");
            it = tableSyncClient.scan(name, scan_key, "card_mcc", 1235l, 0l, "ts", 0);
            row = tableSyncClient.getRow(name, new Object[] {"card0", "mcc0"}, "card_mcc", 1234, "ts", null);

            Map<String, Object> key_map = new HashMap<String, Object>();
            key_map.put("card", "card0");
            key_map.put("mcc", "mcc1");
            row = tableSyncClient.getRow(name, key_map, "card_mcc", 1235, "ts", null);

            data.clear();
            data.put("card", "card0");
            data.put("mcc", "mcc1");
            data.put("amt", 1.6);
            data.put("ts", 1240l);
            tableSyncClient.put(name, data);

            data.clear();
            data.put("card", "card0");
            data.put("mcc", "mcc1");
            data.put("amt", 1.7);
            data.put("ts", 1245l);
            PutFuture pf = tableAsyncClient.put(name, data);
            pf.get();

            ScanFuture sf = tableAsyncClient.scan(name, key_map, "card_mcc", 1245, 0, "ts", 0);
            it = sf.get();
            row = it.getDecodedValue();

            GetFuture gf = tableAsyncClient.get(name, key_map, "card_mcc", 1235, "ts", null);
            row = gf.getRow();

            data.clear();
            data.put("card", "card0");
            data.put("mcc", "");
            data.put("amt", 1.8);
            data.put("ts", 1250l);
            tableSyncClient.put(name, data);
            row = tableSyncClient.getRow(name, new Object[] {"card0", ""}, "card_mcc", 0, "ts", null);
        } catch (Exception e) {
            e.printStackTrace();
        }

    // 遍历全表
    // 建议将removeDuplicateByTime设为true, 避免同一个key下相同ts比较多时引起client死循环
    // 建议将重试次数MaxRetryCnt往大设下, 默认值为1.
    public void Traverse() {
        String name = "test2";
        try {
            for (int i = 0; i < 1000; i++) {
                Map<String, Object> rowMap = new HashMap<String, Object>();
                rowMap.put("card", "card" + (i + 9529));
                rowMap.put("mcc", "mcc" + i);
                rowMap.put("amt", 9.2d);
                boolean ok = tableSyncClient.put(name, i + 9529, rowMap);
            }
            KvIterator it = tableSyncClient.traverse(name, "card");
            while (it.valid()) {
                // 一次迭代只能调用一次getDecodedValue
                Object[] row = it.getDecodedValue();
                System.out.println(row[0] + " " + row[1] + " " + row[2]);
                it.next();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    // relation table
    private static String createRelationalTable(String name) {
        nsc.dropTable(name);
        TableDesc tableDesc = new TableDesc();
        tableDesc.setName(name);
        tableDesc.setTableType(TableType.kRelational);
        List<com._4paradigm.rtidb.client.schema.ColumnDesc> list = new ArrayList<>();
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("id");
            col.setDataType(DataType.BigInt);
            col.setNotNull(true);
            list.add(col);
        }
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("attribute");
            col.setDataType(DataType.Varchar);
            col.setNotNull(true);
            list.add(col);
        }
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("image");
            col.setDataType(DataType.Blob);
            col.setNotNull(false);
            list.add(col);
        }
        tableDesc.setColumnDescList(list);
 
        List<IndexDef> indexs = new ArrayList<>();
        IndexDef indexDef = new IndexDef();
        indexDef.setIndexName("idx1");
        indexDef.setIndexType(IndexType.PrimaryKey);
        List<String> colNameList = new ArrayList<>();
        colNameList.add("id");
        indexDef.setColNameList(colNameList);
        indexs.add(indexDef);
 
        tableDesc.setIndexs(indexs);
        boolean ok = nsc.createTable(tableDesc);
        System.out.println(ok);
        rtidbClusterClient.refreshRouteTable();
 
        return name;
    }
 
    public static void relatioTableOperator() {
        init();
        //表提前在cmd客户端使用上面的建表文件create
        String name = "rtidb_test";
        createRelationalTable(name);
        try {
            List<ColumnDesc> schema = tableSyncClient.getSchema(name);
            System.out.println(schema.size());  // 输出 3
 
            /**
             *  blob数据需要先转换为ByteBuffer
             */﻿﻿
            File afile = new File("/Users/innerpeace/demo/src/main/resources/tmp.jpg");
            long fsize = afile.length();
            FileInputStream inFile = new FileInputStream(afile);
            FileChannel inChannel = inFile.getChannel();
            ByteBuffer buf1 = ByteBuffer.allocate((int) fsize);
            int ret = inChannel.read(buf1);
            System.out.println(ret);
            System.out.println(-1 != ret);
            /*
             *  put 默认覆盖
             */
            WriteOption wo = new WriteOption();
            Map<String, Object> data = new HashMap<String, Object>();
            data.put("id", 11l);
            data.put("attribute", "a1");
//            String imageData1 = "i1";
//            ByteBuffer buf1 = StringToBB(imageData1);
            data.put("image", buf1);
            System.out.println(tableSyncClient.put(name, data, wo).isSuccess());// true
 
            data.clear();
            data.put("id", 12l);
            data.put("attribute", "a2");
            String imageData2 = "i1";
            ByteBuffer buf2 = StringToBB(imageData2);
            data.put("image", buf2);
            System.out.println(tableSyncClient.put(name, data, wo).isSuccess()); //true
 
            ReadOption ro;
            RelationalIterator it;
            Map<String, Object> queryMap;
            /*
             * traverse
             */
            ro = new ReadOption(null, null, null, 1);
            it = tableSyncClient.traverse(name, ro);
 
            System.out.println(it.valid());
            queryMap = it.getDecodedValue();
            System.out.println(queryMap.size() == 3);
            System.out.println(queryMap.get("id").equals(11l));
            System.out.println(queryMap.get("attribute").equals("a1"));
            BlobData bd = (BlobData) queryMap.get("image");
            System.out.println(buf1.equals(bd.getData()));
            // get url suffix method: getUrlMap(columnName)
            System.out.println("blob url is: " + bd.getUrl());
 
            it.next();
            queryMap = it.getDecodedValue();
            System.out.println(queryMap.size() == 3);
            System.out.println(queryMap.get("id").equals(12l));
            System.out.println(queryMap.get("attribute").equals("a2"));
            System.out.println((buf2.equals(queryMap.get("image"))));
 
            it.next();
            System.out.println(!it.valid());
            /*
             * query
             */
            {
                Map<String, Object> index = new HashMap<>();
                index.put("id", 11l);
                Set<String> colSet = new HashSet<>();
                colSet.add("id");
                colSet.add("image");
                ro = new ReadOption(index, null, colSet, 0);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid()); // true
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 2); // true
                System.out.println(queryMap.get("id").equals(11l)); // true
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
            }
            {
                Map<String, Object> index2 = new HashMap<>();
                index2.put("id", 12l);
                ro = new ReadOption(index2, null, null, 0);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid()); // true
 
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3); //true
                System.out.println(queryMap.get("id").equals(12l)); // true
                System.out.println(queryMap.get("attribute").equals("a2")); // true
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
            }
            /*
             * batchQuery
             */
            {
                List<ReadOption> ros = new ArrayList<ReadOption>();
                {
                    Map<String, Object> index = new HashMap<>();
                    index.put("id", 12l);
                    ro = new ReadOption(index, null, null, 0);
                    ros.add(ro);
                }
                {
                    Map<String, Object> index2 = new HashMap<>();
                    index2.put("id", 11l);
                    ro = new ReadOption(index2, null, null, 0);
                    ros.add(ro);
                }
                it = tableSyncClient.batchQuery(name, ros);
                System.out.println(it.getCount() == 2);
 
                System.out.println(it.valid());
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(12l));
                System.out.println(queryMap.get("attribute").equals("a2"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf2.equals(bd.getData()));
 
                it.next();
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(11l));
                System.out.println(queryMap.get("attribute").equals("a1"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
 
                it.next();
                System.out.println(!it.valid());
            }
 
            /*
             * update
             */
            boolean ok = false;
            String imageData3 = "i3";
            UpdateResult updateResult;
            ByteBuffer buf3 = StringToBB(imageData3);
            {
                Map<String, Object> conditionColumns = new HashMap<>();
                conditionColumns.put("id", 11l);
                Map<String, Object> valueColumns = new HashMap<>();
                valueColumns.put("attribute", "a3");
 
                valueColumns.put("image", buf3);
                updateResult = tableSyncClient.update(name, conditionColumns, valueColumns, wo);
                System.out.println(updateResult.isSuccess());
                System.out.println(updateResult.getAffectedCount());
 
                Map<String, Object> index = new HashMap<>();
                index.put("id", 11l);
                ro = new ReadOption(index, null, null, 1);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid());
 
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(11l));
                System.out.println(queryMap.get("attribute").equals("a3"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf3.equals(bd.getData()));
            }
            /*
             * delete
             */
            Map<String, Object> index2 = new HashMap<>();
            index2.put("id", 11l);
            ro = new ReadOption(index2, null, null, 0);
            updateResult = tableSyncClient.delete(name, index2);
            System.out.println(updateResult.isSuccess());
            System.out.println(updateResult.getAffectedCount());
            it = tableSyncClient.query(name, ro);
            System.out.println(!it.valid());
            /*
             * duplicate primary key put
             */
            WriteOption woDup = new WriteOption(true);
            data.clear();
            data.put("id", 12l);
            data.put("attribute", "a2");
            String imageData2 = "i1";
            ByteBuffer buf2 = StringToBB(imageData2);
            data.put("image", buf2);
            System.out.println(tableSyncClient.put(name, data, woDup).isSuccess()); //true
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    private static ByteBuffer StringToBB(String ss) {
        ByteBuffer buf = ByteBuffer.allocate(ss.getBytes().length);
        for (int k = 0; k < ss.getBytes().length; k++) {
            buf.put(ss.getBytes()[k]);
        }
        buf.rewind();
        return buf;
    }
}

```

# 单机模式 {#single}

```java
package example;

import com._4paradigm.rtidb.client.GetFuture;
import com._4paradigm.rtidb.client.PutFuture;
import com._4paradigm.rtidb.client.ScanFuture;
import com._4paradigm.rtidb.client.KvIterator;
import com._4paradigm.rtidb.client.TabletException;
import com._4paradigm.rtidb.client.ha.RTIDBClientConfig;
import com._4paradigm.rtidb.client.ha.impl.RTIDBSingleNodeClient;
import com._4paradigm.rtidb.client.impl.TableAsyncClientImpl;
import com._4paradigm.rtidb.client.impl.TableSyncClientImpl;
import com._4paradigm.rtidb.client.impl.TabletClientImpl;
import com._4paradigm.rtidb.client.schema.ColumnDesc;
import com._4paradigm.rtidb.client.schema.ColumnType;
import com._4paradigm.rtidb.tablet.Tablet;
import com.google.protobuf.ByteString;
import io.brpc.client.EndPoint;

import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SingleModeExample {
    private static EndPoint endpoint = new EndPoint("172.27.128.31:8541"); // 配置节点endpoint
    private static RTIDBClientConfig config = new RTIDBClientConfig();
    private static RTIDBSingleNodeClient snc = new RTIDBSingleNodeClient(config, endpoint);
    private static TabletClientImpl tabletClient = null;
    // 同步client
    private static TableSyncClientImpl tableClient = null;
    // 异步client
    private static TableAsyncClientImpl tableAsyncClient = null;
    static {
        try {
            snc.init();
        } catch (Exception e) {
            e.printStackTrace();
        }
        tableClient = new TableSyncClientImpl(snc);
        tableAsyncClient = new TableAsyncClientImpl(snc);
        tabletClient = new TabletClientImpl(snc);
    }

    public void createKVTable() {
        int tid = 10;
        int pid = 0; // pid设置为0, 不要改成其他值
        // tid和pid唯一确定一张表, 创建表时不能和已有表重复
        int segCnt = 8; // 一般指定为8
        // 通过返回值可以查看是否创建成功
        // 创建absolute表, 过期时间设置为144000分钟. 如果过期时间设置为0表示不过期
        boolean ret = tabletClient.createTable("test1", tid, 0, 144000, segCnt);
        // 创建latest表, 保留最近1条记录
        // latest表不能scan
        ret = tabletClient.createTable("test2", tid + 1, pid, 1, Tablet.TTLType.kLatestTime, segCnt);
    }

    public void createSchemaTable() {
        int tid = 20;
        int pid = 0; // pid设置为0, 不要改成其他值
        // tid和pid唯一确定一张表, 创建表时不能和已有表重复
        int segCnt = 8; // 一般指定为8

        // 设置表schema
        List<ColumnDesc> schema = new ArrayList<ColumnDesc>();
        ColumnDesc desc1 = new ColumnDesc();
        desc1.setName("card");      // 设置字段名
        desc1.setAddTsIndex(true);  // 设置是否为索引列, 如果设置为true可以通过此列来查询数据
        desc1.setType(ColumnType.kString); // 设置类型
        schema.add(desc1);
        ColumnDesc desc2 = new ColumnDesc();
        desc2.setName("mcc");
        desc2.setAddTsIndex(true);
        desc2.setType(ColumnType.kString);
        schema.add(desc2);
        ColumnDesc desc3 = new ColumnDesc();
        desc3.setName("money");
        desc3.setAddTsIndex(false);
        desc3.setType(ColumnType.kFloat);
        schema.add(desc3);
        // 通过返回值可以查看是否创建成功
        // 默认创建的是absolute表, 如果过期时间设置为0表示不过期
        boolean ret = tabletClient.createTable("test3", tid, pid, 144000, segCnt, schema);
        // 创建latest表，保留最近10条记录
        // latest表不能scan
        ret = tabletClient.createTable("test4", tid + 1, pid, 10, Tablet.TTLType.kLatestTime, segCnt, schema);
    }

    // kv表put, scan, get
    public void syncKVTable() {
        int tid = 10;
        int pid = 0;
        long ts = System.currentTimeMillis();
        try {
            // 可以通过返回值来判断是否put成功
            boolean ret = tableClient.put(tid, pid, "key1", ts, "value0");
            ret = tableClient.put(tid, pid, "key1", ts + 1, "value1");
            ret = tableClient.put(tid, pid, "key2", ts + 2, "value3");

            // scan数据
            // 需要传入st和et, 分别表示起始时间和结束时间, 其中起始时间大于结束时间. 返回起始时间和结束时间之间的数据
            // 如果结束时间et设置为0, 则返回从起始时间以来所有没有过期的数据
            KvIterator it = tableClient.scan(tid, pid, "key1", ts + 1, 0);
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println("scan-" + value);
                it.next();
            }

            // 可以通过limit限制最多返回的条数。如果不设置或设置为0，则不限制
            // 如果st和et都设置为0则返回最近N条记录
            it = tableClient.scan(tid, pid, "key1", ts + 1, 0, 1);
            while (it.valid()) {
                byte[] buffer = new byte[it.getValue().remaining()];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println("scan1-" + value);
                it.next();
            }

            // get数据
            ByteString buffer = tableClient.get(tid, 0, "key1", ts);
            // 如果没有查到数据返回null
            if (buffer != null) {
                System.out.println("get" + buffer.toString(Charset.forName("utf-8")));
            }
            // ts设置为0或者不设置返回最新插入的数据
            buffer = tableClient.get(tid, 0, "key1", 0);
            // buffer = tableClient.get(tid, 0, "key1");
            if (buffer != null) {
                System.out.println("get1" + buffer.toString(Charset.forName("utf-8")));
            }
        } catch (TabletException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // schema表put, scan, get
    public void syncSchemaTable() {
        int tid = 20;
        int pid = 0;
        long ts = System.currentTimeMillis();
        try {
            // 通过map传入需要插入的数据, key是字段名, value是该字段对应的值
            Map<String, Object> row = new HashMap<String, Object>();
            row.put("card", "card0");
            row.put("mcc", "mcc0");
            row.put("money", 1.3f);
            // 可以通过返回值来判断是否put成功
            boolean ret = tableClient.put(tid, pid, ts, row);
            row.clear();
            row.put("card", "card0");
            row.put("mcc", "mcc1");
            row.put("money", 15.8f);
            ret = tableClient.put(tid, pid, ts + 1, row);
            // 也可以通过object数组方式插入
            // 数组顺序和创建表时schema顺序对应
            // 通过RTIDBSingleNodeClient可以获取表schema信息
            // List<ColumnDesc> schema = snc.getHandler(tid).getSchema();
            Object[] arr = new Object[]{"card1", "mcc1", 9.15f};
            ret = tableClient.put(tid, pid, ts + 2, arr);

            // scan数据
            // 需要传入st和et, 分别表示起始时间和结束时间, 其中起始时间大于结束时间. 返回起始时间和结束时间之间的数据
            // 如果结束时间et设置为0, 则返回从起始时间以来所有没有过期的数据
            KvIterator it = tableClient.scan(tid, pid, "card0","card", ts + 1, 0);
            while (it.valid()) {
                // 解码出来是一个object数组, 顺序和创建表时的schema对应
                // 通过RTIDBSingleNodeClient可以获取表schema信息
                // List<ColumnDesc> schema = snc.getHandler(tid).getSchema();
                Object[] result = it.getDecodedValue();
                System.out.println("scan-" + result[0]);
                System.out.println("scan-" + result[1]);
                System.out.println("scan-" + result[2]);
                it.next();
            }

            // 可以通过limit限制返回的记录条数
            // 如果st和et都设置为0则返回最近N条记录
            it = tableClient.scan(tid, pid, "card0","card", ts + 1, 0, 1);
            while (it.valid()) {
                Object[] result = it.getDecodedValue();
                System.out.println("scan1-" + result[0]);
                System.out.println("scan1-" + result[1]);
                System.out.println("scan1-" + result[2]);
                it.next();
            }

            // get数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询指定ts的值. 如果不设置ts或者ts设置为0, 返回最新插入的一条数据

            // 解码出来是一个object数组, 顺序和创建表时的schema对应
            // 通过RTIDBSingleNodeClient可以获取表schema信息
            // List<ColumnDesc> schema = clusterClient.getHandler(tid).getSchema();
            Object[] result = tableClient.getRow(tid, pid, "card0", "card", ts);
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println("get-" + result[idx]);
            }
            result = tableClient.getRow(tid, pid, "card0","card", 0);
            // result = tableClient.getRow(tid, pid, "card0","card");
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println("get1-" + result[idx]);
            }
        } catch (TabletException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // kv表异步put, 异步scan, 异步get
    public void aSyncKVTable() {
        int tid = 10;
        int pid = 0;
        long ts = System.currentTimeMillis();
        try {
            PutFuture pf1 = tableAsyncClient.put(tid, pid, "akey1", ts, "value0");
            PutFuture pf2 = tableAsyncClient.put(tid, pid, "akey1", ts + 1, "value1");
            PutFuture pf3 = tableAsyncClient.put(tid, pid, "akey2", ts + 2, "value3");
            // 调用get会阻塞到返回
            // 通过返回值可以判断是否插入成功
            boolean ret = pf1.get();
            ret = pf2.get();
            ret = pf3.get();

            // scan数据, 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            ScanFuture sf = tableAsyncClient.scan(tid, pid, "akey1", ts + 1, 0);
            KvIterator it = sf.get();
            while (it.valid()) {
                byte[] buffer = new byte[6];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }
            // 可以通过limit限制返回的记录条数
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            sf = tableAsyncClient.scan(tid, pid, "akey1", ts + 1, 0, limit);
            it = sf.get();
            while (it.valid()) {
                byte[] buffer = new byte[6];
                it.getValue().get(buffer);
                String value = new String(buffer);
                System.out.println(value);
                it.next();
            }

            // get数据
            // 查询指定ts的值. 如果不设置ts或者设置为0, 返回最新插入的一条数据
            GetFuture gf = tableAsyncClient.get(tid, pid, "akey1", ts);
            ByteString bs = gf.get();
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
            gf = tableAsyncClient.get(tid, pid, "akey1");
            // 如果不设置ts和设置为0是一样的
            // gf = tableAsyncClient.get(tid, pid, "akey1", 0);

            bs = gf.get();
            // 如果没有查询到bs就是null
            if (bs != null) {
                String value = new String(bs.toByteArray());
                System.out.println(value);
            }
        } catch (TabletException e) {
            e.printStackTrace();
        }  catch (Exception e) {
            e.printStackTrace();
        }

    }

    // schema表异步put, 异步scan, 异步get
    public void aSyncSchemaTable() {
        int tid = 20;
        int pid = 0;
        long ts = System.currentTimeMillis();
        try {
            // 通过map传入需要插入的数据, key是字段名, value是该字段对应的值
            Map<String, Object> row = new HashMap<String, Object>();
            row.put("card", "acard0");
            row.put("mcc", "amcc0");
            row.put("money", 1.3f);
            PutFuture pf1 = tableAsyncClient.put(tid, pid, ts, row);
            row.clear();
            row.put("card", "acard0");
            row.put("mcc", "amcc1");
            row.put("money", 15.8f);
            PutFuture pf2 = tableAsyncClient.put(tid, pid, ts + 1, row);

            // 也可以通过object数组方式插入
            // 数组顺序和创建表时schema顺序对应
            // 通过RTIDBSingleNodeClient可以获取表schema信息
            // List<ColumnDesc> schema = snc.getHandler(tid).getSchema();
            Object[] arr = new Object[]{"acard1", "amcc1", 9.15f};
            PutFuture pf3 = tableAsyncClient.put(tid, pid, ts + 2, arr);
            // 调用get会阻塞到返回
            // 通过返回值可以判断是否插入成功
            boolean ret = pf1.get();
            ret = pf2.get();
            ret = pf3.get();

            // scan数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询范围需要传入st和et分别表示起始时间和结束时间, 其中起始时间大于结束时间
            // 如果结束时间et设置为0, 返回起始时间之前的所有数据
            ScanFuture sf = tableAsyncClient.scan(tid, pid, "acard0", "card", ts + 1, 0);
            KvIterator it = sf.get();
            while (it.valid()) {
                // 解码出来是一个object数组, 顺序和创建表时的schema对应
                // 通过RTIDBSingleNodeClient可以获取表schema信息
                // List<ColumnDesc> schema = snc.getHandler(tid).getSchema();
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }
            // 可以通过limit限制返回的记录条数
            // 如果st和et都设置为0则返回最近N条记录
            int limit = 1;
            sf = tableAsyncClient.scan(tid, pid, "acard0", "card", ts + 1, 0, limit);
            it = sf.get();
            while (it.valid()) {
                Object[] result = it.getDecodedValue();
                System.out.println(result[0]);
                System.out.println(result[1]);
                System.out.println(result[2]);
                it.next();
            }

            // get数据
            // key是需要查询字段的值, idxName是需要查询的字段名
            // 查询指定ts的值. 如果不设置ts或者设置为0, 返回最新插入的一条数据

            // 解码出来是一个object数组, 顺序和创建表时的schema对应
            // 通过RTIDBSingleNodeClient可以获取表schema信息
            // List<ColumnDesc> schema = snc.getHandler(tid).getSchema();
            GetFuture gf = tableAsyncClient.get(tid, pid, "acard0", "card", ts);
            Object[] result = gf.getRow();
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
            gf = tableAsyncClient.get(tid, pid, "acard0","card");
            // 如果不设置ts和设置为0一样的
            // gf = tableAsyncClient.get(tid, pid, "acard0","card", 0);
            result = gf.getRow();
            for (int idx = 0; idx < result.length; idx++) {
                System.out.println(result[idx]);
            }
        } catch (TabletException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void testAddTableFiled() {
        String name = "test";
        PartitionMeta pm0_0 = PartitionMeta.newBuilder().setEndpoint(nodes[0]).setIsLeader(true).build();
        PartitionMeta pm0_1 = PartitionMeta.newBuilder().setEndpoint(nodes[1]).setIsLeader(false).build();
        ColumnDesc col0 = ColumnDesc.newBuilder().setName("card").setAddTsIdx(true).setType("string").build();
        ColumnDesc col1 = ColumnDesc.newBuilder().setName("mcc").setAddTsIdx(true).setType("string").build();
        ColumnDesc col2 = ColumnDesc.newBuilder().setName("amt").setAddTsIdx(false).setType("double").build();
        TablePartition tp0 = TablePartition.newBuilder().addPartitionMeta(pm0_0).addPartitionMeta(pm0_1).setPid(0).build();
        TablePartition tp1 = TablePartition.newBuilder().addPartitionMeta(pm0_0).addPartitionMeta(pm0_1).setPid(1).build();
        TableInfo table = TableInfo.newBuilder().addTablePartition(tp0).addTablePartition(tp1)
            .setSegCnt(8).setName(name).setTtl(0)
            .addColumnDesc(col0).addColumnDesc(col1).addColumnDesc(col2)
            .build();    // relation table
    private static String createRelationalTable(String name) {
        nsc.dropTable(name);
        TableDesc tableDesc = new TableDesc();
        tableDesc.setName(name);
        tableDesc.setTableType(TableType.kRelational);
        List<com._4paradigm.rtidb.client.schema.ColumnDesc> list = new ArrayList<>();
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("id");
            col.setDataType(DataType.BigInt);
            col.setNotNull(true);
            list.add(col);
        }
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("attribute");
            col.setDataType(DataType.Varchar);
            col.setNotNull(true);
            list.add(col);
        }
        {
            com._4paradigm.rtidb.client.schema.ColumnDesc col = new com._4paradigm.rtidb.client.schema.ColumnDesc();
            col.setName("image");
            col.setDataType(DataType.Blob);
            col.setNotNull(false);
            list.add(col);
        }
        tableDesc.setColumnDescList(list);
 
        List<IndexDef> indexs = new ArrayList<>();
        IndexDef indexDef = new IndexDef();
        indexDef.setIndexName("idx1");
        indexDef.setIndexType(IndexType.PrimaryKey);
        List<String> colNameList = new ArrayList<>();
        colNameList.add("id");
        indexDef.setColNameList(colNameList);
        indexs.add(indexDef);
 
        tableDesc.setIndexs(indexs);
        boolean ok = nsc.createTable(tableDesc);
        System.out.println(ok);
        rtidbClusterClient.refreshRouteTable();
 
        return name;
    }
 
    public static void test() {
        init();
        //表提前在cmd客户端使用上面的建表文件create
        String name = "rtidb_test";
        createRelationalTable(name);
        try {
            List<ColumnDesc> schema = tableSyncClient.getSchema(name);
            System.out.println(schema.size());  // 输出 3
 
            /**
             *  blob数据需要先转换为ByteBuffer
             */﻿﻿
            File afile = new File("/Users/innerpeace/demo/src/main/resources/tmp.jpg");
            long fsize = afile.length();
            FileInputStream inFile = new FileInputStream(afile);
            FileChannel inChannel = inFile.getChannel();
            ByteBuffer buf1 = ByteBuffer.allocate((int) fsize);
            int ret = inChannel.read(buf1);
            System.out.println(ret);
            System.out.println(-1 != ret);
            /*
             *  put 默认覆盖
             */
            WriteOption wo = new WriteOption();
            Map<String, Object> data = new HashMap<String, Object>();
            data.put("id", 11l);
            data.put("attribute", "a1");
//            String imageData1 = "i1";
//            ByteBuffer buf1 = StringToBB(imageData1);
            data.put("image", buf1);
            System.out.println(tableSyncClient.put(name, data, wo).isSuccess());// true
 
            data.clear();
            data.put("id", 12l);
            data.put("attribute", "a2");
            String imageData2 = "i1";
            ByteBuffer buf2 = StringToBB(imageData2);
            data.put("image", buf2);
            System.out.println(tableSyncClient.put(name, data, wo).isSuccess()); //true
 
            ReadOption ro;
            RelationalIterator it;
            Map<String, Object> queryMap;
            /*
             * traverse
             */
            ro = new ReadOption(null, null, null, 1);
            it = tableSyncClient.traverse(name, ro);
 
            System.out.println(it.valid());
            queryMap = it.getDecodedValue();
            System.out.println(queryMap.size() == 3);
            System.out.println(queryMap.get("id").equals(11l));
            System.out.println(queryMap.get("attribute").equals("a1"));
            BlobData bd = (BlobData) queryMap.get("image");
            System.out.println(buf1.equals(bd.getData()));
            // get url suffix method: getUrlMap(columnName)
            System.out.println("blob url is: " + bd.getUrl());
 
            it.next();
            queryMap = it.getDecodedValue();
            System.out.println(queryMap.size() == 3);
            System.out.println(queryMap.get("id").equals(12l));
            System.out.println(queryMap.get("attribute").equals("a2"));
            System.out.println((buf2.equals(queryMap.get("image"))));
 
            it.next();
            System.out.println(!it.valid());
            /*
             * query
             */
            {
                Map<String, Object> index = new HashMap<>();
                index.put("id", 11l);
                Set<String> colSet = new HashSet<>();
                colSet.add("id");
                colSet.add("image");
                ro = new ReadOption(index, null, colSet, 0);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid()); // true
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 2); // true
                System.out.println(queryMap.get("id").equals(11l)); // true
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
            }
            {
                Map<String, Object> index2 = new HashMap<>();
                index2.put("id", 12l);
                ro = new ReadOption(index2, null, null, 0);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid()); // true
 
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3); //true
                System.out.println(queryMap.get("id").equals(12l)); // true
                System.out.println(queryMap.get("attribute").equals("a2")); // true
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
            }
            /*
             * batchQuery
             */
            {
                List<ReadOption> ros = new ArrayList<ReadOption>();
                {
                    Map<String, Object> index = new HashMap<>();
                    index.put("id", 12l);
                    ro = new ReadOption(index, null, null, 0);
                    ros.add(ro);
                }
                {
                    Map<String, Object> index2 = new HashMap<>();
                    index2.put("id", 11l);
                    ro = new ReadOption(index2, null, null, 0);
                    ros.add(ro);
                }
                it = tableSyncClient.batchQuery(name, ros);
                System.out.println(it.getCount() == 2);
 
                System.out.println(it.valid());
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(12l));
                System.out.println(queryMap.get("attribute").equals("a2"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf2.equals(bd.getData()));
 
                it.next();
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(11l));
                System.out.println(queryMap.get("attribute").equals("a1"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf1.equals(bd.getData()));
 
                it.next();
                System.out.println(!it.valid());
            }
 
            /*
             * update
             */
            boolean ok = false;
            String imageData3 = "i3";
            UpdateResult updateResult;
            ByteBuffer buf3 = StringToBB(imageData3);
            {
                Map<String, Object> conditionColumns = new HashMap<>();
                conditionColumns.put("id", 11l);
                Map<String, Object> valueColumns = new HashMap<>();
                valueColumns.put("attribute", "a3");
 
                valueColumns.put("image", buf3);
                updateResult = tableSyncClient.update(name, conditionColumns, valueColumns, wo);
                System.out.println(updateResult.isSuccess());
                System.out.println(updateResult.getAffectedCount());
 
                Map<String, Object> index = new HashMap<>();
                index.put("id", 11l);
                ro = new ReadOption(index, null, null, 1);
                it = tableSyncClient.query(name, ro);
                System.out.println(it.valid());
 
                queryMap = it.getDecodedValue();
                System.out.println(queryMap.size() == 3);
                System.out.println(queryMap.get("id").equals(11l));
                System.out.println(queryMap.get("attribute").equals("a3"));
                bd = (BlobData) queryMap.get("image");
                System.out.println(buf3.equals(bd.getData()));
            }
            /*
             * delete
             */
            Map<String, Object> index2 = new HashMap<>();
            index2.put("id", 11l);
            ro = new ReadOption(index2, null, null, 0);
            updateResult = tableSyncClient.delete(name, index2);
            System.out.println(updateResult.isSuccess());
            System.out.println(updateResult.getAffectedCount());
            it = tableSyncClient.query(name, ro);
            System.out.println(!it.valid());
            /*
             * duplicate primary key put
             */
            WriteOption woDup = new WriteOption(true);
            data.clear();
            data.put("id", 12l);
            data.put("attribute", "a2");
            String imageData2 = "i1";
            ByteBuffer buf2 = StringToBB(imageData2);
            data.put("image", buf2);
            System.out.println(tableSyncClient.put(name, data, woDup).isSuccess()); //true
            
        } catch (Exception e) {
            e.printStackTrace();
        }
        try {
            if (nsc != null)
                nsc.close();
            if (rtidbClusterClient != null)
                rtidbClusterClient.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    private static ByteBuffer StringToBB(String ss) {
        ByteBuffer buf = ByteBuffer.allocate(ss.getBytes().length);
        for (int k = 0; k < ss.getBytes().length; k++) {
            buf.put(ss.getBytes()[k]);
        }
        buf.rewind();
        return buf;
    }
        boolean create = nsc.createTable(table);
        client.refreshRouteTable();
        System.out.println("create table : " + create);
        try {
            List<com._4paradigm.rtidb.client.schema.ColumnDesc> schema = tableSyncClient.getSchema(name);
            System.out.println("scheme size ：" + schema.size());
            boolean put = tableSyncClient.put(name, 9527, new Object[]{"card0", "mcc0", 9.15d});

            boolean ok = nsc.addTableField(name, "aa", "string");
            System.out.println("add table field : " + ok);
            client.refreshRouteTable();

            List<com._4paradigm.rtidb.client.schema.ColumnDesc> schemaNew = tableSyncClient.getSchema(name);
            System.out.println("schemeNew size ：" + schemaNew.size());
            boolean put1 = tableSyncClient.put(name, 9528, new Object[]{"card1", "mcc1", 9.2d, "aa1"});
            boolean put2 = tableSyncClient.put(name, 9529, new Object[]{"card2", "mcc2", 9.3d});

            Object[] row = tableSyncClient.getRow(name, "card0", 9527);
            System.out.println("card : " + row[0]);
            System.out.println("mcc : " + row[1]);
            System.out.println("amt : " + row[2]);
            System.out.println("aa : " + row[3]);
            System.out.println();

            row = tableSyncClient.getRow(name, "card1", 9528);
            System.out.println("card : " + row[0]);
            System.out.println("mcc : " + row[1]);
            System.out.println("amt : " + row[2]);
            System.out.println("aa : " + row[3]);
            System.out.println();

            row = tableSyncClient.getRow(name, "card2", 9529);
            System.out.println("card : " + row[0]);
            System.out.println("mcc : " + row[1]);
            System.out.println("amt : " + row[2]);
            System.out.println("aa : " + row[3]);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
      
}
```
