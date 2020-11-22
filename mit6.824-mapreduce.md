# MIT6.824 - MapReduce



### Paper: MapReduce \(2004\)

[http://nil.csail.mit.edu/6.824/2020/papers/mapreduce.pdf](http://nil.csail.mit.edu/6.824/2020/papers/mapreduce.pdf)

### Introduction

MapReduce是谷歌提出的一种编程模型，主要目的是为了处理和生成大数据。通过定义map函数来处理key/value对，生成中间键值对，而reduce函数则是用来归并这些中间键值对。

### Example

以经典的word count为例，比如有一个文件，文章中的内容如下：

```text
good better best never it rest 
till good is better and better is best 
```

如果要统计一篇文章中的单词的数目，在MapReduce中是这样的：

[![](https://github.com/sandyhu533/Notes/raw/master/.gitbook/assets/image%20%285%29.png)](https://github.com/sandyhu533/Notes/blob/master/.gitbook/assets/image%20%285%29.png)

1. 首先将文件拆成M个份（M的个数，如何拆可以由用户指定）；
2. 对于M份里的每一份，用map函数处理（这里是将文本按空格拆分，并输出&lt;word, 1&gt;），输出的内容为中间键值对（intermediate key/value pairs）；
3. 对于每份最终产生的键值对，按照一定的规则拆成R组（比如以hash mod R来区分），再把所有的中间键/值对生成的组组合起来；
4. 再对于R份里的每一份，用reduce函数处理（这里是将相同的key组合起来），最终输出；
5. 最终输出的文件往往会被送去做下一步MapReduce；

由于在调Reduce函数之前，相同的key总会被分到同一组。所以每做一次MapReduce，都能将相同的单词整合到一起。

### Implementation

#### Execution Overview

[![Execution overview](https://github.com/sandyhu533/Notes/raw/master/.gitbook/assets/image%20%284%29.png)](https://github.com/sandyhu533/Notes/blob/master/.gitbook/assets/image%20%284%29.png)

1. MapReduce库先将input分成M份\(16MB-64MB\)，然后启动集群上多个机器上的进程；
2. 其中一个进程是master，其它都是worker；
3. 分配了map任务的worker会读取那M份输入的一份，解析键值对，将其传到自定义的Map函数中，产生的中间键值对将会缓存起来；
4. 缓存的内容会被周期性写入到磁盘上，这里磁盘被分成R个区域。写入后的位置信息将会反馈到maser，master再将位置信息传给reduce的worker；
5. reduce的worker将会调用RPC去读取缓存，并根据中间结果的key进行排序，使得相同key的键值对分到一个组；
6. reduce worker将会遍历键值对，然后将key和相关联的values传到自定义的reduce函数里；
7. 当所有任务完成后，master将会从MapReduce中返回；

#### Master Data Structure

* master里面需要存worker的状态\(idle, in-progress, or completed\)和它的身份\(map or reduce\)；
* master需要对中间键值做调度。当map的worker执行完成后会将中间键值的位置告诉master，master会将信息传给reduce的worker；

#### Fault Tolerance

**Worker Failure**

* master会周期性地ping所有的worker状态，将异常的worker标记为failed；
* 下面分情况讨论worker在不同状态failed之后master如何处理
  * idle，此时worker没有执行任何任务，无需做处理；
  * in-progress，此时worker正在执行任务，但都没有完成。计算结果在缓存中，还没有被持久化。所以需要调度一个worker重新执行任务；
  * completed，此时worker已经将任务执行完毕并写入磁盘。对于map worker，因为结果写入的是本地磁盘，所以需要调度一个worker重新执行。对于reduce worker，结果写入了全局共享磁盘，所以不需要重新执行。

**Master Failure**

本可以考虑在增加检查点\(check point\)，当master故障了再从检查点恢复，但考虑到只有一个master，故障的概率比较小所以就不增加了，故障了重新执行一次即可。（这也是MapReduce只适合用来做批处理的原因之一吧）

**Semantics in the Presence of Failures**

当用户的map和reduce函数是确定性的，那么MapReduce产生的结果也是唯一确定的，这是依赖于Map和Reduce任务的原子性提交实现的。

而对于非确定性的Map或者Reduce操作，单个reduce操作的输出对应于整个程序某次序列化输出的结果。

#### Locality

网络带宽是一个很重要的资源，MapReduce充分利用了本地磁盘的好处，将input资源冗余存储，并且尽量让map worker调度到存了input资源的机器，或者隔壁的机器，以此来减少网络上的延迟。

#### Task Granularity

理论上M和R的任务数量应该要比worker机器要多得多，这样使得worker可以执行多种任务，从而提高负载均衡，也可以在某个worker挂掉的时候快速恢复，因为它已经完成的大量map任务都可以重新分配给其它worker机器上执行。

但实际上M和R的数量收到一些限制，因为master进行任务分配决策的复杂度是O\(M+R\)，并且需要在内存中使用O\(M\*R\)大小的空间来保存之前所说的状态。

#### Backup Tasks

MapReduce执行时间长的一个常见原因是“掉队者”\(straggler\)：一台机器在最后几个map或reduce任务上花了很长时间才完成。产生掉队者的原因有：

* 磁盘问题，频繁的可纠正错误使读取性能 30 MB/s ==&gt; 1 MB/s；
* 集群调度系统可能调度了其他任务到该机器上，导致CPU、内存、本地磁盘或网络带宽的争用；
* 初始化代码中的bug：高速缓存被禁用，受影响的计算慢了100倍以上；

缓解掉队者问题的一般机制：

* 当MapReduce计算接近完成时，master调度执行剩余in-progress任务的备份；
* 当原任务或备份之一完成时，该任务就被标记为完成；

### Refinements

#### Partitioning Function

将中间键值分成R组，默认用的是hash mod R的分组算法，可以被重写。

#### Ordering Guarantees

在分组过程中MapReduce会做排序。

#### Combiner Function

用户可以在map完还没有被送去reduce的时候定义一个combiner函数先将数据压缩一遍。

#### Input and Output Types

Mapreduce支持三种文件格式：第一种是逐行读入，key是文件偏移，value是行内容；第二种是key/value读入；第三种是用户自定义reader，可以从文件、数据库或者内存中的数据结构读取。

#### Side-effects

MapReduce允许用户生成额外的输出，但其原子性应该由应用本身来实现

#### Skipping Bad Records

对于一些不好修复的bug，或者确定性的错误。worker通过一个信号处理器来捕获错误，然后在执行Map或者Reduce操作前，MapReduce会存储一个全局序列号，一旦发现了用户代码的错误，信号处理器就会发一个内含序列号的UDP包给master，如果master发现了特定记录有了多次的失败，就会指示该记录应该跳过，不再重试。

#### Local Execution

因为分布式环境调试不方便，MapReduce提供在本机串行化执行MapReduce的接口，方便用户调试。

#### Status Information

master把内部的状态通过网页的方式展示出来

#### Counters

MapReduce提供一个计数器来计算各种时间的发生频率。计数器的值会周期性传达给master。当MapReduce操作完成时，count值会返回给用户程序，需要注意的是，重复执行的任务的count只会统计一次。

### Conclusion

MapReduce容易使用，场景多样且在Google的很多场景得到实践。

在实现它的过程中的体会：

* 限制编程模型能够让分布式系统的实现更简单（包括并行和分布式计算、故障处理等方面）；
* 网络资源很关键，可以从减少网络传输的规模，局部性的优化来缓解网络资源紧张的问题；
* 冗余执行能够被用来减少机器慢、机器故障和数据丢失的影响；

