# redis 的主从复制
可参考：https://www.cnblogs.com/kismetv/p/9236731.html

主从复制是指一个主库下面挂了多台从库，主库的数据单向地复制给从库

主从复制的作用：
1. 数据冗余：实现数据的热备份
2. 故障恢复：主节点挂掉了，可以快速切换到从节点
3. 负载均衡：实现读写分离，减轻主节点的压力，提高 redis 集群的吞吐量
4. 高可用

**主从复制的原理**
主从复制主要过程主要分为三个阶段：
1. 连接建立阶段
2. 数据同步阶段
3. 命令传播阶段

## 连接建立阶段
主从之间建立连接，为数据同步做好准备

流程：客户端调用 slaveof
1. 保存主节点信息，返回客户端 ok
2. 建立 socket 连接
>从节点每秒1次调用复制定时函数replicationCron()，如果发现了有主节点可以连接，便会根据主节点的ip和port，创建socket连接。如果连接成功，则：
从节点：为该socket建立一个专门处理复制工作的文件事件处理器，负责后续的复制工作，如接收RDB文件、接收命令传播等。
主节点：接收到从节点的socket连接后（即accept之后），为该socket创建相应的客户端状态，并将从节点看做是连接到主节点的一个客户端，后面的步骤会以从节点向主节点发送命令请求的形式来进行。

3. 连接建立成功后，发送 ping 命令
> 检查 socket 连接是否可用。如果主节点返回 pong，则可用，可以执行复制过程；如果超时或者返回pong以外的结果，则断开连接，重连

4. 身份验证
5. 发送从节点端口信息给主节点

## 数据同步阶段
redis 2.8 之前发送 sync 命令，执行全量同步；在 redis 2.8 后从节点发送 psync 命令，执行同步。

复制分为两种类型
1. 全量复制
2. 部分复制

**全量同步**
1. 从节点判断无法进行部分复制，发出全量复制的请求；或者从节点发送部分复制的请求，主节点判断无法进行部分复制，则触发全量同步
2. 主节点执行全量同步，执行 bgsave，在后台生成 RDB 文件，并使用一个复制缓冲区记录从现在开始执行的所有写命令
3. 主节点的 bgsave 执行完成后，将 RDB 文件发送给从节点；从节点清除自己的旧数据，然后加载新的 RDB 文件数据
4. 主节点将复制缓冲区的所有写命令发送给从节点，从节点执行写命令，更新数据库到最新状态
5. 如果从节点开启了AOF，则会触发 bgrewrietaof 的执行，保证 aof 文件更新至主节点最新

全量复制是非常重的操作，比如
1. 主节点通过 bgsave fork 子进程进行 RBD 持久化，这个过程非常消耗 CPU、内存（页面复制）、磁盘 IO
2. 主节点发送 RDB 文件给从节点，对主从的带宽会带来很大的消耗
3. 从节点清空老数据、载入 rdb 文件是阻塞，无法响应客户端的命令，从节点执行 bgrewriteaof，也会带来损耗

**部分复制**
理解三个概念

**复制偏移量**
根据偏移量进行复制，主从维护一个复制偏移量，表示主节点向从节点传递的字节数

如果主从的 offset 不一致，则数据状态不一致，则主库需要将 offset 的差量部分的数据发送给从库。

**复制积压缓冲区**
复制积压缓冲区是由主节点维护的、固定长度的、先进先出(FIFO)队列，默认大小1MB；当主节点开始有从节点时创建，其作用是备份主节点最近发送给从节点的数据。注意，无论主节点有一个还是多个从节点，都只需要一个复制积压缓冲区。

在后面的命令传播阶段，主节点除了发写命令给从节点，同时也会将写命令写到复制积压缓冲区，该缓存区是固定长度的，因此当缓冲区满时，则会淘汰掉先写的命令。

因此当从库发起部分复制时，主节点根据offset和缓冲区大小决定能否执行部分复制：
- 如果offset偏移量之后的数据，仍然都在复制积压缓冲区里，则执行部分复制；
- 如果offset偏移量之后的数据已不在复制积压缓冲区中（数据已被挤出），则执行全量复制。

**服务器运行ID（runid）**
redis 节点启动时会生成一个随机 ID（runid），主从节点初次复制后，从节点会保存主节点的 runid，当断线重连时，从节点发送这个 runid，主节点根据 runid 判断能否进行部分复制
- 如果从节点保存的runid与主节点现在的runid相同，说明主从节点之前同步过，主节点会继续尝试使用部分复制(到底能不能部分复制还要看offset和复制积压缓冲区的情况)；
- 如果从节点保存的runid与主节点现在的runid不同，说明从节点在断线前同步的Redis节点并不是当前的主节点，只能进行全量复制。

psync 命令的执行过程
![Alt text](./1593695940152.png)
（1）首先，从节点根据当前状态，决定如何调用psync命令：

- 如果从节点之前未执行过slaveof或最近执行了slaveof no one，则从节点发送命令为psync ? -1，向主节点请求全量复制；
- 如果从节点之前执行了slaveof，则发送命令为psync <runid> <offset>，其中runid为上次复制的主节点的runid，offset为上次复制截止时从节点保存的复制偏移量。
（2）主节点根据收到的psync命令，及当前服务器状态，决定执行全量复制还是部分复制：

- 如果主节点版本低于Redis2.8，则返回-ERR回复，此时从节点重新发送sync命令执行全量复制；
- 如果主节点版本够新，且runid与从节点发送的runid相同，且从节点发送的offset之后的数据在复制积压缓冲区中都存在，则回复+CONTINUE，表示将进行部分复制，从节点等待主节点发送其缺少的数据即可；
- 如果主节点版本够新，但是runid与从节点发送的runid不同，或从节点发送的offset之后的数据已不在复制积压缓冲区中(在队列中被挤出了)，则回复+FULLRESYNC <runid> <offset>，表示要进行全量复制，其中runid表示主节点当前的runid，offset表示主节点当前的offset，从节点保存这两个值，以备使用。

## 命令传播阶段
主从完成复制后，则进入命令传播阶段，该阶段主节点将自己执行的写命令发送给从节点，从节点接受命令并执行，从而保证主从节点数据的一致性。

另外除了发送写命令，主从之间还有心跳机制：PING 和 REPLCONF ACK

**PING**
每隔执行时间（默认10s）主节点发送 PING 给从，该命令主要是为了让从节点进行超时判断

**REPLCONF ACK**
每隔1s，从节点向主节点发送 REPLCONF_ACK{offset} 命令，该命令的作用：
- 检测主从节点的网络状态，主节点用于复制超时的判断
- 检测命令丢失，主节点对比 offset，如果发现从节点数据丢失，主节点会根据 offset 和复制缓冲区来决定需要进行部分复制还是全部复制
- 辅助保证从节点的数量和延迟

## 常见问题
**复制超时问题**
超时判断意义

在复制连接建立过程中及之后，主从节点都有机制判断连接是否超时，其意义在于：

（1）如果主节点判断连接超时，其会释放相应从节点的连接，从而释放各种资源，否则无效的从节点仍会占用主节点的各种资源（输出缓冲区、带宽、连接等）；此外连接超时的判断可以让主节点更准确的知道当前有效从节点的个数，有助于保证数据安全（配合前面讲到的min-slaves-to-write等参数）。

（2）如果从节点判断连接超时，则可以及时重新建立连接，避免与主节点数据长期的不一致。

判断机制

主从复制超时判断的核心，在于repl-timeout参数，该参数规定了超时时间的阈值（默认60s），对于主节点和从节点同时有效；主从节点触发超时的条件分别如下：

（1）主节点：每秒1次调用复制定时函数replicationCron()，在其中判断当前时间距离上次收到各个从节点REPLCONF ACK的时间，是否超过了repl-timeout值，如果超过了则释放相应从节点的连接。

（2）从节点：从节点对超时的判断同样是在复制定时函数中判断，基本逻辑是：

- 如果当前处于连接建立阶段，且距离上次收到主节点的信息的时间已超过repl-timeout，则释放与主节点的连接；
- 如果当前处于数据同步阶段，且收到主节点的RDB文件的时间超时，则停止数据同步，释放连接；
- 如果当前处于命令传播阶段，且距离上次收到主节点的PING命令或数据的时间已超过repl-timeout值，则释放与主节点的连接。