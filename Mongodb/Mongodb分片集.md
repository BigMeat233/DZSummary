# Mongodb分片集

## shard key、chunk、zone和shard的关系

1. 单个shard key无法跨chunk存储（同一个shard key最多存储在一个chunk上）
2. zone是shard间迁移数据的最小单位，zone和shard是多对多关系
3. 一个zone可以包含多个shard key range
4. 指定zone和shard时需要关注**基数**，如果一个zone的基数小于shard的个数，拓展shard个数无意义

## 关于chunk

1. 当chunk大小增长至配置的最大chunk size时，chunk会进行分裂（insert和update操作会触发chunk分裂）
2. 只有一个shard key的chunk不会分裂，此chunk体积会越积越大，变为jumboChunk
3. jumboChunk无法分裂，也无法被balancer迁移
4. chunk操作包括：**创建**、**拆分**、**合并**、**迁移**、**调整体积**  

## 创建chunk

1. 使用场景：在吞吐量大的时候(比如：导入大量单调性的数据)，
   Mongodb没有充足的时间创建足够多的chunk（若要创建chunk并迁移保持数据存储平衡则会导致无法快速分发数据），所以需要提前创建chunk  
2. 只能在空的collection上执行创建chunk操作  
3. 创建chunk使用**split**指令，但是只能使用**middle**作为分割点（find、bounds将会以符合条件chunk的中位数进行分割，空chunk没有中位数）  
4. 说明：当split操作使用**middle**且作用在空collection上时即创建chunk，是拆分chunk的特殊形式  

## 拆分chunk

1. 使用场景：大量的数据集中在一个chunk或shard中时，需要进行手动拆分chunk。
   比如：刚进行了shard操作、导入了大量shard key集中在某范围且这个范围占据了大多数的shard key的数据
2. 使用```sh.splitFind()```或```sh.splitAt()```指令执行拆分，即使用**split**指令的find、middle配置
3. ```sh.splitFind()```将会寻找符合指定条件的chunk，拆分成两个同样体积的chunk
4. ```sh.splitAt()```将会以传入的条件作为新chunk的下界进行拆分，可能拆分出的chunk体积不同

## 合并chunk

1. 合并chunk操作只能合并相邻的一个或多个chunk
2. 使用**mergeChunks**操作进行合并，需要传入合并后chunk的上界和下界作为bounds

## 迁移chunk

1. 使用场景：设置了balancer运行周期，但是想迁移chunk; 
   导入大量数据之前创建了大量chunk想要手动满足存储负载平衡
2. 迁移chunk使用**moveChunk**指令

## 调整chunk体积

1. 只有insert、update操作后chunk体积达到配置上限会触发自动分裂（只有一个shard key的chunk不会分 裂*
2. chunk体积范围为：[1MB 1024MB]
3. chunk体积越小迁移每个chunk越快，但是次数越多; chunk体积越大迁移每个chunk越慢，但是次数越小
4. 当调小chunk体积后，Mongodb将会立即开始分裂超过新上限的chunk，并开始迁移chunk，可能会造成性能影响
5. 当调大chunk体积后，只有在insert、update操作后到达新上限才会触发自动分裂

## chunk迁移过程

1. Balancer Proces仅存在于Config Server Replica Set的Primary节点上
2. Balancer Process向source shard发送moveChunk指令（内部指令）
3. 当source shard收到指令后开始进行chunk迁移，迁移期间的读写操作流量会继续进入source shard
4. destination shard在对应的collection中构建迁移chunk必须的索引，
   然后开始请求和接收迁移前的数据（迁移期间写操作产生的增量数据不会在此阶段进行迁移）
5. 当destination shard接收chunk迁移前的数据完毕后，
   Mongodb会暂停该chunk所在集合的读写流量（更新Config Server Replica Set成功之后恢复），
   然后destination shard对迁移期间写操作产生的增量数据进行同步
6. 当同步完成后，source shard在Config Server Replica Set中对迁移chunk的元数据进行update操作，
   更新成功后，恢复该chunk所在集合的读写流量，此时流量将会进入destination shard中，
   **更新Config Server Replica Set操作的write concern为majority，不受_secondaryThrottle影响**
7. 当改动元数据完成后，当此chunk没有活动cursor时，将会移除source shard上此chunk的备份数据（进入待删除队列）

## 关于balancer

1. Balancer是一个跑在Config Server Replica Set中Primary节点的后台进程，用于监控和迁移每一个shard上的chunk
2. 为了降低性能影响，**cluster维度上**最多有**shard数量/2（取下限）**的chunk进行迁移，**shard维度上**最多有**1个chunk**进行迁移
3. 为了降低性能影响，只有拥有最大数量chunk和拥有最少数量chunk的shard的chunk数量超过阈值时才会触发迁移，这个阈值和chunk总数量相关
4. 为了降低性能影响，在生产环境下通常设置固定时间（业务低峰期）运行balancer
5. Mongodb无法迁移document数量超过1.3倍平均值的chunk
6. balancer结束条件为：某collection在任意两个shard中的chunk数量差异<2或迁移失败

## 关于_secondaryThrottle

1. 这个参数决定何时继续迁移chunk中的下一个document
2. 如果```_secondaryThrottle == [write concern]```，则本次迁移的document在destination shard的写操作满足write concern后才会开启下一个document的迁移  
3. 如果```_secondaryThrottle == true```，则本次迁移的document在destination shard的写入操作需要收到至少一个secondary节点的ACK后才会开启下一次迁移，即```{ w：2 }```
4. 如果```_secondaryThrottle == undefined```，则在迁入的shard中，此document的写入操作不需要收到secondary节点的ACK即可开启下一次迁移，即```{ w：1 }```
5. **设置_secondaryThrottle后需要重启balancer才会生效**

## 迁移chunk时的write concern保证一致性

1. 以下的情况将会无视```_secondaryThrottle```设置
2. 迁移chunk过程中，在所有的document迁移完成后，Mongodb会将chunk的新位置更新至Config Server Replica Set，
   在更新之前，Mongodb会暂停source shard上该chunk所在集合的读写流量，更新成功之后恢复，
   更新chunk新位置操作的write concern为majority
3. 更新Config Server Replica Set
   之前的写操作（destination shard所有迁入document的写操作、oplog的写操作等）和
   之后的写操作（source shard上删除的源document、oplog的写操作等）
   使用的write concern为majority
4. 在source shard完成chunk迁出要删除备份数据的时候，
   只有删除操作只有得到了write concern为majority的ACK后才能继续
   下一次删除备份数据操作（因其他chunk迁出导致的删除）或迁入chunk
