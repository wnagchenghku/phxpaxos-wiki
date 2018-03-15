**PhxPaxos类库本身是一个相对小巧的作品，但难以避免涉及了数据存储以及网络通信，虽然我们认为我们提供的默认存储与网络通信模块已经做得不错，但可能很多开发者自身已经有成熟的存储或者网络组建，他们更希望使用自己的组建来构建paxos算法以便更好的维护。这没问题，PhxPaxos一直秉持非常开放的态度，从第一个版本开始就已经将核心的算法高度抽象，使得存储以及网络都可以由开发者自行定制。**

# 使用自己的存储模块

### 从`include/phxpaxos/storage.h`开始

这个头文件是存储模块的基类，开发者通过继承这个基类来使用自己的存储模块，构建起来非常简单，但也有一些注意事项。

- 对于每个写操作函数，`WriteOptions`传入了写入的要求选项，其中当`bSync`设置为`true`的时候，存储必须要保证这次写操作严格落到磁盘后才能返回。也就是说，当这次写操作成功后，即使发生了机器重启，一样要能读出这个数据，可采用`fsync`等方法保证此类操作。
- 每个函数都有一个参数叫做`iGroupIdx`, 这个一个从`[0, GroupCount)`的编号，`GroupCount`是多少？这个是开发者传给PhxPaxos的，开发者自然知道。
- `GetMaxInstanceID`函数是获得所有写入过的InstanceID里面最大的，必须严格是最大的，不能大概。
- `ClearAllLog`函数是清除掉所有通过`Put`写入过的数据，但不包括`SystemVariables`与`MasterVariables`。
- 所有函数，成功返回0，不成功返回其他。
- 所有`Get`相关函数，数据不存在返回1。
- 必须严格实现所有纯虚函数，缺一不可。

### 通过`include/phxpaxos/options.h`设置自己的存储模块。

选项参数为`Options::poLogStorage`, 注意，poLogStorage必须在PhxPaxos的Node实例之后进行析构。

# 使用自己的网络模块

### 从`include/phxpaxos/network.h`开始

这个头文件是网络通信模块的基类，继承这个类即可实现自定义的网络模块。必须严格实现所有纯虚函数，下面介绍里面的几个函数。

- `OnReceiveMessage`这个函数不需要开发者实现，而是当网络模块收到消息的时候，进行这个父函数的调用，这个函数会将消息放进PhxPaxos的内存队列并立即返回。
- `RunNetWork`启动网络通信函数，在调用此函数之前，网络模块不能进行任何收发消息，也就是不能有调用`OnReceiveMessage`。
- `StopNetWork`停止网络通信函数，调用此函数后，必须停止所有消息接收。
- `SendMessageTCP`通过TCP进行消息发送。
- `SendMessageUDP`通过UDP进行消息发送。


### 通过`include/phxpaxos/options.h`设置自己的网络模块。

选项参数为`Options::poNetWork`, 注意，poNetWork必须在PhxPaxos的Node实例之后进行析构。