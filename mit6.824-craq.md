# MIT6.824 - CRAQ

## Paper: Object Storage on CRAQ -- High-throughput chain replication for read-mostly workloads \(2009\)

[http://nil.csail.mit.edu/6.824/2020/papers/craq.pdf](http://nil.csail.mit.edu/6.824/2020/papers/craq.pdf)

## Chain Replication (CR)

Chain Replication（CR）是一种在多节点之间备份数据，提供强一致性存储接口的方法。

简单来说就是，节点组成一个链表，所有的写请求由链表头部接收，然后向后传导，直到到达尾部节点（此时视为committed）。然后尾部节点将会将响应返回到头部，由头部响应成功（因为实际实现使用的是TCP）。所有的读请求都由链表尾部接收，直接返回。

![5eaf0238c2a9a83be5078e18](https://pic.downk.cc/item/5eaf0238c2a9a83be5078e18.png)

这个模型能够保证是linearizable的，因为write返回的时候会确保所有的replica都收到了，read是从尾部接收的，保证是最新的数据。（在尾部的数据都被所有的副本缓存，不在尾部的数据说明至少没有被尾部节点缓存故没有commit）

当这个模型是没有节点故障的理想模型时

1. write

- 时间复杂度O(n)

- 相比于Raft和ZooKeeper，头节点的网络传输压力较小

  Raft和ZK由leader负责通知所有的Replica，而CRAQ中头节点只需要向后通知即可

2. read

- 时间复杂度O(1)
- 只能由尾节点处理读请求，压力过大

## Chain Replication with Apportioned Queries (CRAQ)

对于读取请求比较多的场景，CRAQ会通过本地读取来尝试提高读取吞吐量。具体设计如下：

1. CRAQ的节点会存储对象的多个版本，并且会标示每个版本是dirty还是clean；
2. 当一个节点得到新版本的写入，会追加到版本列表中；
   1. 如果节点不是尾节点，则标示该版本是dirty的；
   2. 如果是尾节点，则直接标示为clean，然后通过链条去答应通知前面的节点；
3. 前面的节点收到响应后，得知某个版本的节点可以修改为clean；
4. 如果一个节点得到了对象的读取请求；
   1. 如果对象最后一个节点是clean的，则马上响应；
   2. 否则，节点会联系尾节点，询问尾部节点最后一个committed版本。

具体效果如图所示：

![5eafce3cc2a9a83be58fe0d0](https://pic.downk.cc/item/5eafce3cc2a9a83be58fe0d0.png)

## Failure Recovery and Partition

好消息是每个replica都知道每个提交的写操作，但是需要将写了一半的写操作传递下去。
如果head失败，后继节点接管为head，没有提交写丢失。
如果tail失败，前驱接管为tail，没有写丢失。
如果中间节点失败，将节点从链中删除，它的前驱节点可能需要将最新的写操作重新发给它。

CRAQ故障处理相比Raft和ZK来说要更简单，但上面的操作无法应对脑裂的情况。当网络被分成了两个区域，将会有两个leader。

如何安全地使用CRAQ，应对脑裂?

需要用单个configuration manager来管理链路结构（头、链路、尾），所有的服务器和客户端必须听从configuration manager的配置。这是一种常用的模式，如GFS (master)和VMware-FT (test-and-set server)。configuration manager通常用Paxos/Raft/ZK。