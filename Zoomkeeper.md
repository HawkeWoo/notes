# Zoomkeeper

## 使用场景

分布式锁， 配置管理， 集群管理



PERSISTENT	持久化节点
PERSISTENT_SEQUENTIAL	顺序自动编号持久化节点，这种节点会根据当前已存在的节点数自动加 1
EPHEMERAL	临时节点， 客户端session超时这类节点就会被自动删除
EPHEMERAL_SEQUENTIAL	临时自动编号节点

## 崩溃恢复

ZAB协议会让ZK集群进入崩溃恢复模式的情况如下：
   （1）当服务框架在启动过程中
   （2）当Leader服务器出现网络中断，崩溃退出与重启等异常情况。
   （3）当集群中已经不存在过半的服务器与Leader服务器保持正常通信

1. 每个Server发出一个投票。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。

2. 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。

3. 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK规则如下

   ​	优先检查ZXID。ZXID比较大的服务器优先作为Leader。

   ​	如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器。

4. 对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

5. 统计投票。每次投票后，发起投票的服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。

6. 改变服务器状态。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。

7. Leader等待server连接；

8. Follower连接leader，将最大的zxid发送给leader；

9. Leader根据follower的zxid确定同步点；

10. 完成同步后通知follower 已经成为uptodate状态；

11. Follower收到uptodate消息后，又可以重新接受client的请求进行服务了。

## 消息广播

1. Leader 接收到消息请求后，将消息赋予一个全局唯一的 64 位自增 id，叫做：zxid，通过 zxid 的大小比较即可实现因果有序这一特性。
2. Leader 通过先进先出队列（会给每个follower都创建一个队列，保证发送的顺序性）（通过 TCP 协议来实现，以此实现了全局有序这一特性）将带有 zxid 的消息作为一个提案（proposal）分发给所有 follower。
3. 当 follower 接收到 proposal，先将 proposal 写到本地事务日志，写事务成功后再向 leader 回一个 ACK。
4. 当 leader 接收到过半的 ACKs 后，leader 就向所有 follower 发送 COMMIT 命令，同意会在本地执行该消息。
5. 当 follower 收到消息的 COMMIT 命令时，就会执行该消息