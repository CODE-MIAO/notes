在 LangGraph 中，它的核心思想是将 LLM（大语言模型）的驱动流程建模为一个**有向图（Directed Graph）**。

为了让你轻松理解，我们可以把 LangGraph 比作一个**快递集散中心**。下面是它的核心三要素：**State（状态）**、**Node（节点）** 和 **Edge（边）**。

## 核心概念解析

### 1. State (状态)

- **大白话**：就是图里的“全局变量”或者“共享账本”，在快递中心里它就像是**快递包裹本身的信息**（比如：寄件人、目的地、当前到哪了、包裹重量）。
- **核心作用**：图中的每一个节点都可以读取这个 State，也可以修改它。LangGraph 会在节点之间自动传递这个 State。通常它是一个 Pydantic 模型或 Python 字典。

### 2. Node (节点)

- **大白话**：就是图里的“工位”或“处理步骤”，在快递中心里就像是**分拣机、打包员、或者质检员**。
- **核心作用**：Node 本质上是一个 **Python 函数**。它接收当前的 State 作为输入，执行一些逻辑（比如调用 LLM、查数据库），然后输出更新后的 State。

### 3. Edge (边)

- **大白话**：就是连接工位的“传送带”，决定了包裹下一步去哪里。
- **核心作用**：它定义了节点的执行顺序。Edge 分为两种：
  - **普通边 (Normal Edge)**：硬编码的传送带。比如：A 节点执行完，必然去 B 节点。
  - **条件边 (Conditional Edge)**：带分叉的传送带（带有判断逻辑）。比如：质检员（Node）检查完后，如果合格就走“去发货”的边，不合格就走“去重工”的边。这通常由一个特殊的条件函数来决定。

## 简单的编排例子

我们来写一个最简单的例子：**自动化起标题和润色工具**。

- **流程**：输入一个初始想法 -> 节点 1 生成标题 -> 节点 2 加上 Emoji 润色 -> 结束。

### 代码实现

首先，确保你安装了 `langgraph`：

Bash

```
pip install langgraph
```

完整代码如下：

Python

```
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# ==========================================
# 1. 定义 State (状态)
# ==========================================
class AgentState(TypedDict):
    idea: str      # 初始想法
    title: str     # 生成的标题
    output: str    # 最终带有 emoji 的输出

# ==========================================
# 2. 定义 Node (节点函数)
# ==========================================
def generate_title_node(state: AgentState):
    print("--- 节点 1：正在生成标题 ---")
    current_idea = state["idea"]
    # 模拟 LLM 生成标题的行为
    generated_title = f"如何利用 {current_idea} 改变世界"
    
    # 返回更新后的 State (只需要返回改变或增加的键值)
    return {"title": generated_title}

def add_emoji_node(state: AgentState):
    print("--- 节点 2：正在润色加 Emoji ---")
    current_title = state["title"]
    # 模拟润色行为
    final_output = f"🚀 {current_title} 🔥"
    
    return {"output": final_output}

# ==========================================
# 3. 编排图 (构建 Node 和 Edge)
# ==========================================
# 初始化图，并传入我们定义好的 State 结构
workflow = StateGraph(AgentState)

# 添加节点 (工位)
workflow.add_node("generate_title", generate_title_node)
workflow.add_node("add_emoji", add_emoji_node)

# 配置边 (传送带)
# 从起点 START 出发，进入第一个节点
workflow.add_edge(START, "generate_title")
# 从第一个节点执行完后，自动进入第二个节点
workflow.add_edge("generate_title", "add_emoji")
# 从第二个节点结束后，进入终点 END
workflow.add_edge("add_emoji", END)

# 编译图，使其可以运行
app = workflow.compile()

# ==========================================
# 4. 运行测试
# ==========================================
if __name__ == "__main__":
    # 输入初始状态
    initial_input = {"idea": "AI Agent 技术"}
    
    print("开始执行工作流...\n")
    result = app.invoke(initial_input)
    
    print("\n--- 执行结果 ---")
    print(f"最终输出: {result['output']}")
```

### 运行结果

当你运行这段代码，你会看到控制台输出：

Plaintext

```
开始执行工作流...

--- 节点 1：正在生成标题 ---
--- 节点 2：正在润色加 Emoji ---

--- 执行结果 ---
最终输出: 🚀 如何利用 AI Agent 技术改变世界 🔥
```

> **💡 总结**
>
> 在这个例子中，`AgentState` 承载了全场的数据；`generate_title_node` 和 `add_emoji_node` 是干活的工人；而 `workflow.add_edge` 则是把他们串联成一条自动流水线的传送带。

# React模式编排

在 LangGraph 中实现 **ReAct (Reasoning and Acting)** 模式，是展示图结构的绝佳场景。ReAct 的核心逻辑是一个**循环（Loop）**：

1. **思考 (Reason)**：LLM 决定下一步要做什么（调用工具还是直接回答）。
2. **行动 (Act)**：如果是调用工具，则执行工具（节点），并把结果传回给 LLM。
3. **重复**：LLM 拿到工具结果后继续思考，直到它认为拿到了最终答案。

这个“循环”在 LangGraph 中是通过 **条件边 (Conditional Edge)** 来实现的。

下面是一个完整的、可运行的 ReAct 模式代码示例。我们不依赖复杂的外部 LLM API，而是用一个**模拟的 LLM 函数**来演示，这样你不需要 Key 也能直接跑通并理解逻辑。

### 1. 安装依赖

确保安装了最新版的 `langgraph`：

Bash

```
pip install langgraph
```

### 2. 完整代码实现

Python

```
from typing import TypedDict, Annotated, Sequence
import operator
from langgraph.graph import StateGraph, START, END

# ==========================================
# 1. 定义 State (状态)
# ==========================================
class ReActState(TypedDict):
    question: str           # 用户提出的问题
    steps: Annotated[list, operator.add]  # 记录思考和行动的思考链（自动追加列表）
    current_action: str     # 当前决定执行的工具名称，如果没有则为空
    final_answer: str       # 最终输出的答案

# ==========================================
# 2. 定义 Tool (工具函数)
# ==========================================
def search_weather_tool(location: str) -> str:
    """模拟一个天气查询工具"""
    if "北京" in location:
        return "晴天，25°C"
    return "阴天，18°C"

# ==========================================
# 3. 定义 Nodes (节点)
# ==========================================

# 节点 A: LLM 思考节点
def llm_reason_node(state: ReActState):
    print("\n🤖 [LLM 思考中...]")
    question = state["question"]
    steps = state["steps"]
    
    # 模拟 LLM 的 ReAct 思考逻辑
    if not steps:
        # 第一轮思考：发现需要查天气
        print("💡 LLM 想法: 我需要知道北京的天气，我应该使用 search_weather_tool 工具。")
        return {
            "steps": ["思考：查询北京天气"], 
            "current_action": "search_weather_tool" # 决定调用工具
        }
    else:
        # 第二轮思考：拿到了工具结果，可以回答了
        last_step = steps[-1]
        print(f"💡 LLM 想法: 我拿到了工具结果（{last_step}），现在可以回答用户了。")
        return {
            "steps": ["思考：根据天气结果生成答案"],
            "current_action": "", # 清空动作，准备结束
            "final_answer": f"为您查询到，北京目前是{last_step}，非常舒适！"
        }

# 节点 B: 工具执行节点
def action_node(state: ReActState):
    print("🛠️ [工具执行中...]")
    action = state["current_action"]
    
    if action == "search_weather_tool":
        # 执行工具
        tool_result = search_weather_tool("北京")
        print(f"🔩 工具返回结果: {tool_result}")
        return {"steps": [tool_result]} # 将结果追加到 steps 中
    
    return {}

# ==========================================
# 4. 定义 Conditional Router (条件路由函数)
# ==========================================
def should_continue(state: ReActState):
    """根据当前的 action 决定下一步去哪里"""
    if state["current_action"]:
        print("🔗 路由决策: 有执行动作，走向 [工具节点]")
        return "call_tool"
    else:
        print("🔗 路由决策: 无执行动作，走向 [结束]")
        return "end"

# ==========================================
# 5. 编排 ReAct 图
# ==========================================
workflow = StateGraph(ReActState)

# 添加两个核心节点
workflow.add_node("llm_reason", llm_reason_node)
workflow.add_node("action", action_node)

# 设置起点：总是先让 LLM 思考
workflow.add_edge(START, "llm_reason")

# 从工具节点执行完后，必须连回 LLM 节点让其继续思考
workflow.add_edge("action", "llm_reason")

# 在 LLM 节点后面，添加【条件边】
workflow.add_conditional_edges(
    "llm_reason",          # 从 llm_reason 节点出来
    should_continue,       # 根据这个函数的返回值决定去向
    {
        "call_tool": "action",  # 如果返回 'call_tool'，去 action 节点
        "end": END              # 如果返回 'end'，去终点
    }
)

# 编译图
app = workflow.compile()

# ==========================================
# 6. 运行测试
# ==========================================
if __name__ == "__main__":
    inputs = {"question": "今天北京天气怎么样？", "steps": []}
    print("====== 开始执行 ReAct 工作流 ======")
    result = app.invoke(inputs)
    print("\n==================================")
    print(f"🎉 最终回答: {result['final_answer']}")
```

### 3. 运行过程解析

当你运行上述代码，控制台会清晰地打印出 ReAct 模式的**环形跳转**过程：

Plaintext

```
====== 开始执行 ReAct 工作流 ======

🤖 [LLM 思考中...]
💡 LLM 想法: 我需要知道北京的天气，我应该使用 search_weather_tool 工具。
🔗 路由决策: 有执行动作，走向 [工具节点]

🛠️ [工具执行中...]
🔩 工具返回结果: 晴天，25°C

🤖 [LLM 思考中...]
💡 LLM 想法: 我拿到了工具结果（晴天，25°C），现在可以回答用户了。
🔗 路由决策: 无执行动作，走向 [结束]

==================================
🎉 最终回答: 为您查询到，北京目前是晴天，25°C，非常舒适！
```

### 💡 为什么 LangGraph 适合写 ReAct？

在传统的纯代码编写中，处理这种“LLM -> 工具 -> LLM”的死循环和退出条件往往需要写复杂的 `while True` 逻辑。

而在 LangGraph 中：

1. **记忆天然存在**：通过 `ReActState` 里的 `steps`，每一次工具返回的结果都会被自动记录（多亏了 `operator.add` 追加器），LLM 下一次执行时自然能看到上一步发生了什么。
2. **结构异常清晰**：通过 `add_conditional_edges`，图的跳转完全变成了解耦的配置，你可以非常轻松地在中间插入更多的工具或者监控节点。