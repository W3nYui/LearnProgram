# `RpcProvider::OnMessage` 如何找到并执行 RPC 方法

本文以一次 `kvServerRpc.Get` 调用为主线，解释 `RpcProvider::OnMessage` 如何从 TCP 字节流找到正确的 RPC 方法、如何把字节还原为 `GetArgs` / `GetReply`，以及 `done->Run()` 为什么能把结果发回客户端。

结论先说清楚：

1. 业务方法名不是从 C++ 函数指针或字符串拼接中猜出来的，而是来自 `.proto` 生成的 **Protobuf 描述符**（`ServiceDescriptor`、`MethodDescriptor`）。
2. `OnMessage` 先用请求中的 `service_name` 和 `method_name` 查两层表，取得 `google::protobuf::Service*` 与 `MethodDescriptor*`。
3. `service->CallMethod(...)` 是 Protobuf 的反射入口。对于 `kvServerRpc`，生成代码内部按方法下标 `switch`，再通过 C++ 虚函数调用最终的 `KvServer::Get` 或 `KvServer::PutAppend`。
4. `done` 不是业务返回值，而是框架预先绑定好的“响应发送动作”。业务方法完成并调用 `done->Run()` 后，才会序列化 `reply` 并通过原来的 TCP 连接发回。

## 0. 先区分：哪些代码由谁负责

这是理解动态调用最重要的边界。你不需要、也不应该手写“按 `method->index()` 调 `Get`”的代码；那一层已经由 `protoc` 根据 `.proto` 自动生成。

| 部分 | 文件或类型 | 由谁提供 | 你是否应该修改 | 职责 |
| --- | --- | --- | --- | --- |
| RPC 接口契约 | `src/raftRpcPro/kvServerRPC.proto` | 你 | 是，新增/修改 RPC 的唯一入口 | 声明服务、方法、请求和响应消息。 |
| 服务端接口、Stub、描述符、`CallMethod` | `kvServerRPC.pb.h` / `.pb.cc` | `protoc` | 否 | 把 `.proto` 翻译成 C++ 类型、元数据与“描述符 -> 强类型虚函数”的分派代码。 |
| 业务服务 | `KvServer : kvServerRpc`，其 `Get` / `PutAppend` 重写 | 你 | 是 | 写真正的业务逻辑，并在结果写入 `response` 后执行 `done->Run()`。 |
| 通用客户端传输 | `MprpcChannel` | 你 | 是 | 将 Stub 给出的描述符和消息编码为字节，发送并把响应字节写回调用方的 `reply`。 |
| 通用服务端传输与路由 | `RpcProvider` | 你 | 是 | 注册 Service、解帧、查找描述符、创建消息、交给 `Service::CallMethod`、回包。 |
| 网络事件与发送 | Muduo `TcpServer` / `TcpConnection` | Muduo | 通常否 | 在收到字节时调 `OnMessage`，并提供 `conn->send`。 |
| 反射接口、消息序列化、Closure | Protobuf 库 | Protobuf | 否 | 提供 `Service`、`Message`、描述符、`NewCallback` 等基础能力。 |

特别注意两层“动态”：

```text
你写的 RpcProvider：  service_name + method_name -> MethodDescriptor
protoc 生成的代码：   MethodDescriptor::index() -> Get / PutAppend 虚函数
```

前一层由你构建路由表；后一层不由你手写。两层合起来，才把网络中的字符串和字节恢复成对强类型业务函数的调用。

## 1. 参与者与契约

`src/raftRpcPro/kvServerRPC.proto` 是 KV RPC 的唯一接口定义：

```proto
service kvServerRpc {
  rpc PutAppend(PutAppendArgs) returns(PutAppendReply);
  rpc Get(GetArgs) returns(GetReply);
}
```

其中 `option cc_generic_services = true` 很关键：它要求 `protoc` 额外生成旧式 C++ RPC 接口，包括：

- 服务端抽象基类 `raftKVRpcProctoc::kvServerRpc`；
- 客户端代理类 `kvServerRpc_Stub`；
- 服务/方法/入参/出参的描述符；
- `CallMethod`、`GetRequestPrototype`、`GetResponsePrototype` 的实现。

生成文件是由 `.proto` 自动产生的，不应手改。当前工程的实作类是：

```cpp
class KvServer : raftKVRpcProctoc::kvServerRpc { ... };
```

因此一个 `KvServer` 对象同时具有两种身份：

- **业务对象**：处理 Raft、KV 数据和一致性等待；
- **Protobuf Service**：能向框架报告“我有哪些 RPC 方法、每个方法的输入输出类型是什么、如何按描述符调用我”。

## 2. 客户端究竟发送了什么

例如客户端执行：

```cpp
stub->Get(&controller, args, reply, nullptr);
```

这里的 `stub` 是 `kvServerRpc_Stub`。它的生成实现并不直接调用服务器上的 `KvServer::Get`，只做一件事：

```cpp
channel_->CallMethod(descriptor()->method(1), controller, request, response, done);
```

也就是说，Stub 将“调用 `Get`”转化为：

- `method`：`kvServerRpc` 的第 1 个 `MethodDescriptor`；
- `request`：调用者已经填好的 `GetArgs`；
- `response`：调用者准备接收结果的 `GetReply`；
- `channel_`：工程实现的 `MprpcChannel`。

`MprpcChannel::CallMethod` 从描述符读取路由信息，而不是手写 `if (Get)`：

```cpp
const auto* service = method->service();
std::string service_name = service->name(); // "kvServerRpc"
std::string method_name = method->name();   // "Get"
```

然后它序列化 `request`，组装如下请求帧：

```text
+----------------------+------------------------------+---------------------+
| Varint32(header_size) | protobuf(RpcHeader)          | protobuf(GetArgs)   |
+----------------------+------------------------------+---------------------+
                         service_name = "kvServerRpc"
                         method_name  = "Get"
                         args_size    = GetArgs 的字节数
```

这里的 `RpcHeader` 定义在 `src/rpc/rpcheader.proto`。`service_name` 与 `method_name` 是服务器选择业务方法所需的“路由键”；`args_size` 则让服务器知道后面有多少字节属于业务请求。

## 3. 服务注册：动态查找表从哪里来

服务器启动时，`KvServer` 构造函数会做：

```cpp
RpcProvider provider;
provider.NotifyService(this);              // 注册 kvServerRpc
provider.NotifyService(m_raftNode.get());  // 注册 raftRpc
provider.Run(...);
```

注意：同一个 TCP 端口发布了两个 Service，因此服务器必须依据每次请求中的服务名分流，而不能在网络回调中写死 `KvServer::Get`。

`RpcProvider::NotifyService` 做的事情可以近似理解成：

```cpp
auto* service_desc = service->GetDescriptor();
ServiceInfo info;
info.m_service = service;

for (int i = 0; i < service_desc->method_count(); ++i) {
  auto* method_desc = service_desc->method(i);
  info.m_methodMap[method_desc->name()] = method_desc;
}
m_serviceMap[service_desc->name()] = info;
```

对 KV 服务而言，注册后的内存关系是：

```text
m_serviceMap
  └─ "kvServerRpc"
       ├─ m_service   -> 当前节点的 KvServer 对象
       └─ m_methodMap
            ├─ "PutAppend" -> kvServerRpc::descriptor()->method(0)
            └─ "Get"       -> kvServerRpc::descriptor()->method(1)
```

`MethodDescriptor` 不是可执行代码；它是协议元数据，描述了方法名、所属服务、输入消息类型、输出消息类型和下标。真正可执行的分派发生在后面的 `Service::CallMethod`。

## 4. `OnMessage` 的逐步执行

`RpcProvider::OnMessage` 是 Muduo 在连接上有可读数据时调用的回调。以下以包头内容 `{ service_name: "kvServerRpc", method_name: "Get" }` 为例。

### 4.1 从网络缓冲区取出本次字节

```cpp
std::string recv_buf = buffer->retrieveAllAsString();
```

随后代码以 `ArrayInputStream` 和 `CodedInputStream` 在 `recv_buf` 上读取，不需要先把字节转换成某个固定的 C++ 请求类型。

### 4.2 先解帧，再解析通用 RPC 头

```cpp
uint32_t header_size{};
coded_input.ReadVarint32(&header_size);

auto limit = coded_input.PushLimit(header_size);
coded_input.ReadString(&rpc_header_str, header_size);
coded_input.PopLimit(limit);

RPC::RpcHeader rpc_header;
rpc_header.ParseFromString(rpc_header_str);
```

解析成功后得到：

```text
service_name = "kvServerRpc"
method_name  = "Get"
args_size    = 后续 GetArgs 的长度
```

之后 `ReadString(&args_str, args_size)` 读出原始业务参数。此时服务器只知道它是一串字节，尚未把它认作 `GetArgs`。

### 4.3 依据名称找到服务与方法描述符

```cpp
auto service_it = m_serviceMap.find(service_name);
auto method_it = service_it->second.m_methodMap.find(method_name);

google::protobuf::Service* service = service_it->second.m_service;
const google::protobuf::MethodDescriptor* method = method_it->second;
```

对于这个例子，结果是：

```text
service -> KvServer*，但静态类型被擦除为 google::protobuf::Service*
method  -> 描述 kvServerRpc.Get 的 MethodDescriptor
```

这就是“得到 RPC 方法”的第一层含义：先从字符串路由键取得描述符，而非取得 `KvServer::Get` 的成员函数指针。

### 4.4 依据方法描述符创建正确的请求与响应对象

`OnMessage` 无法在编译期写出某个固定的消息类型，因为同一个 Provider 也可能收到 `raftRpc.AppendEntries`。所以它使用 Service 的原型对象：

```cpp
google::protobuf::Message* request =
    service->GetRequestPrototype(method).New();
request->ParseFromString(args_str);

google::protobuf::Message* response =
    service->GetResponsePrototype(method).New();
```

对 `kvServerRpc.Get`，生成的 `kvServerRpc::GetRequestPrototype` 等价于：

```cpp
return raftKVRpcProctoc::GetArgs::default_instance();
```

因此 `.New()` 的实际结果是 `new GetArgs`。同理，响应原型为 `GetReply`，于是实际结果是 `new GetReply`。不过变量的静态类型保持为共同基类 `google::protobuf::Message*`，Provider 不必了解每一种业务消息。

这就是“解析远端 `KvServiceRPC` 的参数”的精确含义：不是把“远端服务对象”反序列化过来，而是根据远端传来的服务名/方法名选择本地同一 `.proto` 生成的 `GetArgs` 类型，再把参数字节反序列化到这个本地对象中。

### 4.5 创建 `done`：把“发响应”包装成一次性回调

```cpp
google::protobuf::Closure* done =
    google::protobuf::NewCallback<RpcProvider,
        const muduo::net::TcpConnectionPtr&,
        google::protobuf::Message*>(
        this, &RpcProvider::SendRpcResponse, conn, response);
```

可以把上面这一段读成概念等价代码：

```cpp
done = [this, conn, response] {
  this->SendRpcResponse(conn, response);
};
```

但实际类型不是 C++ lambda，而是 Protobuf 的 `Closure` 对象。它保存了三类信息：

- 要调用的成员函数：`RpcProvider::SendRpcResponse`；
- 成员函数所属对象：当前 `RpcProvider` 的 `this`；
- 预先绑定的实参：当前连接 `conn` 与刚创建的 `response`。

所以业务层只需在适当时机执行 `done->Run()`。它完全不需要知道 Muduo 连接、序列化或 TCP 发送的细节。

### 4.6 反射入口调用到最终的 `KvServer::Get`

最后 `OnMessage` 调用：

```cpp
service->CallMethod(method, nullptr, request, response, done);
```

这行是第二层、也是最容易被误解的“动态调用”。`service` 的静态类型是 `google::protobuf::Service*`，动态对象实际是 `KvServer`；而 `CallMethod` 的实现来自 `protoc` 生成的 `kvServerRpc` 基类。生成代码的核心逻辑如下：

```cpp
switch (method->index()) {
  case 0:
    PutAppend(controller,
              DownCast<const PutAppendArgs*>(request),
              DownCast<PutAppendReply*>(response), done);
    break;
  case 1:
    Get(controller,
        DownCast<const GetArgs*>(request),
        DownCast<GetReply*>(response), done);
    break;
}
```

本例中 `method->index() == 1`，因此发生的实际调用是：

```text
KvServer::Get(RpcController*, const GetArgs*, GetReply*, Closure*)
```

`KvServer` 重写的这个四参数版本是一个适配层：

```cpp
void KvServer::Get(RpcController*, const GetArgs* request,
                   GetReply* response, Closure* done) {
  KvServer::Get(request, response);  // 调用项目自己的业务实现
  done->Run();                       // 通知框架：response 已写好
}
```

业务实现会经过 Raft 提交与等待；最终把 `Err`、`Value` 写入 `GetReply`。完成后才运行 `done`，这保证 `SendRpcResponse` 序列化的是已经填充好的结果。

## 5. `reply` 如何回到调用方

服务端的 `done->Run()` 调用上面绑定的 `SendRpcResponse(conn, response)`：

```cpp
void RpcProvider::SendRpcResponse(const TcpConnectionPtr& conn,
                                  google::protobuf::Message* response) {
  std::string response_str;
  if (response->SerializeToString(&response_str)) {
    conn->send(response_str);
  }
}
```

客户端的 `MprpcChannel::CallMethod` 此前一直同步阻塞在 `recv`。收到字节后，它执行：

```cpp
response->ParseFromArray(recv_buf, recv_size);
```

这里的 `response` 正是最开始调用方传给 Stub 的 `GetReply* reply`。因此所谓“得到远端结果”是：

```text
服务端 response (新建的 GetReply)
  -> SerializeToString
  -> TCP
  -> 客户端 recv
  -> ParseFromArray
  -> 调用方原有的 GetReply* reply 被填充
```

并没有跨网络传递 C++ 对象，更没有让两个进程共享 `reply` 指针；跨进程传递的永远是 Protobuf 字节。

## 6. 与 `rpcExample` 的关系：它其实使用同一套动态分派

`example/rpcExample` 看起来更“静态”，因为业务代码明确写了：

```cpp
stub.AddFriend(...);
stub.GetFriendsList(...);
```

但它和 KV 服务的底层机制相同：

```text
调用方的静态语法              Stub 的描述符调用              Provider 的动态分派
------------------           ---------------------          ----------------------
stub.Get(...)          ->    channel->CallMethod(...)   ->  service->CallMethod(...)
stub.AddFriend(...)    ->    channel->CallMethod(...)   ->  service->CallMethod(...)
```

`FriendService` 也继承了 `FiendServiceRpc`，注册后同样通过 `NotifyService` 建立“服务名 -> 方法描述符”的表。客户端 Stub 同样会把其 `MethodDescriptor` 交给 `MprpcChannel`。所以 RPC 示例不是另一套静态网络调用实现，而是同一套框架的较小协议与较简单业务。

真正的区别在于**代码所处的一侧**：

| 位置 | 是否在编译期知道具体方法 | 为何 |
| --- | --- | --- |
| 客户端业务代码 | 是 | 调用者写的是 `stub.Get(...)`，参数类型也是 `GetArgs` / `GetReply`。 |
| `MprpcChannel` | 否 | 一个 Channel 必须承载所有服务和方法，只能读 `MethodDescriptor`。 |
| `RpcProvider::OnMessage` | 否 | 同一端口同时可能收到 KV RPC 或 Raft RPC，必须从包头路由。 |
| 最终业务类 `KvServer` | 是 | 生成的 `switch` 已将描述符下标还原为强类型 `GetArgs*`，再以虚函数调用重写方法。 |

因此可以把整个框架理解为“外层动态、内层恢复静态类型”：网络边界只传名称和字节，Provider 用描述符选择类型，生成代码再以已知的 `switch + 虚函数` 调到真正的强类型业务函数。

## 7. 一次 `Get` 的完整时序

```text
Clerk / raftServerRpcUtil
  | stub->Get(controller, &args, &reply, nullptr)
  v
kvServerRpc_Stub::Get
  | descriptor()->method(1)
  v
MprpcChannel::CallMethod
  | 组装 RpcHeader("kvServerRpc", "Get", args_size) + GetArgs 字节
  | TCP send
  v
RpcProvider::OnMessage
  | 解包 RpcHeader 与 args_str
  | m_serviceMap["kvServerRpc"].m_methodMap["Get"]
  | request = GetArgs 原型.New(); request.ParseFromString(args_str)
  | response = GetReply 原型.New()
  | done = SendRpcResponse(conn, response) 的 Closure
  v
kvServerRpc::CallMethod(method #1, ..., done)   [protoc 生成]
  | switch(1) -> 虚函数 Get(...)
  v
KvServer::Get(RpcController*, GetArgs*, GetReply*, Closure*)
  | 业务 Get：提交/等待 Raft，填充 GetReply
  | done->Run()
  v
RpcProvider::SendRpcResponse
  | SerializeToString(GetReply) + conn->send(...)
  v
MprpcChannel::CallMethod
  | recv + reply.ParseFromArray(...)
  v
调用方读取 reply.err() / reply.value()
```

## 8. 阅读源码时建议按这个顺序

1. `src/raftRpcPro/kvServerRPC.proto`：确定服务、方法、请求、响应的契约。
2. `src/raftRpcPro/kvServerRPC.pb.h` 和 `.pb.cc`：只读地查看生成出的 `Stub`、`CallMethod`、原型函数，理解桥接机制；不要修改它们。
3. `src/rpc/mprpcchannel.cpp`：理解客户端如何把描述符和消息编码为 TCP 请求。
4. `src/rpc/rpcprovider.cpp`：理解服务端如何反向解码、查表、创建消息对象和设置回调。
5. `src/raftCore/kvServer.cpp`：看 `KvServer` 的四参数 RPC 适配函数如何调用实际业务逻辑并执行 `done->Run()`。

## 9. 当前实现需要知道的边界

本文解释的是现有实现的意图与路径。若继续把它用于并发或大消息测试，还需注意以下工程限制：

- `OnMessage` 直接 `retrieveAllAsString()` 并假设收到一个完整请求；TCP 是字节流，真实网络中一次回调可能只有半个请求，也可能包含多个请求。生产实现应保留缓冲区并按完整帧循环解析。
- 服务端响应没有长度前缀；客户端只用一次固定大小 `recv(..., 1024, ...)`。响应超过 1024 字节或被 TCP 分段时会解析失败或截断。
- 客户端 `CallMethod` 是同步阻塞模型，并且当前连接上没有请求 ID/响应匹配机制，因此一个 Channel 不适合并发复用多个未完成请求。
- `OnMessage` 用 `.New()` 创建的 `request` 与 `response` 没有在当前代码路径中显式释放。需要明确所有权，或改用智能指针/在回调完成时回收。
- `NewCallback` 创建的是一次性 Closure；业务方法应对每次请求恰好调用一次 `done->Run()`。漏调会让客户端一直等，重复调会重复发送响应。

这些限制不改变“描述符查找 -> 原型建对象 -> `CallMethod` 分派 -> `done` 回传”的核心原理，但决定了框架在真实网络条件下还需要补齐的部分。

## 10. 关键源码定位

- 服务表结构与注册：`src/rpc/include/rpcprovider.h`、`src/rpc/rpcprovider.cpp` 的 `NotifyService`。
- 请求解码、查表、反射调用、Closure 绑定：`src/rpc/rpcprovider.cpp` 的 `OnMessage`。
- 回包：`src/rpc/rpcprovider.cpp` 的 `SendRpcResponse`。
- 客户端封包与回包解码：`src/rpc/mprpcchannel.cpp` 的 `MprpcChannel::CallMethod`。
- KV 服务注册与适配层：`src/raftCore/kvServer.cpp` 的构造函数及四参数 `Get` / `PutAppend`。
- 生成的动态分派实现：`src/raftRpcPro/kvServerRPC.pb.cc` 的 `kvServerRpc::CallMethod`。

## 11. 以后新增一个业务服务：你实际要写的内容

假设要增加订单服务，不需要改 `RpcProvider::OnMessage`，也不需要触碰任何 `*.pb.*`。正确流程如下。

### 第一步：修改协议，并重新生成代码

```proto
syntax = "proto3";
package example;
option cc_generic_services = true;

message CreateOrderRequest { string product_id = 1; }
message CreateOrderReply { string order_id = 1; string err = 2; }

service OrderServiceRpc {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderReply);
}
```

运行项目既有的 Protobuf 生成流程后，生成器会提供：

```cpp
class OrderServiceRpc : public google::protobuf::Service { ... };
class OrderServiceRpc_Stub : public OrderServiceRpc { ... };
```

以及 `OrderServiceRpc::CallMethod`。**这部分只读，不手改。**

### 第二步：实现业务类，并重写生成的四参数函数

```cpp
class OrderService : public example::OrderServiceRpc {
 public:
  void CreateOrder(google::protobuf::RpcController* controller,
                   const example::CreateOrderRequest* request,
                   example::CreateOrderReply* response,
                   google::protobuf::Closure* done) override {
    // 1. 使用 request 执行业务逻辑。
    // 2. 将业务结果写入 response。
    response->set_order_id(Create(request->product_id()));
    // 3. 精确调用一次：触发 Provider 绑定的回包动作。
    done->Run();
  }
};
```

这一步由你实现。`request`、`response`、`done` 均由 `RpcProvider` 在收到请求后创建或绑定；业务类只使用它们，不负责网络编码。

若业务是异步的，不能在函数返回时立刻调用 `done`，而要等异步工作真正完成后再调用一次。此时必须让 `response`、`done` 和必要的连接状态存活到异步回调结束，且必须设计取消、超时和仅回包一次的保护。

### 第三步：在启动时注册实例

```cpp
RpcProvider provider;
OrderService order_service;
provider.NotifyService(&order_service);
provider.Run(...);
```

调用 `NotifyService` 后，新的 `OrderServiceRpc` 与 `CreateOrder` 自动进入通用表。`OnMessage` 不需要知道订单服务的存在：它只依赖请求中带来的 `"OrderServiceRpc"` 和 `"CreateOrder"`。

## 12. 以后自己构建或扩展 `RpcProvider`：必须具备的逻辑

对于当前这种基于 Protobuf 通用 Service 的设计，一个可扩展 Provider 的核心不是“拿字符串调用成员函数”，而是维持下列边界清晰的七步。

```text
1. 注册：Service* -> 服务描述符 -> 服务表 / 方法表
2. 解帧：TCP Buffer -> 一条完整 RPC 请求
3. 解析头：请求 -> service_name、method_name、args_size
4. 路由：两个名称 -> Service*、MethodDescriptor*
5. 建对象：MethodDescriptor* -> 正确的 request / response Message
6. 分派：service->CallMethod(...) -> protoc 生成的 switch -> 业务虚函数
7. 完成：done->Run() -> 序列化 response -> 在同一连接发送响应
```

### 12.1 第 1 步：注册表必须保存什么

最小结构就是当前 `RpcProvider` 头文件中的形式：

```cpp
struct ServiceInfo {
  google::protobuf::Service* service;
  std::unordered_map<std::string,
                     const google::protobuf::MethodDescriptor*> methods;
};
std::unordered_map<std::string, ServiceInfo> services;
```

注册时不要自己维护一份 `"Get" -> 1` 的下标表。应始终从 `service->GetDescriptor()` 读取服务名和方法描述符，这样 `.proto` 变动后，描述符和生成的 `CallMethod` 仍然保持一致。

还要定义服务对象的所有权：当前工程保存的是裸指针，因此 `RpcProvider` 假定注册的 Service 在 Provider 运行期间一直存活。若未来 Provider 拥有服务实例，注册表应改为 `std::shared_ptr<google::protobuf::Service>` 或在外层明确管理生命周期，不能让 `m_serviceMap` 悬空。

### 12.2 第 2 和第 3 步：先解决 TCP 分帧，才谈解析

Provider 的协议必须能够判断“这是不是一条完整请求”。当前头部只携带 `args_size`，因此完整请求长度是：

```text
Varint32(header_size) 实际占用的字节数
+ header_size
+ args_size
```

一个稳健的 `OnMessage` 应保留尚未消费的 Buffer 内容：数据不足时返回等待下一次可读事件；数据完整时只消费一帧；若 Buffer 还有下一帧则继续循环。这是动态调用能够正确运行的前提，因为不能把半个 `GetArgs` 错当成完整请求。

同时应校验：`ReadVarint32`、读取 header、`RpcHeader::ParseFromString`、`args_size` 上限及完整参数读取的返回值。未知服务或未知方法也应转化为明确的错误响应，而不是仅打印后让客户端无期限阻塞。

### 12.3 第 4 到第 6 步：动态只到框架边界为止

分派器的核心应保持为下面的抽象形态：

```cpp
Service* service = FindService(header.service_name());
const MethodDescriptor* method = FindMethod(service, header.method_name());

auto call = std::make_shared<CallContext>();
call->request.reset(service->GetRequestPrototype(method).New());
call->response.reset(service->GetResponsePrototype(method).New());
call->request->ParseFromString(args);

// done 持有 call，直到它发送响应并完成清理。
Closure* done = MakeResponseClosure(conn, call);
service->CallMethod(method, controller, call->request.get(),
                    call->response.get(), done);
```

实际的所有权写法可以不同，但职责不能混淆：Provider 只做类型无关的 `Message` 与 `Service` 操作；它绝不为每一个新业务服务增加 `if (service == "...")` 或 `if (method == "...")`。强类型转换和具体虚函数调用留给 `protoc` 生成的 `CallMethod`。

`CallContext` 是说明所有权的伪类型：其中保存本次调用的 `request` 与 `response`。同步业务中，`CallMethod` 返回后即可回收请求；异步业务中，请求、响应和 `done` 都必须活到异步完成。当前代码没有处理这两个对象的释放，因此扩展时应该先明确这个规则。

### 12.4 第 7 步：回调是“完成协议”，不是装饰

`done` 的契约应写清楚：

- 业务成功、业务错误、超时或取消都必须有终止路径；
- 每条请求至多发送一次响应；
- 回调执行前，`response` 已填好；
- 回调执行后，框架可以释放 `response`、`request` 与 Closure；
- 客户端连接已经关闭时，回包应安全失败，不得访问悬空连接。

这样设计后，同步服务只需“填 `response`，调用 `done`”；异步服务也可在未来完成时调用同一 `done`。业务代码不依赖 Muduo，Provider 也不依赖任何具体业务类型。

### 12.5 一个判断标准

当你添加新 RPC 方法时，若需要修改 `RpcProvider::OnMessage` 的业务分支，说明框架的动态边界被破坏了。正确情况下，你只修改 `.proto`、重新生成、重写服务虚函数并注册服务；Provider 的查表、原型创建、`CallMethod` 和回包逻辑应该原封不动地支持它。
