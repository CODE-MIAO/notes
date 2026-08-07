---
title: "mcp改造方案"
weight: 2
---
这是一个基于 **MCP (Model Context Protocol)** 架构的改造方案。我们将从你目前的“内置 Agent”模式，进化为**“分布式代理架构”**（类似 Figma 或 Cursor 的实现方式）。

---

### 一、 改造方案架构图

我们将架构分为三层，通过 **“SSE + Socket”** 双协议链条打通。

```text
[ 表现层：AI 客户端 ] (Claude Desktop / Your UI)
       ↑ 
       | (协议：HTTP / SSE) —— 标准 MCP 协议
       ↓
[ 代理层：MCP Proxy (Python) ] —— “翻译官”
       ↑
       | (协议：TCP Socket / Local Pipe) —— 自定义快速通道
       ↓
[ 执行层：C++ Workbench ] —— “本体” (包含你的 Executor)
```

---

### 二、 改造详细步骤

#### 1. 执行层 (C++): 暴露 Socket 接口
不再让 C++ 直接去连 DeepSeek，而是让 C++ 变成一个“等待指令”的服务端。
*   **新增功能**：在程序启动时开启一个 `QTcpServer` (或 `QLocalServer`)。
*   **注册机制**：定义一个工具注册表，把你的 `WorkbenchAgentExecutor` 能做的操作（如 `openPage`, `editCode`）打包成 JSON。
*   **逻辑**：
    *   收到指令：`{"action": "call", "tool": "openPage", "params": {...}}`
    *   执行逻辑：调用你原有的 `WorkbenchAgentExecutor` 逻辑。
    *   返回结果：通过 Socket 写回 JSON 结果。

#### 2. 代理层 (Python): 桥接 Claude 与 C++
使用 Python MCP SDK 编写一个通用的代理服务器。
*   **动态发现**：Python 启动后先连 C++ Socket，问：“你现在有哪些工具？”
*   **包装工具**：根据 C++ 返回的清单，动态生成 MCP 的 `@mcp.tool` 声明。
*   **透传指令**：
    *   当 Claude 调用工具时，Python 触发 `call_tool`。
    *   Python **不做任何计算**，直接把参数转成 JSON 发给 C++ Socket。
    *   收到 C++ 回复后，原样转交给 Claude。

#### 3. 配置层 (JSON): 注册到 AI 客户端
在 `claude_desktop_config.json` 中添加你的 Python 代理：
```json
"mcpServers": {
  "my_workbench": {
    "command": "python",
    "args": ["path/to/proxy_server.py"]
  }
}
```

---

### 三、 改造后的核心代码逻辑 (伪代码)

**C++ 端 (发送工具清单):**
```cpp
// 只要 C++ 这边多写一个函数，AI 那边自动就能用
json tool_list = {
    {"name": "create_component", "description": "在UI中创建一个新组件", "params": {...}}
};
socket->write(tool_list.dump());
```

**Python 代理端 (通用透传):**
```python
@mcp.tool()
async def universal_handler(name: str, arguments: dict):
    # 将 Claude 的请求通过 Socket 转发给 C++
    cpp_socket.send(json.dumps({"tool": name, "args": arguments}))
    # 等待 C++ 执行结果
    result = await cpp_socket.recv()
    return result
```

---

### 四、 改造后的效果预测

#### 1. **“热插拔”式的功能更新**
*   **改造前**：你在 C++ 增加一个 UI 操作功能，需要修改 AI 调用逻辑、重新处理 Prompt、甚至重新编译整个 Agent 模块。
*   **改造后**：你在 C++ 里写好功能并往 `tool_list` 里加个名字，**重启软件的一瞬间**，Claude 的工具栏里就自动多出了这个按钮。

#### 2. **极致的解耦（Figma 体验）**
*   **改造前**：Agent 嵌在 UI 里，UI 卡顿 Agent 就卡顿；DeepSeek 网络波动，UI 界面也会跟着转圈。
*   **改造后**：Agent 逻辑跑在独立的 Python 进程（Proxy）里。即便你的 C++ Workbench 还没打开，Proxy 也能告诉 AI：“Workbench 离线中，无法操作”。一旦你打开软件，工具立刻“点亮”。

#### 3. **多客户端支持**
*   **改造前**：你的 Agent 只能在你自己写的这个 UI 窗口里用。
*   **改造后**：
    *   你可以在你自己的 UI 里用。
    *   你也可以直接在 **Claude 官方桌面端**里操作你的 Workbench。
    *   你甚至可以在 **Cursor** 里写代码时，直接调用你 Workbench 里的 UI 预览功能。

#### 4. **性能与稳定性**
*   **改造前**：C++ 需要处理复杂的 HTTP 协议、SSL 证书、长连接重连。
*   **改造后**：C++ 只负责最擅长的内存操作和本地 Socket，每秒处理上万次指令毫无压力。复杂的网络握手和 MCP 协议细节全丢给 Python 代理层。

### 五、 总结建议
这个方案将你的 **C++ 程序定义为“器官”**，将 **Python Proxy 定义为“神经接口”**。
这不仅是技术的重构，更是思维的转变：**不要让 AI 住在你的程序里，而是为你的程序开一个标准化的“后门”（MCP），让 AI 从外面伸手进来操作。**

