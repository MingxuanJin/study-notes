# Task2 打卡



# 笔记

## Chap.3 虚拟文件系统

虚拟文件系统是Deep Agents区别于其他Agent框架的最核心的创新部分

它提供了一整套文件操作工具，Agent 可以像人类一样按需读取、结构化存储、搜索定位

### 工具概览

| 工具         | 用途                                                         | 类比                           |
| ------------ | ------------------------------------------------------------ | ------------------------------ |
| `ls`         | 列出目录中的文件和元信息（大小、修改时间）                   | 打开文件夹看看有什么           |
| `read_file`  | 读取文件内容，支持偏移量和限制条数；原生支持多模态格式（图片、视频、音频、PDF/PPT） | 翻开某份资料阅读               |
| `write_file` | 创建文件，或完整覆盖已有文件                                 | 写一份新的备忘录或重写整份草稿 |
| `edit_file`  | 对已有文件做精确字符串替换                                   | 用红笔修改文档                 |
| `delete`     | 删除文件或目录                                               | 清理不再需要的资料             |
| `glob`       | 按模式匹配查找文件（如 `**/*.py`）                           | 在文件柜中按标签找             |
| `grep`       | 搜索文件内容，按字面量匹配；支持内容输出和计数               | 全文检索                       |

`write_file` 和 `edit_file` 默认都是普通文件操作，但可以通过 `FilesystemPermission` 对敏感路径直接拒绝，或在执行前暂停等待人工审批

#### read_file

##### **分片读取**

对于大文件，`read_file` 支持按偏移量和行数读取，避免一次性把整个文件塞进上下文，默认状态下最多读取前100行

```
# 从第 100 行开始，读取 50 行
read_file("/workspace/report.md", offset=100, limit=50)
```

##### **原生多模态支持**

不再局限于纯文本工作流，可以处理图片、视频、音频

#### grep

支持三种输出模式

- **`files_with_matches`**：只返回匹配的<span style="color: #b8860b">文件路径</span>（快速定位）
- **`content`**：<span style="color: #b8860b">返回匹配行及上下文（深入查看）</span>
- **`count`**：返回匹配数量（概览统计）

```
# 找到所有包含 "TODO" 的 Python 文件
grep("TODO", glob="**/*.py", output_mode="files_with_matches")

# 查看匹配内容
grep("def create_agent", output_mode="content")
```

### 上下文自动管理

虚拟文件系统最大的价值不在于”存文件”本身，而在于它与 Deep Agents 的**上下文自动管理机制**紧密配合

> [!NOTE]
>
> 用虚拟文件系统当作 Agent 的“外置扩展硬盘（虚拟内存）”，避免大模型被海量数据直接“撑爆”上下文窗口
>
> 虚拟文件系统不是最终的存储目的，而是一种让Agent在有限上下文窗口内处理超大规模数据的方法和路径



当工具调用的输入或输出超过 20,000 tokens 时（可通过 `tool_token_limit_before_evict` 配置），Deep Agents 会自动：

1. 将完整内容**写入虚拟文件系统**

2. 在对话历史中**替换为文件路径引用** + 前 10 行预览

3. Agent 需要时可以**按需读回**

   

> 如果只是把文件写进磁盘或者内存，那只是一次简单的I/O操作
>
> 在长链路的 Agent 任务中，工具经常会返回巨量文本，导致上下文膨胀、Lost in the Middle、成本飙升
>
> 因此每当单次工具调用的输入或者输出超出阈值时，框架就会介入



```b
[ 工具返回 50,000 tokens 数据 ]
               │
               ▼
   [ 触发 Evict 驱逐机制 ]
   ┌───────────┴───────────┐
   ▼                       ▼
【写入虚拟文件系统】       【改写对话历史 Context】
存入: /tmp/output_123.txt   仅保留:
                           - 文件路径: /tmp/output_123.txt
                           - 前 10 行预览内容
                           - 数据元信息（如总行数、大小）
```

> Agent 看到前 10 行预览和路径后，明白“完整数据已经在那个文件里了”。它不会盲目地把整个文件再全部读回上下文，而是根据需要进行**靶向操作**：
>
> - **切片读取**：用读取工具只读指定行（如 `read_lines(path, start=100, end=200)`）。
> - **外部过滤**：调用 grep、python 脚本或 SQL 查询工具，在文件系统层面进行检索或统计，只将最后的关键计算结果返回到上下文。
> - **分块流式处理**：如果需要全量分析，可以分批次按块读取，处理完一块丢弃一块。

### 对话历史总结

当上下文达到配置的阈值时，Deep Agents 会启动**自动总结**。`create_deep_agent()` 在模型 profile 提供窗口大小时，默认使用窗口的 85%；缺少该信息时使用固定 token 阈值，也可以自行配置：

1. 将待总结的旧消息**写入 Backend**，保存供后续查阅
2. 用 LLM 生成结构化摘要（意图、产出物、下一步），并附上历史保存路径
3. 将**本次发给模型的内容**换成“摘要 + 近期消息”；正常摘要时，State 中的原始消息仍然保留

这里要区分“保存的消息历史”和“模型这次看到的内容”。摘要缩短了模型输入，历史文件则用于回查细节；能否<span style="color: #b8860b">读回还取决于 Backend 是否成功保存、文件是否仍然可用</span>。

![上下文自动管理：大结果卸载为文件引用；摘要中间件保存旧消息，并将模型输入改为摘要加近期消息；自动摘要阈值可配置](https://datawhalechina.github.io/deepagents-in-action/imgs/08-flowchart-context-management.png)

图源 https://datawhalechina.github.io/deepagents-in-action/chapters/ch03-virtual-filesystem/

### 可插拔的存储后端

具体的文件存到哪里，由**后端（Backend）**决定，可以根据场景选择不同的储存策略

#### StateBackend（默认）：临时存储

文件存在 LangGraph 的 Agent State 中。特点：

- 同一个对话线程内**持久化**（多轮对话不丢失）
- 对话结束后**丢失**（换一个 thread 就没了）
- 主 Agent 和子 Agent **共享**文件

适合场景：大多数情况下的默认选择，Agent 的”草稿纸”。

> 在 LangGraph 和大多数 Agent 框架中，**`thread`（线程）代表一段独立隔离的“会话实例”或“对话窗口”**. 换了一个thread之后就是换了一个全新的独立对话窗口
>
> eg.使用网页端ChatGPT，**在同一个聊天窗口里继续提问**：这就是在**同一个 `thread`** 内。你前两轮让它生成的分析表格、传的文件（存在 State 里），后续几轮接着问它依然记得。
>
> **点击左上角“+ New Chat（开启新对话）”**：这就是**换了一个新的 `thread`**。在新窗口里，前一个窗口上传的文件、草稿和变量全部清空，Agent 对上一场对话一无所知。
>
> 为什么是草稿纸：
>
> 一旦会话完成、开启新任务（新 `thread`），这些临时切片、中间计算结果自然失效，不会污染全局存储，也不需要开发者手动写清理逻辑去删垃圾文件



#### FilesystemBackend：本地磁盘

文件直接读写**本地文件系统**。特点：

- `root_dir` 指定 Agent 可访问的根目录；相对路径会被解析为绝对路径（`Path(root_dir).resolve()`），`"."` 即当前工作目录
- `virtual_mode=True` 启用路径沙箱（阻止 `..`、`~` 及越界的绝对路径），**强烈建议开启**；若为默认的 `virtual_mode=False`，即使设了 `root_dir` 也不提供任何越界保护
- 文件修改是**永久的、不可逆的**

适合场景：本地开发 CLI（编程助手）、CI/CD 流水线。



> [!WARNING]
>
> Agent 可以读取 `root_dir` 下所有文件，包括 `.env`、密钥等敏感文件。Web 服务或 API 场景中切勿使用此后端，应改用沙箱后端。建议配合 Human-in-the-Loop 使用



#### LocalShellBackend：本地 Shell 执行

`LocalShellBackend` 是 `FilesystemBackend` 的扩展，在文件系统工具之外**额外提供 `execute` 工具**，可直接在宿主机运行 Shell 命令。特点：

- 命令通过 `subprocess.run(shell=True)` 执行，**无任何沙箱隔离**
- 支持 `timeout`（默认 120 秒）、`max_output_bytes`（默认 100,000）、`env` 等参数
- `root_dir` 作为命令的工作目录，<span style="color: #b8860b">但命令可访问系统上任意路径</span>

适合场景：本地开发环境的编程助手、你完全信任 Agent 行为的个人开发机。

> 基础的 `FilesystemBackend` 只负责文件I/O, 在只有文件系统工具时，Agent 只能“看”和“改”静态文件，**无法让代码跑起来**。而 `LocalShellBackend` 在这些读写工具的基础上，**额外给 Agent 一个名为 `execute` 的通用命令行执行工具**，相当于给了一把终端的钥匙，Agent不仅能编辑代码，还可以自己编译、运行、测试、排错



> [!NOTE]
>
> 高收益但高风险，只适合在个人本机或一次性测试机器上使用，并且你完全清楚并信任这个 Agent 的提示词与行为



> [!CAUTION]
>
> Agent 可执行任意 Shell 命令，包括删除文件、外传数据、消耗资源。**绝对不要在生产环境或多用户系统中使用。** 沙箱后端是生产环境的安全替代方案。



如果确实要在个人开发机中临时使用，至少做几层防护：

- 将 `root_dir` 指向一个专门的临时工作区，而不是用户主目录或整个仓库上级目录

- 显式设置 `virtual_mode=True`，并用最小化的 `env` / `PATH` 降低命令可见范围

- 不把 `.env`、私钥、云凭证、生产配置文件放进 Agent 可访问目录

- 对 `rm`、`mv`、安装依赖、修改配置、访问网络等高风险操作增加 Human-in-the-Loop 审批

- 需要运行不可信代码、处理用户上传文件或对外提供服务时，直接换用沙箱后端，不要用 `LocalShellBackend`

  

#### StoreBackend：跨会话持久化

长期记忆存储，用于实现**跨会话的持久化虚拟文件系统或用户级长期状态存储**

文件存在 LangGraph 的 Store 中。特点：

- **跨线程持久化**——不同对话都能访问同一份文件
- `namespace` 参数控制数据隔离：`lambda rt: (rt.server_info.user.identity,)` 按用户隔离，防止数据混用
- 开发用 `InMemoryStore`，部署到 LangSmith 时省略 `store` 参数（平台自动配置）

适合场景：长期记忆、跨会话的用户偏好、累积的知识库

state vs. store

| **维度**     | **Agent State（检查点存储）**          | **LangGraph Store（长期存储）**            |
| ------------ | -------------------------------------- | ------------------------------------------ |
| **隔离维度** | `thread_id`（按单个对话窗口隔离）      | `namespace`（按用户、组织或业务维度隔离）  |
| **生命周期** | 仅随当前会话存在，换 Thread 即失效     | 永久保存，跨越无数个独立会话               |
| **角色定位** | 会话内的“临时草稿纸”、中间执行状态     | 跨会话的“个人网盘 / 用户长期记忆库”        |
| **典型数据** | 工具单次超长输出切片、当前任务步骤跟踪 | 用户上传的历史资产、持久化配置、全局知识库 |

> **`namespace` 按用户隔离数据**：
>
> - **作用**：防止多租户或多用户环境下发生“数据串门”。
> - **机制**：通过 `namespace=lambda rt: (rt.server_info.user.identity,)`，系统自动提取当前登录用户的唯一身份标识（User ID）。
> - **效果**：系统在底层将数据划分到独立的命名空间（如 `("user_123", "files")`）。张三只能访问张三的文件，即使李四请求相同文件名，也绝不会读到张三的数据。
>
> **存储后端的渐进式迁移**：
>
> - **开发环境（`InMemoryStore`）**：单机开发调试时直接保存在内存中，无需部署外部数据库，启动即用，便于本地快速写测试。
> - **生产部署（LangSmith / LangGraph Cloud）**：上线到托管平台时，平台会自动挂载企业级的分布式后端（基于 Postgres），开发者无需手动维护连接池或显式传入 `store` 实例。

#### CompositeBackend：混合路由

最灵活的方案——**不同路径走不同后端**



#### 沙箱后端：安全代码执行

当使用沙箱后端（Modal、Daytona、Runloop 等）时，除了文件系统工具外，Agent 还会获得一个额外的 `execute` 工具，可以在隔离环境中执行 Shell 命令：

![五种存储后端对比：StateBackend（临时）→ FilesystemBackend（本地磁盘）→ StoreBackend（跨会话持久化）→ CompositeBackend（混合路由）→ 沙箱后端（安全执行），从临时到持久化的渐进选择](https://datawhalechina.github.io/deepagents-in-action/imgs/09-comparison-backends.png)

### 自定义后端与安全策略

#### 声明式权限：FilesystemPermission

最简单的路径访问控制方式是使用 `FilesystemPermission`，无需修改后端代码：

```python
from deepagents import create_deep_agent, FilesystemPermission

agent = create_deep_agent(
    model=model,
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/memories/": StoreBackend(
                namespace=lambda rt: (rt.server_info.user.identity,),
            ),
        },
    ),
    permissions=[
        FilesystemPermission(
            operations=["write"],
            paths=["/policies/**"],
            mode="deny",           # 禁止写入 /policies/ 下的任何文件
        ),
    ],
)
```

权限规则在工具调用前按声明顺序求值，采用 first-match-wins：第一个同时匹配 `operations` 和 `paths` 的规则决定结果；如果没有规则匹配，则默认允许。因此配置权限时，应将更具体的规则放在更宽泛的规则之前。

权限规则在工具调用前按声明顺序求值，采用 first-match-wins：第一个同时匹配 operations 和 paths 的规则决定结果；如果没有规则匹配，则默认允许。因此配置权限时，<span style="color: #b8860b">应将更具体的规则放在更宽泛的规则之前</span>。

| `mode`        | 行为                     | 适用场景                             |
| ------------- | ------------------------ | ------------------------------------ |
| `"allow"`     | 允许操作继续执行         | 为特定路径设置显式例外               |
| `"deny"`      | 直接拒绝，不执行文件操作 | 无论谁发起都不应访问的路径           |
| `"interrupt"` | 暂停并等待人工决策       | 可以操作，但必须先经过审批的敏感路径 |

#### 实现自定义后端

此部分可以直接参考教程

#### 安全策略：PolicyWrapper

这里也可以直接看教程