# Agent Runtime 就是 Harness 吗？

## 问题

Agent Runtime 是不是就是 Harness？如果不是，两者到底差在哪？

## 核心结论

**不是。Harness 和 Runtime 高度相关，但不能直接画等号。**

更合适的理解是：

```text
Agent Runtime
    └── Agent Harness
          └── Agent Loop
                ├── LLM
                └── Tools
```

Harness 更接近 Agent 的“核心执行器”，Runtime 则是承载并管理这个执行器的更完整运行环境。

## Harness 主要负责什么？

Harness 关注的是 Agent 如何真正执行一轮又一轮任务。

典型职责包括：

- 构建当前 Context；
- 调用 LLM；
- 解析模型输出；
- 识别 Tool Call；
- 执行 Tool；
- 把 Tool Result 作为 Observation 放回上下文；
- 判断是否继续执行；
- 控制最大循环次数、预算和终止条件。

也就是：

```text
Context
  ↓
LLM
  ↓
Tool Call
  ↓
Tool Execution
  ↓
Observation
  ↓
Context
  ↓
LLM
  ...
```

这部分是 Agent 最核心的执行闭环，因此可以称为 Harness。

## Runtime 比 Harness 多了什么？

如果只实现 Harness，你已经可以让一个 Agent 跑起来。

但要把它变成稳定的产品，还需要很多外围能力：

```text
Agent Runtime
├── Session
├── Lifecycle
├── Harness
│   └── Agent Loop
├── Sandbox
├── Permission / Approval
├── Workspace
├── Memory Persistence
├── Scheduling
├── Recovery
├── Resource Management
└── Trace / Metrics / Logs
```

这些能力通常不属于狭义 Harness，却属于 Runtime。

## 一个例子：Codex

假设 Codex 收到：

```text
帮我修改 hello.py，然后运行测试。
```

Harness 关心：

```text
1. 当前上下文是什么？
2. 下一步是否需要读文件？
3. 模型产生哪个 Tool Call？
4. Tool Result 是什么？
5. 是否继续调用模型？
6. 任务是否完成？
```

Runtime 还需要处理：

```text
这个 Session 属于谁？
Workspace 在哪里？
命令是否需要进 Sandbox？
哪些目录可以写？
能不能联网？
命令是否需要用户批准？
任务中断后如何恢复？
执行记录怎么 Trace？
```

因此：

```text
Harness = 怎么执行 Agent
Runtime = 在什么环境、规则和生命周期下执行 Agent
```

## 为什么这两个概念很容易混？

因为很多 Agent 项目规模较小时，会把所有逻辑写在一起：

```text
main.py
├── while loop
├── call_llm()
├── call_tool()
├── memory
├── session
└── permission
```

这时 Harness 和 Runtime 在代码结构上几乎重叠。

但随着系统平台化，边界会越来越明显。

例如云端 Agent 平台会出现：

```text
Platform
  ↓
Runtime Control Plane
  ↓
Runtime Instance / Sandbox
  ↓
Harness
  ↓
Agent Loop
```

此时 Harness 只是 Runtime 内部的一部分。

## 一句话记忆

> Harness 是 Agent 的“执行引擎”，Runtime 是让这个执行引擎能够安全、稳定、可管理运行的完整运行时。
