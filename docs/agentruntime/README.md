# Agent Runtime / Codex 学习笔记

这个目录用于整理关于 Agent Runtime、Codex Runtime、Harness、Rust、命令执行和 Sandbox 的讨论。

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
- Agent Loop、Harness、Runtime、Platform 之间如何分层。

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

### 04. 为什么 Codex 使用 Rust？

文档：[`04-why-codex-uses-rust.md`](./04-why-codex-uses-rust.md)

主要回答：

- Codex 为什么不只是一个普通 LLM 客户端；
- 为什么 Coding Agent 的 Runtime 很接近系统软件；
- Rust 为什么适合 Process、PTY、Filesystem、Sandbox 等底层能力；
- Rust 的价值为什么主要在 Agent 执行层，而不是模型推理层。

---

### 05. Rust 用来做 Agent Runtime，真正的优势是什么？

文档：[`05-rust-advantages-for-agent-runtime.md`](./05-rust-advantages-for-agent-runtime.md)

主要回答：

- Native、Memory Safety、Ownership、无 GC 的实际意义；
- Rust 为什么适合资源生命周期和并发管理；
- Rust async 为什么适合 Tool Runtime；
- Rust 与 Java / Python / Node.js 在 Agent 系统中的优势区间；
- Java Control Plane + Rust Runtime 这种架构为什么合理。

---

### 06. Codex 的命令明明在宿主机执行，为什么还叫 Sandbox？

文档：[`06-how-codex-sandbox-runs-on-host.md`](./06-how-codex-sandbox-runs-on-host.md)

主要回答：

- Sandbox 是否会复制 / 移动 workspace；
- 宿主机真实进程为什么仍然可以被隔离；
- 最终权限为什么可以理解为“用户权限 ∩ Sandbox Policy”；
- OS Kernel 如何真正拦截文件和网络访问；
- macOS 和 Linux 上的隔离思路；
- bind mount、namespace 与真实 workspace 的关系；
- Sandbox、Docker、虚拟机有什么区别。

---

## 推荐阅读顺序

```text
01 Agent Runtime
      ↓
02 Runtime vs Harness
      ↓
04 为什么 Codex 使用 Rust
      ↓
05 Rust 的 Runtime 优势
      ↓
03 命令如何决定 Sandbox / Approval
      ↓
06 Sandbox 为什么仍然运行在宿主机
```

读完之后，可以形成一条比较完整的理解链路：

```text
Agent
  ↓
Agent Loop
  ↓
Harness
  ↓
Runtime
  ↓
Tool / exec_command
  ↓
Execution Policy / Approval
  ↓
Sandbox
  ↓
OS Process
  ↓
Kernel
```
