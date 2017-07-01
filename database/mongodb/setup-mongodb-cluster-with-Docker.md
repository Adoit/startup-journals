# 使用Docker搭建MongoDB Shard集群

## 背景

上一次创业时，使用MongoDB存储业务数据，当时我们使用Docker部署的MongoDB Shard Cluster，当时使用的MongoDB的版本是3.0，目前最新（截止2017-06-25）版本是3.4中，从部署上来看，除了在3.2版本时config server使用了[replica set](https://docs.mongodb.com/v3.2/tutorial/upgrade-config-servers-to-replica-set/)这一变化外，没有其它的变化。

## 目标

1. 搭建一个三节点的MongoDB Cluster
2. 使用3 Replica Sets + 2 Shards的方式。MongoDB自3.2起放弃了Master-Slave的架构，较之于Replica Sets的方式，主-从模式唯一的优势在于从节点的数量没有限制，但主从模式的缺点也很明显，太多的从节点会加大主节点的负担，主节点会成为瓶颈，而且主从模式不支持自动的failover。
3. 不同的shards存储在不同的物理磁盘上。通常实际的生产环境中，会把不同的shard存储在不同的物理磁盘上，避免一块物理磁盘的损坏，对数据的完整性产生较大的影响。
4. 使用MongoDB版本3.4
5. 使用Docker来搭建。Docker的优势之一是做到资源的隔离，并且可以做到资源尽可能多的复用。

## 说明

1. 假设三台测试服务器信息

   ```
   192.168.1.80  test001.localcluster
   192.168.1.81  test002.localcluster
   192.168.1.82  test003.localcluster
   ```

2. 两块磁盘，**disk1**与**disk2**，分别挂载在**/data/disk1**和**/data/disk2**两个目录下，为了测试，仅需要创建这两个目录即可。


## 初始化

1. 下载最新的Mongo Docker镜像（当前版本3.4），`docker pull mongo:3.4`

2. 创建数据文件夹

   ```shell
   # test001
   mkdir -p /data/disk1/mongodb-config-1/
   mkdir -p /data/disk1/mongodb-shard0000-data1/
   mkdir -p /data/disk2/mongodb-shard0001-data1/
   mkdir -p /data/disk2/mongodb-mongos-1/

   # test002
   mkdir -p /data/disk1/mongodb-config-2/
   mkdir -p /data/disk1/mongodb-shard0000-data2/
   mkdir -p /data/disk2/mongodb-shard0001-data2/
   mkdir -p /data/disk2/mongodb-mongos-2/

   # test003
   mkdir -p /data/disk1/mongodb-config-3/
   mkdir -p /data/disk1/mongodb-shard0000-data3/
   mkdir -p /data/disk2/mongodb-shard0001-data3/
   mkdir -p /data/disk2/mongodb-mongos-3/
   ```


## 部署

### Create the Config Server Replica Set

1. 启动服务，**注：如果重新部署，一定要删除相应目录下的数据文件，否则会因为数据不一致引起错误**

  ```shell
  # test001
  rm -rf /data/disk1/mongodb-config-1/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-config-1 -p 30000:27017 -v /data/disk1/mongodb-config-1:/data/db mongo mongod --port 27017 --configsvr --replSet confgShard

  # test002
  rm -rf /data/disk1/mongodb-config-2/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-config-2 -p 30000:27017 -v /data/disk1/mongodb-config-2:/data/db mongo mongod --port 27017 --configsvr  --replSet confgShard

  # test003
  rm -rf /data/disk1/mongodb-config-3/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-config-3 -p 30000:27017 -v /data/disk1/mongodb-config-3:/data/db mongo mongod --port 27017 --configsvr  --replSet confgShard
  ```

2. 配置configsvr集群

   ```shell
   rs.initiate(
     {
       _id: "confgShard",
       configsvr: true,
       members: [
         { _id : 0, host : "test001.localcluster:30000" },
         { _id : 1, host : "test002.localcluster:30000" },
         { _id : 2, host : "test003.localcluster:30000" }
       ]
     }
   )
   ```




### Create the Shard Replica Sets

1. 启动`shard0`服务

   ```shell
   # test001
   rm -rf /data/disk1/mongodb-shard0000-data1/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard000-data1 -p 28000:27017 -v /data/disk1/mongodb-shard0000-data1:/data/db mongo mongod --shardsvr --replSet shard0  --port 27017

   # test002
   rm -rf /data/disk1/mongodb-shard0000-data2/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard000-data2 -p 28000:27017 -v /data/disk1/mongodb-shard0000-data2:/data/db mongo mongod --port 27017 --shardsvr --replSet shard0 

   # test003
   rm -rf /data/disk1/mongodb-shard0000-data3/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard000-data3 -p 28000:27017 -v /data/disk1/mongodb-shard0000-data3:/data/db mongo mongod --port 27017 --shardsvr --replSet shard0 
   ```

2. 配置`shard0` repliSets

   ```shell
   # 登录至其中一台mongo服务
   docker exec -it mongodb-shard000-data3 mongo

   rs.initiate(
     {
       _id : "shard0",
       members: [
         { _id : 0, host : "test001.localcluster:28000", priority: 1 },
         { _id : 1, host : "test002.localcluster:28000", priority: 2 },
         { _id : 2, host : "test003.localcluster:28000", priority: 3 }
       ]
     }
   )
   ```

3. 启动`shard1`服务

   ```shell
   # test001
   rm -rf /data/disk2/mongodb-shard0001-data1/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard001-data1 -p 28010:27017 -v /data/disk2/mongodb-shard0001-data1:/data/db mongo mongod --port 27017 --shardsvr --replSet shard1

   # test002
   rm -rf /data/disk2/mongodb-shard0001-data2/* && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard001-data2 -p 28010:27017 -v /data/disk2/mongodb-shard0001-data2:/data/db mongo mongod --port 27017 --shardsvr --replSet shard1 

   # test003
   rm -rf /data/disk2/mongodb-shard0001-data3 && docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-shard001-data3 -p 28010:27017 -v /data/disk2/mongodb-shard0001-data3:/data/db mongo mongod --port 27017 --shardsvr --replSet shard1 

   ```

4. 配置`shard1`repliSets

   ```shell
   # 登录至其中一台mongo服务
   docker exec -it mongodb-shard001-data3 mongo

   rs.initiate(
     {
       _id : "shard0",
       members: [
         { _id : 0, host : "test001.localcluster:28010", priority: 1 },
         { _id : 1, host : "test002.localcluster:28010", priority: 2 },
         { _id : 2, host : "test003.localcluster:28010", priority: 3 }
       ]
     }
   )
   ```


   ​

### Connect a mongos to the Sharded Cluster

```shell
# test001
docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-mongos-1 -p 40000:27017 -v /data/disk2/mongodb-mongos-1/:/data/db mongo mongos --port 27017 --configdb confgShard/test001.localcluster:30000,test002.localcluster:30000,test003.localcluster:30000

# test002
docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-mongos-2 -p 40000:27017 -v /data/disk2/mongodb-mongos-2/:/data/db mongo mongos --port 27017 --configdb confgShard/test001.localcluster:30000,test002.localcluster:30000,test003.localcluster:30000

# test003
docker run -d --add-host test001.localcluster:192.168.1.80 --add-host test002.localcluster:192.168.1.81 --add-host test003.localcluster:192.168.1.82 --name mongodb-mongos-3 -p 40000:27017 -v /data/disk2/mongodb-mongos-3/:/data/db mongo mongos --port 27017 --configdb confgShard/test001.localcluster:30000,test002.localcluster:30000,test003.localcluster:30000
```



### Add Shards to the Cluster

```shell
# 访问其中一台mongos服务
docker exec -it mongodb-mongos-1 mongo

sh.addShard( "shard0/test001.localcluster:28000")
sh.addShard( "shard1/test001.localcluster:28000")

# 查看状态
```



### Enable Sharding for a Database

```shell
# ycsb为database 名称
sh.enableSharding('ycsb')
```



### Shard a Collection

```shell
# usertable为collection名称
sh.shardCollection('ycsb.usertable',  { "_id": "hashed" })
```



### Verify Shard Status

```
sh.status()

--- Sharding Status ---
  sharding version: {
	"_id" : 1,
	"minCompatibleVersion" : 5,
	"currentVersion" : 6,
	"clusterId" : ObjectId("58da3060901545b0e5d4f1d1")
}
  shards:
	{  "_id" : "shard1",  "host" : "shard1/test001.localcluster:28010,test002.localcluster:28010,test003.localcluster:28010",  "state" : 1 }
	{  "_id" : "shard0",  "host" : "shard0/test001.localcluster:28000,test002.localcluster:28000,test003.localcluster:28000",  "state" : 1 }
  active mongoses:
	"3.4.2" : 1
 autosplit:
	Currently enabled: yes
  balancer:
	Currently enabled:  yes
	Currently running:  no
		Balancer lock taken at Tue Mar 28 2017 09:44:02 GMT+0000 (UTC) by ConfigServer:Balancer
	Failed balancer rounds in last 5 attempts:  0
	Migration Results for the last 24 hours:
		2 : Success
  databases:
	{  "_id" : "ycsb",  "primary" : "shard1",  "partitioned" : true }
		ycsb.usertable
			shard key: { "_id" : "hashed" }
			unique: false
			balancing: true
			chunks:
				shard0	2
				shard1	2
			{ "_id" : { "$minKey" : 1 } } -->> { "_id" : NumberLong("-4611686018427387902") } on : shard1 Timestamp(2, 2)
			{ "_id" : NumberLong("-4611686018427387902") } -->> { "_id" : NumberLong(0) } on : shard1 Timestamp(2, 3)
			{ "_id" : NumberLong(0) } -->> { "_id" : NumberLong("4611686018427387902") } on : shard0 Timestamp(2, 4)
			{ "_id" : NumberLong("4611686018427387902") } -->> { "_id" : { "$maxKey" : 1 } } on : shard0 Timestamp(2, 5)
```



[^adoit]: 转载请注明出处 [adoit](https://github.com/adoit)