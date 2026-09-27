# Codex 执行命令时，怎么判断是否进入 Sandbox？

## 问题

例如 Codex 执行：

```bash
python3 hello.py
```

这个命令到底是不是在 Sandbox 里执行？Codex 是不是所有命令都会无条件放进 Sandbox？审批和 Sandbox 又是什么关系？

## 核心结论

**Codex 并不是简单地“所有命令一律 Sandbox”或者“只有危险命令才 Sandbox”。命令执行通常要经过两类不同判断：执行策略判断，以及 Sandbox / Permission 判断。**

最重要的一点是：

> **Approval 和 Sandbox 是两个不同维度，不能混为一谈。**

一个命令可以：

- 不需要用户审批，但仍然在 Sandbox 中执行；
- 需要用户审批，获批后以更高权限执行；
- 被策略直接禁止；
- 在允许范围内自动执行。

## 可以把执行链路理解成

```text
LLM 产生 Tool Call
        ↓
exec_command
        ↓
Execution Policy
        ↓
允许 / 需要审批 / 禁止
        ↓
Permission Profile
        ↓
Sandbox Manager
        ↓
构造受限执行环境
        ↓
启动真实 OS Process
```

## 第一层：Exec Policy

这一层解决的是：

> “这个动作能不能直接执行？是否要先问用户？”

概念上可能得到三类结果：

```text
Skip / Allow
NeedsApproval
Forbidden
```

例如一个普通的：

```bash
python3 hello.py
```

如果当前策略允许，通常可以直接执行。

而一个明显要求提升权限或突破当前边界的动作，则可能触发审批。

## 第二层：Sandbox / Permission

即使命令不需要审批，也不代表它拥有宿主用户的全部权限。

例如当前模式是：

```text
workspace-write
```

那么：

```bash
python3 hello.py
```

通常可以自动运行，但进程仍然会受到 Sandbox 限制。

它可能具有：

```text
workspace       READ + WRITE
系统依赖目录      READ
其他敏感路径      DENY / READ ONLY
network         DENY / LIMITED
```

因此：

```text
不需要审批
≠
不在 Sandbox
```

这是最容易混淆的地方。

## `workspace-write` 可以怎么理解？

它表达的是一种权限配置：

```text
项目工作区：允许读写
工作区之外：大幅收紧
网络：根据策略限制
```

所以 Codex 可以真的修改你的项目文件，但不能因此随意修改整个用户目录。

例如：

```text
/project/hello.py       RW
/project/test.py        RW
~/.ssh/id_rsa           DENY
~/Documents/...         NO WRITE / DENY
```

具体细节取决于平台和当前 Sandbox 配置，但设计目标是一致的：

> 保证 Agent 能完成开发任务，同时缩小它对宿主环境的影响范围。

## `danger-full-access` 又是什么？

当运行模式允许更大的宿主权限时，Codex 可以减少或绕过本地 Sandbox 限制。

因此可以粗略理解为：

```text
workspace-write
→ 默认仍然通过 Sandbox 收窄权限

更高权限 / escalation
→ 获得用户批准后扩大执行能力

full access
→ Sandbox 限制显著减少或不再作为主要边界
```

## 为什么 Approval 和 Sandbox 必须分开？

因为二者解决的问题不同。

Approval 解决：

```text
“这个动作是否应该让用户明确同意？”
```

Sandbox 解决：

```text
“即使允许执行，这个进程实际上能访问什么？”
```

所以完整模型应该是：

```text
是否允许执行？
      ↓
是否需要用户批准？
      ↓
以什么权限执行？
      ↓
Sandbox 给进程哪些 OS 权限？
```

而不是：

```text
安全命令 → 不进 Sandbox
危险命令 → 才进 Sandbox
```

## `python3 hello.py` 的典型过程

假设当前是普通 workspace-write 模式：

```text
用户要求运行 hello.py
        ↓
LLM 生成 exec_command
        ↓
Exec Policy 判断：可自动执行
        ↓
无需 Approval
        ↓
Sandbox Manager 构造 workspace-write 环境
        ↓
启动 python3
        ↓
python3 是宿主机真实进程
        ↓
只能在 Sandbox Policy 允许的边界内访问文件和网络
```

因此最终结论是：

> `python3 hello.py` 完全可能“无需审批但仍在 Sandbox 中运行”。

## 一句话记忆

> Approval 决定“要不要先问你”，Sandbox 决定“命令真正跑起来后能干什么”。两者是正交的。
