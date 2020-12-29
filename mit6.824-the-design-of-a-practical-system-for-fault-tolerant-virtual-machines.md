# MIT6.824 - VMware Fault-Tolerant Virtual Machines

## Paper: The Design of a Practical System for Fault-Tolerant Virtual Machines\(2010\)

[http://nil.csail.mit.edu/6.824/2020/papers/vm-ft.pdf](http://nil.csail.mit.edu/6.824/2020/papers/vm-ft.pdf)

## ABSTRACT

一个容错虚拟机分布式系统的设计

## INTRODUCTION

对于分布式系统而言，有很多通用的容错方法：

* 主备服务器：在主服务器挂掉了，由备份服务器接管工作。需要大量带宽在主备间传输状态
* 状态机方法：让两台机器初始化为相同状态，然后接受相同的输入，使得两台机器保持同步。保持两台机器同步的额外信息数量远少于改变主服务器的状态量；然而可能存在一些不确定的操作（如读取时钟），因此必须同步这些不确定操作的结果
* primary和backup之间传递deterministic operation + non-deterministic operation’s result

## Basic FT Design

![](.gitbook/assets/image%20%283%29.png)

上面是FT VMs典型的setup，Primary VM和Backup VM运行在同一个虚拟机上，Backup VM与Primary VM保持完全同步。由于两个VM公用物理存储，所以Primary VM和Backup VM的输入和输出都是可以互通的。但只有Primary VM会暴露给外界使用，所以从外部看就只有一台VM。

所有Primary VM接收到的input都会经过一个叫Logging channel的网络传输给Backup VM，有两种需要被传输的输入：

* dominant input: 来自网络和磁盘的输入
* non-deterministic event and operation: 如软件中断、时钟函数

所有Backup的输入都来自于Hypervisor的转发，Backup的输出都会被Hypervisor拦截并且丢弃，所以对外只表现为Primary VM这一台机器。

对Primary VM或Backup VM的故障检测结合了心跳机制和对Logging channel的监视。

### Deterministic Replay Implementation

正如上文提到过的，让两台机器处于相同的初始状态，然后以相同的顺序提供相同的输入，这样两台机器就能经历相同的状态序列并产生相同的输出。

但由于存在非确定性的事件\(虚拟中断\)或者操作\(读取处理器时钟技术器\)，这样会影响VM的状态。

这里的挑战在于：

1. 如何将input和non-deterministic的信息完全捕获下来
2. 如何将input和non-deterministic正确地作用于Backup VM
3. 如何在实现后不影响效率

实现方式：

1. 将input记录在logging channel中，并已相同的顺序作用于Backup VM
2. 对于non-deterministic，将指令本身和环境信息都记录下来，在replay的过程中event被作用在指令流与Primary VM的相同位置

### FT Protocol

FT协议是用于logging channel的协议

* 输出要求：

> 如果primary宕机了，backup会接管它的工作，并且backup会执行与primary一致的输出

* 输出规则：

> 在backup VM收到并应答所有的日志之前，primary都不会把输出发送给外部

并且，基于这个输出规则来说，primary VM不会停止执行，它只是延迟发送输出。

FT协议的流程如下图

![](.gitbook/assets/image%20%286%29.png)

但这里存在一个小问题，如果 primary 宕机了，backup 不能判断它是在发送了 output 之前还是之后宕机的，因此 backup 会再发送一次 output，但可以通过以下方式解决：

* 诸如TCP等网络协议能够检查丢失或者重复的数据包；

### Detecting and Responding to Failure

如果是backup宕机，primary会停止发送日志。如果primary宕机，情况复杂一点，backup会接替它的工作，在执行完接收到的日志记录之后，成为primary真正对外输出。

存在一些方法检测宕机，比如通过UDP heartbeat来检测primary与backup之间是否正常通信。另外，还会监控logging channel的日志流量。

但这些方法仍然无法解决split-brain问题，即primary和backup同时宕机。为了解决这个问题，该设计使用了共享存储，提供了一个原子操作test-and-set，primary和backup无法同时在该区域操作。





