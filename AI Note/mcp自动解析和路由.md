这是一个非常深刻的问题。答案是：**MCP 服务器确实做到了“自动解析和路由”，但并不是完全不需要写代码，而是通过 SDK 把繁琐的“脏活”给屏蔽了。**

如果你使用官方提供的 **SDK**（比如 Python 的 `mcp` 库或 Node.js 的 SDK），它实现这种“自动化”效果的核心秘密在于：**装饰器（Decorators/Annotations）** 和 **类型自省（Introspection）**。

下面我为你拆解它是如何实现“不需要手动写解析逻辑”的：

### 1. 它是怎么做的？（以 Python FastMCP 为例）

当你写下这段代码时，你其实已经完成了所有工作：

```python
# 开发者只需要写这几行逻辑代码
@mcp.tool()
def calculate_weather(city: str, unit: str = "celsius") -> str:
    """获取指定城市的温度"""
    return f"{city} 的温度是 25 {unit}"
```

**背后发生的自动化流程如下：**

1.  **自动生成声明（Registration）**：
    *   SDK 会利用 Python 的 `inspect` 库读取你的函数名、参数名、参数类型（`str`）和文档字符串（Docstring）。
    *   它自动把这些信息转换成符合 MCP 标准的 **JSON Schema**。这样客户端（Claude）就知道这个工具长什么样了。

2.  **自动路由（Routing/Dispatching）**：
    *   当 MCP 服务器收到客户端发来的 `tools/call` 请求（一个 JSON）：
        ```json
        { "method": "tools/call", "params": { "name": "calculate_weather", "arguments": {"city": "Beijing"} } }
        ```
    *   **SDK 内部维护了一个字典**（Registry）。它看到 `name` 是 `calculate_weather`，就直接去字典里找对应的函数引用。

3.  **自动解析与校验（Parsing & Validation）**：
    *   SDK 会把 JSON 里的参数（`{"city": "Beijing"}`）映射到函数的形参上。
    *   如果类型不符，SDK 会在调用函数前直接拦截并报错，不需要你写 `if not isinstance(city, str): ...`。

4.  **自动打包返回（Result Wrapping）**：
    *   函数执行完 return 的结果，SDK 会自动把它包装成 MCP 规定的 `content` 格式发送回客户端。

---

### 2. 为什么你觉得“不需要手动写代码”？

是因为 MCP SDK 帮你处理了 **RPC（远程过程调用）** 的底层逻辑。

**如果没有 MCP SDK，你需要手动写成这样（伪代码）：**

```python
# 手动处理的痛苦过程
def handle_request(json_data):
    data = json.loads(json_data)
    if data['method'] == 'tools/call':
        tool_name = data['params']['name']
        args = data['params']['arguments']
        
        # 你得手写一堆 if-else 来匹配函数
        if tool_name == "calculate_weather":
            # 你得手动检查参数齐不齐
            if "city" not in args:
                return error_response("Missing city")
            # 运行函数
            result = calculate_weather(args['city'])
            # 手动拼成 JSON 返回
            return json.dumps({"content": [{"type": "text", "text": result}]})
```

**MCP SDK 把上面的逻辑全部封装在了 `@mcp.tool()` 这个装饰器里。**

---

### 3. 实现这种“自动化”的核心技术

如果你想在自己的 Agent 框架里实现类似的效果，你需要掌握以下技术点：

1.  **类型检查与提取**：
    *   **Python**: 使用 `inspect.signature` 获取函数的参数名和类型注解，使用 `pydantic` 做数据校验。
    *   **TypeScript**: 使用 `Zod` 库来定义模式（Schema）并自动推导类型。

2.  **注册中心模式 (Registry Pattern)**：
    *   创建一个单例对象，维护一个 `dict[str, Callable]`，键是函数名，值是函数指针。

3.  **JSON-RPC 协议处理**：
    *   MCP 底层遵循 JSON-RPC 2.0。你需要一个统一的解析层，负责把收到的字符串转成对象，并根据 `method` 分发任务。

### 总结

MCP 服务器收到的 `tool_call` **不会**凭空自动调用工具。

它之所以看起来“自动”，是因为 **MCP SDK 充当了“翻译官”和“调度员”**：
*   **翻译官**：把你的函数代码翻译成 AI 能听懂的 JSON Schema。
*   **调度员**：把 AI 发回来的 JSON 指令，根据名字“派送”给对应的函数去执行。

**如果你在写自己的 Agent 系统，强烈建议你模仿这种“装饰器”风格。** 这样你后续增加新工具时，只需要写业务函数，而不需要去改动复杂的 `AgentExecutor` 逻辑。