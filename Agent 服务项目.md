## 如果要完成Agent项目 需要先搭建HttpServer框架
HttpServer框架是上层应用层与底层网络通信层沟通的框架。
## C++ HttpServer 项目架构分析
整个项目的核心可以高度抽象为一个输入输出模型：`Raw TCP Bytes -> HttpRequest -> Router -> Middleware -> Handler -> HttpResponse -> Raw TCP Bytes`。
学习路径建议采用 **“入口切入 -> 核心骨架 -> 边缘扩展”** 的顺序：
1. **主干（必须优先吃透）**：`main.cpp` -> `router` -> `http` 解析 -> `handlers`。
2. **基石（决定系统性能）**：网络 I/O 模型与并发（Epoll + ThreadPool）。
3. **组件（工程化能力）**：`session`、`db` 连接池、`middleware`。
---
### 模块拆解与学习指南
#### 1. 框架入口与核心管理 (`GomokuServer/main.cpp` & `GomokuServer.cpp`)
- **简介**：程序的生命周期起点。负责初始化服务器配置、建立监听端口、注册路由规则，以及启动事件循环（Event Loop）。
- **重点学什么**：
    - **API 设计**：观察框架向应用层暴露了什么样的接口。是类似 Express/Gin 的链式调用，还是基于宏或面向对象的注册？
    - **资源管理**：单例模式（Singleton）的应用，以及核心资源的初始化顺序。
- **怎么学**：找到 `main()` 函数，重点看 `server.start()` 之前的路由注册代码和 `server.start()` 内部的 socket 创建逻辑（`socket()`, `bind()`, `listen()`）。

#### 2. HTTP 协议与解析层 (`HttpServer/http`)
- **简介**：网络通信的翻译官。负责将底层 Socket 读到的无边界字节流（Byte Stream）切分并解析为结构化的 `HttpRequest` 对象，并将业务生成的 `HttpResponse` 序列化为字节流发回客户端。
- **重点学什么**：
    - **有限状态机 (FSM)**：这是 HTTP 解析的核心。如何通过状态机依次解析 Request Line（请求行）、Headers（请求头）和 Body（请求体）。
    - **报文边界处理**：如何处理 `\r\n`，如何根据 `Content-Length` 读取 Body，如何处理粘包/半包问题。
    - **长连接机制**：`Connection: keep-alive` 是如何通过定时器或时间轮（Time Wheel）维护的。
- **怎么学**：找到处理 `recv()` 数据的函数，画出它解析 HTTP 报文的状态流转图。重点关注指针移动和字符串提取的效率（是否涉及频繁的内存拷贝）。

#### 3. 路由分发中心 (`HttpServer/router`)
- **简介**：请求的调度枢纽（Router）。根据 `HttpRequest` 中的 URL 路径和 Method（GET/POST），将请求精准投递给对应的业务函数（Handler）。
- **重点学什么**：
    - **数据结构**：路由表是如何存储的？
        - _静态路由_（精确匹配，如 `/login`）：通常使用 `std::unordered_map`。
        - _动态路由/RESTful_（如 `/user/:id`）：通常使用前缀树（Trie Tree）或正则表达式匹配。
    - **回调机制**：C++11 中 `std::function` 和 `std::bind`（或 Lambda）在存储 Handler 时的应用。
- **怎么学**：查看 `Router::addRoute()` 和 `Router::dispatch()`（或同等命名）的实现，搞清楚一个 URL 是如何对应到一个具体的 C++ 函数地址的。

#### 4. 中间件系统 (`HttpServer/middleware`)
- **简介**：请求处理流水线上的拦截器。在路由分发前后执行通用逻辑，如日志记录、跨域处理（CORS）、身份鉴权（Token 校验）等。
- **重点学什么**：
    - **设计模式**：通常采用**责任链模式 (Chain of Responsibility)**。
    - **控制流**：多个中间件是如何链式执行的？如何在中间件中提前终止请求（如鉴权失败直接返回 401，不再进入 Handler）。
- **怎么学**：看框架是否提供了 `use(middleware)` 这样的接口，以及请求遍历中间件队列的循环结构。

#### 5. 状态管理 (`HttpServer/session`)
- **简介**：打破 HTTP 无状态特性的组件。负责在多次独立的 HTTP 请求之间识别“这是同一个用户”，从而维持登录态或游戏对局态。
- **重点学什么**：
    - **Cookie-Session 机制**：服务端如何生成唯一的 SessionID，如何通过 HTTP Header 发给前端（Set-Cookie），前端请求时如何解析。
    - **并发安全**：由于 Session 通常存在内存中的全局 Map 里，高并发下对这个 Map 的读写如何加锁（如读写锁 `std::shared_mutex`）或通过锁分离优化。
    - **过期剔除**：过期的 Session 是如何被清理的（惰性删除 vs 定时任务删除）。
- **怎么学**：结合 `LoginHandler`，观察登录成功后 Session 对象是如何创建、存储，并将 ID 写入 Response 的。

#### 6. 通用工具与持久化 (`HttpServer/utils/db`)
- **简介**：基础设施组件，通常包含 JSON 解析、文件操作和数据库交互。
- **重点学什么**：
    - **数据库连接池 (Connection Pool)**：极其重要。学习如何利用队列管理 MySQL 连接，避免频繁建立/释放连接的开销；RAII 机制在获取和归还连接中的应用。
    - **JSON 处理**：如何将 C++ 结构体与 JSON 字符串进行序列化/反序列化。
- **怎么学**：重点研读 `MysqlUtil.h/cpp`，查看连接池的初始化、获取连接（`getConnection`）和归还连接（`releaseConnection`）的锁机制和条件变量（`std::condition_variable`）的使用。

#### 7. 业务应用层 (`WebApps/GomokuServer`)
- **简介**：基于上述框架搭建的实际应用。定义了五子棋的规则、API 接口和 AI 算法。
- **重点学什么**：
    - **前后端交互**：`LoginHandler`, `AiGameMoveHandler` 等接口是如何解析前端传入的 JSON 数据，操作数据库或调用 AI 后，再组装 JSON 返回的。
    - **核心算法**：`AiGame.cpp` 中的博弈算法（通常是 Minimax 算法配合 Alpha-Beta 剪枝，以及对棋盘局势的启发式评估函数）。
- **怎么学**：将焦点从网络 I/O 转移到业务逻辑，结合五子棋规则，分析 AI 的算力瓶颈与深度限制。
### 调用结构图
#### 服务器启动结构
``` 
main()
  ↓
GomokuServer::GomokuServer()
  ↓
GomokuServer::initialize()
  ├─ MysqlUtil::init()  → 数据库连接池
  ├─ initializeSession() → 会话管理
  ├─ initializeMiddleware() → 中间件
  └─ initializeRouter() → 路由
  ↓
GomokuServer::start()
  ↓
HttpServer::start()
  ↓
TcpServer::start() → 开始监听端口！
  ↓
EventLoop::loop() → 进入事件循环，等待请求
```
#### 客户端返回响应图
```
客户端发送HTTP请求
  ↓
TcpServer::onMessage() → 收到网络数据
  ↓
HttpServer::onMessage() → 开始处理
  ├─ SSL 处理（如果启用）
  └─ HttpContext::parseRequest() → 解析HTTP报文
       ↓
    解析完成 → HttpServer::onRequest()
       ↓
    HttpServer::handleRequest()
       ├─ 1. MiddlewareChain::processBefore() → 中间件前处理
       ├─ 2. Router::route() → 路由匹配
       │    ├─ 静态路由匹配
       │    └─ 动态路由匹配 + 路径参数提取
       ├─ 3. 执行对应的 Handler → 业务逻辑
       └─ 4. MiddlewareChain::processAfter() → 中间件后处理
       ↓
    HttpResponse → 发送回客户端
```
#### 响应时序图
``` mermaid
sequenceDiagram
    participant Client as 客户端
    participant TcpServer
    participant HttpServer
    participant HttpContext
    participant Router
    participant Handler
    participant Session
    participant DB

    Client->>TcpServer: HTTP请求
    TcpServer->>HttpServer: onMessage
    HttpServer->>HttpContext: parseRequest
    HttpContext-->>HttpServer: 解析完成
    HttpServer->>Router: handleRequest
    Router->>Handler: processBefore
    Handler-->>Router: 
    Router->>Handler: route()
    Router->>Handler: handle()
    Handler->>Session: getSession()
    Session-->>Handler: Session
    Handler->>DB: query()
    DB-->>Handler: 结果
    Handler-->>Router: 返回
    Router->>Handler: processAfter
    Handler-->>Router: 
    HttpServer-->>TcpServer: 发送响应
    TcpServer-->>Client: HTTP响应
```