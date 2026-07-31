# 一次 `Put` 命令的 Raft 全链路

> 本文以当前仓库的实际代码为准，解释一次客户端 `Put(key, value)` 如何经过 RPC、Raft 日志复制、多数派提交、状态机 Apply，最后返回客户端。
> 文中同时标出标准 Raft 语义与本项目实现的差异。不要把“当前代码能够做到什么”和“Raft 协议要求什么”混为一谈。

## 1. 先给结论：一条 Put 的标准流程

可以先记住下面这条因果链：

```text
Clerk.Put
  -> 选择最近一次知道的节点，发送 kvServerRpc.PutAppend
  -> 目标 KvServer 把请求封装为 Op
  -> Raft.Start 仅在本地 Leader 的 m_logs 追加并持久化
  -> Leader 的 AppendEntries（本项目由心跳 ticker 驱动）向 Follower 复制
  -> Follower 校验 term 和 prevLog，写入日志并返回 success
  -> Leader 更新 matchIndex / nextIndex；多数副本持有当前 term 的日志后推进 commitIndex
  -> Raft::applierTicker 把 commitIndex 之后的日志生成 ApplyMsg
  -> KvServer 消费 ApplyMsg，执行 Put，更新去重表
  -> KvServer 按 logIndex 唤醒原 PutAppend RPC
  -> 校验返回的 (ClientId, RequestId) 仍是原命令，返回 OK
```

最重要的三个边界是：

| 边界 | 发生了什么 | 还不能说明什么 |
| --- | --- | --- |
| RPC 到达 | `RpcProvider` 已经找到并调用 `KvServer::PutAppend` | 不代表是 Leader，也不代表写入成功 |
| `Start` 返回 | Leader 本地追加了一条日志并持久化 | 不代表多数派复制，更不代表状态机已修改 |
| `waitApplyCh` 被唤醒 | 本节点状态机已经按该索引执行 | 不代表所有副本此刻都已经 Apply；客户端只需要 Leader 的线性化确认 |

### 一句话“标答”

`Put` 不是“写入 Leader 的 Log 就成功”，而是“请求先由 Leader 追加到本地日志，再通过 AppendEntries 复制到多数派；Leader 推进 commitIndex 后，Raft 将日志按序 Apply 给 KvServer，KvServer 执行状态机并唤醒等待该日志索引的 RPC，最后客户端才得到 OK”。

## 2. 你的原流程中需要纠正的地方

你的大方向是对的，但有几处概念错位：

1. **两种 RpcUtil 要分开。** `Clerk` 持有的是 `raftServerRpcUtil`，它为每个节点包装 `kvServerRpc_Stub`，只负责 Clerk -> KvServer。每个 `KvServer` 内部另外创建 `RaftRpcUtil`，包装 `raftRpc_Stub`，负责 Raft 节点之间的 `AppendEntries`、`RequestVote`、`InstallSnapshot`。前者不会替客户端“转发给其他节点”。
2. **`KvServer` 不是泛化的 Server。** 它继承生成的 `raftKVRpcProctoc::kvServerRpc`，重写 Protobuf 的 `PutAppend`；同一个 `RpcProvider` 还注册了 `Raft` 对象，因此一个进程同时暴露两个 service。
3. **`Start` 不负责多数派确认。** `Raft::Start` 只做 Leader 身份检查、序列化 `Op`、追加 `m_logs`、持久化，并返回 `logIndex/term/isLeader`。复制由后续 `doHeartBeat` -> `sendAppendEntries` 完成。
4. **AppendEntries 的成功响应不是“回调修改 commitIndex”。** `done->Run()` 是 Protobuf RPC 的网络响应回调，只表示本地 `AppendEntries1` 执行完、可以把 reply 发回去。Leader 在 `sendAppendEntries` 收到 reply 后，持锁更新 `matchIndex/nextIndex`，再按多数派规则推进 `commitIndex`。
5. **多数派确认与状态机 Apply 是两个阶段。** 多数派复制只能使日志“已提交”；`applierTicker` 随后才把提交日志放入 `applyChan`，KvServer 才真正修改跳表。
6. **不是只在 Leader 的心跳里“检查 logsindex”。** Leader 为每个 Follower 维护 `nextIndex[peer]` 和 `matchIndex[peer]`，每轮根据某个 peer 的 `nextIndex` 单独选择 `prevLogIndex/prevLogTerm/entries`。
7. **客户端等待的不是 Raft RPC 返回，而是状态机 Apply。** `KvServer::PutAppend` 在 `waitApplyCh[raftIndex]` 上阻塞，只有 `GetCommandFromRaft` 执行完命令并调用 `SendMessageToWaitChan` 才唤醒它。
8. **超时不等于日志被取消。** `timeOutPop` 超时后原日志仍可能在稍后被提交；客户端会用同一个 `(ClientId, RequestId)` 重试，状态机去重保证不会重复执行。
9. **“表决”要分术语。** `RequestVote` 是选举投票；`AppendEntries` 返回的是日志复制确认（ack）。两者都使用多数派，但不是同一个阶段。

## 3. 启动时各对象如何连接

每个进程的 `KvServer` 构造函数（`src/raftCore/kvServer.cpp:393`）做了这些事：

1. 创建 `Persister`、`applyChan` 和 `Raft` 对象。
2. 启动 `RpcProvider` 线程，注册 `KvServer` 和 `Raft` 两个 Protobuf service。
3. 从配置文件读取所有节点地址，为其他节点建立 `RaftRpcUtil` stub。
4. 调用 `Raft::init`，传入 peer stub、节点编号、持久化器和 `applyChan`。
5. 启动 `KvServer::ReadRaftApplyCommandLoop`，持续消费 `applyChan`。

`Raft::init`（`src/raftCore/raft.cpp:1008`）启动三个长期循环：

| 循环 | 代码位置 | 当前实现 | 作用 |
| --- | --- | --- | --- |
| Leader 心跳 ticker | `leaderHearBeatTicker` | `IOManager` 中的 fiber | Leader 定时调用 `doHeartBeat`，发送空心跳或日志复制 RPC |
| 选举超时 ticker | `electionTimeOutTicker` | `IOManager` 中的 fiber | 非 Leader 超时后调用 `doElection` |
| Apply ticker | `applierTicker` | 独立 `std::thread` | 检查 `commitIndex`，把未 Apply 的日志送入 `applyChan` |

所以“两个协程 + 一个循环线程”这个观察是正确的，但第二个协程不是“另一个心跳线程”，而是**选举超时检查**。

## 4. 阶段一：Clerk 发送 Put

入口是 `Clerk::Put`（`src/raftClerk/clerk.cpp:74`），它调用 `PutAppend(key, value, "Put")`。

### 4.1 请求身份和重试

`Clerk::PutAppend`（`src/raftClerk/clerk.cpp:41`）第一次执行时：

1. `m_requestId++`，为这次逻辑请求生成一个序号。
2. 读取 `m_clientId`。同一个 Clerk 生命周期内它不变。
3. 从 `m_recentLeaderId` 开始尝试节点。
4. 构造 `PutAppendArgs{key, value, op, clientId, requestId}`。
5. 如果 TCP/RPC 失败，或者返回 `ErrWrongLeader`，按环形顺序尝试下一个节点。
6. 收到 `OK` 后把该节点记为新的 `m_recentLeaderId` 并返回。

因此这是“至少一次发送 + 服务端按请求 ID 去重”：网络超时可能发生在服务端已经执行之后，客户端不能凭超时判断请求是否到达，所以重试必须复用同一个 `(ClientId, RequestId)`。

### 4.2 Clerk 侧 RPC 封装

`raftServerRpcUtil::PutAppend`（`src/raftClerk/raftServerRpcUtil.cpp:24`）只做三件事：

1. 创建 `MprpcController`。
2. 调用生成的 `kvServerRpc_Stub::PutAppend`。
3. 用 `controller.Failed()` 区分网络/编解码失败。

业务层的 `ErrWrongLeader` 在 Protobuf reply 里，不是 `controller.Failed()`。这两个错误通道必须分开理解。

## 5. 阶段二：RPC 到达 KvServer

自定义 RPC 的字节路径是：

```text
MprpcChannel::CallMethod
  -> varint(header_size) + RpcHeader(service, method, args_size) + protobuf(args)
  -> TCP send
  -> RpcProvider::OnMessage
  -> 按 service_name / method_name 查表并反序列化
  -> kvServerRpc::CallMethod
  -> KvServer::PutAppend(controller, request, response, done)
```

`KvServer` 的 Protobuf 重载（`kvServer.cpp:381`）调用业务重载 `KvServer::PutAppend(args, reply)`，业务函数返回后执行 `done->Run()`。这里的 `done` 只负责把 response 发回 Clerk，**不是 Raft 的提交通知**。

## 6. 阶段三：KvServer 把请求交给 Raft

`KvServer::PutAppend`（`src/raftCore/kvServer.cpp:220`）先把参数转换为普通 C++ 对象：

```text
Op.Operation = "Put"
Op.Key       = args->key()
Op.Value     = args->value()
Op.ClientId  = args->clientid()
Op.RequestId = args->requestid()
```

然后调用：

```cpp
m_raftNode->Start(op, &raftIndex, &term, &isLeader);
```

### 6.1 `Raft::Start` 做什么

`Raft::Start`（`src/raftCore/raft.cpp:966`）持有 `m_mtx`：

1. 若 `m_status != Leader`，返回 `isLeader=false`、索引和 term 为 `-1`；KvServer 立即返回 `ErrWrongLeader`。
2. 调用 `Op::asString()`，用 Boost 文本归档把完整命令序列化成字符串。
3. 创建 `raftRpcProctoc::LogEntry`：
   - `Command = serialized Op`
   - `LogTerm = m_currentTerm`
   - `LogIndex = getNewCommandIndex()`，即当前逻辑尾索引 + 1
4. 追加到 `m_logs`。
5. 调用 `persist()`，保存 currentTerm、votedFor、快照边界和日志。
6. 返回新日志索引、term 和 `isLeader=true`。

这里的 `m_logs` 不是简单的“从 0 开始的 vector 下标”。快照裁剪后，日志有一个逻辑索引边界 `m_lastSnapshotIncludeIndex`；物理下标通过：

```cpp
physicalIndex = logIndex - m_lastSnapshotIncludeIndex - 1;
```

转换，代码封装在 `getSlicesIndexFromLogIndex`（`raft.cpp:759`）。

### 6.2 KvServer 建立等待点

只有 Leader 的 `Start` 成功后，KvServer 才创建：

```text
waitApplyCh[raftIndex] -> LockQueue<Op>
```

随后调用 `timeOutPop(CONSENSUS_TIMEOUT, &raftCommitOp)`，当前超时配置是 500 ms。它等待的条件不是“AppendEntries 返回”，而是后面状态机线程把同一索引的 `Op` 推入这个队列。

## 7. 阶段四：Leader 的心跳判断与日志选择

### 7.1 Leader 何时发送

`leaderHearBeatTicker`（`src/raftCore/raft.cpp:489`）只在 `m_status == Leader` 时等待心跳间隔；当前配置：

```text
HeartBeatTimeout = 25 ms
minRandomizedElectionTime = 300 ms
maxRandomizedElectionTime = 500 ms
```

睡醒后它检查 `m_lastResetHearBeatTime` 是否在等待期间被更新：

- 已更新：说明这段时间内已有一次心跳触发，继续等待；
- 未更新：调用 `doHeartBeat()`。

`doHeartBeat()` 为每一个 Follower 单独构造一个 AppendEntries 请求。即使 `entries` 为空，它仍然是 AppendEntries RPC，只是通常称作“空心跳”。

### 7.2 每个 Follower 的三个复制状态

Leader 对 peer `i` 维护：

| 状态 | 含义 |
| --- | --- |
| `m_nextIndex[i]` | 下次发给该 Follower 的第一条日志逻辑索引 |
| `m_matchIndex[i]` | 已知该 Follower 与 Leader 匹配到的最大逻辑索引 |
| Leader 的 `m_logs` | Leader 自己的完整（快照边界之后）日志 |

新 Leader 当选后，代码将每个 `nextIndex` 初始化为 `lastLogIndex + 1`，`matchIndex` 置为 0；这意味着先乐观地认为 Follower 与自己一样新，失败后再回退。

### 7.3 `doHeartBeat` 如何选择 `prevLog` 和 `entries`

对 Follower `i`：

1. 若 `m_nextIndex[i] <= m_lastSnapshotIncludeIndex`，旧日志已经被快照裁掉，发送 `InstallSnapshot`，不发送 AppendEntries。
2. 否则调用 `getPrevLogInfo(i, &preLogIndex, &preLogTerm)`：
   - 通常 `preLogIndex = nextIndex[i] - 1`；
   - `preLogIndex` 正好位于快照边界时，`preLogTerm` 使用 `m_lastSnapshotIncludeTerm`。
3. 若 `preLogIndex` 不是快照边界，复制 `m_logs` 中从 `nextIndex[i]` 到 Leader 尾部的所有条目。
4. 若没有新条目，`entries_size() == 0`，这就是空心跳。
5. `leadercommit = m_commitIndex` 一并发送，让 Follower 知道 Leader 已提交到哪里。

所以“logs 选择”不是把整个日志无条件发送，而是以 `nextIndex` 为起点，发送：

```text
prevLogIndex = nextIndex[peer] - 1
prevLogTerm  = Leader 在 prevLogIndex 的 term
entries      = Leader 在 nextIndex[peer] ... lastLogIndex 的后缀
```

## 8. 阶段五：Follower 的 AppendEntries 判断

Follower 入口是 `Raft::AppendEntries1`（`src/raftCore/raft.cpp:8`）。判断顺序非常重要。

### 8.1 任期检查与心跳重置

1. `args.term < m_currentTerm`：拒绝，返回当前 term，**不重置选举定时器**。过期 Leader 的消息不能阻止新选举。
2. `args.term > m_currentTerm`：更新 term、清空 `votedFor`、转为 Follower，并持久化。
3. term 相等时也强制转为 Follower：同一 term 收到合法 Leader 的 AppendEntries，Candidate 不能继续竞选。
4. 通过前面的 term 检查后才执行 `m_lastResetElectionTime = now()`。

这就是“心跳判断”的核心：心跳不是特殊消息，仍然必须经过 term 和日志前缀检查；只有当前或更新任期的 Leader 消息才能重置 Follower 的选举计时器。

### 8.2 前置日志检查

Follower 必须确认 `prevLogIndex/prevLogTerm` 与自己的日志相同：

- `prevLogIndex > getLastLogIndex()`：Follower 太短，返回 `lastLogIndex + 1`。
- `prevLogIndex` 位于快照之前：理论上需要让 Leader 走快照路径。
- `matchLog(prevLogIndex, prevLogTerm)` 为真：前缀匹配，接受 entries。
- 前缀存在但 term 不匹配：拒绝，并返回一个更靠前的 `UpdateNextIndex`。

只有前缀匹配时，Follower 才会逐条检查收到的 entries：索引超出本地尾部就追加；同索引 term 不同就覆盖该位置。最后将：

```text
m_commitIndex = min(LeaderCommit, Follower 的 lastLogIndex)
```

这样 Follower 不会因为 Leader 的 commitIndex 超过自己已经保存的日志而越界。

## 9. “快速恢复”到底是什么

这里的“快速恢复”通常指 Raft 论文之外的 **conflict optimization / fast backup**：日志冲突时，不让 Leader 每次只减 1 个索引。

### 9.1 标准思想

如果 Follower 在 `prevLogIndex` 处 term 不同，Follower 可以告诉 Leader：

```text
conflictTerm  = 我在 prevLogIndex 的 term
conflictIndex = 这个 term 在我日志中第一次出现的位置
```

Leader 收到后：

1. 如果自己也有 `conflictTerm`，跳到该 term 最后一个位置之后；
2. 否则直接跳到 `conflictIndex`；
3. 重新发送 AppendEntries。

这样一次失败可以跨过一整段冲突 term。

### 9.2 本项目的实现

本项目没有 `conflictTerm` 字段，只有 `AppendEntriesReply.UpdateNextIndex`：

- Follower 日志太短：返回 `getLastLogIndex() + 1`；
- `prevLog` term 冲突：从 `prevLogIndex` 向前扫描，找到 term 变化处，返回该段起点之后的索引；
- Leader 在 `sendAppendEntries` 失败分支直接设置 `m_nextIndex[server] = reply.updatenextindex()`。

它比每次 `nextIndex--` 快，但不等同于论文中最完整的 `(conflictTerm, conflictIndex)` 优化。当前代码也不是立刻在失败分支递归重发，而是等待下一轮 heartbeat ticker 再按新的 `nextIndex` 发送。

### 9.3 快速回退和快照不是一回事

不要把两个“恢复”混在一起：

| 名称 | 触发条件 | 解决的问题 |
| --- | --- | --- |
| 快速回退 | Follower 有日志，但 `prevLog` term/index 冲突 | 找到共同日志前缀 |
| InstallSnapshot | `nextIndex <= Leader 的快照边界` | Follower 缺失的日志已经被 Leader 裁剪，无法逐条发送 |
| 本地持久化恢复 | 进程重新启动 | 从磁盘恢复 term、投票、日志和快照边界 |

## 10. 阶段六：Leader 如何认定“已提交”

`sendAppendEntries` 收到成功回复后，在锁内执行：

1. `m_matchIndex[server] = max(old, prevLogIndex + entries_size)`；
2. `m_nextIndex[server] = m_matchIndex[server] + 1`；
3. 本轮成功数达到多数派时，若本轮最后一条日志的 term 等于当前 term，推进 `m_commitIndex`。

为什么要限制“当前 term”？这是 Raft 的关键安全规则：Leader 不能仅凭多数副本存有旧任期日志，就直接把旧任期日志标成 committed；只要当前任期有一条日志提交，之前任期的日志才会因日志顺序一起安全提交。

标准实现通常根据所有 `matchIndex` 计算最大的多数派索引 `N`，再检查 `log[N].term == currentTerm`。本项目保留了 `leaderUpdateCommitIndex()`（`raft.cpp:571`），但当前 `sendAppendEntries` 主要使用本轮 `appendNums` 和本轮 entries 尾索引推进，学习时应知道这是实现策略而非 Raft 唯一写法。

还要注意：Leader 推进 `m_commitIndex` 后，不会通过一个独立“提交回调”通知 KvServer；`applierTicker` 下一次循环才观察到这个变化。

## 11. 阶段七：提交日志如何 Apply 到 KV

### 11.1 Raft 的 Apply ticker

`applierTicker`（`src/raftCore/raft.cpp:155`）循环调用 `getApplyLogs()`：

```text
while (m_lastApplied < m_commitIndex):
    m_lastApplied++
    读取该逻辑索引的 LogEntry
    生成 ApplyMsg{CommandValid=true, Command, CommandIndex}
```

它严格按索引递增生成消息，然后把每条消息推入共享的 `applyChan`。`ApplyMsg` 是 Raft 与状态机之间的接口，不是网络 RPC。

### 11.2 KvServer 应用和去重

`KvServer::ReadRaftApplyCommandLoop` 阻塞 `applyChan->Pop()`，收到 `CommandValid` 后调用 `GetCommandFromRaft`：

1. 从 `message.Command` 反序列化 `Op`。
2. 查询 `m_lastRequestId[ClientId]`。
3. 若 `RequestId <= lastRequestId`，视为重复，不再次修改状态机。
4. 否则执行 `ExecutePutOpOnKVDB`（或 `ExecuteAppendOpOnKVDB`），并更新去重水位。
5. 调用 `SendMessageToWaitChan(op, message.CommandIndex)`，唤醒等待该索引的 RPC。

这说明幂等性属于 KV 服务层，而不是 Raft 层。Raft 只复制一串命令字节，并不知道 `ClientId`、`RequestId` 的业务含义。

### 11.3 原 PutAppend RPC 如何结束

等待线程从 `timeOutPop` 返回后还要比较：

```text
raftCommitOp.ClientId == op.ClientId
raftCommitOp.RequestId == op.RequestId
```

原因是 Leader 可能在等待期间失去领导权，新 Leader 可能在同一 `raftIndex` 写入了不同命令。索引相同并不代表命令相同；比较请求身份可以避免旧 Leader 对错误命令返回 OK。

如果 500 ms 内没有收到 Apply：

- 若 `ifRequestDuplicate` 已经看到该请求，返回 OK；这表示命令可能已经应用，只是通知晚了；
- 否则返回 `ErrWrongLeader`，Clerk 换节点重试。

超时路径不会撤销日志。日志是否最终提交由 Raft 复制和选举结果决定。

## 12. 快照、日志裁剪与落后节点

### 12.1 本地生成快照

KvServer 在 Apply 后检查 Raft 状态大小；超过阈值时：

1. `MakeSnapShot()` 序列化跳表数据和 `m_lastRequestId`；
2. 调用 `Raft::Snapshot(appliedIndex, snapshot)`；
3. Raft 删除该索引及之前的日志，保存 `lastSnapshotIncludeIndex/Term` 与快照。

裁剪后必须同时记住快照最后一条日志的 index 和 term，因为它充当后续 AppendEntries 的“虚拟前置日志”。

### 12.2 Leader 给过于落后的 Follower 安装快照

`doHeartBeat` 发现 `nextIndex[peer] <= lastSnapshotIncludeIndex` 时，启动 `leaderSendSnapShot`：

1. 从 `Persister` 读取快照；
2. 发送 `InstallSnapshot`；
3. 成功后令 `matchIndex[peer] = snapshotIndex`、`nextIndex[peer] = snapshotIndex + 1`；
4. Follower 截断旧日志，更新提交/应用索引，向自己的 `applyChan` 投递 `SnapshotValid`；
5. KvServer 调用 `CondInstallSnapshot` 并恢复跳表和去重表。

这也是“日志选择”的第三种结果：不是选择 entries，而是选择 snapshot。

## 13. 当前代码中必须特别标注的实现边界

下面这些内容对理解代码很重要，但不是标准 Raft 可以直接推出的结论：

1. **Follower 追加逻辑没有完整截断多余后缀。** `AppendEntries1` 逐条覆盖冲突位置，但当前实现没有在“Leader entries 更短、Follower 仍有旧尾部”时显式删除旧尾部；标准 Raft 必须删除冲突点及其后缀，否则可能保留不应存在的日志。
2. **快照边界分支存在风险。** 当 `prevLogIndex < m_lastSnapshotIncludeIndex` 时，代码设置了 `UpdateNextIndex`，但没有立即 `return`，随后仍可能进入 `matchLog`，而 `matchLog` 要求索引不小于快照边界。这个路径需要修正或由更严格的快照发送条件保证不可达。
3. **高 term 的 AppendEntries 回复未完整持久化。** `sendAppendEntries` 发现 `reply.term() > m_currentTerm` 时会转 Follower、更新 term 和 vote，但该分支没有像其他路径一样调用 `persist()`；标准实现应持久化 term/vote。
4. **`Start` 后不是立即复制。** 当前代码注释明确说明新命令等待下一次心跳；这增加客户端延迟，但不改变基本因果链。
5. **`appendNums` 是按一次心跳批次统计成功数。** 标准做法通常直接用所有 `matchIndex` 计算提交点；当前实现保留了 `leaderUpdateCommitIndex`，但没有在主路径使用它。
6. **持久化文件的重启恢复需要谨慎验证。** `Persister` 构造函数会以 truncate 模式清空对应文件，`ReadRaftState`/`ReadSnapshot` 也使用 `operator>>` 读取文本；因此 `readPersist` 的设计意图存在，但不能直接据此断言当前示例已实现可靠的进程重启恢复。
7. **项目中的 `Append` 当前是覆盖而非字符串拼接。** `ExecuteAppendOpOnKVDB` 和 `ExecutePutOpOnKVDB` 都调用 `SkipList::insert_set_element`；学习 Put 流程时不要把接口名误当成实际语义。
8. **Apply 队列和等待表是本地内存机制。** 它们解决的是一个节点内“哪个 RPC 等哪个日志索引”，不参与 Raft 多数派协议，也不会跨进程传输。

## 14. 按故障场景复盘一次 Put

### 场景 A：请求打到 Follower

```text
Clerk -> Follower KvServer -> Start 返回 isLeader=false
       <- ErrWrongLeader
Clerk 换下一个节点，复用同一个 ClientId/RequestId
```

### 场景 B：Leader 已追加但复制超时

```text
Start 已把日志写入 Leader 本地并持久化
AppendEntries 未获得多数派
KvServer 等待 500 ms 后返回 ErrWrongLeader
Clerk 重试
原日志可能后来提交，也可能被新 Leader 覆盖
```

### 场景 C：旧 Leader 等待期间失去领导权

```text
旧 Leader 在 index=N 追加 Op-A
新 Leader 在 index=N 形成 Op-B
旧 Leader 收到 ApplyMsg(index=N, Op-B)
比较 ClientId/RequestId 失败，不能给原请求返回 OK
```

### 场景 D：Follower 断线后恢复

```text
Leader 对该 peer 的 AppendEntries 失败或收到冲突
nextIndex[peer] 根据 UpdateNextIndex 回退
下一轮 heartbeat 从更早的 prevLog 开始重试
若 nextIndex 已落入快照边界，则改发 InstallSnapshot
```

## 15. 最终背诵版

1. Clerk 为一次逻辑 Put 生成稳定的 `(ClientId, RequestId)`，向最近 Leader 发 `PutAppend`。
2. RPC 到达 KvServer；KvServer 把参数封装成 `Op`，调用 Raft `Start`。
3. 非 Leader 立即返回 `ErrWrongLeader`；Leader 将 `Op` 序列化成 LogEntry，追加并持久化，返回逻辑 `raftIndex`。
4. KvServer 为 `raftIndex` 建等待队列，并等待 Apply，不等待某个单独的“提交回调”。
5. Leader heartbeat ticker 对每个 peer 根据 `nextIndex` 选择 `prevLogIndex/prevLogTerm/entries`；必要时改发快照。
6. Follower 检查 term、重置选举计时器、校验前置日志；匹配后追加/覆盖日志并返回成功，同时按 `leaderCommit` 更新自己的 commitIndex。
7. Leader 收到多数成功 ack，确认当前 term 日志达到多数派，推进 commitIndex。
8. 每个节点的 applier ticker 顺序产生 ApplyMsg；KvServer 消费消息、按请求 ID 去重、执行 Put。
9. Leader KvServer 用 `raftIndex -> waitApplyCh` 唤醒原 RPC，并检查 `(ClientId, RequestId)` 防止同索引错配。
10. Clerk 收到 `OK` 返回；若失败或超时则继续轮询节点，重试仍使用同一个请求身份。

只要记住“**追加不等于提交，提交不等于 Apply，Apply 后才返回成功**”，这条 Put 的完整因果链就不会错位。

## 16. 关键函数速查

| 函数 | 作用 | 学习重点 |
| --- | --- | --- |
| `Clerk::PutAppend` | 客户端请求与重试 | 稳定请求 ID、最近 Leader、环形轮询 |
| `raftServerRpcUtil::PutAppend` | Clerk 侧 stub 包装 | 网络失败与业务错误分离 |
| `KvServer::PutAppend` | RPC -> `Op` -> 等待 Apply | `waitApplyCh`、超时、同索引身份校验 |
| `Raft::Start` | Leader 本地追加日志 | 序列化、逻辑索引、持久化、仅 Leader 接收 |
| `leaderHearBeatTicker` | 心跳定时判断 | Leader 状态、心跳间隔、触发 `doHeartBeat` |
| `doHeartBeat` | 构造每个 peer 的 AE/快照 | `nextIndex`、前缀、entries 选择 |
| `AppendEntries1` | Follower 接收复制 | term、选举计时器、日志匹配、leaderCommit |
| `sendAppendEntries` | Leader 处理 reply | term 检查、快速回退、match/next、提交 |
| `leaderUpdateCommitIndex` | 通过 matchIndex 计算提交点 | 多数派与当前 term 约束 |
| `applierTicker` | commit -> ApplyMsg | `lastApplied` 与 `commitIndex` 的顺序差 |
| `KvServer::GetCommandFromRaft` | ApplyMsg -> 状态机 | 反序列化、去重、快照触发、唤醒等待者 |
| `SendMessageToWaitChan` | 唤醒指定索引的 RPC | 本地关联，不是 Raft 网络回调 |
| `Raft::Snapshot` / `leaderSendSnapShot` | 日志裁剪与落后节点恢复 | 逻辑索引、快照边界、InstallSnapshot |

