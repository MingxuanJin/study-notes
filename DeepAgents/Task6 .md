# Task6 打卡

# 打卡内容

**做了什么**

- 搭了三个容器:State(存草稿)、Checkpointer(按 `thread_id` 恢复旧线程)、Store(存偏好)。
- 用 `CompositeBackend` 把 `/memories/` 路由到 Store,其余路径留在 State。
- 预置了 Alice 和 Bob 两个用户的偏好文件,然后跑了四个场景:保存、新线程、旧线程、其他用户。

**验证了什么**

1. **偏好跨线程共享。** Alice 的新线程能读到更新后的偏好。Store 内容、`read_file` 返回和首次模型请求这三处都严格等于同一个 JSON 对象。
2. **草稿线程内隔离。** 新线程读草稿会报 `not found`,这是预期的边界,不是失败。
3. **旧线程能恢复草稿。** 复用 `alice-original` 后,checkpoint 里仍有草稿。
4. **用户间互不串。** Bob 只读到自己的偏好。
5. **写入落在正确的后端。** 偏好只在 Store 里,草稿只在 State 里。

**没有证明的部分**

- 数据都在内存里,重启就没了。
- 没有测访问控制,用户身份应由认证过的入口提供。
- offline 模式只验证数据边界,不证明真实模型会自己选对工具。



# 笔记

以下内容来自 https://datawhalechina.github.io/deepagents-in-action/chapters/ch09-human-in-the-loop/ 的学习笔记

两章其实共用同一套"地基":**Checkpointer(存档)+ thread_id(存档编号)**。记忆靠它,暂停恢复也靠它。

------

## 第一部分:记忆(第 8 章)

### 1. 短期记忆 vs 长期记忆

|            | 短期记忆(Thread-scoped)                             | 长期记忆(Cross-thread)               |
| ---------- | --------------------------------------------------- | ------------------------------------ |
| 范围       | 同一个对话线程内                                    | 跨对话、跨线程                       |
| 靠什么实现 | **Checkpointer** + `StateBackend`                   | **Store** + `StoreBackend`           |
| 比喻       | 每段对话一张**工作桌**,换桌就看不到上一张桌上的东西 | 一个**公共储物柜**,谁来都能存取      |
| 例子       | 你刚说"我叫张三",后面问"我叫啥"它能答               | "我喜欢简洁代码风格"下次新对话还记得 |

> **thread(线程)**:一段独立的对话。每段对话有个 `thread_id`,相当于存档编号。
>  换了 `thread_id` = 换了一段新对话 = 短期记忆全部看不到。

### 2. Checkpointer:短期记忆的基础

- 每执行一步,自动**存档**(消息历史、文件状态、Todo 任务清单等)
- 下次用**同一个 `thread_id`** 调用,自动读档
- `MemorySaver`:存在内存里,重启就没了,**开发用**
- `PostgresSaver`:存在数据库里,**生产用**
- 关键限制:**只在同一个 thread_id 内有效**,不同对话完全隔离

### 3. 对话太长怎么办(上下文窗口)

对话越聊越长,可能超过 LLM 的**上下文窗口(context window)**。三种策略:

| 策略               | 做法                            | 啥时候用              |
| ------------------ | ------------------------------- | --------------------- |
| **Trim 裁剪**      | 只留最近 N 条,老的直接扔        | 不需要历史的简单场景  |
| **Delete 删除**    | 用 `RemoveMessage` 精准删某几条 | 要删敏感信息          |
| **Summarize 总结** | 让 LLM 把老消息压成摘要         | 要保留语义,**最推荐** |

Deep Agents 已经**内置了 Summarize**(`SummarizationMiddleware`):

- 知道模型窗口大小时,用到 **85%** 就触发
- 不知道就用固定 token 阈值
- 压缩的只是"发给模型的那份",State 里的原始消息还在

> ⚠️ 想替换默认的 `SummarizationMiddleware`,是**整个实例替换**,不是只改某个参数。触发条件、保留条数、摘要提示词要一起重新配。

#### 自定义裁剪:`@before_model`

```python
@before_model
def trim_messages(state, runtime):
    ...
```

这里的 `@before_model` 就是**装饰器**(详见 [装饰器(Decorator)](装饰器(Decorator))):把你的函数注册成一个"钩子",框架会在**每次调用模型之前**自动执行它。对应的还有 `@after_model`(模型调用之后执行)。它们属于 **Middleware(中间件)** 体系。

### 4. 自定义 AgentState 和 ToolRuntime

- **AgentState**:Agent 的"状态本"。默认有 `messages`,你可以继承它加自己的字段(比如 `user_id`、`preferences`)
- **ToolRuntime**:一个**隐藏参数**。模型看不到它,但工具函数能通过它访问 Agent 的完整状态
  - `runtime.state[...]`:**读**状态
  - `Command(update={...})`:**写**状态
  - 这样工具除了返回结果给模型,还能直接改 Agent 的短期记忆

### 5. 长期记忆的核心:CompositeBackend

**核心思路**:Agent 对文件的操作永远是 `write_file` / `read_file`,**不关心存哪**。存哪由**路径前缀**决定。

- **Backend**:文件到底存在哪里的"后端"
- **CompositeBackend**:组合后端,**按路径路由**到不同的 Backend
- **StateBackend**:存在 Agent State 里,属于短期,新线程看不到
- **StoreBackend**:存在 **Store** 里,属于长期,跨线程保留

```python
CompositeBackend(
    default=StateBackend(),            # 默认:短期
    routes={"/memories/": StoreBackend(...)},   # /memories/ 开头:长期
)
```

于是:

- `write_file("/notes.txt", ...)` → 短期,新对话看不到
- `write_file("/memories/preferences.md", ...)` → 长期,新对话还在

#### namespace:决定"谁能看到这份记忆"

`namespace` 就是 Store 里的"分柜子的标签"。标签怎么设,决定记忆的**作用域(scope)**:

| 作用域       | namespace         | 谁能看                    | 例子                       |
| ------------ | ----------------- | ------------------------- | -------------------------- |
| **Agent 级** | `(assistant_id,)` | 这个 Agent 的所有用户共享 | `AGENTS.md`,Agent 自我改进 |
| **用户级**   | `(user_id,)`      | 只有这个用户              | `preferences.md`           |
| **组织级**   | `(org_id,)`       | 整个组织,**通常只读**     | `compliance.md` 合规政策   |

> 💡 **默认用用户级**。如果 A 用户能写、B 用户能读同一份记忆,恶意用户就能往里**注入指令**(提示词注入)。

#### `rt.server_info` 的坑

部署到 LangSmith / LangGraph Server 时,`rt.server_info` 能给你 `assistant_id` 和登录用户。但**本地 `agent.invoke()` 里它可能是空的**,所以要写兜底逻辑(默认 `"local-user"` 之类),并通过 `context_schema` + `context=` 手动传身份。

### 6. memory= 和 skills= 到底怎么用

- `memory=["/memories/preferences.md"]`:**读取配置**,启动时把这些文件内容塞进**系统提示词**
- `skills=["/skills/"]`:传的是**目录**,先只注入元数据,正文由 Agent **按需读取**

**容易踩的坑(重点!)**:

1. `memory=` 只管**读**,**不会自动创建文件**。文件不存在就直接跳过
2. 所以要先用 `store.put(namespace, key, create_file_data(...))` **预置文件**
3. 写入靠 **`system_prompt` 约定**(告诉它"要记偏好就 `edit_file` 更新哪个文件"),光写 `memory=` 不保证它会用这个文件名
4. `write_file` 会**完整覆盖**,追加或局部修改要用 `edit_file`
5. 验证要用**新 `thread_id`**,不能在同一线程里继续聊就当成验证了(0.7.10 会复用 state 里已有的 `memory_contents`)
6. **别只信模型说"已记住"**,要去 Store 里 `store.get(...)` 核对真的写进去了

### Store key 的路由前缀坑

`CompositeBackend` 会**去掉路由前缀**再存进 Store:

| 配置                            | Agent 看到的路径     | Store 里的 key       |
| ------------------------------- | -------------------- | -------------------- |
| 直接用 `StoreBackend`           | `/skills/x/SKILL.md` | `/skills/x/SKILL.md` |
| 挂在 `routes={"/skills/": ...}` | `/skills/x/SKILL.md` | `/x/SKILL.md`        |

如果你外部预填时 key 多写了 `/skills/`,会变成 `/skills/skills/x/SKILL.md`,找不到。

外部写入要用 `create_file_data` 辅助函数,别手写底层 JSON(v2 格式的 `content` 是字符串,带 `encoding`、`created_at`、`modified_at`)。

### 7. 记忆的六个维度

| 维度     | 问的问题     | 选项                                                         |
| -------- | ------------ | ------------------------------------------------------------ |
| 持续时间 | 留多久       | 短期 / 长期                                                  |
| 信息类型 | 记什么       | **情景记忆**(经历)/ **程序性记忆**(Skills)/ **语义记忆**(事实、偏好) |
| 作用域   | 谁能看       | 用户 / Agent / 组织                                          |
| 更新策略 | 何时写       | 对话中 / 对话间(后台整合)                                    |
| 检索方式 | 怎么读       | 启动加载(`memory=`)/ 按需读取(Skills)                        |
| 权限控制 | Agent 能写吗 | 读写 / 只读                                                  |

三种记忆人话版:

- **语义记忆**:"我是谁、我喜欢啥"(事实)
- **情景记忆**:"上次那个 bug 我是怎么一步步解决的"(完整经历)
- **程序性记忆**:"遇到这类任务应该怎么做"(Skills)

### 8. 高级玩法

- **情景记忆搜索**:Checkpointer 本来就存了所有对话,包一个 `search_past_conversations` 工具就能搜过去的对话。注意 `threads.search()` 只能按元数据筛选,**不能文本搜索**,所以在客户端按关键词匹配
- **后台整合(Background Consolidation)**:另起一个"整合 Agent",在对话间隙审查最近对话、提取事实、合并进记忆,用 **Cron Job** 定时触发(Cron 间隔要和回溯窗口一致,比如都是 6 小时)

|      | 热路径(对话中写) | 后台整合(对话间写)          |
| ---- | ---------------- | --------------------------- |
| 优点 | 即时可用         | 用户无感知延迟,可跨对话综合 |
| 缺点 | 增加延迟         | 下次对话才生效,需额外 Agent |

- **并发写入**:多个线程同时改**同一个文件**会 **last-write-wins**(后写覆盖先写)。对策:
  - 按主题拆文件
  - 对话中追加日志,后台再合并
  - 共享记忆只让后台任务写
  - 敏感记忆加审批

### 9. Store 升级路径

| 场景           | 用啥                                       |
| -------------- | ------------------------------------------ |
| 本地开发       | `InMemoryStore`(重启丢)                    |
| 生产           | `PostgresStore`(要先 `store.setup()` 建表) |
| LangSmith 部署 | 平台自动配,**不用写 store 参数**           |

最佳实践:路径起描述性名字、按主题拆文件、多 Agent 部署在 namespace 里加 `assistant_id`、用 LangSmith trace **审计**记忆写入(审计回答"发生了什么",权限回答"是否允许发生")。

------

# 第二部分:Human-in-the-Loop(第 9 章)

## 1. 为什么要 HITL

Agent 越自主越危险:删错文件、发错邮件、调用贵的 API。所以要一个**可控的中间地带**:
 **Agent 在执行特定操作前暂停,等人类审批 / 修改 / 拒绝 / 直接回答,然后再继续。**

## 2. `interrupt_on`:配置哪些工具要审批

```python
interrupt_on={
    "delete_file": {"allowed_decisions": ["approve", "edit", "reject"]},
    "read_file": False,
    "send_email": {"allowed_decisions": ["approve", "reject"]},
}
```

配置它后,Deep Agents 会自动加上 `HumanInTheLoopMiddleware`。如果运行中途被中断/取消,`PatchToolCallsMiddleware` 会自动修复消息历史。

三种配置值:

| 值                             | 含义                  |
| ------------------------------ | --------------------- |
| `True`                         | 要中断,四种决策都允许 |
| `False`                        | 不中断,直接执行       |
| `{"allowed_decisions": [...]}` | 要中断,只允许指定决策 |

## 3. 四种决策(重点)

| 决策      | 含义                                        | 场景                      |
| --------- | ------------------------------------------- | ------------------------- |
| `approve` | 批准,用 Agent 原参数执行                    | "确认删除"                |
| `edit`    | 改参数后再执行                              | "收件人改一下再发"        |
| `reject`  | 不执行,并把拒绝原因告诉 Agent               | "不要删除"                |
| `respond` | 不执行工具,**人的回答直接当成功的工具结果** | `ask_user` 这类"问人"工具 |

**一个最容易搞混的点:`reject` vs `respond`**

> 记口诀:
>
> - **不同意执行** → `reject`(写清原因和下一步)
> - **同意但要改参数** → `edit`(只改必要参数,别大改)
> - **工具本来就是问人** → `respond`

为什么删文件/发邮件不能用 `respond`?因为 `respond` 的内容会被模型当成**一次成功的 ToolMessage**,模型可能以为"删除成功了"。用 `reject` 才是明确告诉它"没执行"。

`reject` 时建议写清楚 `message`:

> "用户拒绝删除该文件。不要再次尝试删除,请询问是否改为归档文件。"

## 4. 条件中断:`when` 谓词

默认只要工具名在 `interrupt_on` 里,**每次**调用都暂停。加个 `when` 函数可以只拦截危险的:

```python
def writes_outside_workspace(request: ToolCallRequest) -> bool:
    path = request.tool_call["args"].get("file_path", "")
    return not path.startswith("/workspace/")   # True=中断,False=放行
```

需要 `langchain>=1.3.3`。

## 5. 中断与恢复流程(核心)

```
1. Agent 跑,碰到敏感工具
2. 暂停,返回中断信息(result.interrupts)
3. 人看一眼,做决策
4. 用【同一个 thread_id】+ Command(resume=...) 恢复
```

四个关键要求:

1. **必须配 Checkpointer**(没存档就没法恢复)
2. **必须用同一个 `thread_id`**
3. 返回格式:用 `version="v2"` 时读 `result.interrupts` 和 `result.value`;默认 `invoke()` 也支持,中断在返回字典的 `__interrupt__` 里
4. **`decisions` 数量和顺序必须和 `action_requests` 一一对应**

小细节(字段名不一致):

- 展示参数时,可能是 `action_request["arguments"]` 或 `["args"]`,两个都兜底读
- 但 `edit` 决策里的 `edited_action` **永远用 `args`**,且必须带工具名 `name`

**批量调用**:Agent 一次调多个需审批的工具,所有中断会**打包成一个**,你按顺序给每个工具一个决策。

## 6. 子 Agent 独立配置

子 Agent(subagent)可以有**自己的 `interrupt_on`**,覆盖主 Agent 的。典型场景:主 Agent 可信,但子 Agent 处理更敏感的数据,要更严格(比如子 Agent 读文件也要审批)。

## 7. 按风险分层(最佳实践)

| 风险       | 工具                       | 配置                  | 理由                                  |
| ---------- | -------------------------- | --------------------- | ------------------------------------- |
| 高         | 删除、覆盖写入、发送、部署 | `approve/edit/reject` | 不可逆或影响外部,**不开放 `respond`** |
| 中         | 写入、调外部 API           | `approve/reject`      | 能审批但不用改参数                    |
| 低         | 读取、搜索、列表           | `False`               | 只读无副作用                          |
| 人工输入型 | 询问偏好                   | `respond`             | 工具本来就是问人                      |

## 8. 文件系统权限中断

除了 `interrupt_on`,还能用权限规则让内置文件工具触发中断(需 `deepagents>=0.6.8`):

```python
FilesystemPermission(operations=["write"], paths=["/secrets/**"], mode="interrupt")
```

- 触发的中断格式和普通工具审批**一样**,恢复方式也一样
- 会和 `interrupt_on` **合并**,一次审查能覆盖两者
- v0.7 的 `write_file` 会**完整覆盖**文件,内置 `delete` 还能**递归删目录**,所以 `operations=["write"]` 保护的包含覆盖和删除
- 真实文件系统的敏感路径优先用**路径权限**;`interrupt_on` 更适合控制自定义工具

------

# 第三部分:底层机制(第 9 章后半,最难懂的地方)

## 1. `interrupt()` 是什么

- `interrupt_on` 是 Deep Agents 封装的**便捷入口**
- 底层是 **LangGraph 的 `interrupt()` 原语**
- 它**不是普通的 `input()`**:是把控制权交还给调用方,并把恢复所需信息存起来

流程:

1. 节点调用 `interrupt(value)`
2. 运行时包成 `Interrupt` 对象(含给人看的数据 + 稳定的 `id`)
3. 通过 Checkpointer 保存状态、下一步节点、待恢复中断
4. 调用方拿到 `result.interrupts`
5. 人类返回 `Command(resume=...)`
6. 用同一个 `thread_id` 找回 checkpoint,把恢复值送回 `interrupt()`

> 所以 HITL 的关键不是"弹个确认框",而是**几秒后、几小时后、甚至换个进程也能安全恢复**,前提是 Checkpointer 还在、`thread_id` 一致。

## 2. 最关键的一点:恢复时节点从头重放 ⚠️

`interrupt()` 内部是**抛一个特殊异常**来暂停的。恢复时,LangGraph 加载存档后,**从这个节点的第一行重新执行**,而不是从 `interrupt()` 下一行继续。

> 比喻:你看书看到一半被打断,回来后**不是接着那一页**,而是**从这一章开头重读**,只不过读到之前被打断的位置时,会直接拿到你的答案,不再停。

所以 `interrupt()` **之前**的代码会**再跑一次**。由此引出五条规则:

| #    | 规则                                       | 为什么                                                       |
| ---- | ------------------------------------------ | ------------------------------------------------------------ |
| 1    | **不要用裸 `try/except` 包 `interrupt()`** | 它靠异常暂停,裸 `except Exception` 会把它吞掉,暂停就失效。只捕获具体异常(如 `NetworkError`) |
| 2    | **`interrupt()` 之前的副作用必须幂等**     | 重放会重复执行。比如之前 `db.create_log(...)` 会创建多条记录。解决:用 upsert,或把副作用放到 `interrupt()` **之后** |
| 3    | **不要动态改变 `interrupt()` 的调用顺序**  | 同一节点里多个 `interrupt()` 是**按索引**匹配答案的,条件跳过会错位 |
| 4    | **并行中断用 ID 映射恢复**                 | 多个并行节点同时中断时,用 `Interrupt.id` 绑定每个答案,别靠顺序 |
| 5    | **resume 后会重放整个节点**                | 和规则 2 呼应:不能重复执行的动作都放 `interrupt()` 之后      |

> **幂等(idempotent)**:同一个操作做一次和做很多次,结果一样。比如"把状态设为已审批"是幂等的,"新增一条日志"不是。

> **Wrap-style 为什么不推荐做中断**:它的 `handler` 可能被跳过、执行一次或重试多次,在 `handler()` 之后中断,恢复时整个节点重放,模型/工具调用可能**再来一遍**。

## 3. 自定义 Middleware 中的 interrupt()

`interrupt_on` 只管"工具调用前审批"。如果要:

- 发布最终答复前**审稿**
- 进入下一阶段前**确认**
- **跨多个工具**统一策略

就在自定义 Middleware 里直接调 `interrupt()`,通过 `middleware=[...]` 注入。

Middleware 有两类 Hook:

- **Node-style**(`before_model`、`after_model`、`before_agent`、`after_agent`):会成为图中的**独立节点**,**推荐**用来中断
- **Wrap-style**(`wrap_model_call`、`wrap_tool_call`):包在 model/tools 节点内部,适合**重试、缓存、降级**,不适合做人工中断边界

| 需求                   | 推荐位置                             |
| ---------------------- | ------------------------------------ |
| 模型调用前补充缺失信息 | `before_model`                       |
| 模型生成后统一审查     | `after_model`                        |
| 整次运行前后确认       | `before_agent` / `after_agent`       |
| 重试/缓存/模型降级     | `wrap_model_call` / `wrap_tool_call` |

官方的 `HumanInTheLoopMiddleware` 本身就是这样实现的:在 `after_model` 里检查工具调用再 `interrupt()`。

## 4. 单个中断 vs 并行中断

- 只有一个中断:`Command(resume="yes")` 直接传值
- 并行多个中断:用 **`Interrupt.id` → 答案** 的字典,避免回答错配

```python
resume_map = {intr.id: answer for intr in stream.interrupts}
graph.invoke(Command(resume=resume_map), config=config)
```

## 5. 怎么看卡在哪(排查"为什么没继续")

`graph.get_state(config)` 返回快照:

- `values`:当前状态
- `next`:下一步节点
- `interrupts`:待处理的中断
- `checkpoint`:检查点标识

排错思路:`interrupts` 还在 → 还缺 resume;`thread_id` 不对 → 看到的是别的线程。

## 6. 输入校验模式

别在节点里写 `while True` 反复 `interrupt()`(每次恢复都重放,前几轮跟着重跑)。正确做法:**每次节点只 `interrupt()` 一次**,把下一轮问题写回状态,用**条件边**决定是否回到同一节点。

## 7. 静态中断 vs 动态中断

|        | 动态中断 `interrupt()` | 静态中断                                                 |
| ------ | ---------------------- | -------------------------------------------------------- |
| 写在哪 | 节点代码里             | `compile(interrupt_before=[...], interrupt_after=[...])` |
| 恢复   | `Command(resume=...)`  | `invoke(None, config)`                                   |
| 用途   | **生产**的人机协作     | **开发调试**                                             |

## 8. 什么时候用什么

| 场景                   | 方式                                   |
| ---------------------- | -------------------------------------- |
| 给某个工具加审批       | `interrupt_on`                         |
| 按工具参数决定是否审批 | `interrupt_on` + `when`                |
| 模型调用前后统一审查   | Node-style Middleware + `interrupt()`  |
| 节点中收集缺失信息     | 直接 `interrupt()`                     |
| 多轮表单/输入校验      | `interrupt()` + 条件边                 |
| 调试图执行路径         | `interrupt_before` / `interrupt_after` |
| 并行人工任务汇总       | `interrupt()` + ID 映射                |

------

# 两章怎么串起来

```
          ┌──────────── Checkpointer(存档)+ thread_id ────────────┐
          │                                                        │
   第 8 章:记忆                                           第 9 章:人机协作
   - 短期:同 thread 内读档                                - 暂停:存档
   - 长期:Store 跨 thread                                 - 恢复:同 thread_id 读档
          │                                                        │
          └──── 交集:敏感记忆路径加 HITL 审批 / 权限只读 ─────────┘
```

**交集点**(第 8 章也提到):

- 组织级记忆要**只读**,防提示词注入
- 敏感路径写入前加 **Human-in-the-Loop** 审批
- 用 `FilesystemPermission(mode="interrupt")` 保护敏感记忆路径

------

# 术语速查表

| 术语                                  | 一句话解释                                         |
| ------------------------------------- | -------------------------------------------------- |
| **Thread / thread_id**                | 一段独立对话 / 它的存档编号                        |
| **Checkpointer**                      | 存档器,保存每一步状态,支撑短期记忆和中断恢复       |
| **MemorySaver / PostgresSaver**       | 内存版(开发)/ 数据库版(生产)的 Checkpointer        |
| **Store**                             | 跨线程的共享存储,支撑长期记忆                      |
| **InMemoryStore / PostgresStore**     | 内存版 / 数据库版的 Store                          |
| **StateBackend**                      | 文件存在 Agent State,短期                          |
| **StoreBackend**                      | 文件存在 Store,长期                                |
| **CompositeBackend**                  | 按路径路由到不同 Backend                           |
| **namespace**                         | Store 里的"柜子标签",决定记忆作用域                |
| **Agent 级 / 用户级 / 组织级**        | 记忆的三种作用域                                   |
| **语义 / 情景 / 程序性记忆**          | 事实偏好 / 完整经历 / 做事方法(Skills)             |
| **context window**                    | LLM 一次能看的最大内容长度                         |
| **Trim / Delete / Summarize**         | 裁剪 / 删除 / 总结,三种控制对话长度的策略          |
| **SummarizationMiddleware**           | 内置的自动总结中间件                               |
| **Middleware(中间件)**                | 在 Agent 流程中插入逻辑的机制                      |
| **装饰器 `@before_model`**            | 把函数注册成"模型调用前"的钩子                     |
| **AgentState**                        | Agent 的状态结构,可扩展自定义字段                  |
| **ToolRuntime**                       | 工具的隐藏参数,用来读写 Agent 状态                 |
| **热路径 / 后台整合**                 | 对话中写记忆 / 对话间由整合 Agent 写               |
| **Cron Job**                          | 定时任务                                           |
| **last-write-wins**                   | 并发写同一文件时后写覆盖先写                       |
| **HITL**                              | Human-in-the-Loop,人机协作                         |
| **`interrupt_on`**                    | 配置哪些工具需要人工审批                           |
| **approve / edit / reject / respond** | 批准 / 改参数 / 拒绝 / 人代答                      |
| **`when` 谓词**                       | 条件中断,只拦截特定调用                            |
| **`Command(resume=...)`**             | 带着人的决策恢复执行                               |
| **`interrupt()`**                     | LangGraph 底层暂停原语,靠抛异常实现                |
| **`Interrupt.id`**                    | 每个中断的唯一 ID,并行恢复时用                     |
| **幂等**                              | 重复执行结果不变                                   |
| **Node-style / Wrap-style Hook**      | 独立节点式钩子(适合中断)/ 包裹式钩子(适合重试缓存) |
| **静态中断**                          | `interrupt_before/after`,调试用                    |
| **FilesystemPermission**              | 文件系统权限规则,可设 `mode="interrupt"`           |
| **提示词注入**                        | 恶意内容混进记忆/输入,诱导 Agent 执行指令          |
| **LangSmith**                         | LangChain 的部署/追踪平台,可审计记忆写入           |

------

# 自己在学的时候弄混了点

1. **Checkpointer 和 Store 不是一回事**:前者按 `thread_id` 隔离(短期),后者跨线程共享(长期)
2. **`memory=` 只读不建文件,也不保证写入**:要预置文件 + 在提示词里约定写入路径,并去 Store 核对
3. **`reject` 和 `respond` 别混**:危险操作用 `reject`,`respond` 会被当成"成功"
4. **恢复是从节点开头重放**,不是接着往下:`interrupt()` 前的副作用要幂等
5. **别裸 `try/except` 包 `interrupt()`**:它靠异常暂停,会被吞掉