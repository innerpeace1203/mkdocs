# python sdk 文档

## pip 安装

```
pip3 install -U -i http://pypi.4paradigm.com/4paradigm/dev --trusted-host=pypi.4paradigm.com rtidb==1.6.3.0

```

## 使用demo

python client 目前只支持集群模式

集群模式必须要有zk,tablet,nameserver 这三个角色

 如果有二进制数据(图片 mp3等)还需要blob 和 blob_proxy 这两个角色

schema 如下

```bash
name : "test2"
table_type : "Relational"
column_desc {
      name : "id"
        type : "bigint"
        not_null : true
}
column_desc {
      name : "filename"
        type : "varchar"
        not_null : true
}
column_desc {
      name : "content"
        type : "blob"
        not_null : false
}
index {
       index_name : "idx1"
       col_name : "id"
       index_type : "PrimaryKey"
}
```



```python
import rtidb
import os
try:
  nsc = rtidb.RTIDBClient("172.27.128.81:3181", "/rtidb_cluster") #初始化集群，初始化参数分别是zk server 地址(多个zk server 使用英文,隔开 ) zk 路径, 初始化成功后会返回一个rtidb client
except Exception as e:
  print(e)
  exit(1)

table="test2" # python client 不支持建表和查表功能，需要实现通过命令行或者java client 建表

# put
id = 1
for fileName in os.listdir("images"):
  with open("images/{}".format(fileName), "rb") as f:
    binaryData = f.read()
  data = {"id": id, "filename": fileName, "content": binaryData} #构造数据
  putResult = nsc.put(table, data, None) #put 数据, 第一个参数是表名，第二个参数是data，第三个参数是readoption,当前未开发传None即可, put 函数调用后，返回一个 put result
  if not putResult.success(): # 通过调用success() 判断是否put 成功
    print("put %s error" % fileName);
    exit(1)
  id += 1
  #如果索引是auto gen 类型，可以通过 putResult.get_auto_gen_pk() 获取 自动生成的key

# query
ro = rtidb.ReadOption() #创建read option
ro.index.update({"id":1}) #指定查询索引 key
try:
  resp = nsc.query(table, ro) #第一个参数表名，第二个参数read option，返回一个query 迭代器
except Exception as e:
  print("query error ", e)
  exit(1)
print("query size ", resp.count()) #如果有数据查询到，count返回结果是1
for m in resp: # 循环query 迭代器，迭代器每次返回一个map, map的key 说schema names
  for k in m:
    if k == "content":
      blobData = m[k]
      print(blobData.getUrl()) #获取http proxy 访问后缀
      print(len(blobData.getData())) #获取blob 数据
      print(blobData.getKey()) #获取blob key
    else:
      print("key: ", k, ",value: ", m[k])

# batch_query batch_query 可以一次查询多个key
ros = []
for idx in range(1, 10): # 构造9个 read option
  ro = rtidb.ReadOption()
  ro.index.update({"id":1})
  ros.append(ro)

try:
  resp = nsc.batch_query(table, ros) #第一个参数表名，第二个参数read options，返回一个batch query 迭代器
except Exception as e:
  print("batch query error ", e)
  exit(1)
print("batch query size ", resp.count())
for m in resp: # 循环batch query 迭代器，迭代器每次返回一个map, map的key 说schema names
  for k in m:
    if k == "content":
      blobData = m[k]
      print(blobData.getUrl()) #获取http proxy 访问后缀
      print(len(blobData.getData())) #获取blob 数据
      print(blobData.getKey()) #获取blob key
    else:
      print("key: ", k, ",value: ", m[k])

# traverse 全表查询
try:
  resp = nsc.traverse(table, None)
except Exception as e:
  print(e)
  exit(1)
# traverse do not support count, beacuse traverse count is not precise
traverse_id = 1;
for m in resp:
  for k in m:
    if k == "content":
      blobData = m[k]
      print(blobData.getUrl()) #获取http proxy 访问后缀
      print(len(blobData.getData())) #获取blob 数据
      print(blobData.getKey()) #获取blob key
    else:
      print("key: ", k, ",value: ", m[k])
  if m["id"] != traverse_id:
    raise Exception("data error")
  traverse_id+=1
  if (traverse_id > 10):
      break

# update
cond = {"id":"10"}
value = {"filename": "123"}
try:
  update_result = nsc.update(table, cond, value, None) #更新id 10 的filename
except Exception as e:
  print(e)
  exit(1)
if not update_result.success(): #通过success 判断是否更新成功
  print("update fail")
  exit(1)
print(update_result.affected_count()) # 获取有多少行 被更新工程


# query update result #验证update 结果
ro = rtidb.ReadOption()
ro.index = cond
try:
  resp = nsc.query(table, ro)
except Exception as e:
  print("query errir", e)
  exit(1)

for m in resp:
  if m["filename"] != "123":
    print("update failed")
    exit(1)

# delete
try:
  resp = nsc.delete(table, cond)
except Exception as e:
  print(e)
  exit(1)

ro.index = cond #验证是否删除成功
try:
  resp = nsc.quey(table, ro)
except Exception as e:
  print("delete ok, because query {} fail".format(cond["id"]))

# 主键重复put, 目前只有索引类型为primarykey，在设置write option 的情况下，允许重复put
dup_id = 1 
for fileName in os.listdir("images"):
  with open("images/{}".format(fileName), "rb") as f:
    binaryData = f.read()
  data = {"id": dup_id, "filename": "dup_put", "content": binaryData} 
  wo = rtidb.WriteOption()
  wo.updateIfExist = True #  设置write 参数
  putResult = nsc.put(table, data, wo)
  if not putResult.success():
    print("put %s error" % fileName);
    exit(1)
  break
ro = rtidb.ReadOption() #验证结果
ro.index = {"id": dup_id}
try:
  resp = nsc.query(table, ro)
except Exception as e:
  print("query errir", e)
  exit(1)
for m in resp:
  if m["filename"] != "dup_put":
    print("dup_put query error")
    exit(1)
```

