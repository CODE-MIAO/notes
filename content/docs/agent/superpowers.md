---
title: "superpowers"
weight: 11
---
已为你启用 **Superpowers** 插件。它包含以下内容，并会在本会话中自动协助你：

### 你获得了什么

- **一组开发技能（Skills）**：覆盖从需求澄清、方案设计、实现、调试、评审到收尾的完整流程  
  - `using-superpowers`（会话起始引导）
  - `brainstorming`（创意/功能修改前先澄清需求）
  - `writing-plans`、`executing-plans`（先写计划再执行）
  - `test-driven-development`（TDD）
  - `systematic-debugging`（系统化排错）
  - `verification-before-completion`（完成前强制验证）
  - `requesting-code-review`、`receiving-code-review`（评审闭环）
  - `subagent-driven-development`、`dispatching-parallel-agents`（多代理并行）
  - `using-git-worktrees`（隔离工作目录）
  - `finishing-a-development-branch`（分支收尾）
  - `writing-skills`（编写/维护技能）

- **一个子代理（Subagent）**
  - `code-reviewer`：适合在完成一个较大步骤后做实现质量与计划一致性审查

- **一个 Hook**
  - `sessionstart`：会话启动时自动触发相关引导行为

- **命令支持**
  - 可通过 `/` 调用能力（例如技能、子代理）
  - 旧命令 `/brainstorm`、`/write-plan`、`/execute-plan` 已弃用，建议改用对应的 Superpowers 技能

### 使用方式（简要）

- 你可以直接输入 `/技能名` 或 `/子代理名` 来显式调用。  
- 也可以不手动调用：我会根据任务阶段自动选择合适技能（如先 brainstorm、再计划、再实现、再验证）。  
- 规则与 hooks 会自动生效，无需额外配置。





## 1. 会话入口（元流程）

```
using-superpowers
```

- 作用：约定「怎么发现和引用技能」，强调在需要用流程约束时优先走 Skill，而不是随口回答。
- 位置：新对话开始 / 你希望工作方式「按 Superpowers 规范」时。

------

## 2. 需求与设计（先想清楚再动手）

| 顺序感 | 技能            | 典型流程意图                                                 |
| :----- | :-------------- | :----------------------------------------------------------- |
| ①      | `brainstorming` | 做功能、改行为、有较大设计空间前：先把意图、约束、取舍聊清楚，减少返工。 |
| ②      | `writing-plans` | 已有多步骤需求/规格时：先写可实现、可检查的步骤，再允许动代码。 |

------

## 3. 计划执行（两种方式）

| 情形                           | 技能                          | 流程要点                                                     |
| :----------------------------- | :---------------------------- | :----------------------------------------------------------- |
| 有计划、可能要「分段验收」     | `executing-plans`             | 按书面计划逐步实现，中途用检查点回看是否偏离计划。           |
| 计划里有多块 互不依赖 的活     | `dispatching-parallel-agents` | 并行起多个代理/任务，缩短总耗时（前提：共享状态少、边界清楚）。 |
| 当前会话内要拆解成多个独立任务 | `subagent-driven-development` | 用子代理拆分执行实现计划里的独立项。                         |

------

## 4. 实现质量：TDD / 调试 / 隔离

| 场景                           | 技能                      | 流程要点                                                    |
| :----------------------------- | :------------------------ | :---------------------------------------------------------- |
| 新功能或修 bug（强调测试先行） | `test-driven-development` | 先测试/失败用例 → 再实现 → 再重构，减少「写完才发现不对」。 |
| 测试挂、运行结果异常           | `systematic-debugging`    | 先复盘现象与证据，再 hypothesis，避免拍脑袋改一行。         |
| 需要与主分支隔离做功能         | `using-git-worktrees`     | 单独 worktree/目录做实验，降低污染当前工作区风险。          |

------

## 5. Code Review（给出去 / 接回来）

| 方向                         | 技能                     | 流程要点                                                     |
| :--------------------------- | :----------------------- | :----------------------------------------------------------- |
| 大块做完、合并前             | `requesting-code-review` | 主动对照需求和规范做一次「可当 PR 描述」的自我审查视角。     |
| 收到评审意见但不清晰或有疑问 | `receiving-code-review`  | 先核实技术正确性，再决定采纳方式，反对「为表态而全盘照改」。 |

子代理 `code-reviewer`：适合在「完成一个逻辑整块」之后，跑一次更独立的审查（和上面「请求/接收 review」技能是互补关系）。

------

## 6. 收尾与合线

| 技能                             | 流程要点                                                     |
| :------------------------------- | :----------------------------------------------------------- |
| `verification-before-completion` | 在声称「做完了 / 修好了 / CI 过了」之前：先跑验证命令、对输出有证据，再下结论。 |
| `finishing-a-development-branch` | 实现与测试都就绪后：整理合并/PR/清理分支等选项，把分支收束到可集成状态。 |

------

## 7. 维护技能库本身

| 技能             | 用途                                     |
| :--------------- | :--------------------------------------- |
| `writing-skills` | 编写、修改、部署前自检 Skill 文档 时用。 |

------

## 8. 串成一条「常见主路径」（示意）

用一句话串起来就是：

入口（using-superpowers）→ 有设计空间先 brainstorming → 多步则 writing-plans → 执行用 executing-plans / 并行用 dispatching-parallel-agents / 本会话拆子任务用 subagent-driven-development → 实现侧优先 TDD，出问题走 systematic-debugging，需要隔离用 worktrees → 大块完成用 requesting-code-review，别人意见用 receiving-code-review → 收尾前 verification-before-completion → 合线用 finishing-a-development-branch；维护技能文档用 writing-skills。

如果你希望某一条技能写成「逐步清单」（例如 TDD 或 systematic-debugging 里每一步具体问什么、做什么），告诉我技能英文名，我可以按插件对外描述再拆成更细的步骤说明。

