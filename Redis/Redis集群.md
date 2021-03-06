### Redis Cluster TCP ports

Redis集群节点间使用TCP进行通信。6379端口用于客户端连接，16379端口用于集群节点之间的通信，包括故障侦测，配置更新，故障切换等。



### Redis Cluster data sharding

Redis Cluster 不使用一致性hash算法，而是使用hash slot（哈希槽）。

总共有16384个槽，平均分配给集群中每个节点，计算key对应的槽方式为，

```
HASH_SLOT = CRC16(key) mod 16384
```

#### Moved重定向

1.每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系 

2.客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16383取余，计算对应的槽和节点 

3.如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端 

4.如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常 

5.客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息 

6.客户端向目标节点发送命令，获取命令执行结果

#### ask重定向

在对集群进行扩容和缩容时，需要对槽及槽中数据进行迁移

如果此时正在进行集群扩展或者缩空操作，当客户端向正确的节点发送命令时，槽及槽中数据已经被迁移到别的节点了，就会返回ask，这就是ask重定向机制

1.客户端向目标节点发送命令，目标节点中的槽已经迁移支别的节点上了，此时目标节点会返回ask转向给客户端 

2.客户端向新的节点发送Asking命令给新的节点，然后再次向新节点发送命令 

3.新节点执行命令，把命令执行结果返回给客户端

moved异常与ask异常的相同点和不同点：

moved异常：槽已经确定迁移，即槽已经不在当前节点 ，将来所有的对该槽的请求都将发往新的节点。

ask异常：槽还在迁移中，只有下一次请求需要去请求目标节点。

#### 智能客户端

是一种提高性能的方式，客户端也知道槽位与节点的对应关系，知道所有节点的地址，并且实时更新槽位变化，每次请求都将发往正确的节点，减少重定向次数。

#### hash_tag

使用{}对key进行处理，那么在计算hash时，只对{}内的内容进行计算。这样能够保证在{}内拥有相同内容的key被分到同一个节点上。

### Redis Cluster master-slave model

每个主节点都有1-N个从节点，主节点负责读写，从节点负责从主节点拉取数据备份。

从节点的选举和提升都是由从节点处理的，主节点会投票要提升哪个从节点。一个从节点的选举是在主节点被至少一个从节点标记为 FAIL 状态的时候发生的，发起选举的从节点必须具有成为主节点的先决条件。

当以下条件满足时，一个从节点可以发起选举：

- 该从节点的主节点处于 FAIL 状态。
- 这个主节点负责的哈希槽数目不为零。
- 从节点和主节点之间的重复连接（replication link）断线不超过一段给定的时间，这是为了确保从节点的数据是可靠的。
- 一个从节点想要被推选出来，那么第一步应该是提高它的 currentEpoch 计数，并且向主节点们请求投票。

从节点通过广播一个 FAILOVER_AUTH_REQUEST 数据包给集群里的每个主节点来请求选票。然后等待回复（最多等 NODE_TIMEOUT 这么长时间）。一旦一个主节点给这个从节点投票，会回复一个 FAILOVER_AUTH_ACK，并且在 NODE_TIMEOUT * 2 这段时间内不能再给同个主节点的其他从节点投票。在这段时间内它完全不能回复其他投票请求。

从节点会忽视所有带有的时期（epoch）参数比 currentEpoch 小的回应（ACKs），这样能避免把之前的投票的算为当前的合理投票。

一旦某个从节点收到了大多数主节点的回应，那么它就赢得了选举。否则，如果无法在 NODE_TIMEOUT 时间内访问到大多数主节点，那么当前选举会被中断并在 NODE_TIMEOUT * 4 这段时间后由另一个从节点尝试发起选举。



从节点并不是在主节点一进入 FAIL 状态就马上尝试发起选举，而是有一点点延迟，这段延迟是这么计算的：

```
DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds +
        SLAVE_RANK * 1000 milliseconds.
```

固定延时（fixed delay）确保我们会等到 FAIL 状态在集群内广播后，否则若从节点尝试发起选举，主节点们仍然不知道那个主节点已经 FAIL，就会拒绝投票。

随机延时（random delay）是用来添加一些不确定因素以减少多个从节点在同一时间发起选举的可能性，因为若同时多个从节点发起选举或许会导致没有任何节点赢得选举，要再次发起另一个选举的话会使集群在当时变得不可用。

SLAVE_RANK：从节点根据自己的数据与主节点的差异大小排名，差异越小，rank越小，越早开始投票。

一旦有从节点赢得选举，它就会开始用 ping 和 pong 数据包向其他节点宣布自己已经是主节点，并提供它负责的哈希槽，设置 configEpoch 为 currentEpoch（选举开始时生成的）。

其他节点会检测到有一个新的主节点（带着更大的configEpoch）在负责处理之前一个旧的主节点负责的哈希槽，然后就升级自己的配置信息。 旧主节点的从节点，或者是经过故障转移后重新加入集群的该旧主节点，不仅会升级配置信息，还会配置新主节点的备份。



### Redis Cluster consistency guarantees

Redis cluster 不保证强一致性，在failover的过程中可能会丢失数据，有以下两种情况

1. 主节点写完数据返回给客户端确认信息，在备份到从节点之前宕机
2. 网络发生分区，例如ABC三个节点，有A1B1C1三个从节点，B节点与AC失去联系，但是客户端还在往B写数据，之后网络故障恢复，在这期间B1被提升为主节点，B加入后成为从节点，请求与B1进行数据同步，因此就会丢失之前客户端写入的数据。

### Redis Cluster configuration parameters

### 失效检测（Failure detection）

Redis 集群失效检测是用来识别出大多数节点何时无法访问某一个主节点或从节点。当这个事件发生时，就提升一个从节点来做主节点；若如果无法提升从节点来做主节点的话，那么整个集群就置为错误状态并停止接收客户端的查询。

每个节点都有一份跟其他已知节点相关的标识列表。其中有两个标识是用于失效检测，分别是 PFAIL 和 FAIL。PFAIL 表示可能失效（Possible failure），这是一个非公认的（non acknowledged）失效类型。FAIL 表示一个节点已经失效，而且这个情况已经被大多数主节点在某段固定时间内确认过的了。

**PFAIL 标识:**

当一个节点在超过 NODE_TIMEOUT 时间后仍无法访问某个节点，那么它会用 PFAIL 来标识这个不可达的节点。无论节点类型是什么，主节点和从节点都能标识其他的节点为 PFAIL。

Redis 集群节点的不可达性（non reachability）是指，发送给某个节点的一个活跃的 ping 包（active ping）(一个我们发送后要等待其回复的 ping 包)已经等待了超过 NODE_TIMEOUT 时间，那么我们认为这个节点具有不可达性。为了让这个机制能正常工作，NODE_TIMEOUT 必须比网络往返时间（network round trip time）大。节点为了在普通操作中增加可达性，当在经过一半 NODE_TIMEOUT 时间还没收到目标节点对于 ping 包的回复的时候，就会马上尝试重连接该节点。这个机制能保证连接都保持有效，所以节点间的失效连接通常都不会导致错误的失效报告。

**FAIL 标识:**

单独一个 PFAIL 标识只是每个节点的一些关于其他节点的本地信息，它不是为了起作用而使用的，也不足够触发从节点的提升。要让一个节点真正被认为失效了，那需要让 PFAIL 状态上升为 FAIL 状态。 在本文的节点心跳章节有提到的，每个节点向其他每个节点发送的 gossip 消息中有包含一些随机的已知节点的状态。最终每个节点都能收到一份其他每个节点的节点标识。使用这种方法，每个节点都有一套机制去标记他们检查到的关于其他节点的失效状态。

当下面的条件满足的时候，会使用这个机制来让 PFAIL 状态升级为 FAIL 状态：

- 某个节点，我们称为节点 A，标记另一个节点 B 为 PFAIL。
- 节点 A 通过 gossip 字段收集到集群中大部分主节点标识的节点 B 的状态信息。
- 大部分主节点标记节点 B 为 PFAIL 状态，或者在 NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT 这个时间内是处于 PFAIL 状态。

如果以上所有条件都满足了，那么节点 A 会：

- 标记节点 B 为 FAIL。
- 向所有可达节点发送一个 FAIL 消息。

FAIL 消息会强制每个接收到这消息的节点把节点 B 标记为 FAIL 状态。

注意，FAIL 标识基本都是单向的，也就是说，一个节点能从 PFAIL 状态升级到 FAIL 状态，但要清除 FAIL 标识只有以下两种可能方法：

- 节点已经恢复可达的，并且它是一个从节点。在这种情况下，FAIL 标识可以清除掉，因为从节点并没有被故障转移。
- 节点已经恢复可达的，而且它是一个主节点，但经过了很长时间（N * NODE_TIMEOUT）后也没有检测到任何从节点被提升了。

### Redis Cluster configuration parameters

- **cluster-enabled<yes/no>**: 对单个redis实例而言，是否开启集群模式
- **cluster-config-file\<filename>:** 指定配置文件名，这个文件不是给用户编辑的，而是redis在每次配置改变时持久化到磁盘上的，供重启时读取。
- **cluster-node-timeout\<milliseconds>:** redis节点不可用的最大时间，在此期间内不会被认为不可用。当某节点与大多数节点失去连接超过该时间则会拒绝接受请求。
- **cluster-slave-validity-factor\<factor>:** 如果设置为0，从节点会在与主节点失去连接后立马开始故障切换，而忽略node-timeout值。如果是正数，则在node-timeout * factor时间内不会进行故障切换。
- **cluster-migration-barrier\<count>:**主节点必须与多少个从节点保持连接。
- **cluster-require-full-coverage\<yes/no>:** 如果设置为yes，当某些槽位不在任何一个节点上时，集群拒绝写操作。
- **cluster-allow-reads-when-down\<yes/no>:** 是否允许节点在处于fail状态时还可以对外提供读服务。

### Nodes handshake

以下两种情况下，节点会认为其他节点是集群的一部分：

1. 如果节点信息出现在MEET message中，MEET message会让接收者节点强制认为这个节点是集群的一部分。节点只会在系统管理者发送以下命令时发送MEET message, 

   ```
   CLUSTER MEET ip port
   ```

2. 如果A已经与B建立了信任，B与C建立了信任，B会将AC的信息发送给AC，这样A和C就会主动尝试去连接对方。

