# Kafka

## 简介

kafka是一个分布式消息队列。具有高性能、持久化、多副本备份、横向扩展能力。生产者往队列里写消息，消费者从队列里取消息进行业务逻辑。一般在架构设计中起到解耦、削峰、异步处理的作用。

kafka对外使用topic的概念，生产者往topic里写消息，消费者从读消息。为了做到水平扩展，一个topic实际是由多个partition组成的，遇到瓶颈时，可以通过增加partition的数量来进行横向扩容。单个parition内是保证消息有序。

**broker**

Kafka 集群包含一个或多个服务器，服务器节点称为broker。

broker存储topic的数据。如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个partition。

如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一个partition，剩下的M个broker不存储该topic的partition数据。

如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致Kafka集群数据不均衡。

**Topic**

每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。（物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处）

类似于数据库的表名

**Partition**

topic中的数据分割为一个或多个partition。每个topic至少有一个partition。每个partition中的数据使用多个segment文件存储。同一个partition中的数据是有序的，不同partition间的数据丢失了数据的顺序。如果topic有多个partition，消费数据时就不能保证数据的顺序。在需要严格保证消息的消费顺序的场景下，需要将partition数目设为1。

**Producer**

生产者即数据的发布者，该角色将消息发布到Kafka的topic中。broker接收到生产者发送的消息后，broker将该消息**追加**到当前用于追加数据的segment文件中。生产者发送的消息，存储到一个partition中，生产者也可以指定数据存储的partition。

**Consumer**

消费者可以从broker中读取数据。消费者可以消费多个topic中的数据。

**Consumer Group**

每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。

**Leader**

每个partition有多个副本，其中有且仅有一个作为Leader，Leader是当前负责数据的读写的partition。

**Follower**

Follower跟随Leader，所有写请求都通过Leader路由，数据变更会广播给所有Follower，Follower与Leader保持数据同步。如果Leader失效，则从Follower中选举出一个新的Leader。当Follower与Leader挂掉、卡住或者同步太慢，leader会把这个follower从“in sync replicas”（ISR）列表中删除。



## 安装

```shell
# 本机安装
$ brew install kafka
# 本机启动服务
$ brew services start zookeeper
$ brew services start kafka
# 本机停止服务，需要注意次序
$ brew services stop kafka
$ brew services stop zookeeper
```



## 为什么性能高

1. 写：每新写一条消息，kafka就是在对应的文件append写，属于顺序写磁盘，所以性能非常高。

2. partition提升了并发

3. 读：zero-copy：

   传统的读取文件数据并发送到网络的步骤如下：
    （1）操作系统将数据从磁盘文件中读取到内核空间的页面缓存；
    （2）应用程序将数据从内核空间读入用户空间缓冲区；
    （3）应用程序将读到数据写回内核空间并放入socket缓冲区；
    （4）操作系统将数据从socket缓冲区复制到网卡接口，此时数据才能通过网络发送。

   “零拷贝技术”只用将磁盘文件的数据复制到页面缓存中一次，然后将数据从页面缓存直接发送到网络中（发送给不同的订阅者时，都可以使用同一个页面缓存），避免了重复复制操作。

4. 生产者消息聚集batch发送

6. 页缓存



## 生产者

创建一条记录，记录中一个要指定对应的topic和value，key和partition可选。 先序列化，然后按照topic和partition，放进对应的发送队列中。kafka produce都是批量请求，会积攒一批，然后一起发送，不是调send()就进行立刻进行网络发包。
如果partition没填，那么情况会是这样的：

1. key有填
    按照key进行哈希，相同key去一个partition。（如果扩展了partition的数量那么就不能保证了）
2. key没填
    round-robin来选partition

**生产消息可靠性**

生产者生产消息的时候，通过request.required.acks参数来设置数据的可靠性。

| acks | what happen                                                  |
| ---- | ------------------------------------------------------------ |
| 0    | which means that the producer never waits for an acknowledgement from the broker.发过去就完事了，不关心broker是否处理成功，可能丢数据。 |
| 1    | which means that the producer gets an acknowledgement after the leader replica has received the data. 当写Leader成功后就返回,其他的replica都是通过fetcher去同步的,所以kafka是异步写，主备切换可能丢数据。 |
| -1   | which means that the producer gets an acknowledgement after all in-sync replicas have received the data. 要等到isr里所有机器同步成功，才能返回成功，延时取决于最慢的机器。强一致，不会丢数据。 |

在acks=-1的时候，如果ISR少于min.insync.replicas指定的数目，那么就会返回不可用。

这里ISR列表中的机器是会变化的，根据配置replica.lag.time.max.ms，多久没同步，就会从ISR列表中剔除。以前还有根据落后多少条消息就踢出ISR，**在1.0版本后就去掉了**，因为这个值很难取，在高峰的时候很容易出现节点不断的进出ISR列表。

从ISA中选出leader后，follower会从把自己日志中上一个高水位后面的记录去掉，然后去和leader拿新的数据。因为新的leader选出来后，**follower上面的数据，可能比新leader多，所以要截取**。这里高水位的意思，对于partition和leader，就是所有ISR中都有的最新一条记录。消费者最多只能读到高水位；

从leader的角度来说高水位的更新会延迟一轮，例如写入了一条新消息，ISR中的broker都fetch到了，但是ISR中的broker只有在下一轮的fetch中才能告诉leader。

也正是由于这个高水位延迟一轮，在一些情况下，kafka会出现丢数据和主备数据不一致的情况，0.11开始，使用leader epoch来代替高水位。

思考：
当acks=-1时

1. 是follwers都来fetch就返回成功，还是等follwers第二轮fetch？
2. leader已经写入本地，但是ISR中有些机器失败，那么怎么处理呢？

 

## Broker

当存在多副本的情况下，会尽量把多个副本，分配到不同的broker上。**kafka会为该partition的集合选出一个leader，之后所有该partition的请求，实际操作的都是leader，然后再同步到其他的follower。**当一个broker歇菜后，所有leader在该broker上的partition都会重新选举，选出一个leader。（这里不像分布式文件存储系统那样会自动进行复制保持副本数）

### controller

关于partition的分配，还有leader的选举，总得有个执行者。在kafka中，这个执行者就叫**controller**。kafka中所有broker会在zk上争夺注册一个节点，注册成功的那个broker就是controller，用于partition分配和leader选举。

controller会在Zookeeper的/brokers/ids节点上注册Watch，一旦有broker宕机，它就能知道。当broker宕机后，controller就会给受到影响的partition选出新leader。

作用

- Controller在Zookeeper上注册Watch，一旦有Broker宕机，其在Zookeeper对应的znode会自动被删除，Zookeeper会触发 Controller注册的watch，Controller读取最新的Broker信息

- Controller确定set_p，该集合包含了宕机的所有Broker上的所有Partition

- 对set_p中的每一个Partition，选举出新的leader、Isr，并更新结果。

  1. 从/brokers/topics/[topic]/partitions/[partition]/state读取该Partition当前的ISR

  2. 决定该Partition的新Leader和Isr。如果当前ISR中有至少一个Replica还幸存，则选择其中一个作为新Leader，新的ISR则包含当前ISR中所有幸存的Replica。否则选择该Partition中任意一个幸存的Replica作为新的Leader以及ISR（该场景下可能会有潜在的数据丢失）

  3. 更新Leader、ISR、leader_epoch、controller_epoch，写入/brokers/topics/[topic]/partitions/[partition]/state

- 直接通过RPC向set_p相关的Broker发送LeaderAndISRRequest命令。Controller可以在一个RPC操作中发送多个命令从而提高效率。

### leader选举

选举分区leader的基本思路是按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中。一个分区的AR集合在分配的时候就被指定，并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，而分区的ISR集合中副本的顺序可能会改变。注意这里是根据AR的顺序而不是ISR的顺序进行选举的。这个说起来比较抽象，有兴趣的读者可以手动关闭/开启某个集群中的broker来观察一下具体的变化。

还有一些情况也会发生分区leader的选举，比如当分区进行重分配（reassign）的时候也需要执行leader的选举动作。这个思路比较简单：从重分配的AR列表中找到第一个存活的副本，且这个副本在目前的ISR列表中。

当ISR中的Replica全部宕机时，可以通过如下方式处理：

  　　1. 等待ISR中任一Replica恢复，并选它为Leader。

　　缺点：等待时间较长，降低可用性（因为不能使用所有集群节点），因此或ISR中的所有Replica都无法恢复或者数据丢失，则该Partition将永不可用。

  　　2. 选择第一个恢复的Replica为新的Leader，无论它是否在ISR中。

　　缺点：并未包含所有已被之前Leader Commit过的消息（因为它不在之前的ISR中），因此会造成数据丢失，但是它提高了可用节点的范围，可用性比较高。

### partition分配

topic是Kafka中的逻辑概念，而partition是物理实现，每个topic可以由多个partition组成，每个partion分布存储在不同的broker上（也可以放在相同的broker，但不是最佳实践）。

partition的分配

1. 将所有Broker（假设共n个Broker）和待分配的Partition排序
2. 将第i个Partition分配到第（i mod n）个Broker上 （这个就是leader）
3. 将第i个Partition的第j个Replica分配到第（(i + j) mode n）个Broker上

为什么topic要分成多个partition？简单的来说，是为了允许消费者消费一个topic可以水平扩展。分区是否越多越好呢？显然也不是，因为每个分区都有自己的开销。（1. 客户端/服务器端需要使用的内存就越多；2. 如果你有10000个分区，10个broker，也就是说平均每个broker上有1000个分区。此时这个broker挂掉了，那么zookeeper和controller需要立即对这1000个分区进行leader选举。比起很少的分区leader选举而言，这必然要花更长的时间，并且通常不是线性累加的。如果这个broker还同时是controller情况就更糟了。）

### offset

当生产者把数据写到每个分区时，每条消息在所在分区是有一个递增的序号的，序号从 0 开始，这个序号就是 offset，**offset 既反映了每条消息在分区中的相对位置，也唯一标识了一条消息**，offset越小，消息越旧，最新写入的消息的 offset在当前分区总是最大的。

**消息的 offset 和消费者的 offset**，用一张表总结一下：

|                 |                           理论概念                           |                           物理实现                           |
| :-------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  消息的 offset  |    分区内每条消息都有一个递增的序号，可以看做是消息的 ID     | 通过一个索引文件（`.index` 后缀），可以查到指定 offset 的 消息在数据文件（`.log`后缀）中的位置 |
| 消费者的 offset | 消费者在一个分区内所消费的最新消息的 offset，表示消费者在该分区内的消费进度 | 通过名为 `__consumer_offsets` 的特殊 topic，消费者将自己的 offset 发送到此 topic |



### 数据同步

- 同步复制：要求所有能工作的Follower都复制完，这条消息才会被认为commit，这种复制方式极大的影响了吞吐率
- 异步复制：Follower异步的从Leader pull数据，data只要被Leader写入log认为已经commit，这种情况下如果Follower落后于Leader的比较多，如果Leader突然宕机，会丢失数据
- ISR

Kafka结合同步复制和异步复制，使用ISR（与Partition Leader保持同步的Replica列表）的方式在确保数据不丢失和吞吐率之间做了平衡。Producer只需把消息发送到Partition Leader，Leader将消息写入本地Log。Follower则从Leader pull数据。Follower在收到该消息向Leader发送ACK。一旦Leader收到了ISR中所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向Producer发送ACK。这样如果leader挂了，只要Isr中有一个replica存活，就不会丢数据。

Leader会跟踪ISR，如果ISR中一个Follower宕机，或者落后太多，Leader将把它从ISR中移除。这里所描述的“落后太多”指Follower超过一定时间（replica.lag.time.max.ms）未向Leader发送fetch请求。

Producer在发布消息到某个Partition时，先通过ZooKeeper找到该Partition的Leader，然后无论该Topic的Replication Factor为多少，Producer只将该消息发送到该Partition的Leader。Leader会将该消息写入其本地Log。每个Follower都从Leader pull数据。这种方式上，Follower存储的数据顺序与Leader保持一致。Follower在收到该消息并写入其Log后，向Leader发送ACK。一旦Leader收到了ISR中的所有Replica的ACK，该消息就被认为已经commit了，Leader将增加HW并且向Producer发送ACK。

为了提高性能，每个Follower在接收到数据后就立马向Leader发送ACK，而非等到数据写入Log中。因此，对于已经commit的消息，**Kafka只能保证它被存于多个Replica的内存中**，而不能保证它们被持久化到磁盘中，也就不能完全保证异常发生后该条消息一定能被Consumer消费。

Kafka分区下有可能有很多个副本(replica)用于实现冗余，从而进一步实现高可用。副本根据角色的不同可分为3类：

- leader副本：响应clients端读写请求的副本
- follower副本：被动地备份leader副本中的数据，不能响应clients端读写请求。
- ISR副本：包含了leader副本和所有与leader副本保持同步的follower副本——如何判定是否与leader同步后面会提到

每个Kafka副本对象都有两个重要的属性：LEO和HW。**注意是所有的副本，而不只是leader副本。**

- LEO：即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息！也就是说，如果LEO=10，那么表示该副本保存了10条消息，位移值范围是[0, 9]。另外，leader LEO和follower LEO的更新是有区别的。我们后面会详细说
- HW：即上面提到的水位值。对于同一个副本对象而言，其HW值不会大于LEO值。小于等于HW值的所有消息都被认为是“已备份”的（replicated）。

### LEO, HW, leader epoch

Kafka分区下有可能有很多个副本(replica)用于实现冗余，从而进一步实现高可用。副本根据角色的不同可分为3类：

- leader副本：响应clients端读写请求的副本
- follower副本：被动地备份leader副本中的数据，不能响应clients端读写请求。
- ISR副本：包含了leader副本和所有与leader副本保持同步的follower副本——如何判定是否与leader同步后面会提到

每个Kafka副本对象都有两个重要的属性：LEO和HW。**注意是所有的副本，而不只是leader副本。**

- LEO：即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息！也就是说，如果LEO=10，那么表示该副本保存了10条消息，位移值范围是[0, 9]。另外，leader LEO和follower LEO的更新是有区别的。我们后面会详细说
- HW：即上面提到的水位值。对于同一个副本对象而言，其HW值不会大于LEO值。小于等于HW值的所有消息都被认为是“已备份”的（replicated）。

**一、follower副本何时更新LEO？**

如前所述，follower副本只是被动地向leader副本请求数据，具体表现为follower副本不停地向leader副本所在的broker发送FETCH请求，一旦获取消息后写入自己的日志中进行备份。那么follower副本的LEO是何时更新的呢？首先我必须言明，Kafka有两套follower副本LEO(**明白这个是搞懂后面内容的关键，因此请多花一点时间来思考**)：1. 一套LEO保存在follower副本所在broker的副本管理机中；2. 另一套LEO保存在leader副本所在broker的副本管理机中——换句话说，leader副本机器上保存了所有的follower副本的LEO。

为什么要保存两套？这是因为Kafka使用前者帮助follower副本更新其HW值；而利用后者帮助leader副本更新其HW使用。下面我们分别看下它们被更新的时机。

**1 follower副本端的follower副本LEO何时更新？**（原谅我有点拗口~~~~~）

follower副本端的LEO值就是其底层日志的LEO值，也就是说每当新写入一条消息，其LEO值就会被更新(类似于LEO += 1)。当follower发送FETCH请求后，leader将数据返回给follower，此时follower开始向底层log写数据，从而自动地更新LEO值

**2 leader副本端的follower副本LEO何时更新？**

leader副本端的follower副本LEO的更新发生在leader在处理follower FETCH请求时。一旦leader接收到follower发送的FETCH请求，它首先会从自己的log中读取相应的数据，但是在给follower返回数据之前它先去更新follower的LEO(上一次？)(即上面所说的第二套LEO)

**二、follower副本何时更新HW？**

follower更新HW发生在其更新LEO之后，一旦follower向log写完数据，它会尝试更新它自己的HW值。具体算法就是比较当前LEO值与FETCH响应中leader的HW值，取两者的小者作为新的HW值。这告诉我们一个事实：如果follower的LEO值超过了leader的HW值，那么follower HW值是不会越过leader HW值的。

**三、leader副本何时更新LEO？**

和follower更新LEO道理相同，leader写log时就会自动地更新它自己的LEO值。

**四、leader副本何时更新HW值？**

收到ISR所有的follow返回的ACK后。leader broker上保存了一套follower副本的LEO以及它自己的LEO。当尝试确定分区HW时，它会选出所有满足条件的副本，比较它们的LEO(当然也包括leader自己的LEO)，并选择最小的LEO值作为HW值。

**leader epoch**



## 消费者

订阅topic是以一个消费组来订阅的，一个消费组里面可以有多个消费者。同一个消费组中的两个消费者，不会同时消费一个partition。换句话来说，**就是一个partition，只能被消费组里的一个消费者消费**，但是可以同时被多个消费组消费。因此，如果消费组内的消费者如果比partition多的话，那么就会有个别消费者一直空闲。

### 分配partition--reblance

生产过程中broker要分配partition，**消费过程这里，也要分配partition给消费者。类似broker中选了一个controller出来，消费也要从broker中选一个coordinator，用于分配partition。**
 下面从顶向下，分别阐述一下

1. 消费者leader选举。
2. 怎么选coordinator。
3. 交互流程。
4. reblance的流程。

#### 消费者leader选举

组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader，这个选举的算法也很简单，分两种情况分析。如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader。如果某一时刻leader消费者由于某些原因退出了消费组，那么会重新选举一个新的leader，这个重新选举leader的过程又更“随意”了，相关代码如下：

```scala
//scala code.
private val members = new mutable.HashMap[String, MemberMetadata]
var leaderId = members.keys.head
```

解释一下这2行代码：在GroupCoordinator中消费者的信息是以HashMap的形式存储的，其中key为消费者的member_id，而value是消费者相关的元数据信息。leaderId表示leader消费者的member_id，它的取值为HashMap中的第一个键值对的key，这种选举的方式基本上和随机无异。总体上来说，消费组的leader选举过程是很随意的。

#### 选coordinator

kafka集群由于默认的__consumer_offsets这个topic的默认的副本数为1

第一步：看offset保存在那个partition

第二步：该partition leader所在的broker就是被选定的coordinator

这里我们可以看到，consumer group的coordinator，和保存consumer group offset的partition leader是同一台机器。

#### 交互流程

把coordinator选出来之后，就是要分配了
 整个流程是这样的：

1. consumer启动、或者coordinator宕机了，consumer会任意请求一个broker，发送ConsumerMetadataRequest请求，broker会按照上面说的方法，选出这个consumer对应coordinator的地址。
2. consumer 发送heartbeat请求给coordinator，返回IllegalGeneration的话，就说明consumer的信息是旧的了，需要重新加入进来，进行reblance。返回成功，那么consumer就从上次分配的partition中继续执行。

#### reblance流程

1. consumer给coordinator发送JoinGroupRequest请求。
2. 这时其他consumer发heartbeat请求过来时，coordinator会告诉他们，要reblance了。
3. 其他consumer发送JoinGroupRequest请求。
4. 所有记录在册的consumer都发了JoinGroupRequest请求之后，coordinator就会在这里consumer中随便选一个leader。然后回JoinGroupRespone，这会告诉consumer你是follower还是leader，对于leader，还会把follower的信息带给它，让它根据这些信息去分配partition

5. consumer向coordinator发送SyncGroupRequest，其中leader的SyncGroupRequest会包含分配的情况。
6. coordinator回包，把分配的情况告诉consumer，包括leader。

当partition或者消费者的数量发生变化时，都得进行reblance。
 列举一下会reblance的情况：

1. 增加partition
2. 增加消费者
3. 消费者主动关闭
4. 消费者宕机了
5. coordinator自己也宕机了

分配策略：

1. Range strategy
2. RoundRobin strategy

## ZK在kafka中作用

1. Broker注册
2. Topic注册
3. 生产者负载均衡
4. 消费者负载均衡
5. 消费者注册

## 消息投递语义

kafka支持3种消息投递语义

1. At most once：最多一次，消息可能会丢失，但不会重复
2. At least once：最少一次，消息不会丢失，可能会重复
3. Exactly once：只且一次，消息不丢失不重复，只且消费一次（0.11中实现，仅限于下游也是kafka）

在业务中，常常都是使用At least once的模型，如果需要可重入的话，往往是业务自己实现。

**At least once**

先获取数据，再进行业务处理，业务处理成功后commit offset。
 1、生产者生产消息异常，消息是否成功写入不确定，重做，可能写入重复的消息
 2、消费者处理消息，业务处理成功后，更新offset失败，消费者重启的话，会重复消费

**At most once**

先获取数据，再commit offset，最后进行业务处理。
 1、生产者生产消息异常，不管，生产下一个消息，消息就丢了
 2、消费者处理消息，先更新offset，再做业务处理，做业务处理失败，消费者重启，消息就丢了

**Exactly once**

思路是这样的，首先要保证消息不丢，再去保证不重复。所以盯着At least once的原因来搞。 首先想出来的：

1. 生产者重做导致重复写入消息----生产保证幂等性
2. 消费者重复消费---消灭重复消费，或者业务接口保证幂等性重复消费也没问题

**由于业务接口是否幂等，不是kafka能保证的，所以kafka这里提供的exactly once是有限制的，消费者的下游也必须是kafka。**所以一下讨论的，没特殊说明，消费者的下游系统都是kafka（注:使用kafka conector，它对部分系统做了适配，实现了exactly once）

**幂等性**

不丢消息

- 首先kafka保证了对已提交消息的at least保证
- Sender有重试机制
- producer业务方在使用producer发送消息时，注册回调函数。在onError方法中重发消息
- consumer 拉取到消息后，处理完毕再commit，保证commit的消息一定被处理完毕

不重复

- consumer拉取到消息先保存，commit成功后删除缓存数据

为实现Producer的幂等性，Kafka引入了Producer ID（即PID）和Sequence Number。对于每个PID，该Producer发送消息的每个<Topic, Partition>都对应一个单调递增的Sequence Number。同样，Broker端也会为每个<PID, Topic, Partition>维护一个序号，并且每Commit一条消息时将其对应序号递增。对于接收的每条消息，如果其序号比Broker维护的序号）大一，则Broker会接受它，否则将其丢弃：

- 如果消息序号比Broker维护的序号差值比一大，说明中间有数据尚未写入，即乱序，此时Broker拒绝该消息，Producer抛出InvalidSequenceNumber
- 如果消息序号小于等于Broker维护的序号，说明该消息已被保存，即为重复消息，Broker直接丢弃该消息，Producer抛出DuplicateSequenceNumber
- Sender发送失败后会重试，这样可以保证每个消息都被发送到broker

业务方对 Kafka producer的优化

- 增大producer数量
- ack配置
- batch



## 对比

### 与rabbitmq对比



### 与阿里云日志服务对比

