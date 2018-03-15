# 为何需要Checkpoint

状态机是通过由PhxPaxos选出的有序Chosen value(PaxosLog)系列进行状态转移得到的，而这些状态本质上也是一堆数据，但是这堆数据是由开发者自己进行维护的，我们不能要求开发者在这份数据上**时刻**做到极其严格的正确性, 那么这份数据很有可能因为机器的一些异常或者重启导致丢失，从而影响了数据的正确性。

所以我们的做法是只相信自己的数据，于是乎每次启动(`RunNode`)的时候都会从0开始将所有的PaxosLog重新输入状态机(**重演**)，从而保证状态机数据的完整性。但这个做法缺陷是非常明显的，第一：PaxosLog是无限增长的，每次都重新遍历，重启效率必然很低。第二：由于每次都要遍历，那么这些数据无法删除，从而要求无限的磁盘空间，这是无法做到的。

# 以状态机的Checkpoint来启动PhxPaxos

Checkpoint的出现用于解决这个问题。我们不能要求开发者做到**时刻**的状态机数据正确，但要求开发者生成一个状态机的Checkpoint的时候，这份数据必须是正确的完整的。一个Checkpoint代表着一份**某一时刻**被固化下来的状态机数据，它通过`sm.h`下的`StateMachine::GetCheckpointInstanceID()`函数反馈它的精确时刻，于是每次启动的**重演**，我们只需要从这个时刻所指向PaxosLog位置开始而不是从0开始。下图描述了这一过程：

![](https://github.com/tencent-wechat/phxpaxos/blob/master/doc/images/start_with_checkpoint.jpg)

# 控制PhxPaxos删除PaxosLog

有了一份Checkpoint，我们即可完成PaxosLog的删除，因为`StateMachine::GetCheckpointInstanceID()`之前的PaxosLog都是不再需要的了。

![](https://github.com/tencent-wechat/phxpaxos/blob/master/doc/images/delete_log_before_checkpoint.jpg)

`Node::PausePaxosLogCleaner()`和`Node::ContinuePaxosLogCleaner()`这两个函数控制删除PaxosLog这个线程的运行。因为删除操作是从磁盘中清除数据，所以肯定要占用一些I/O资源，如果开发者不希望在业务高峰期等进行这些操作，则可以通过这两个函数，控制删除操作的运行时间。

`Node::SetHoldPaxosLogCount()`控制我们需要保留多少在`StateMachine::GetCheckpointInstanceID()`之前的PaxosLog。为何我们仍然希望开发者保留一定量的PaxosLog，这里涉及到一个数据对齐的问题。

# 自动传输Checkpoint到其他节点

在没有Checkpoint的情况下，节点之间数据的对齐只需要进行PaxosLog的Learn即可，而加入了Checkpoint，则需要一个Checkpoint+PaxosLog的数据对齐。PaxosLog的对齐是有PhxPaxos自动完成的，但Checkpoint数据是掌握在开发者手里的，所以我们需要`sm.h`里面一些函数来协助我们获取这些数据，以完成一个Checkpoint+PaxosLog的自动数据对齐。

下图展示一个自动对齐数据的流程：

![](https://github.com/tencent-wechat/phxpaxos/blob/master/doc/images/transfer_checkpoint_automatic.jpg)

图示首先展示了需要传输Checkpoint的情况，NodeA所需的PaxosLog在NodeB已经被删除了，这种情况需要进行Checkpoint的传输。这也回答了上文为啥我们建议开发者保留一定量的PaxosLog，因为Checkpoint的传输往往是一个非常重的操作，并且这里还要涉及到非常多的流程，所以我们建议能不出现Checkpoint的自动对齐则不要出现，以保持服务的稳定性。

Checkpoint自动传输流程

> 如开发者使用了Checkpoint，则以下4个函数都需由开发者重载实现。

 - `StateMachine::LockCheckpointState()`用于开发者锁定状态机Checkpoint数据，这个锁定的意思是指这份数据文件不能被修改，移动和删除，因为接下来PhxPaxos就要将这些文件发送给其他节点，而如果这个过程中出现了修改，则发送的数据可能乱掉。
 - `StateMachine::GetCheckpointState()`是Phxpaxos获取Checkpoint的文件列表，从而将文件发送给其他节点。
 - `StateMachine::UnLockCheckpointState()`是当PhxPaxos发送Checkpoint到其他节点完成后，会调用的解锁函数，解除对开发者的状态机Checkpoint数据的锁定。
 - `StateMachine::LoadCheckpointState()`当一个节点获得来自其他节点的Checkpoint数据时，会调用这个函数，将这份数据交由开发者进行处理(开发者往往要做的事情就是将这份Checkpoint数据覆盖当前节点的数据)，当调用此函数完成后，PhxPaxos将会进行进程自杀操作，通过重启来完成一个新Checkpoint数据的启动。