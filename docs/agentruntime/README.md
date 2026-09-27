# Agent Runtime / Codex 学习笔记

这个目录用于整理本次关于 Agent Runtime、Codex Runtime、Harness、Agent Loop、Rust、命令执行和 Sandbox 的讨论。

整理原则：**一个问题，一篇文档。**

## 文档目录

### 01. 什么是 Agent Runtime？

文档：[`01-what-is-agent-runtime.md`](./01-what-is-agent-runtime.md)

主要回答：

- Agent Runtime 是什么；
- Runtime 和普通模型调用代码有什么区别；
- Runtime 通常包含哪些能力；
- Agent Loop、Tool Runtime、Memory、Sandbox、Session、Observability 分别位于什么位置。

---

### 02. Agent Runtime 就是 Harness 吗？

文档：[`02-agent-runtime-vs-harness.md`](./02-agent-runtime-vs-harness.md)

主要回答：

- Harness 和 Runtime 是否等价；
- Harness 负责什么；
- Runtime 为什么比 Harness 更大；
- 为什么可以把 Harness 看成 Runtime 中的核心执行器。

---

### 03. Codex 执行命令时，怎么判断是否进入 Sandbox？

文档：[`03-how-codex-decides-sandbox-execution.md`](./03-how-codex-decides-sandbox-execution.md)

主要回答：

- `python3 hello.py` 是否会进入 Sandbox；
- 是否所有命令都会无条件 Sandbox；
- Execution Policy、Permission、Sandbox 的关系；
- Approval 和 Sandbox 为什么是两个正交维度；
- `workspace-write`、权限提升、full access 应该怎么理解。

---

### 04. Codex 明明在宿主机执行命令，Sandbox 是怎么隔离的？

文档：[`04-how-codex-sandbox-isolates-host-processes.md`](./04-how-codex-sandbox-isolates-host-processes.md)

主要回答：

- Sandbox 是否会复制或移动 Workspace；
- 宿主机真实进程为什么仍然可以被隔离；
- 为什么最终权限可以理解为“用户权限 ∩ Sandbox Policy”；
- Kernel 如何真正拦截文件和网络访问；
- macOS Seatbelt 与 Linux namespace / bind mount / bubblewrap / seccomp 的思路；
- Sandbox 和 Docker 的区别。

---

### 05. Agent Runtime、Harness、Agent Loop 到底是什么关系？

文档：[`05-runtime-harness-agent-loop-relationship.md`](./05-runtime-harness-agent-loop-relationship.md)

主要回答：

- Agent Loop 是什么；
- Harness 如何把 Loop 工程化；
- Runtime 如何给 Harness 提供 Session、Workspace、Sandbox、权限和生命周期；
- Platform、Runtime、Harness、Agent Loop、LLM、Tool 应该如何分层；
- Tool 的声明和真正执行为什么不是一回事。

---

### 06. 为什么 Codex 使用 Rust？

文档：[`06-why-codex-uses-rust.md`](./06-why-codex-uses-rust.md)

主要回答：

- Codex 为什么不只是一个普通 LLM 客户端；
- 为什么 Coding Agent 的 Runtime 很接近系统软件；
- Rust 为什么适合 Process、PTY、Filesystem、Sandbox 等底层能力；
- 为什么 Rust 的价值主要在 Agent 执行层，而不是模型推理层；
- Java、Python 与 Rust 分别适合 Agent 系统的哪些部分。

---

### 07. Rust 用来做 Agent Runtime，真正的优势是什么？

文档：[`07-rust-advantages-for-agent-runtime.md`](./07-rust-advantages-for-agent-runtime.md)

主要回答：

- Native、Memory Safety、Ownership、无 GC 的实际意义；
- Rust 为什么适合资源生命周期和并发管理；
- Rust async 为什么适合 Tool Runtime；
- Rust 的类型系统为什么适合 Runtime 状态机和错误处理；
- Rust 与 Java / Python / Node.js 在 Agent 系统中的优势区间；
- Java Control Plane + Rust Runtime 这种架构为什么合理。

---

## 推荐阅读顺序

```text
01 Agent Runtime 是什么
      ↓
02 Runtime vs Harness
      ↓
05 Runtime / Harness / Agent Loop 分层
      ↓
06 为什么 Codex 使用 Rust
      ↓
07 Rust 的 Runtime 优势
      ↓
03 命令如何决定 Sandbox / Approval
      ↓
04 宿主机上的 Sandbox 到底怎么隔离
```

读完之后，可以形成一条完整的理解链路：

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
      ↓
exec_command
      ↓
Execution Policy / Approval
      ↓
Sandbox
      ↓
OS Process
      ↓
Kernel
```

最值得记住的几个关系是：

```text
Harness = Agent 的核心执行引擎
Runtime = Harness + Session / Workspace / Sandbox / Permission / Lifecycle 等运行环境
Approval = 要不要先问用户
Sandbox = 真正执行后能访问什么
Rust = 更适合实现靠近 OS 的 Agent 执行基础设施
```
