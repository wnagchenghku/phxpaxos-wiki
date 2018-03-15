**本页会持续补充微信自身在使用PhxPaxos上遇到的一些问题以及一些经验**

### 关于磁盘的选择

PhxPaxos的写磁盘会产生比较多的随机写，所以对sata并不友好，建议在SSD的基础上使用PhxPaxos。
当然如果仍然想在sata上使用，则需要开发者自行实现LogStorage(参见[如何使用自己的存储以及网络模块来构建PhxPaxos](https://github.com/tencent-wechat/phxpaxos/wiki/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%E8%87%AA%E5%B7%B1%E7%9A%84%E5%AD%98%E5%82%A8%E4%BB%A5%E5%8F%8A%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9D%97%E6%9D%A5%E6%9E%84%E5%BB%BAPhxPaxos))，以更好的适配sata磁盘。

### fdatasync的性能问题

在对磁盘进行分区的时候，如果不进行pagesize的对齐，会导致fdatasync的性能大幅下降，所以要先检查磁盘分区是否已进行pagesize对齐。

### fdatasync带来的写放大问题

fdatasync每次刷盘是以pagesize为最小单位进行（可能是4KB），那么在进行一些小数据的写入的时候，每次刷盘都会放大为4KB，从而使得IO出现瓶颈。解决这个问题可以通过调整options参数控制刷盘间隔，但这不是一个很好的做法，更好的做法是将多个写入合并成一次写入，从而使得一次fdatasync能写入更多的请求。目前我们内部版本已经支持合并写(`BatchPropose`)，我们会尽快merge到github上来。

### 关于PhxPaxos在LoadCheckpointState后会进行自杀

首先这里自杀的目的是为了方便程序以新的Checkpoint状态机数据来进行重启，那么会涉及到如何重启的问题。PhxPaxos只负责自杀，不负责重启，开发者需要自行解决重启的问题。我们微信内部一般会通过守护进程的方式来自动拉起工作进程。

其次当你使用到PhxPaxos多个Group的特性的时候，那么当多个Group整体落后非常多的时候，每个Group都需要各自进行Checkpoint的对齐，那么每个Group都要经历一次自杀的操作，想象如果有100个Group，那么程序可能要经过100次重启才能完成Checkpoint的对齐，效率非常低下。这时候开发者需要根据自己的业务特性，在`LoadCheckpointState`的函数过程中进行一些延缓等待操作，使得一次自杀可以完成更多Group的Checkpoint对齐。

### 关于SetHoldPaxosLogCount的设置

[状态机Checkpoint详解](https://github.com/tencent-wechat/phxpaxos/wiki/%E7%8A%B6%E6%80%81%E6%9C%BACheckpoint%E8%AF%A6%E8%A7%A3)提到，PaxosLog保留越多，就越少几率触发Checkpoint的同步，在有条件的情况下必然使保留越多越好。实际情况应该根据业务请求数据大小情况，算一下这个值对应的数据有多少，并根据自身的磁盘情况来决定，还有就是要算一下这个值对应的产生这么多请求需要的时间，实际就是你能容忍机器离线落后并不进行Checkpoint拉取的时间。

### 使用BatchPropose带来的checkpoint更新问题

一般Checkpoint InstanceID(后面简称CPID)的更新，可在状态机Execute执行成功后，则更新到当前执行的InstanceID。

但由于BatchPropose使得多个Propose请求获得相同的InstanceID，那么在状态机Execute阶段，会连续Execute相同的InstanceID里面的不同Value，这时候如果第一个Value就更新了CPID，而后面一些Value执行失败，而刚好这时候又发生重启，那么这个InstanceID的Values将不再会被Execute，从而导致丢失数据。这是因为CPID代表着[0, CPID]的Instance都已经被完全Execute完了。

如何解决这个问题，比较简易的做法是，始终更新CPID为当前Execute成功的InstanceID - 1.

相关讨论Issue [#56](https://github.com/Tencent/phxpaxos/issues/56)

### 更严格的使用Master功能

目前PhxPaxos提供的Master具有严格唯一性，这使得我们在限定写入操作只发生在Master上的话，那么Master上就可以立即读出最新数据，使得读取效率非常高。但其实这还不够严格，因为限定在Master写入，不代表限定了在Master上生效，而只有限定在Master上生效，才能满足Master读出最新数据的要求。

这里列一个写入步骤，然后进行问题分析。
 - 1. 通过`GetMaster`获得Master信息，判定自己是否Master，如果是则继续，否则退出。
 - 2. 进行`Propose`操作进行写入。
 - 3. 进行状态机的`Execute`操作。

这里可以看到一个时间差，在`Propose`之前是Master，不代表在`Execute`的时候还是Master（为什么会出现这个偏差，这是跨线程同步原因，具体大家可以分析），这样就使得一个写入操作在非Master上生效，而这时候最新的Master上则不一定能立即读出这个写入的数据。

一个简单的修改即可解决问题，使得最终我们达到严格限定写入在Master生效的要求。修改后的步骤如下：
 - 1. 通过`GetMasterWithVersion`获得Master信息和Master版本号，判定自己是否Master，如果是则继续，否则退出。
 - 2. 进行`Propose`操作进行写入，写入数据携带Master版本号。
 - 3. 进行状态机的`Execute`操作， Execute过程中判断携带Master版本号是否与当前Master版本号相同，如果不同则拒绝这次写入。

### 关于PhxPaxos会产生巨量的日志

由于Paxos算法的复杂性，为方便于调试和学习，PhxPaxos在各个环节都输出了非常详细的日志，这些日志都是调试时候必不可少的。

但大量的日志输出会损耗很大的性能，特别是直接使用比如glog这种同步写磁盘的方式，打日志将卡住大部分的函数运行时间，使得整体吞吐严重下降。
对于线上有高吞吐要求的生产环境的话，有两个建议：
 - 1. 使用异步打日志方法，比如将日志先缓存到内存，然后通过其他线程进行刷写磁盘，这样写日志不会卡住函数的运行。
 - 2. 直接关闭日志。

### 待持续补充