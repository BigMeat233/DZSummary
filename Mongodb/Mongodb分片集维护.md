# Mongodb分片集

## 更换Config Server副本集中的某台机器

1. 使用原Config Server副本集信息（副本集名）启动一台新的Config Server
2. 连接原有副本集并添加**步骤1**中启动的Config Server，添加时需要设置**priority**、**votes**为**0**
3. 等待新加入的Config Server进行数据同步（状态变为```SECONDARY```）后，在副本集中进行reconfig操作，重设此机器的**priority**、**votes**
4. 若更换的是副本集中的Primary机器，需要先使Primary进行```rs.stepDown()```
5. 在副本集中移出需要替换的Config Server
6. 更新mongos实例上的configDB配置使流量进入新的Config Server

## 更换Shard副本集中的某台机器
  
1. 若迁移的是Primary机器，需要先使Primary进行```rs.stepDown()```
2. 连入将要被替换的Shard实例，使用```shutdown```指令关闭mongod实例
3. 迁移dbpath指定路径下的文件到新的机器上
4. 在新机器上启动mongod实例
5. 连入Shard副本集的Primary，进行reconfig操作，重设新机器的host值

## 查看分片集群信息

1. 查看已分片的Database：

   ```javascript
   use config
   db.databases.find( { "partitioned": true } )
   ```

2. 查看所有的shard：

   ```javascript
   db.adminCommand( { listShards : 1 } )
   ```

## 关闭/启动分片集群

1. 关闭分片集群
    * 1.1 关闭balancer，在mongos上使用：

      ```javascript
      sh.stopBalancer()
      ```

    * 1.2 关闭所有Mongos实例，在每台mongos实例上使用：

      ```javascript
      use admin
      db.shutdownServer()
      ```

    * 1.3 关闭所有Shard实例，在每台Shard实例上使用：

      ```javascript
      use admin
      db.shutdownServer()
      ```

    * 1.4 关闭所有Config Server实例，在每台Config Server实例上使用：

      ```javascript
      use admin
      db.shutdownServer()
      ```

2. 启动分片集群
   * 2.1 开启Config Server副本集
   * 2.2 开启所有Shard实例
   * 2.3 开启所有mongos实例
   * 2.4 开启balancer，在mongos上使用：

     ```javascript
     sh.startBalancer()
     ```

## 新增分片

1. 新增分片会立刻触发balancer进行chunk迁移，将会带来因迁移chunk导致的存储空间和网络性能影响
2. 新增分片需要连入mongos，使用```sh.addShard("[副本集名]/[机器地址]:[机器端口]，")```进行操作

## 删除分片

1. 删除分片会**导致change stream关闭，并可能无法恢复**
2. 删除分片需要**确保balancer功能处于启用状态**
3. 删除分片需要连入mongos，使用```db.adminCommand( { removeShard: "[分片名]" } )```
4. 删除分片时，如果某Database的Primary Shard在此分片上，则必须迁移该数据库的Primary Shard，
   使用```db.adminCommand( { movePrimary: "[数据库名]"，to: "[目标shard名]" })```，
   迁移完成后( movePrimary后需要在所有的mongos和shard机器上重启或flushRouterConfigmongos )，
   继续使用```db.adminCommand( { removeShard: "[分片名]" } )```尝试删除
5. 删除分片时，如果有jumboChunk，则必须手动迁移或拆分jumboChunk，当没有jumboChunk时，
   继续使用```db.adminCommand( { removeShard: "[分片名]" } )```尝试删除
