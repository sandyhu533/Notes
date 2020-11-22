# MIT6.824 - GFS

## Paper: GFS \(2003\)

[http://nil.csail.mit.edu/6.824/2020/papers/gfs.pdf](http://nil.csail.mit.edu/6.824/2020/papers/gfs.pdf)

## Introduction

GFS是由Google设计出来的一个文件系统，用来满足Google快速增长的大数据处理需要。在Google的常年实践中观察出了一些结论：

* 组件的失效是很常见的
* 文件很大\(TB级\)，相比于传统的文件系统设计\(KB级\)，显然系统需要被重构
* 大部分文件的修改都是在后面附加，不改前面的内容
* 开发了很多辅助的插件，增加使用灵活性

## Design Overview

### Architecture

![](.gitbook/assets/image%20%289%29.png)

* 系统由一个master和多个chunkserver构成，被外部的许多client访问
* 文件被分为了很多chunk，被一个全局唯一的chunk handle标识
* 一个chunk会被很多个chunkserver冗余存储在linux文件系统中，默认存储3份
* master管理整个GFS系统的元信息\(metadata\)，包括命名空间、权限管理、文件到chunk的映射、chunk存储的位置；也管理系统级别的活动包括chunk租期管理、对孤儿chunk的垃圾回收以及chunk迁移；还会定期接收chunkserver的心跳并对它下发指令
* client会向master拿chunk handle和它的位置，所有的数据传输都在client和chunkserver之间直接进行

重要设计

* 数据流和控制流分开
* client和chunkserver都不缓存数据
  * 数据太大了，而且chunkserver可以用linux buffer来缓存

### Single Master

client读的全过程：

1. client将文件和偏移制定到具体的某个chunk；
2. client向master发送请求，包含块序号和文件名
3. master相应回chunk的chunk handle以及副本们的位置
4. client向其中一个副本发请求，包含chunk handle和要访问的数据范围
5. chunkserver向client传输数据

### Chunk Size

确定chunk的的大小是一个讲究的事情，目前用了64MB，比block要大得多。chunk size偏大的好处与坏处：

* 好处在于减少master的网络请求负担，降低整体系统的网络链接数量，减少metadata的存储量
* 坏处在于可能会给单个chunkserver带来较大的压力

### Metadata

* master存储的metadata主要有三类，file和chunk的namespace、file到chunk的映射以及每个chunk副本的位置；
* master将metadata都存在了内存里，不仅操作地更快，还方便master对GFS的整个状态做轮询，用来实现垃圾回收、存副本、chunkserver故障的处理以及chunk的迁移、动态平衡等功能
* namespace和file到chunk的映射还会存在operation log里，operation log不仅对这部分信息做了持久化，还支持了同步。master如需要恢复，也会借助于operation log
  * 为避免operation log过长，会用check point定期持久化master并清理operation log；
  * operation log很重要，除了在master本地有存外，还会在别的机器上备份
* file到chunk的映射不在master里面写死，而是在chunkserver发心跳的时候把信息更新给master

### \#Consistency Model

#### Guarantees by GFS

* 命名空间的修改（如创建文件）由GFS来维护原子性和正确性，用命名空间锁和log来保证
* 命名空间内文件的修改取决于修改的类型（是否成功或是否是同步修改）

![](.gitbook/assets/image%20%282%29.png)

* consistent：任意副本内的数据保持一致
* defined：consistent，且修改可见

#### Implication for Applications

* 应用需要分辨defined和undefined
  * 尽量多的做record append而不是write
  * 应用级的checkpoint
  * 做自我验证 自我标识的记录

## System Interaction

interactions包括

* File Read
* File Mutations
  * Write：在应用指定的位置修改
  * Record append：记录被添加至少一次
  * Snapshot：建立快照

系统里优化的目标：

* 保证一致性
* 充分利用网络带宽
  * 减少master的控制
  * 将一些跟数据流相关的控制代理给chunkserver/client

### Lease and Mutation Order

Master给其中一个replica颁发chunk lease，这个replica就变成了the primary

* Primary在所有的修改里面选一个顺序，其他的replica跟随Primary所选的顺序
* 由此整个系统的修改首先被master选中的primary确定，然后由primary管理
* Lease会有一个过期时间，默认为60秒

### Write Access

![](.gitbook/assets/image%20%281%29.png)

1. Client向Master询问Primary和其他副本的位置（如果没有Primary，Master会先选中一个）
2. Master向Client返回Primary和其他副本的位置，Client将这个信息缓存下来（只有当Primary不可答或不再拥有Lease的时候再向Master请求）
3. Client向所有的Replica传输数据（用广播的形式，Client向最近的Replica发送，然后Replica再广播到与自己最近的Replica），chuck server用LRU策略缓存数据信息
4. 当所有的Replica都收到数据后，Client向Primary发出修改请求。Primary首先为这个修改分配顺序，并对自己本地的数据按照既定的顺序做修改
5. Primary将修改请求转发到其他的Replica，Replica跟随Primary的修改顺序做修改
6. 修改成功的Replica向Primary返回成功
7. Primary根据返回的情况，判断修改是成功或者是失败的，向Client返回

* 如果Write的数据太大了，超过了chunk size的限制，GFS客户端会将单个请求拆成小份的多个请求
* 对于一个chunk的同步修改会导致这个region处于undefined状态

### Record Append

* 一个将记录添加到文件末尾的原子操作，由GFS系统保证consistent和defined
* 记录会被GFS至少一次地添加到文件末尾（如果一个mutation只修改了一部分Replica，会返回failed。客户端将retry，那么有一个部分Replica会有两次记录）
* 步骤与Write Access类似，在第4步Primary会检查Append完之后的chunk如果超过了chunk size，将会返回并让Client Retry

### Snapshots

* Snapshot可以让客户端建立分支或者是checkpoint
* GFS使用了Copy-On-Write机制来提高效率、降低延迟
* 在建立Snapshot之后，Master会给另一个Replica颁发新的lease（新分支）。当Client请求Primary的时候，Master不会直接返回，而是询问要哪个版本

## Master Operation

### Namespace Management and Locking

* 对于文件命名空间的修改都会给命名空间上锁，每个目录都有读写锁
* 通过读写锁来控制文件创建/删除/修改的并发性

由于很多master操作会花费很多时间，为了避免master阻塞，允许多种master操作同时running，我们使用锁保证序列化。

GFS逻辑上将namespace表示为**完整路径名映射到元数据的查找表**，并且通过前缀压缩保证了其在内存中的使用，namespace树中的每个节点都有一个读写锁。

每个master操作前都会获取一组锁，如果设计了路径**/d1/d2/…/dn/leaf**，那么就会获得一组关于/d1, /d1/d2, …,  
/d1/d2/…/dn， /d1/d2/…/dn/leaf的锁。

举个例子，当/home/user被快照到/save/user的时候，/home/user/foo的创建是被禁止的。因为快照操作获取/home和/save上的读锁，以及/home/user和/save/user上的写锁。文件创建需要/home和/home/user上的读锁，以及/home/user /foo上的写锁。其中，/home/user的锁产生冲突。

这种方案的一个好处是保障其可以在同一个文件目录并发执行多个文件创建。

### Replica Placement

在GFS集群中，通常有数百个chunk server分布在许多rack上。副本的放置策略有两个目的：最大化数据可靠行和可用性，并最大化网络带宽的利用率。我们必须把chunk的副本分发到不同的rack，这样即使整个rack故障了，这些副本仍然可以存活可用。而且这样在读取的时候也可以利用多个rack的聚合带宽。

* chunk的创建，会考虑以下几种因素：
  * 希望chunk server低于平均磁盘空间利用率；
  * 限制每个chunk server最近创建的数量，因为创建chunk往往意味着后续会有大量写入；
  * 希望在rack上分散chunk的副本；
* re-replication
  * 一旦可用副本的数量低于用户指定的目标，主服务器就会重新复制一个数据块。需要重新复制的每个块根据几个因素进行优先级排序，一个是它与复制目标的距离（比如优先复制丢失了更多副本的块），另外就是优先重新复制活动文件的块，而不是属于最近删除的文件的块。最后，为了最大限度地减少故障对运行应用程序的影响，我们提高了阻止客户端进度的任何块的优先级。 
* rebalancing
  * 协调存储空间的利用
  * 协调网络资源的利用
  * 多个rack的部署来保证HA

### Garbage Collections

GFS使用的是惰性的垃圾回收，不会主动地去清理垃圾。

* 在文件层，将原本文件的chunk以及其位置对应的文件改成hidden的。在master定期扫描文件系统的期间，如果发现其存在已经超过一定间隔（三天），它将删除此类隐藏文件
* 在chunk层，孤儿chunk（不指向可用文件的chunk）会在chunk server发送心跳的时候被master告知，chunk server自己可以将chunk释放
* 垃圾回收只会在master空闲的时候进行，减轻了master的负担

这种回收方法有许多优势：

* 不用担心副本删除信息的丢失，因为heartbeat消息携带了相关信息，可以重试；
* GC被放在master的后台活动中，和定期命名空间扫描等活动一起，使得cost均摊，master可以更加迅速回应其它紧急请求；
* GC的延迟提供了防止意外、不可逆删除的保障；

### Stale Replica Deletion

* 当chunkserver fail了，或者错过了一次mutation，就变成了stale chunk replica
* master会维持一个replica的版本号，在每颁发一个新的grant的时候更新
* Primary也会向其他的Secondary更新版本号
* 如果master接收到一个旧的版本号，会告知将对应的chunk垃圾回收掉

## High Availability

### Fast Recovery

* Master通过operation logs和checkpoint做恢复，恢复到checkout然后重新执行operation logs里记录的操作
* Chunkserver启动的时候会自动向Master发心跳

### Chunk Replication

每个chunk都被复制在不同rack的多个chunkserver上，client可以指定其复制级别（默认为3），master则根据需要克隆现有的副本。

### Master Replication

* operation log和checkpoint会存很多副本
* 只有当一个mutation的所有东西被持久化到了磁盘上，并且被master的所有副本更新后，才被认为是完成了的
* master有主从模式，所有的从master都根据operation log更新自己，并且只读

### Data Integrity

* 每64KB都有32bit的校验和
* 每次数据被读的时候都会验校验和
* 在chunkserver空闲的时候也会扫描验证校验和

### Diagnosis Tools and Analysis of Logs

GFS服务器生成诊断日志，记录许多重要事件，比如上下游的chunkservers，RPC的请求和回复。













