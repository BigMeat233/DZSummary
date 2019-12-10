# Mongodb副本集

## 关于priority、votes的关系

1. 若```priority > 0```，则一定有```votes > 0```
2. 可能存在```priority == 0```且```votes > 0```的情况，比如：arbiter节点
3. 可能存在```votes == 0```且```priority > 0```的情况，
   在一个Replica Set中，Mongodb仅允**7台**```votes > 0```的节点，所以可能会出现此情况，
   此时该节点可以当选为Primary，但是无法参与选举，也无法发出ACK

## 关于PSA的priority、hidden、votes

1. 若```prioriry == 0```，则该节点无法发起选举，无法当选为Primary
2. 若```hidden == true```，则```priority == 0```；
   反之```prioriry > 0```，不一定有```hidden == true```，比如：热备节点
3. 在Arbiter节点中，```priority == 0```且```votes == 1```
4. ```votes > 0```的节点参与选举和发出ACK，
   ```votes == 0```的节点无法参与选举和发出ACK
5. 无论是选举还是write concern的majority，
   都需要超过Replica Set中所有节点```votes```总和的半数才记为有效（防脑裂）

## 关于catchUpTakeoverDelayMillis

> ### 附录（若满足所有情况将会发起新的选举）
>
> 1. 此节点还是领先于当前的Primary节点
> 2. 此节点是当前所有可用节点中最新的节点
> 3. Primary节点正在追赶此节点

1. 当确定某个节点领先于当前的Primary节点时（一般发生在选举完成后），
   等待```catchUpTakeoverDelayMillis```然后判断**附录**中描述的3种情况
2. 若满足所有的条件，则发起新的选举，选举**附录**中3种情况都满足的节点为新的Primary
3. 若```catchUpTakeoverDelayMillis = -1```，则禁用此选项，
   即即使Replica Set有领先当Primary的节点，也不选举此节点成为新的Primary，此节点上的数据终将回滚
4. 若```catchUpTimeoutMillis = 0```，则选举之后会立即回滚各个节点领先当前Primary的数据，
   即选举后所有节点将与Primary同步，不会存在领先当前Primary的节点，此时```catchUpTakeoverDelayMillis```设置将无效

## 关于catchUpTimeoutMillis

1. 当选举完成后，等待```catchUpTimeoutMillis```时间后，将回滚所有的节点与领先当前Primary的数据，使所有节点与Primary同步
2. 在```catchUpTimeoutMillis```时间内，当前Primary节点将从Replica Set中拥有最新数据的节点同步数据，此时Primary将会拒绝所有的写操作导致可用性降低（**```catchUpTimeoutMillis```与可用性负相关，与一致性正相关**）  
3. 若```catchUpTimeoutMillis == 0```，则选举成功后将会立即回滚，不保留任何```write: 1```的数据（**此时集群可用性最高，一致性最低**）
4. 若```catchUpTimeoutMillis == -1```，则选举成功后则当前Primary与Replica Set中拥有最新数据的节点同步完成后才可以接受写流量，当前Primary最后会同步到所有```write: 1```的数据（**此时集群可用性最低，一致性最高**）

## 关于heartbeatTimeoutSecs

1. 如果某个节点在```heartbeatTimeoutSecs```内没有应答，则其他节点将会把此节点标记为不可到达

## 关于electionTimeoutMillis

1. 认定Primary不可用的最大时间
2. 若```electionTimeoutMillis```设置增大，则Primary对网络波动的敏感度会降低，故障转移将会更慢
3. 若```electionTimeoutMillis```设置减少，则Primary对网络波动的敏感度会增大，故障转移将会更快
