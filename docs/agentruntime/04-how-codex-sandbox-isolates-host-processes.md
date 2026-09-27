# Codex 明明在宿主机执行命令，Sandbox 是怎么隔离的？

## 问题

Codex 执行的命令不还是在宿主机上运行吗？

如果 `python3 hello.py` 用的就是宿主机上的 Python，项目文件也还是原来的 Workspace，那么 Sandbox 到底隔离了什么？难道只是把 Workspace 挪到另一个地方？

## 核心结论

**Sandbox 不等于虚拟机，也不等于复制或搬迁 Workspace。**

Codex 的命令通常仍然是在宿主机上启动真实进程，只不过操作系统会给这棵进程附加额外的权限边界。

可以记成：

```text
进程最终权限
=
宿主用户原有权限
∩
Sandbox Policy
```

Sandbox 的作用不是给进程更多权限，而是继续把权限收窄。

## 普通 Terminal 和 Codex 的区别

假设你正常打开 Terminal：

```bash
python3 hello.py
```

这个 Python 进程通常继承当前用户权限。

如果当前用户本身有权限，它可能尝试访问：

```text
workspace
Desktop
Documents
~/.ssh
网络
```

而 Codex 启动同样的 Python 时，会额外施加 Sandbox Policy：

```text
宿主用户权限
      ↓
Sandbox 再收窄
      ↓
最终进程权限
```

例如：

```text
workspace        READ + WRITE
/usr             READ
/System          READ
~/.ssh           DENY
Desktop          NO WRITE / DENY
Documents        NO WRITE / DENY
Network          DENY / LIMITED
```

## Workspace 通常没有搬家

假设真实项目路径是：

```text
/Users/gary/work/demo
```

Codex 在 Sandbox 中执行：

```bash
python3 hello.py
```

访问的通常仍然是：

```text
/Users/gary/work/demo/hello.py
```

而不是先复制到：

```text
/tmp/codex-sandbox-xxx/demo
```

因此 Sandbox 内执行：

```bash
echo hello > a.txt
```

如果 `a.txt` 位于允许写入的 Workspace 中，那么宿主机上的真实文件会立即发生变化。

这正是 Coding Agent 需要的效果：

```text
真实修改项目代码
+
限制项目之外的危险访问
```

## 操作系统是怎么拦住它的？

关键在于：进程访问文件、网络等资源最终都必须经过操作系统内核。

例如 Python 执行：

```python
open("/Users/gary/.ssh/id_rsa").read()
```

最终会走到系统调用：

```text
Python
  ↓
libc / runtime
  ↓
syscall
  ↓
Kernel
  ↓
检查普通 Unix 权限
  ↓
检查 Sandbox Policy
  ↓
Allow / Deny
```

因此限制不是简单地写在 Agent 代码里：

```text
if path.startsWith(workspace): allow
```

而是由更底层的操作系统能力执行。

这样无论 Agent 最终启动的是：

- Python；
- Bash；
- Node.js；
- Java；
- Rust；
- Git；

只要它们处于同一 Sandbox 约束下，都必须遵守相同的 OS 权限边界。

## macOS：Seatbelt

在 macOS 上，可以把模型理解成：

```text
Codex
  ↓
sandbox-exec / Seatbelt Policy
  ↓
zsh / bash
  ↓
python3 hello.py
```

Python 仍然是本机 Python，项目仍然是真实项目。

但 Sandbox Policy 可以规定哪些路径：

```text
允许读取
允许写入
禁止访问
```

以及是否允许网络等能力。

## Linux：更像一个轻量级隔离文件系统视图

Linux 可以使用 namespace、bind mount、bubblewrap、seccomp 等机制构造受限环境。

例如宿主机真实目录：

```text
/home/gary/project
```

可以通过 bind mount 映射进 Sandbox：

```text
Sandbox                         Host
/home/gary/project  ─────────→  /home/gary/project
        RW                         真实目录
```

文件没有复制。

Sandbox 内对该目录的修改，就是对宿主机真实目录的修改。

与此同时，Sandbox 可以让其他目录：

```text
只读
不可见
不可写
```

还可以通过 network namespace 等能力限制网络。

## 为什么系统目录还能被读取？

如果 Sandbox 真的只允许访问 Workspace，很多程序根本启动不了。

例如 Python 启动需要：

```text
Python executable
  ↓
动态链接库
  ↓
系统 Framework / libc
  ↓
Python stdlib
  ↓
site-packages
  ↓
你的 hello.py
```

所以常见策略并不是：

```text
workspace 之外全部不可见
```

而更接近：

```text
系统运行依赖：READ
Workspace：READ + WRITE
敏感位置：DENY / READ ONLY
网络：按策略限制
```

## Sandbox 和 Docker 的区别

Docker 往往提供一个更完整的容器环境：

```text
独立 root filesystem
namespace
network
cgroup
container image
独立 userspace
```

而 Codex 本地 Sandbox 的核心目标更像：

> 使用你的真实开发环境，但把 Agent 进程能做的事情限制在安全边界内。

因为 Coding Agent 往往需要直接使用：

```text
本机 JDK
本机 Python
本机 Node
本机 Git
本机 Maven / Gradle
真实 Workspace
已有依赖缓存
```

如果每次都创建一个完全干净的容器，开发环境的一致性和使用体验都会变差。

## 最容易记住的模型

错误理解：

```text
宿主机 Workspace
       ↓ copy
另一个 Sandbox Workspace
       ↓
执行命令
```

更接近真实情况的理解：

```text
同一台宿主机
│
├── 你的 Terminal
│     └── python3
│           权限≈你的用户权限
│
└── Codex
      └── Sandbox Policy
            └── python3
                  权限=用户权限∩Sandbox权限
```

## 一句话记忆

> Codex Sandbox 不是把 Agent 搬到另一台机器，而是在同一台机器上给 Agent 启动出的进程“戴上手铐”：Workspace 仍然是真实 Workspace，工具仍然是本机工具，但它能访问的文件、网络和系统能力被操作系统限制住。
