# MIT6.824 - Zookeeper

## Paper: ZooKeeper: Wait-free coordination for Internet-scale systems\(2010\)

[http://nil.csail.mit.edu/6.824/2020/papers/zookeeper.pdf](http://nil.csail.mit.edu/6.824/2020/papers/zookeeper.pdf)

## Introduction

The ZooKeeper interface enables a high-performance service implementation. In addition to the wait-free property, ZooKeeper provides a per client guarantee of FIFO execution of requests and linearizability for all requests that change the ZooKeeper state. These design decisions enable the implementation of a high performance processing pipeline with read requests being satisfied by local servers.

### Raft Review

回顾Raft，Raft确实是一个能够保证linearizability的系统，但当Raft增加了机器的数量时，他的效率得到提高了么？

对于写和读两种操作，都需要系统里大部分的机器得到复制之后才能commit，所以他的时间复杂度是O\(n\)，意味着每增加一个机器就要慢一点。

这样的系统虽然保证了局部容错性（CAP里面的P），但效率缺大打折扣。

那是否能够通过只在一台机器上面读来增加读的效率呢（尝试将读的时间复杂度变成O\(1\)）？但我们发现只在一个副本上读时，会有一些问题：

1. 可能这个副本不在达成共识的大多数上面，导致stale read
2. client commit了一个写操作后，可能读不到自己的结果
3. 这个副本可能已经跟系统剥离了
4. 如果client先看到的是一个新的副本，然后再看到一个旧的副本，会发现自己原本能看到的一些东西后面又看不到了

在这个基础上ZooKeeper希望能够提高效率，提高效率的方式是改变强一致性的定义。

### Zookeeper

在大多数场景下，我们只需要考虑一个客户看到的操作是linearizability就行，对于不同的客户有的看到的相对新，有的看到的相对旧其实没有那么关键。也就是在上面提到的有关Raft粗暴版本的read优化里的几个问题，对于问题1是能够容忍的，主要解决2、3、4。保证单个client读操作的顺序。

## Ordering Guarantees

* Linearizable writes

  clients send writes to the leader

  the leader chooses an order, numbered by "zxid"

  sends to replicas, which all execute in zxid order

  this is just like the labs\(raft\)

* FIFO client order

  each client specifies an order for its operations \(reads AND writes\)

  * writes:

    writes appear in the write order in client-specified order

    this is the business about the "ready" file in 2.3 \(?\)

    client可以一次发很多个来write异步地执行，在client端为这些writes标记好了顺序，server端按照client的顺序来做。

  * reads:

    each read executes at a particular point in the write order

    a client's successive reads execute at non-decreasing points in the order，即使当client切换replica的时候也不会读到旧的版本。因为client本地会保存当前读到的最新的'zxid'，换replica的时候会讲自己读到的zxid号发过去。

    a client's read executes after all previous writes by that client.

### 

