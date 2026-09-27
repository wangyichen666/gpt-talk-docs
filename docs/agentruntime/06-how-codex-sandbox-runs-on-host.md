# Codex 的命令明明在宿主机执行，为什么还叫 Sandbox？

## 问题

Codex 执行：

```bash
python3 hello.py
```

不还是在我的宿主机上启动 Python 进程吗？既然命令没有跑到另一台机器，也没有把 workspace 复制走，那“Sandbox 执行”到底是什么意思？它究竟是怎么隔离的？

## 核心结论

**Sandbox 不等于虚拟机，也不等于复制一份 workspace。**

Codex 的命令很多时候确实仍然在宿主机上启动真实 OS Process，但这个进程会被额外施加一层由操作系统内核强制执行的权限边界。

可以记成：

```text
最终有效权限
=
宿主用户本来拥有的权限
∩
Sandbox Policy 允许的权限
```

也就是说：

> 进程还是宿主机进程，但它不能自动继承宿主用户可以做的所有事情。

---

## 1. Sandbox 不是“把 Workspace 搬到另一个目录”

先假设项目位于：

```text
/Users/gary/work/demo
```

Codex 在项目中执行：

```bash
python3 hello.py
```

一种容易产生的误解是：

```text
宿主机 workspace
      ↓ copy
/tmp/codex-sandbox-123/workspace
      ↓
在副本里执行
```

但很多本地 Sandbox 并不是这样工作的。

更准确的模型是：

```text
真实宿主机
│
├── /Users/gary/work/demo
│       └── hello.py
│
└── Codex
        ↓
   创建受限制进程
        ↓
   python3 hello.py
```

`hello.py` 还是原文件。

`python3` 也还是宿主机上的 Python。

区别只是：

```text
这个 python3 进程被限制了能访问哪些资源。
```

---

## 2. 普通 Terminal 中的进程通常继承用户权限

假设用户直接在 Terminal 执行：

```bash
python3 hello.py
```

那么 Python 通常继承当前用户的权限。

如果当前用户本来可以访问：

```text
~/project
~/Desktop
~/Documents
~/.ssh
```

Python 进程通常也可以尝试访问这些地方。

概念上：

```text
User Permission
      ↓
Terminal
      ↓
Python
      ↓
继承用户能够使用的大部分资源
```

对于 Coding Agent 来说，这个默认权限太大。

因为模型如果错误地产生：

```bash
rm -rf ~/Documents
```

从传统 Unix 用户权限角度看：

```text
这个命令可能完全有权限执行。
```

Sandbox 就是在用户权限之上再收紧一层。

---

## 3. Sandbox 做的事情是“给进程戴手铐”

假设 Gary 本人拥有：

```text
/Users/gary/project      RW
/Users/gary/Desktop      RW
/Users/gary/Documents    RW
/Users/gary/.ssh         R
Network                  YES
```

Codex 可以进一步要求 Sandbox 中的进程只有：

```text
/Users/gary/project      RW
/usr                     R
/System                  R
/Users/gary/Desktop      DENY / NO WRITE
/Users/gary/Documents    DENY / NO WRITE
/Users/gary/.ssh         DENY
Network                  DENY / LIMITED
```

因此：

```text
Gary 的权限
        ↓
Sandbox 再过滤一次
        ↓
Python 最终权限
```

Sandbox 只能收窄权限，不能凭空给进程增加宿主用户本来没有的权限。

---

## 4. 真正拦截访问的是操作系统内核

假设 Sandbox 中的 Python 执行：

```python
open("/Users/gary/.ssh/id_rsa").read()
```

最终仍然会走系统调用：

```text
Python
  ↓
libc / runtime
  ↓
syscall
  ↓
Kernel
```

内核会判断：

```text
Unix 文件权限允许吗？
Sandbox Policy 允许吗？
其他安全机制允许吗？
```

只有全部允许时，访问才真正发生。

所以 Sandbox 不是类似：

```python
if path.startswith(workspace):
    allow()
```

这样的应用层约定。

真正有价值的地方在于：

> 限制在比 Python、Node、Java、Bash 更底层的位置执行。

因此无论工具最终启动的是：

```text
Python
Node.js
Java
Rust binary
Shell script
```

都必须经过 OS 的权限检查。

---

## 5. 为什么 Workspace 中的修改又是真实的？

假设 Sandbox 明确允许项目目录读写：

```text
/Users/gary/project    READ + WRITE
```

Codex 执行：

```bash
echo hello > a.txt
```

实际写入的就是：

```text
/Users/gary/project/a.txt
```

所以退出 Codex 后，文件仍然存在。

这不是写到了一个临时副本里。

可以理解为：

```text
Host filesystem

/Users/gary/project
        ↑
        │ 真实读写
        │
Sandbox process
```

Sandbox 控制的是：

```text
哪些真实路径允许读
哪些真实路径允许写
哪些路径完全不能访问
```

而不是强制要求把所有文件复制一份。

---

## 6. 为什么 Python 还能正常启动？

如果 Sandbox 只允许访问 workspace：

```text
/project
```

而其他所有路径都不可读，那么 Python 本身可能都启动不了。

因为执行：

```bash
python3 hello.py
```

还需要读取：

```text
Python executable
动态链接库
系统 Framework
Python standard library
site-packages
证书
配置
```

因此真实 Sandbox Policy 通常不是：

```text
workspace 之外全部隐藏
```

而更像：

```text
系统依赖目录      READ
开发工具目录      READ / EXEC
workspace         READ + WRITE
敏感用户目录      DENY
其他目录          READ ONLY / DENY
network           根据策略决定
```

这也是为什么 Sandbox 可以既让开发工具正常工作，又限制它修改用户系统。

---

## 7. macOS 上可以怎么理解？

macOS 有系统级 Sandbox 机制，Codex 的本地执行可以利用类似 Seatbelt / `sandbox-exec` 的能力为子进程附加策略。

概念上原始命令：

```bash
python3 hello.py
```

可以理解成经过一层 Sandbox Launcher：

```text
Codex
  ↓
Sandbox Policy
  ↓
Shell / Python
  ↓
Kernel enforcement
```

进程树概念上类似：

```text
Codex
  └── sandbox launcher
        └── shell
              └── python3 hello.py
```

注意：

```text
python3 仍然是本机 Python
hello.py 仍然是本机项目文件
```

只是访问系统资源时，Kernel 会额外执行 Sandbox Policy。

---

## 8. Linux 上会更像“轻量容器视图”

Linux 有更多 Namespace 和 Mount 能力。

Sandbox 可以给进程创建独立的：

```text
mount namespace
network namespace
user namespace
```

然后通过 bind mount 构造一个受限制的文件系统视图。

例如真实宿主机：

```text
/
├── usr
├── etc
└── home
    └── gary
        ├── .ssh
        └── project
```

Sandbox 进程看到的可以变成：

```text
/
├── usr             RO
├── etc             RO
└── home
    └── gary
        ├── .ssh     不可见 / 不可访问
        └── project  RW
```

这里很关键的一点是：

> “视图不同”不代表“文件被复制”。

`project` 完全可以通过 bind mount 指向宿主机同一个真实目录。

概念上：

```text
Sandbox view
/home/gary/project
        │
        └──────────────► Host
                         /home/gary/project
```

所以 Sandbox 中修改项目文件，宿主机立即可以看到。

---

## 9. 网络也可以被隔离

文件系统不是唯一风险。

一个 Coding Agent 还可能执行：

```bash
curl ...
npm install
pip install
ssh ...
```

因此 Sandbox 还可能控制：

```text
是否能联网
能访问哪些地址
是否能访问 localhost
是否能访问宿主网络
```

Linux 中 Network Namespace 就可以让进程拥有一个不同的网络视图。

概念上：

```text
Host
└── eth0 → Internet

Sandbox
└── network namespace
      └── no route / limited route
```

于是进程即使在宿主机上运行，也不代表它拥有和宿主机相同的网络能力。

---

## 10. Sandbox 和 Docker 有什么区别？

二者都可以利用 OS 隔离能力，但目标不同。

Docker 通常强调：

```text
独立 root filesystem
container image
mount namespace
PID namespace
network namespace
cgroup
容器生命周期
```

你通常会得到：

```text
一个相对独立的 userspace
```

Codex 本地 Sandbox 更关注的是：

```text
继续使用用户现有开发环境
+
限制 Agent 权限
```

Coding Agent 很依赖本机已有环境，例如：

```text
你的 JDK
你的 Maven
你的 Node
你的 Python
你的 Git
你的编译器
你的项目缓存
```

如果每次命令都进入完全干净的新容器：

```text
JDK 没了
~/.m2 没了
npm cache 没了
本机工具链没了
```

体验会明显变差。

所以本地 Agent Sandbox 的思路更接近：

```text
复用宿主开发环境
+
OS 级最小权限
```

而不是：

```text
每次创建一套完全独立的机器环境
```

---

## 11. Sandbox 与虚拟机又有什么区别？

虚拟机通常是：

```text
Host Kernel
   ↓
Hypervisor
   ↓
Guest Kernel
   ↓
Guest Userspace
```

Sandbox 通常仍共享宿主 Kernel：

```text
Host Kernel
   ↑
Sandboxed Process
```

所以 Sandbox 的特点通常是：

```text
更轻量
启动更快
更容易复用宿主开发环境
```

代价则是：

```text
隔离边界通常不像完整 VM 那么彻底
```

这也是为什么 Sandbox Policy 本身必须设计得非常谨慎。

---

## 12. 一个完整例子

宿主机：

```text
/Users/gary/project
├── hello.py
└── test.py

/Users/gary/.ssh/id_rsa
/Users/gary/Documents/private.txt
```

Codex 在 workspace-write 模式下执行：

```bash
python3 hello.py
```

可以把它理解成：

```text
Codex Agent
    ↓
Exec Runtime
    ↓
Sandbox Policy
    ↓
启动宿主机真实 python3
    ↓
hello.py
```

Python 访问：

```text
/project/test.py
```

如果策略允许：

```text
✅ READ / WRITE
```

Python 访问：

```text
~/.ssh/id_rsa
```

策略可以返回：

```text
❌ Permission denied / Operation not permitted
```

Python 尝试写：

```text
~/Documents/a.txt
```

也可以被拒绝。

同一个：

```text
python3
```

同一个：

```text
宿主操作系统
```

差别只在于：

```text
这个进程带着哪一套 Sandbox Policy
```

---

## 13. 最容易记住的模型

不要把 Sandbox 想成：

```text
把 Codex 搬到另一套房子
```

更像：

```text
Codex 仍然在你的房子里
但被限制：

书房 / workspace   ✅ 可以工作
系统工具            ✅ 可以使用
卧室 / 私密目录     ❌ 不能进入
部分公共区域         👀 只能看
外网                 ❌ / 受限
```

房子还是同一套房子。

项目也还是原项目。

真正变化的是：

> Codex 启动出来的那棵进程树，能做什么、能访问什么，被操作系统收窄了。

---

## 一句话记忆

> Codex Sandbox 的核心不是“把命令搬到别处执行”，而是“在宿主机启动真实进程，再用 OS 内核级机制限制这棵进程树的文件、网络和系统权限”；workspace 可以继续指向真实项目目录，因此代码修改会立即反映到宿主机。
