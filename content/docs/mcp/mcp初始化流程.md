---
title: "mcp初始化流程"
weight: 1
---
MCP初始化流程

我们可以通过 MCP 的官方初始化流程来解构这三次交互：

Cursor                          WorkbenchMcpServer
  |                                                                                  | 
  |-- POST /mcp  initialize ------------->|
  |<-- 200  JSON-RPC result -----------|
  |                                      |
  |-- POST /mcp  notifications/init -->|
  |<-- 202  Accepted ---------------------|
  |                                      |
  |-- GET  /mcp  (SSE) ----------------->|
  |<-- 200  text/event-stream ----------|  ← 长连接保持
  |                                      |
  |-- POST /mcp  tools/list ------------>|
  |<-- 200  {tools: [...]} -------------------|
  |                                      |
  |  （之后用户触发时才 POST tools/call）  |

### 1. 第一步：`initialize` 请求 (Client -> Server)

- **动作**：客户端（如 Claude Desktop、Chainlit 宿主）向服务端发送一个 `initialize` 请求。
- **内容**：客户端会说明自己支持的 MCP 协议版本（`protocolVersion`），以及自己的客户端信息和支持的能力（`capabilities`，比如是否支持 roots、sampling 等）。
- **性质**：这是一个 **JSON-RPC Request**（必须带 `id`，等待回复）。

### 2. 第二步：`initialize` 响应 (Server -> Client)

- **动作**：服务端收到后，返回一个响应。
- **内容**：服务端同样告知客户端自己所采用的协议版本（必须与客户端匹配）、自身的服务端信息（如名字、版本），以及自己能提供什么能力（`capabilities`，比如支持哪些 tools、prompts、resources）。
- **性质**：这是一个 **JSON-RPC Response**（带上一步的 `id`，代表对请求的成功回应）。

### 3. 第三步：`initialized` 通知 (Client -> Server)

- **动作**：客户端收到服务端的响应并确认无误后，发送一个 `initialized` 通知给服务端。
- **内容**：这个通知不需要任何具体的 payload，它的存在只是为了告诉服务端：“我已经收到你的能力报告了，现在初始化正式完成，你可以开始处理我的常规请求（如 `tools/list`）了。”
- **性质**：这是一个 **JSON-RPC Notification**（**没有 `id`**，属于单向告知，服务端不需要也不应该对这个通知进行任何回复）。

如果你平时用的是 Python 的 `fastmcp`（或者官方的 TypeScript SDK），**这些初始化流程全部都被框架在底层封装好了**。你只需要写 `@mcp.tool()`，框架会自动监听输入、处理这三步初始化，然后直接把数据喂给你的函数。

但是，如果你准备在 **Qt (C++)** 程序中**内置（手写）一个 MCP 服务端或客户端**，情况就完全不同了。

## 1. 在 Qt 中内置 MCP，你需要自己写什么？

因为目前 MCP 官方并没有提供现成的 C++ SDK（大部分是 Python, TS, Go 和 Java），所以在 Qt 中实现 MCP，你确实需要**自己处理初始化流程**。

不过别慌，这并不意味着你要从零造所有轮子。你可以把任务拆解为以下几个层次：

### ⚙️ 传输层（Transport）— 借助 Qt 现成组件

不管是 `stdio` 还是 `sse`，Qt 都有很好的支持，你不需要自己写底层的 Socket 或进程管道通信：

- **如果用 `stdio` 方式**：用 `QProcess`。通过读写 `QProcess` 的标准输入输出流来收发数据。
- **如果用 `sse` 方式**：用 `QNetworkAccessManager`。利用 HTTP 长连接来监听服务端推过来的事件。

### 📦 协议层（JSON-RPC 2.0）— 借助 Qt JSON

你需要解析和构建符合 JSON-RPC 2.0 规范的文本。

- 你可以使用 Qt 自带的 `QJsonDocument`、`QJsonObject` 和 `QJsonValue` 来手动解析 `id`、`method`、`params`。
- **关键点**：你需要自己写一个简单的消息分发器（Dispatcher）。根据收到的 `method`（比如 `initialize`），路由到对应的 C++ 处理函数。

### 🤝 应用层（MCP 初始化）— 必须自己手动实现

这里就是你说的“三步初始化”了。以你在 Qt 中实现一个 **MCP Client（客户端）** 为例，你的代码逻辑大致需要这样写：

C++

```
// 伪代码示例：在 Qt 中实现 MCP 客户端初始化

// 1. 发送 initialize 请求
void McpClient::sendInitializeRequest() {
    QJsonObject request;
    request["jsonrpc"] = "2.0";
    request["id"] = ++m_currentId; // 自增 ID
    request["method"] = "initialize";

    QJsonObject params;
    params["protocolVersion"] = "2024-11-05"; // 写入当前 MCP 版本
    
    QJsonObject clientInfo;
    clientInfo["name"] = "MyQtAppClient";
    clientInfo["version"] = "1.0.0";
    params["clientInfo"] = clientInfo;
    
    // 声明你的 Qt 客户端支持的能力（按需填写）
    QJsonObject capabilities; 
    params["capabilities"] = capabilities;

    request["params"] = params;

    // 发送给服务端（通过 QProcess 或 QNetworkAccessManager）
    sendRawJson(request); 
}

// 2. 接收服务端的 Response 并处理
void McpClient::onMessageReceived(const QJsonObject& jsonDoc) {
    // 检查是不是对 initialize 的响应
    if (jsonDoc["id"].toInt() == m_initializeId) {
        QJsonObject result = jsonDoc["result"].toObject();
        QString serverVersion = result["protocolVersion"].toString();
        
        // 校验版本并记录服务端的能力 (Tools, Resources 等)
        this->serverCapabilities = result["capabilities"].toObject();

        // 3. 触发第三步：发送 initialized 通知
        sendInitializedNotification();
    }
}

// 3. 发送 initialized 通知
void McpClient::sendInitializedNotification() {
    QJsonObject notification;
    notification["jsonrpc"] = "2.0";
    notification["method"] = "notifications/initialized"; // 注意：没有 id！

    sendRawJson(notification);
    
    // 到这里，初始化正式完成！接下去你可以开始调用 tools/list 了
    emit mcpReady(); 
}
```

## 2. 避坑指南：在 Qt 中实现 MCP 的核心注意事项

1. **多线程与异步**：

   MCP 的 JSON-RPC 是异步的。比如你发送了一个 `tools/call` 请求，可能要过几秒钟大模型或工具才会返回响应。在 Qt 中，绝对不能用阻塞（Block）的方式去等待回复，一定要利用 **Qt 的信号与槽（Signals and Slots）机制**，通过 `id` 来匹配请求和响应。

2. **标准输出的污染问题（如果是 `stdio` 方式）**：

   如果你把 Qt 程序作为 MCP Server 供外部调用，一定要注意：**不要在代码里随便使用 `qDebug()`**！因为 `qDebug()` 默认会把调试信息输出到 `stdout` 或 `stderr`。如果污染了 `stdout`，对方的 MCP Client 就会因为解析到非 JSON 文本而直接崩溃。在 MCP Server 中，所有的调试日志必须重定向输出到本地文件。

3. **微服务架构的对齐**：

   如果你的 Qt 程序本身采用的是模块化或微服务设计（例如内部用了类似 CppMicroServices 的架构），你可以把 MCP 抽象成一个独立的 Service 模块。让它负责和外界（比如 Python 写的 AI Agent 宿主）通信，收到 Tool 调用指令后，再通过 Qt 内部的事件总线分发给具体的 3D 重建或图像处理业务模块。

