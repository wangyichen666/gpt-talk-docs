# 什么是 Agent Runtime？

## 问题

什么是 Agent Runtime？它和普通的 Agent Framework、模型调用代码有什么区别？

## 核心结论

**Agent Runtime 是“让 Agent 真正运行起来，并且能够持续、稳定、安全地运行”的运行时基础设施。**

如果说 Agent Framework 主要解决“怎么开发一个 Agent”，那么 Agent Runtime 主要解决的是：

- Agent 如何启动、运行和结束；
- 一轮任务如何不断调用 LLM 和工具；
- 上下文、会话状态和 Memory 如何保存；
- Tool / Skill / MCP 如何执行；
- Agent 在什么环境中执行命令；
- 权限、Sandbox、网络访问如何限制；
- 任务失败后如何恢复；
- 多个 Agent 如何调度和隔离；
- 整个执行过程如何 Trace、监控和审计。

## 可以把 Runtime 理解成什么？

最小的 Agent 可能只是：

```text
User
  ↓
LLM
  ↓
Answer
```

但真正的 Agent 往往是：

```text
User Request
    ↓
Agent Runtime
    ↓
Load Session / Context / Memory
    ↓
Agent Loop
    ↓
LLM Reasoning
    ↓
Tool Call
    ↓
Tool Execution
    ↓
Observation
    ↓
继续 Agent Loop
    ↓
完成任务
```

Runtime 就是承载这整条执行链路的环境。

## Runtime 通常包含哪些能力？

### 1. Agent Loop

负责不断执行：

```text
理解任务
  ↓
决定下一步
  ↓
调用 Tool
  ↓
获取 Observation
  ↓
判断是否完成
  ↓
未完成则继续
```

### 2. Model Runtime

负责：

- 模型请求；
- Streaming；
- Retry；
- Token / Context Window 管理；
- 模型切换；
- 推理参数。

### 3. Context / Memory

负责：

- 当前会话历史；
- Working Memory；
- Long-term Memory；
- 上下文裁剪、压缩和恢复。

### 4. Tool Runtime

负责：

- Tool Schema；
- Function Calling；
- MCP；
- CLI；
- API；
- 参数校验；
- Tool Result 转换成 Observation。

### 5. Execution Environment

例如 Codex 中的命令执行环境：

```text
Agent
  ↓
Tool Call
  ↓
exec command
  ↓
Sandbox
  ↓
OS Process
```

这里通常还涉及：

- Workspace；
- 文件读写范围；
- 网络权限；
- 环境变量；
- 进程权限。

### 6. Session / Lifecycle

负责：

```text
Create
  ↓
Running
  ↓
Idle / Suspend
  ↓
Resume
  ↓
Terminate
```

云端 Agent 平台中，这一层尤其重要。

### 7. Safety / Permission

包括：

- Sandbox；
- Tool 白名单；
- Approval；
- 权限提升；
- Secret 隔离；
- 危险操作拦截。

### 8. Observability

例如：

```text
一次 Agent Task
 ├── LLM call #1
 ├── Tool call #1
 ├── Tool result
 ├── LLM call #2
 ├── Tool call #2
 └── Final Answer
```

需要能被 Trace、记录和排查。

## Agent Runtime 和 Agent Framework 的区别

可以简单理解为：

```text
Framework
= 帮你“写 Agent”

Runtime
= 帮你“跑 Agent”
```

例如 Framework 更关注：

- Agent 定义；
- Prompt；
- Tool 注册；
- Workflow；
- Memory API。

而 Runtime 更关注：

- 实际执行；
- 生命周期；
- Sandbox；
- Session；
- 调度；
- 权限；
- 持久化；
- Recovery；
- Trace。

两者会有重叠，但关注点不同。

## 一个更完整的分层

可以把一个 Agent 系统想成：

```text
Agent Platform
    ↓
Agent Runtime
    ↓
Agent Harness
    ↓
Agent Loop
    ↓
LLM + Tools
```

其中 Runtime 不是单纯的一段 Loop 代码，而是包围整个 Agent 执行过程的运行时环境。

## 一句话记忆

> Agent Runtime = 让 Agent 从“一段能调用模型的代码”，变成“一个可以长期、安全、可恢复、可观测运行的系统”的运行时基础设施。
