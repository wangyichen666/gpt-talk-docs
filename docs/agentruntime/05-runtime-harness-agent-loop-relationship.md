# Agent Runtime、Harness、Agent Loop 到底是什么关系？

## 问题

Agent Runtime、Harness、Agent Loop、LLM、Tool 这些概念经常一起出现，它们到底分别在哪一层？Codex 这样的 Coding Agent 可以怎么理解？

## 核心结论

可以先记住这条分层：

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

这不是所有项目都必须严格采用的代码目录结构，而是一种很实用的职责划分方式。

## 1. Agent Loop：最小执行闭环

Agent Loop 是最核心、最小的一层。

典型过程：

```text
用户任务
  ↓
构建上下文
  ↓
调用 LLM
  ↓
模型决定回答或调用 Tool
  ↓
执行 Tool
  ↓
得到 Observation
  ↓
放回上下文
  ↓
再次调用 LLM
  ↓
直到完成
```

伪代码可以写成：

```python
while not finished:
    response = llm(context)

    if response.tool_call:
        result = execute_tool(response.tool_call)
        context.append(result)
    else:
        finished = True
```

Agent Loop 回答的是：

> Agent 下一步做什么，以及什么时候停止。

## 2. Harness：把 Loop 工程化

真实系统不会只有一个 `while`。

Harness 通常会把 Agent Loop 周围的执行逻辑封装起来，例如：

- Context 构造；
- System Prompt；
- Tool Schema 注入；
- Tool Call 解析；
- Tool Result 转换；
- 最大迭代次数；
- Token / Cost Budget；
- Retry；
- 错误处理；
- 中断；
- Streaming；
- Completion 判断。

因此可以理解成：

```text
Harness
└── 管理和驱动 Agent Loop
```

Harness 回答的是：

> 这个 Agent 执行引擎具体怎么跑。

## 3. Runtime：给 Harness 提供真实运行环境

Runtime 范围更大。

除了 Harness，本身还可能负责：

```text
Session
Workspace
Sandbox
Permission
Approval
Memory Persistence
Environment
Process Lifecycle
Scheduling
Recovery
Tracing
Resource Management
```

例如 Codex 要执行：

```bash
python3 hello.py
```

Harness 会关心：

```text
模型是否选择了 exec_command？
命令参数是什么？
命令执行结果是什么？
结果如何作为 Observation 返回给模型？
```

Runtime 还要关心：

```text
在哪个 Workspace 执行？
当前 Session 是什么？
是否 workspace-write？
命令需要审批吗？
是否进入 Sandbox？
哪些路径可以写？
网络是否允许？
进程退出码如何记录？
任务中断后是否能够恢复？
```

Runtime 回答的是：

> 这个 Agent 执行引擎在什么环境、生命周期和权限边界里运行。

## 4. Platform：管理多个 Runtime

如果继续向上一层，就是 Agent Platform。

例如一个云端 Agent 平台可能需要：

```text
用户申请 Agent
    ↓
创建 Sandbox / Runtime
    ↓
初始化 Workspace
    ↓
注入模型配置
    ↓
启动 Agent Daemon
    ↓
建立 WebSocket
    ↓
用户开始交互
```

Platform 还可能负责：

- 多租户；
- Runtime 创建和回收；
- 资源配额；
- 调度；
- 预热池；
- TTL；
- 网关；
- 动态路由；
- 计费；
- 管理后台。

因此：

```text
Platform 管很多 Runtime
Runtime 承载 Harness
Harness 驱动 Agent Loop
Agent Loop 调用 LLM 和 Tools
```

## 5. 用 Codex 举一个完整例子

用户说：

```text
帮我修改 hello.py，并运行测试。
```

整体链路可以理解成：

```text
User
  ↓
Codex Platform / Client
  ↓
Runtime
  ├── Session
  ├── Workspace
  ├── Permission
  └── Sandbox
        ↓
Harness
  ↓
Agent Loop
  ↓
LLM
  ↓
read_file Tool
  ↓
Observation
  ↓
LLM
  ↓
edit_file Tool
  ↓
Observation
  ↓
LLM
  ↓
exec_command: python3 test.py
  ↓
Runtime 检查权限 / Sandbox
  ↓
宿主机受限进程
  ↓
Tool Result
  ↓
Observation
  ↓
LLM
  ↓
Final Answer
```

可以看到：

- Loop 决定下一步；
- Harness 驱动整个循环；
- Runtime 提供真实执行环境和边界；
- Platform 管理 Runtime 的创建、连接和生命周期。

## 6. Tool 在哪一层？

Tool 是 Agent Loop 能调用的能力接口。

例如：

```text
read_file
write_file
exec_command
git
browser
MCP tool
```

但 Tool 的“声明”和“执行”是两件事。

模型看到的是：

```text
Tool Schema
```

真正执行时则可能经过：

```text
Tool Call
  ↓
Harness
  ↓
Permission / Policy
  ↓
Runtime
  ↓
Sandbox / External Service
  ↓
Result
```

因此 Tool Calling 并不是简单的“LLM 直接执行函数”。

## 7. 为什么理解这个分层很重要？

因为很多概念混淆都来自把不同层的问题放在一起。

例如：

```text
“为什么模型能执行 shell？”
```

这是 Tool + Harness 问题。

```text
“shell 为什么不能读 ~/.ssh？”
```

这是 Runtime / Sandbox 问题。

```text
“任务执行到一半断线后怎么恢复？”
```

这是 Runtime / Session / Persistence 问题。

```text
“1000 个用户怎么申请独立 Agent？”
```

这是 Platform / Scheduling 问题。

```text
“模型调用完 Tool 后为什么还会继续思考？”
```

这是 Agent Loop 问题。

分层之后，每个问题应该由哪一层负责会清楚很多。

## 一句话记忆

> Agent Loop 是循环，Harness 是执行引擎，Runtime 是运行环境，Platform 是管理这些运行环境的平台；LLM 和 Tools 则是 Loop 中真正被调用的能力。
