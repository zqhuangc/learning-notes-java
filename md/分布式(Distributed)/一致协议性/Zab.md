## 背景
在分布式系统中实现一致性是件有挑战的事。经典的二阶段提交、三阶段提交都不能完美的解决这一问题，有关传统的的分布式系统一致性问题可以看这里。Paxos 算法能完美地达到分布式系统的一致性，但由于较为复杂，在实际工程上不是很合适，Zab 协议借鉴了 Paxos 的思想，并进行了改进，以满足工程上的实际需求。

## 设计目标
* 一致性
* 有序性：有序性是 Zab 协议与 Paxos 协议的一个核心区别。Zab 的有序性主要表现在两个方面：  
1. 全局有序：如果消息 a 在消息 b 之前被投递，那么在任何一台服务器，消息 a都会在消息 b 之前被投递。
2. 因果有序：如果消息 a 在消息 b 之前发生（a 导致了 b），并被一起发送，则 a 始终在 b 之前被执行。
* 容错性：有 2f+1 台服务器，只要有大于等于 f+1 台的服务器正常工作，就能完全正常工作。
协议内容

### Zab 协议分为两大块：

广播（boardcast）：Zab 协议中，所有的写请求都由 leader 来处理。正常工作状态下，leader 接收请求并通过广播协议来处理。

恢复（recovery）：当服务初次启动，或者 leader 节点挂了，系统就会进入恢复模式，直到选出了有合法数量 follower 的新 leader，然后新 leader 负责将整个系统同步到最新状态。

## 广播（boardcast）

广播的过程实际上是一个简化的二阶段提交过程：

Leader 接收到消息请求后，将消息赋予一个全局唯一的 64 位自增 id，叫做：zxid，通过 zxid 的大小比较即可实现因果有序这一特性。  
Leader 通过先进先出队列（通过 TCP 协议来实现，以此实现了全局有序这一特性）将带有 zxid 的消息作为一个提案（proposal）分发给所有 follower。  
当 follower 接收到 proposal，先将 proposal 写到硬盘，写硬盘成功后再向 leader 回一个 ACK。
当 leader 接收到合法数量的 ACKs 后，leader 就向所有 follower 发送 COMMIT 命令，同事会在本地执行该消息。
当 follower 收到消息的 COMMIT 命令时，就会执行该消息


### 广播过程

相比于完整的二阶段提交，Zab 协议最大的区别就是不能终止事务，follower 要么回 ACK 给 leader，要么抛弃 leader，在某一时刻，leader 的状态与 follower 的状态很可能不一致，因此它不能处理 leader 挂掉的情况，所以 Zab 协议引入了恢复模式来处理这一问题。从另一角度看，正因为 Zab 的广播过程不需要终止事务，也就是说不需要所有 follower 都返回 ACK 才能进行 COMMIT，而是只需要合法数量（2f+1 台服务器中的 f+1 台） 的follower，也提升了整体的性能。

### 恢复（recovery）

由于之前讲的 Zab 协议的广播部分不能处理 leader 挂掉的情况，Zab 协议引入了恢复模式来处理这一问题。为了使 leader 挂了后系统能正常工作，需要解决以下两个问题：  
* 已经被处理的消息不能丢
* 被丢弃的消息不能再次出现

这一情况会出现在以下场景：当 leader 收到合法数量 follower 的 ACKs 后，就向各个 follower 广播 COMMIT 命令，同时也会在本地执行 COMMIT 并向连接的客户端返回「成功」。但是如果在各个 follower 在收到 COMMIT 命令前 leader 就挂了，导致剩下的服务器并没有执行都这条消息。

如图 1-1，消息 1 的 COMMIT 命令 Server1（leader）和 Server2（follower） 上执行了，但是 Server3 还没有收到消息 1 的 COMMIT 命令，此时 leader Server1 已经挂了，客户端很可能已经收到消息 1 已经成功执行的回复，经过恢复模式后需要保证所有机器都执行了消息 1。


图 1-1
为了实现已经被处理的消息不能丢这个目的，Zab 的恢复模式使用了以下的策略：  

选举拥有 proposal 最大值（即 zxid 最大） 的节点作为新的 leader：由于所有提案被 COMMIT 之前必须有合法数量的 follower ACK，即必须有合法数量的服务器的事务日志上有该提案的 proposal，因此，只要有合法数量的节点正常工作，就必然有一个节点保存了所有被 COMMIT 消息的 proposal 状态。  
新的 leader 将自己事务日志中 proposal 但未 COMMIT 的消息处理。  
新的 leader 与 follower 建立先进先出的队列， 先将自身有而 follower 没有的 proposal 发送给 follower，再将这些 proposal 的 COMMIT 命令发送给 follower，以保证所有的 follower 都保存了所有的 proposal、所有的 follower 都处理了所有的消息。  
通过以上策略，能保证已经被处理的消息不会丢
被丢弃的消息不能再次出现

这一情况会出现在以下场景：当 leader 接收到消息请求生成 proposal 后就挂了，其他 follower 并没有收到此 proposal，因此经过恢复模式重新选了 leader 后，这条消息是被跳过的。 此时，之前挂了的 leader 重新启动并注册成了 follower，他保留了被跳过消息的 proposal 状态，与整个系统的状态是不一致的，需要将其删除。  

如图 1-2 ，在 Server1 挂了后系统进入新的正常工作状态后，消息 3被跳过，此时 Server1 中的 P3 需要被清除。  



Zab 通过巧妙的设计 zxid 来实现这一目的。一个 zxid 是64位，高 32 是纪元（epoch）编号，每经过一次 leader 选举产生一个新的 leader，新 leader 会将 epoch 号 +1。低 32 位是消息计数器，每接收到一条消息这个值 +1，新 leader 选举后这个值重置为 0。这样设计的好处是旧的 leader 挂了后重启，它不会被选举为 leader，因为此时它的 zxid 肯定小于当前的新 leader。当旧的 leader 作为 follower 接入新的 leader 后，新的 leader 会让它将所有的拥有旧的 epoch 号的未被 COMMIT 的 proposal 清除。  

## 总结

个人认为 Zab 协议设计的优秀之处有两点，一是简化二阶段提交，提升了在正常工作情况下的性能；二是巧妙地利用率自增序列，简化了异常恢复的逻辑，也很好地保证了顺序处理这一特性。



----------------------------------------
ZAB协议所定义的三种节点状态：  
Looking ：选举状态。  
Following ：Follower节点（从节点）所处的状态。  
Leading ：Leader节点（主节点）所处状态。  

最大ZXID的概念：  
最大ZXID也就是节点本地的最新事务编号，包含epoch和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term

1.Leader election  

选举阶段，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID）。

接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的ZXID所属节点。

每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准Leader，状态变为Leading。其他节点的状态变为Following。

这就相当于，一群武林高手经过激烈的竞争，选出了武林盟主。

2.Discovery

发现阶段，用于在从节点中发现最新的ZXID和事务日志。或许有人会问：既然Leader被选为主节点，已经是集群里数据最新的了，为什么还要从节点中寻找最新事务呢？

这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。

所以这一阶段，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。

各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。

3.Synchronization

同步阶段，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。

自此，故障恢复正式完成。

写数据：
什么是Broadcast呢？简单来说，就是Zookeeper常规情况下更新数据的时候，由Leader广播到所有的Follower。其过程如下：  
1. 客户端发出写入数据请求给任意Follower。
2. Follower把写入数据请求转发给Leader。
3. Leader采用二阶段提交方式，先发送Propose广播给Follower。
4. Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。
5. Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。


应用场景
1. 分布式锁
这是雅虎研究员设计Zookeeper的初衷。利用Zookeeper的临时顺序节点，可以轻松实现分布式锁。
2. 服务注册和发现
利用Znode和Watcher，可以实现分布式服务的注册和发现。最著名的应用就是阿里的分布式RPC框架Dubbo。
3. 共享配置和状态信息
Redis的分布式解决方案Codis，就利用了Zookeeper来存放数据路由表和 codis-proxy 节点的元信息。同时 codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy。
此外，Kafka、HBase、Hadoop，也都依靠Zookeeper同步节点信息，实现高可用。