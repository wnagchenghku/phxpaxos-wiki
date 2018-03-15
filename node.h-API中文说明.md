# class Node

> 所有API无特别说明，均视返回值0为成功，否则异常。更多的返回值定义在`def.h`.

> 所有API参数之一`iGroupIdx`含义均为Paxos Group的Idx。

> 所有API无特别说明，返回值为`Paxos_GroupIdxWrong`均为参数`iGroupIdx`错误。

> 所有API无特别说明，`llInstanceID`均指PhxPaxos所确定的全局有序系列中各个提议对应的ID，这个ID为一个从0开始严格有序连续向上增长的整数。

#### `static int RunNode(const Options & oOptions, Node *& poNode)`
#### 参数
 - **入参**`oOptions`: 运行PhxPaxos的参数设定类，定义在`options.h`.
 - **回参**`poNode`: 调用成功则该值指向成功运行的PhxPaxos实例。

#### 返回值
 - 0: 成功。
 - 非0: 失败。

## 基础API
#### `int Propose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID)`

往集群写入一个值，并且不带状态机。这个API适用范围较窄，一般将PhxPaxos当作纯粹的有序队列来使用才会调用这个API.

#### 参数
 - **入参** `sValue`: 提议的值。
 - **回参** `llInstanceID`: 如果该提议成功被**chosen**，则该值被设为这个提议的全局有序ID。

#### 返回值
 - `PaxosTryCommitRet_OK`: 成功。
 - `PaxosTryCommitRet_Conflict`: 提议与其他节点同时发起的提议冲突，导致没有被chosen，建议对提议重新发起`Propose`.
 - `PaxosTryCommitRet_Follower_Cannot_Commit`: 该节点被设置为follower模式，不允许`Propose`.
 - `PaxosTryCommitRet_Im_Not_In_Membership`: 该节点已被集群剔除，不允许`Propose`.
 - `PaxosTryCommitRet_Value_Size_TooLarge`: 提议的值超过大小限制。
 - `PaxosTryCommitRet_Timeout`: 超时。
 - `PaxosTryCommitRet_TooManyThreadWaiting_Reject`: 超过设置的最大等待线程数，直接拒绝`Propose`.

#### `int Propose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID, SMCtx * poSMCtx)`
往集群写入一个值，并指定状态机和上下文。开发者使用这个API可以进行非常丰富的扩展开发。

> 该API与上面介绍的`Propose`API共享相同的参数与返回值说明。

#### 参数
 - **入参** `poSMCtx`: 状态机的ID以及上下文。

#### 返回值
 - `PaxosTryCommitRet_ExecuteFail`: 状态机转移函数失败。

#### `const uint64_t GetNowInstanceID(const int iGroupIdx)`
获取当前PhxPaxos正在进行的提议所对应的ID，也可认作当前节点所见的最大的ID。

#### `const uint64_t GetMinChosenInstanceID(const int iGroupIdx)`
获取当前节点所保留的Paxos log的最小InstanceID， 也就是当前节点可通过`GetInstanceValue`接口读出的Instance范围的最小值。

#### `const nodeid_t GetMyNodeID() const`
获取当前节点的IP/PORT信息。

## 批量Propose API

### `int BatchPropose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID, uint32_t & iBatchIndex, SMCtx * poSMCtx)`

批量Propose接口，调用此接口的参数含义与上面的Propose一致。需要在`options`设置`bUserBatchPropose`为true才能使用批量接口。

调用此接口后，请求将会进入等待队列，当队列请求个数超过指定阈值或队列堆积时间超过指定阈值，库会自动将这些请求合并成一个Propose请求进行Paxos协议提交。这些请求提交成功后将会获得相同的`InstanceID`，但不同的`BatchIndex`。`BatchIndex`标识这些请求在状态机的确定的执行顺序。`BatchIndex`从0开始。

### `void SetBatchCount(const int iGroupIdx, const int iBatchCount)`

设置批量等待队列最大个数。

### `void SetBatchDelayTimeMs(const int iGroupIdx, const int iBatchDelayTimeMs)`

设置批量等待队列最大等待时间。

## 状态机相关API
#### `void AddStateMachine(StateMachine * poSM)`
添加一个状态机到所有Group.

#### `void AddStateMachine(const int iGroupIdx, StateMachine * poSM)`
添加一个状态机到指定Group.

## Checkpoint相关API
关于Checkpoint的定义与作用请参考资料[状态机Checkpoint详解](https://github.com/tencent-wechat/phxpaxos/wiki/%E7%8A%B6%E6%80%81%E6%9C%BACheckpoint%E8%AF%A6%E8%A7%A3)

#### `void SetHoldPaxosLogCount(const uint64_t llHoldCount)`
设置在Checkpoint之前需要保留的PaxosLog数量。

#### `PauseCheckpointReplayer()`
暂停镜像状态机的状态转移。(需要在Options里面设置`bUseCheckpointReplayer`为`true`才会开启镜像状态机模块)

#### `ContinueCheckpointReplayer()`
继续镜像状态机的状态转移。

#### `PausePaxosLogCleaner()` 
暂停删除PaxosLog, 由于删除PaxosLog也要暂用一定的I/O资源，所有开发者可以随意通过API控制删除PaxosLog的时间。

#### `ContinuePaxosLogCleaner()`
继续删除PaxosLog.

## 成员管理相关API

#### 成员管理相关API返回值统一说明
 - `Paxos_MembershipOp_GidNotSame`: 修改成员的过程中节点GID改变了(GID为集群初始化随机生成的唯一证书)。这错误很罕见，一般是由于数据被人为篡改或其他拜占庭问题导致。
 - `Paxos_MembershipOp_VersionConflit`: 修改冲突，一般由于修改的过程中成员出现变化了，比如其他节点也同时在发起修改成员的操作。针对此错误码建议开发者重新获取当前成员列表以评估是否要再进行重试修改。
 - `Paxos_MembershipOp_NoGid`: 没开启成员管理的集群禁止进行成员管理相关API的调用。集群初始化默认**不开启**成员管理，在Options里面设置`bUseMembership`为`true`则会开启成员管理功能。

#### `int ShowMembership(const int iGroupIdx, NodeInfoList & vecNodeInfoList)`
获取当前集群所有成员列表。

#### `AddMember(const int iGroupIdx, const NodeInfo & oNode)`
往集群添加一个成员。

#### 返回值
 - `Paxos_MembershipOp_Add_NodeExist`: 尝试添加的成员已存在。

#### `RemoveMember(const int iGroupIdx, const NodeInfo & oNode)`
剔除一个成员。

#### 返回值
 - `Paxos_MembershipOp_Remove_NodeNotExist`: 尝试剔除的成员不存在。

#### `ChangeMember(const int iGroupIdx, const NodeInfo & oFromNode, const NodeInfo & oToNode)`
替换一个成员

#### 返回值
 - `Paxos_MembershipOp_Change_NoChange`: 该替换操作对集群成员列表并未产生实质变化(被替换节点不存在且替换节点已存在)。

## Master相关API
Master选举模块也是基于普通状态机实现，开发者需要在状态机`GroupSMInfo`里面设置`bIsUseMaster`才会开启Master选举。

#### `NodeInfo GetMaster(const int iGroupIdx)`
获取当前Master节点信息。

#### `NodeInfo GetMasterWithVersion(const int iGroupIdx, uint64_t & llVersion)`
获取当前Master节点信息以及Master版本号信息。

#### `const bool IsIMMaster(const int iGroupIdx)`
检查当前节点是否Master.

#### `SetMasterLease(const int iGroupIdx, const int iLeaseTimeMs)`
设置Master的租约时间。

#### `DropMaster(const int iGroupIdx)`
放弃当前节点为Master.

## 过载保护相关API

#### `void SetTimeoutMs(const int iTimeoutMs)`
设置`Propose`API的超时时间（注意超时并不意味着该次Propose失败）。

#### `SetMaxHoldThreads(const int iGroupIdx, const int iMaxHoldThreads)`
设置`Propose`API最多挂起的线程。对于每个Group的`Propose`在内部是通过线程锁来进行串行化的，通过设置这个值，使得在过载的情况下不会挂死太多线程。

#### `SetProposeWaitTimeThresholdMS(const int iGroupIdx, const int iWaitTimeThresholdMS)`
设置一个线程挂起等待时间的阈值，系统会根据这个阈值进行`Propose`API的自适应过载保护，当被系统判定需要进行过载保护时，会随机的进行一些`Propose`请求的直接快速拒绝，使得这些请求不会进入线程等待。

## 其他API

#### `SetLogSync(const int iGroupIdx, const bool bLogSync)`
动态设置PhxPaxos写磁盘是否要附加fdatasync，非常有风险的接口，建议由高端开发者谨慎使用。

#### `GetInstanceValue(const int iGroupIdx, const uint64_t llInstanceID, std::string & sValue, int & iSMID)`
通过`llInstanceID`获取对应的提议的值。仅当将PhxPaxos当作纯粹的有序队列来使用才会调用这个API.