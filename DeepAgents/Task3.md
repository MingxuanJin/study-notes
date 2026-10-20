# Task3打卡内容

检查模板的 Todo 配置；启用后观察计划生成与状态变化。

显式启用todolistmiddleware，用固定资料完成多步骤任务

## 检查了模版的todo配置

```bash
sed -n '15,25p;155,175p' src/research_deepagent/agent.py
grep -n "write_todos" src/research_deepagent/prompts.py
```

确认了以下几点：

1. 导入存在（第 20 行）；
2. `build_graph` 里 `middleware=[TodoListMiddleware()]` 只出现**一次**；
3. 提示词要求先计划：`prompts.py:7`（Plan first）、`prompts.py:21`（一次响应最多一次 `write_todos`）；
4. middleware 只给工具和 state，不强制模型调用——所以提示词和 middleware 缺一不可。

## 运行时确认（offline）

```bash
uv run --frozen python - <<'EOF'
import sys; sys.path.insert(0, "tests")
from langchain_core.messages import AIMessage
from scripted_model import ScriptedModel
from research_deepagent.agent import build_graph

model = ScriptedModel(replies=[AIMessage(content="ok")])
graph = build_graph(model)
graph.invoke({"messages": [{"role": "user", "content": "x"}]})
print(sorted(model.seen_tools))
assert "write_todos" in model.seen_tools
EOF
```

关于测试脚本代码：

**`python - <<'EOF'`**：通过标准输入（stdin）把多行脚本传给 Python 解释器执行（Here-Doc 语法），无需先保存成单独的 `.py` 文件

`sys.path.insert(0, "tests")`：把 `tests` 目录加入到 Python 模块搜索路径最前端，以便直接导入测试专用的辅助模块

`ScriptedModel`：测试中常用的 **Mock 模型（模拟模型）**。它不会真正调用 OpenAI/Anthropic 等外部 API，而是直接按预设剧本返回内容，同时记录模型被调用时的参数（例如绑定的工具）

`build_graph`：待测试的核心业务代码，用来构建一个智能体计算图（Agent Workflow/Graph）

`graph.invoke(...)`：向图输入一条测试消息 `{"role": "user", "content": "x"}`，触发 Agent 执行一步

`assert "write_todos" in model.seen_tools`：**断言检查**。验证 `write_todos` 这个待办事项记录工具是否成功绑定给了模型。如果不在列表中，程序会抛出 `AssertionError` 报错；如果在，则脚本静默正常退出

> 断言检查：认为此处条件绝对成立，如果不成立，代码直接报错停止
>
> 语法：assert 条件表达式, "错误提示信息（可选）"

返回的结果：

```bash
['delete', 'edit_file', 'glob', 'grep', 'ls', 'read_file', 'task', 'tavily_search', 'think_tool', 'write_file', 'write_todos']
```

确认了工具里有write_todos~

## 观察计划生成与状态变化(offline)

用现有的测试确认了状态真的进了图

```bash
uv run --frozen pytest tests/test_planning.py -q
```

**PASS**：`1 passed`

断言流里出现过 `in_progress`、最后是 `completed`，并且 checkpoint 里的 `todos` 也是 `completed`

## Live验证

使用模型deepseek-flash，使用的prompt写了一个强制多步的请求：

先写一个 3 步计划，然后按顺序执行：用 tavily_search 查 LangGraph 1.0 的两个新特性，写 /report.md，读回 /report.md 核对，全部完成后再把 todo 标成 completed。

在前端可以观察到：使用tool write_todos,返回以下的内容

```python
{
  "todos": [
    {
      "content": "查询 LangGraph 1.0 的两个新特性（tavily_search）",
      "status": "in_progress"
    },
    {
      "content": "把两个新特性写入 /report.md",
      "status": "pending"
    },
    {
      "content": "读回 /report.md 核对内容，全部完成后将 todo 标记 completed",
      "status": "pending"
    }
  ]
}
```

```python
{
  "todos": [
    {
      "content": "查询 LangGraph 1.0 的两个新特性（tavily_search）",
      "status": "completed"
    },
    {
      "content": "把两个新特性写入 /report.md",
      "status": "completed"
    },
    {
      "content": "读回 /report.md 核对内容，全部完成后将 todo 标记 completed",
      "status": "completed"
```

和课程中学的一样！

UI只能用来观察，判定需要以thread state为准

测试用代码：

```bash
uv run --frozen python - <<'EOF'
from langgraph_sdk import get_sync_client
c = get_sync_client(url="http://127.0.0.1:2024")
for t in c.threads.search(limit=3):
    s = c.threads.get_state(t["thread_id"])["values"]
    print(t["thread_id"], [x["status"] for x in s.get("todos", [])], list(s.get("files", {})))
EOF
```

返回的结果

```bash
3376dd79-1b56-4f33-b410-099a37b36ccc ['completed', 'completed', 'completed'] ['/research_request.md', '/report.md']
```

说明todo确实完成啦！

## 固定资料+多步骤任务

不联网，只用固定资料进行验证，prompt如下：

只使用我给的固定资料，不要调用 tavily_search，也不要补充资料外的事实。

固定资料（合成课程记录）：
M1. 2026-10-18 记录：deepagents 0.7 必须显式传入 TodoListMiddleware，write_todos 才会出现在工具列表；不传则没有该工具。
M2. 同一次记录：一个模型响应里最多允许一次 write_todos，并行的多次调用会被 middleware 拒绝。
M3. 同一次记录：todos 随 checkpoint 持久化，重启服务后仍能从 thread state 读回；但 todo 标成 completed 不代表报告已经写入。

请按以下步骤执行，每步完成后再做下一步：
1. write_todos 建立计划（至少 3 项，第一项 in_progress）
2. write_file 把上面的固定资料原文保存到 /materials/brief.md
3. task 交给 research-agent，要求它 read_file('/materials/brief.md') 后按 M1/M2/M3 整理三段要点，不要搜索
4. 写 /final_report.md，三个小节分别对应 M1/M2/M3，引用用 [1][2][3]
5. read_file 读回 /final_report.md 和 /materials/brief.md，核对后把对应 todo 标 completed

返回的结果：

```python
五个步骤已按顺序完成，全程只使用你给出的固定资料，未调用 tavily_search，未补充资料外的事实。

报告内容（/final_report.md 正文要点）

M1：write_todos 需显式传入 TodoListMiddleware — deepagents 0.7 必须显式传入 TodoListMiddleware，write_todos 才会出现在工具列表；不传则没有该工具 [1]。资料未说明不传时的报错形式、middleware 的传参位置与配置写法，也未提及其他版本行为。

M2：一个模型响应里最多一次 write_todos — 上限的作用范围是“一个模型响应里”，最多一次；并行的多次调用会被 middleware 拒绝 [2]。资料未说明拒绝的具体形式（报错内容、是否重试）、跨响应或跨轮次如何计数，也未对非并行的多次调用作说明。

M3：todos 随 checkpoint 持久化，completed ≠ 报告已写入 — todos 随 checkpoint 持久化，重启服务后仍能从 thread state 读回；但 todo 标成 completed 不代表报告已经写入 [3]。该条只是提示二者不等价，既未断言报告内容正确，也未断言报告实际未写入，更没有说明历史是否被删除或保留。

报告另有一节说明三条记录分别指向不同的条件与结论，资料本身没有给出三者之间更多的因果说明。

执行与核对记录

计划：4 项 todo，第一项一开始设为 in_progress，并逐项推进至全部 completed。
资料：/materials/brief.md 已保存 M1/M2/M3 原文（含“合成课程记录”标注），并读回逐字核对无误。
子代理：research-agent 按要求先 read_file('/materials/brief.md')，仅作整理、未联网，返回的三段要点与原文口径一致。
核对：读回 /final_report.md 与 /materials/brief.md 逐条比对，M1/M2/M3 的措辞与条件、结论均与原文对应；这 3 个编号的来源都是同一份固定资料 M1/M2/M3 条，并非不同网址。
Sources
[1] 合成课程记录 brief.md，M1 条（2026-10-18）：/materials/brief.md [2] 合成课程记录 brief.md，M2 条（2026-10-18）：/materials/brief.md [3] 合成课程记录 brief.md，M3 条（2026-10-18）：/materials/brief.md


```

进行state验收：

```bash
uv run --frozen python - <<'EOF'
from langgraph_sdk import get_sync_client
c = get_sync_client(url="http://127.0.0.1:2024")
t = c.threads.search(limit=1)[0]["thread_id"]
s = c.threads.get_state(t)["values"]
print("todos:", [(x["status"], x["content"]) for x in s.get("todos", [])])
print("files:", list(s.get("files", {})))
print(s["files"]["/final_report.md"]["content"])
EOF
```

返回的结果：

```bash
todos: [('completed', '把固定资料原文保存到 /materials/brief.md'), ('completed', '委派 research-agent 读取 /materials/brief.md 并按 M1/M2/M3 整理三段要点（不搜索）'), ('completed', '写 /final_report.md，三个小节对应 M1/M2/M3，使用 [1][2][3] 引用'), ('completed', '读回 /final_report.md 与 /materials/brief.md 核对，并标记 todo 完成')]
files: ['/materials/brief.md', '/final_report.md']
# deepagents 0.7 中 write_todos 的三点记录整理

本报告仅依据合成的课程记录整理，未引入资料以外的内容，也未参考任何外部资料。全部信息来自 `/materials/brief.md` 的 M1、M2、M3 三条记录，三条同属 2026-10-18 的同一次记录。

## M1：write_todos 需要显式传入 TodoListMiddleware 才会出现

该条记录指出，在 deepagents 0.7 中，必须显式传入 TodoListMiddleware，`write_todos` 才会出现在工具列表；如果不传，就没有这个工具 [1]。这里描述的是"是否显式传入 TodoListMiddleware"与"`write_todos` 是否出现在工具列表"之间的对应关系。

该条没有说明的是：不传时调用该工具会出现什么错误或提示、该 middleware 的传参位置与配置写法，也没有提及其他版本的行为 [1]。

## M2：一个模型响应里最多允许一次 write_todos

该条记录指出，一个模型响应里最多允许一次 `write_todos`，并行的多次调用会被 middleware 拒绝 [2]。也就是说，次数上限的作用范围是"一个模型响应里"，上限是"最多一次"，被拒绝的情形是"并行的多次调用"，拒绝方是 middleware。

该条没有说明的是：拒绝的具体形式（如报错内容、是否重试）、跨响应或跨轮次如何计数，也没有对非并行的多次调用作出说明 [2]。

## M3：todos 随 checkpoint 持久化，但 completed 不代表报告已写入

该条记录指出，todos 随 checkpoint 持久化，重启服务后仍能从 thread state 读回；但 todo 标成 completed 不代表报告已经写入 [3]。这里区分了两件事：一是 todos 的持久化载体是 checkpoint、重启后可从 thread state 读回；二是 `completed` 状态与"报告已经写入"之间不构成等价关系。

需要保持的边界是：该条只是提示"completed 不代表报告已经写入"，并未断言报告内容正确，也未断言报告实际未写入；同时没有说明历史是否被删除或保留，没有说明 checkpoint 的存放位置、thread state 的具体读取方式，也没有给出 completed 的判定标准 [3]。因此，把 todo 标为 completed 只能说明该待办项被标记完成，不能作为报告已生成的证据。

## 三点之间的关系

三条记录分别涉及三个不同层面：`write_todos` 是否出现取决于是否显式传入 TodoListMiddleware [1]；该工具出现之后，一个模型响应里最多允许一次调用，并行多次调用会被 middleware 拒绝 [2]；todos 随 checkpoint 持久化、重启后可从 thread state 读回，但 completed 标记不代表报告已经写入 [3]。三条记录各自指向不同的条件与结论，资料本身没有给出三者之间更多的因果说明。

### Sources
[1] 合成课程记录 brief.md，M1 条（2026-10-18）：/materials/brief.md
[2] 合成课程记录 brief.md，M2 条（2026-10-18）：/materials/brief.md
[3] 合成课程记录 brief.md，M3 条（2026-10-18）：/materials/brief.md
```

验证的结论：

- 过程里出现过 `in_progress`，最终所有 todo 为 `completed`；
- `files` 同时包含 `/materials/brief.md` 和 `/final_report.md`；
- 报告含三个小节、`[1][2][3]` 和 `### Sources`，Sources 映射到 M1/M2/M3；
- 报告里**没有 http(s) 链接**，说明没有偷用搜索结果；
- 因为 research 模板是 **StateBackend**，文件在 thread state 里，不在磁盘上，所以 `ls` 找不到

# 笔记

声明：本笔记内容来源于对datawhialechina deep-agents-in-action项目第一章以及第二章内容的学习，相关著作权归作者本人

教程链接：https://datawhalechina.github.io/deepagents-in-action/

Chap.4

从 v0.7 开始，write_todos按需启用，不再是每个 Agent 的固定配置，可以帮助 Agent 拆解任务、追踪进度

## 为什么 Agent 需要”规划”能力？

### 简单任务 vs 复杂任务

对于简单的任务，Agent一步就可以做完，比如问北京的天气，调用一次tool就可以解决；而复杂的任务需要多步，比如“帮我调研 LangGraph 的技术架构，对比三个竞品，写一份 3000 字的分析报告”

缺少明确的任务清单时，Agent 更容易遗漏步骤或重复搜索。规划可以帮助它追踪进度，具体效果仍要通过实际任务验证

就像如果人有todo list就更容易有规划地完成需要做的内容，不遗漏，agent也一样

### 没有规划的 Agent 会怎样？

- **遗漏关键步骤**：直接开始写报告，忘了先搜索竞品信息
- **重复劳动**：搜索了同一个关键词三次，因为它”忘记”已经搜过了
- **半途而废**：上下文太长后，Agent 失去了对整体进度的把控
- **质量不稳定**：有时做得很好，有时莫名跳过重要环节



## 在 v0.7 中显式启用任务规划

`TodoListMiddleware` 会同时注入 `write_todos` 工具、`todos` 状态和规划提示词。v0.7 默认不再安装它，需要时应明确传入

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_deep_agent(
    model=model,
    middleware=[TodoListMiddleware()],
)
```

| 场景                            | 建议                                   |
| ------------------------------- | -------------------------------------- |
| 单步问答、短工具调用            | 保持关闭，避免计划比任务本身还长       |
| 长程、多阶段、容易漏步骤的任务  | 启用，并用真实任务检查完成率和轨迹长度 |
| 能力较弱、容易失去主线的模型    | 先做 A/B 评测，通常值得尝试            |
| UI 需要展示计划、当前步骤和进度 | 启用；此时 `todos` 也是产品状态协议    |

## `write_todos` 工具详解

启用后，Deep Agents 会获得 `write_todos` 工具，让 Agent 可以创建和管理任务清单。工具存在<span style="color: #b8860b">不代表每个任务都会调用</span>；具体行为仍由模型、提示词和任务复杂度共同决定

### 任务的数据结构

每个任务包含content + status, status分为三种：pending, in_progress, completed

<span style="color: #b8860b">`completed` 是 Agent 写入的进度标记。应用仍需检查产物：例如报告是否生成、引用能否核实，或修改后的代码是否通过测试。仅看清单全部完成，不能判断任务质量。</span>

### Agent 怎么用 write_todos？

**第一步：制定计划**

```
Agent 思考：这个任务比较复杂，我先拆解一下。
Agent 调用 write_todos：
  1. [pending] 搜索 LangGraph 官方文档和核心概念
  2. [pending] 搜索三个竞品（Temporal、Inngest、Prefect）
  3. [pending] 对比分析各产品的优劣势
  4. [pending] 撰写报告大纲
  5. [pending] 撰写完整报告
```

**第二步：逐步执行**

```
Agent 更新任务 1 状态为 in_progress
Agent 调用 internet_search("LangGraph architecture")
Agent 调用 write_file("/workspace/langgraph_notes.md", ...)
Agent 更新任务 1 状态为 completed

Agent 更新任务 2 状态为 in_progress
Agent 调用 internet_search("Temporal vs LangGraph")
...
```

**第三步：动态调整**

在执行过程中，Agent 可能发现需要额外的步骤：

```
Agent 思考：搜索时发现 Prefect 不太合适，应该换成 Durable Objects。
Agent 调用 write_todos 更新列表：
  1. [completed] 搜索 LangGraph 官方文档和核心概念
  2. [in_progress] 搜索三个竞品
  3. [pending] 对比分析各产品的优劣势
  4. [pending] 撰写报告大纲
  5. [pending] 撰写完整报告
  6. [pending] 补充 Durable Objects 的资料 ← 新增
```

### 任务清单的持久化

<span style="color: #b8860b">任务清单保存在 **Agent State 的 `todos` 字段**中，和消息历史分开管理</span>

#### In-Run Memory: 单次执行内的状态保留

- 同一次运行中，后续步骤可以继续访问这份清单；默认的对话总结不会删除 `todos` 字段 

  > 这一点和Agent State的消息历史管理不一样，在同一次调用（一次 `invoke()` 或一次运行流程）中，Agent 执行多步工具调用或多轮思考时，清单字段 `todos` 是一个持久存在于图状态（Graph State）中的键，任何后续 Step 都能随时读取或更新
  >
  > 当对话轮次过长触发系统自动“摘要/修剪”（Conversation Summarization）时，被修剪的通常只是历史消息（`messages` 列表），而类似 `todos` 这种专用状态字段会被显式保留，不会被当成旧对话扔掉



#### Cross-Turn Persistence: 跨多次调用的持久化

- 多次 `invoke()` 之间要自动接续清单，需要配置 Checkpointer，并复用同一个 `thread_id`；只添加 `TodoListMiddleware` 不会自动接续上次运行的状态
- `InMemorySaver` 只在当前进程中保存检查点；进程重启后还要恢复，需要使用数据库等持久化 Checkpointer。部署到 Agent Server 时，由平台提供相应的持久化能力

> TodoListMiddleware只负责管理todo，本身不负责在跨轮次调用之间保存数据，要实现“用户发两句不同的话，Agent 依然记得上次做了一半的清单”，必须传入 `checkpointer，且每次调用必须指定相同的 `thread_id`
>
> <span style="color: #b8860b">状态接续需要：thread_id相同并且配置checkpointer</span>



#### Middleware Inheritance

- 默认配置下，`general-purpose` 子 Agent 和 `subagents=[...]` 声明的子 Agent 都不会继承主 Agent 的 `TodoListMiddleware`。需要规划能力时，应在子 Agent 自己的 `middleware` 字段中启用；默认 `general-purpose` 需用[同名 spec 覆盖](https://datawhalechina.github.io/deepagents-in-action/chapters/ch05-subagents/#为-general-purpose-启用-todo)

> 主 Agent 配置了 `TodoListMiddleware`，并不代表它生出的“通用子 Agent”或自定义 `subagents` 也能自动使用清单工具
>
> 如果某个子任务需要子 Agent 自主拆解步骤做规划，必须在该子 Agent 的配置定义中**显式添加** `TodoListMiddleware`；如果是系统内置的同名通用规格，需要通过配置覆盖来开启

#### 子Agent的数据完全隔离

- 在默认的隔离模式下，子 Agent 的清单保存在自己的 `todos` 状态中，不会自动读取主 Agent 的清单，也不会将清单写回主 Agent

>  主 Agent 与子 Agent 各自拥有一份独立的 `todos` 空间

总之，清单在单次运行中默认安全，跨会话要靠 Checkpointer 存档，而在父子 Agent 之间互相隔离



## 揭开引擎盖：LangChain 中间件

还记得第 1 章的三层架构吗？Deep Agents（Harness）构建在 LangChain（Framework）之上。而 LangChain 提供了一套**中间件（Middleware）**系统——它是 Agent 能力的插件机制。`create_deep_agent()` 内部做的事情，本质上就是把一组中间件**自动组装**到了 Agent 上。



### 先分清两类 Hook

LangChain 中间件提供两类执行边界。它们都叫 Hook，但适合解决的问题并不相同：

| 风格           | Hook                                                         | 执行方式                                                     | 适合场景                         | 能否跳过被调对象                                             | 能否重复调用                                                |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| **Node-style** | `before_agent`、`before_model`、`after_model`、`after_agent` | 编译成 Agent 图中的独立节点，按生命周期顺序运行              | 校验、状态更新、审计、人工中断   | 通常不能随意“拦截跳过”，主要是读取/修改状态后传递给下一节点  | 只能单次流转（除非图自身定义了循环边                        |
| **Wrap-style** | `wrap_model_call`、`wrap_tool_call`                          | 包裹一次模型或工具调用；可以不调用、调用一次或多次 `handler` | 重试、缓存、降级、请求或响应转换 | **可以**。它可以选择根本不触发底层的模型/工具调用（直接返回缓存） | **可以**。底层失败时可以立刻捕获异常并在原地 `retry` 若干次 |

这个区别对 `interrupt()` 尤其重要：Node-style Hook 有清晰的图节点边界，暂停与恢复时更容易推断哪些逻辑会重放；Wrap-style Hook 位于 model/tools 节点内部，恢复时可能连同 `handler` 一起重新执行。因此自定义人工中断优先放在 Node-style Hook 中，Wrap-style 即使技术上可以调用，也不适合作为默认中断边界。



> Hook是一种设计模式，相当于在程序执行主流程的特定关卡上预留的挂钩或者插槽
>
> 主程序在执行到某个阶段的时候会停下来，看挂钩上有没有自定义逻辑，有的话就执行自定义的，没有的话就继续主流程
>
> 核心作用是实现关注点分离

> Node-style Hook是图状态机生命周期的一部分，主要用于**处理状态（State）的变化与合规检查**。
>
> - **校验与阻断（before_model）：** 检查用户输入是否包含敏感词或违规指令；如果不合规，直接改变状态转向安全警告。
> - **状态更新与注入（before_agent）：** 在 Agent 启动前，读取外部会话历史或动态将用户权限、环境参数写入当前 Graph State。
> - **审计与日志（after_model / after_agent）：** 模型生成完毕或整轮会话结束后，记录消耗的 Token、耗时与调用轨迹到审计数据库。
> - **人工介入（Human-in-the-loop）：** 在关键操作前设置断点挂起（Interrupt），等待人工审核确认后再继续流程。

> Wrap-style Hook直接接管了“单次模型推理”或“单次工具执行”的控制权，主要用于**提高调用的健壮性、效率与输入输出变形**。

```python
# Wrap-style 的直观逻辑示意
def wrap_tool_call(request, handler):
    # 1. 缓存拦截：如果不调用 handler，直接命中返回
    if hit_cache(request):
        return get_from_cache(request)

    # 2. 重试机制：可以调用 0 次、1 次或多次 handler
    for attempt in range(3):
        try:
            response = handler(request)
            return response
        except NetworkError:
            continue

    # 3. 降级方案
    return fallback_response()
```

> [!NOTE]
>
> 如果想**观察或修改对话上下文、保存状态、审计流程、或者让人工审核介入流程**，选 **Node-style**
>
> 如果想**防网络抖动（重试）、提速降本（缓存）、故障保活（降级）或拦截具体的 API 调用**，选 **Wrap-style**



在 v0.7 中，可以按配置方式理解 `create_deep_agent()` 组装的中间件

### 按照配置方式理解

下面三组说明<span style="color: #b8860b">能力从哪里加入</span>，不代表三个执行层级

#### **默认中间件**

- `FilesystemMiddleware` — 注入 7 个文件工具，并执行 `permissions` 权限规则
- `SummarizationMiddleware` — 上下文自动压缩，触发阈值可配置
- `PatchToolCallsMiddleware` — 补齐消息历史中缺失的工具响应
- Prompt caching 等模型相关能力 — 是否启用取决于模型和 Harness profile



#### **通过专用参数配置**

`skills=`、`memory=`、`interrupt_on=` 等参数用于配置相应能力，由 Deep Agents 组装对应的中间件。

- `SubAgentMiddleware` — 由 `subagents=` 配置，默认包含 `general-purpose` 子 Agent；提供 `task` 工具

- `SkillsMiddleware` — 传入 `skills=` 参数时启用，注入技能包

- `AsyncSubAgentMiddleware` — 传入异步子 Agent 时启用

- `MemoryMiddleware` — 传入 `memory=` 参数时启用，注入 AGENTS.md 记忆

- `HumanInTheLoopMiddleware` — 传入 `interrupt_on=` 参数时启用，拦截指定工具调用等待人工审批

  

#### **通过 `middleware` 配置**（传入 `middleware=[...]`）

- `TodoListMiddleware` 等可选策略可以按需加入
- 与默认 Middleware 同名的实例会在原位置换默认实例，而不是在末尾再叠加一个
- 原位置换是完整实例替换，不会把新旧配置按字段合并



理解这些配置方式，你就能：

- 看懂 Deep Agents 内部是怎么拼装出来的
- 自己按需添加新能力（PII 脱敏、模型降级、调用次数限制……）
- 在更底层的 LangChain `create_agent()` 上搭建定制化的 Agent

### PatchToolCallsMiddleware：补齐哪一种“缺口”？

模型发出工具调用后，消息历史通常还需要一条对应的工具响应

如果这次运行在工具响应写回前被取消，历史里可能只剩下调用请求。下一次 Agent 运行开始前，`PatchToolCallsMiddleware` 会在 `before_agent` Hook 中检查历史，为缺失的响应补上一条 `ToolMessage`，说明这次调用被取消；对于参数损坏或截断的调用，则说明它未能执行。

假设 Agent 准备搜索 LangGraph 的资料：

1. **发起调用**：模型请求执行 `internet_search("LangGraph")`，调用 ID 为 `call_1`。
2. **运行被取消**：工具响应还没写回，消息历史里缺少 `call_1` 对应的响应。
3. **补齐响应**：下一次 Agent 运行开始前，`PatchToolCallsMiddleware` 补上一条 `tool_call_id` 为 `call_1` 的 `ToolMessage`，说明上次调用已取消。

<span style="color: #b8860b">它补齐的是消息记录，不会重新执行搜索，也不会把错误结果改成正确结果。</span>

已经有对应工具响应的调用，即使返回的是错误信息，也不属于这种缺口。补上的取消说明也不能证明外部操作已撤销，例如服务端可能已经处理了请求，只是客户端没有拿到结果。

| 遇到什么情况                     | 谁来处理                                | 需要注意什么                                         |
| -------------------------------- | --------------------------------------- | ---------------------------------------------------- |
| 历史里有工具调用，却缺少对应响应 | `PatchToolCallsMiddleware` 补充说明消息 | 不重试工具，不验证结果是否正确                       |
| 工具调用遇到可重试的异常         | 应用显式配置的 `ToolRetryMiddleware`    | 是否重试、重试次数由策略决定；写操作还要考虑重复执行 |
| 下一次运行继续上一次的状态       | Checkpointer 配合同一个 `thread_id`     | 跨进程恢复需要持久化 Checkpointer                    |
| 工具返回了内容，但内容是否正确   | 应用的校验、评测或人工审核              | 调用成功、清单完成都不等于结果正确                   |

**Checkpointer 属于 LangGraph 的运行时持久化机制**，通过 `checkpointer=` 配置，不是 `middleware=[...]` 中的一项。

### TodoListMiddleware：write_todos 的真身

`write_todos` 的底层实现是 LangChain 的 `TodoListMiddleware`。无论使用 Deep Agents 的 `create_deep_agent()`，还是更底层的 `create_agent()`，v0.7 都需要显式添加这项能力

```python
import os
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware
from deepagents.middleware import FilesystemMiddleware

model = ChatOpenAI(
    # 任务规划属于复杂推理场景，建议使用能力较强、支持工具调用的模型
    model="zai-org/GLM-5.2",
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

agent = create_agent(
    model=model,
    tools=[],
    middleware=[
        TodoListMiddleware(),
        FilesystemMiddleware(),   # 自动注入 read_file / write_file 等文件工具
    ],
)
```

添加 `TodoListMiddleware` 后，Agent 会自动获得：

1. **`write_todos` 工具** — 创建和管理任务清单
2. **`todos` 状态** — 保存任务及其状态，供后续步骤和 UI 使用；跨次调用的接续由 Checkpointer 负责
3. **规划指导提示词** — 引导 Agent 在面对复杂任务时先规划再执行

> [!NOTE]
>
> TodoListMiddleWare会提供：用于创建和管理任务清单的工具（write_todos），todos状态和规划指导提示词

#### 自定义配置

`TodoListMiddleware` 支持两个可选参数

```python
TodoListMiddleware(
    system_prompt="...",      # 自定义规划指导提示词
    tool_description="...",   # 自定义 write_todos 工具的描述
)
```

大多数情况下，默认配置就够了。只有当你发现 Agent 的规划行为需要特别引导时（比如”总是先写测试再写代码”），才需要自定义 `system_prompt`

### SummarizationMiddleware：上下文压缩的真身

第 3 章讲的“对话历史自动总结”，由 Deep Agents 的 `SummarizationMiddleware` 负责。LangChain 也有一个同名中间件，Deep Agents 复用了它的摘要逻辑，但两者的执行位置和消息处理方式不同。

| 对比项           | LangChain 版本                                               | Deep Agents 版本                                             |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 导入位置         | `langchain.agents.middleware`                                | `deepagents.middleware`                                      |
| 执行位置         | `before_model` / `abefore_model`，在图中形成独立节点         | `wrap_model_call` / `awrap_model_call`，在 `model` 节点内部执行 |
| 正常摘要后的消息 | 将当前 `state["messages"]` 中的旧消息替换为摘要，保留近期消息 | 保留原始消息，另外记录摘要和截断位置；本次模型输入使用“摘要 + 近期消息” |
| 历史保存         | 不负责将旧消息写入 Backend                                   | 将待总结的旧消息写入 Backend，摘要中附上保存路径             |

#### LangChain vs. LangGraph

因此，用 `agent.get_graph()` 查看结构时，LangChain 版本会出现 `SummarizationMiddleware.before_model` 节点；Deep Agents 版本的摘要逻辑位于 `model` 节点内部，不会单独出现摘要节点。图展示的是节点结构，不会把节点内部的包装逻辑全部展开。

> **LangChain 方式（Node-style）：** 摘要逻辑被编译为一个显式的图节点（`before_model`）。在拓扑图上，它是流水线中独立的一环：`[输入] -> [摘要检查节点] -> [模型节点]`。
>
> **Deep Agents 方式（Wrap-style）：** 摘要逻辑被直接包在 `model` 节点的外层（类似上一问讲的 `wrap_model_call`）.图中看不到摘要节点不代表没有摘要功能，只是被封装在了内部

#### Deep Agents 的摘要实现细节与配套中间件

Deep Agents 需要在长任务中压缩模型输入，同时保留历史供后续查阅。它通过 `request.override(messages=...)` 调整本次请求，并将摘要记录在单独的状态字段中；这不等于只注入工具和提示词。需要按路径读回历史时，还要配置使用同一 Backend 的 `FilesystemMiddleware`。提供 `compact_conversation` 工具的则是另一个 `SummarizationToolMiddleware`，与这里的自动摘要分开配置。

这里厘清了三个不同维度的职责：

1. **输入裁剪与状态分离：** Deep Agents 在摘要时，通过 `request.override(messages=...)` 临时替换发给 LLM 的请求上下文（给模型瘦身），但原始详细记录并没有被粗暴丢弃，而是归档到单独的状态字段。
2. **读回历史与文件系统中间件（`FilesystemMiddleware`）：** 模型如果后续想按路径查阅被压缩/归档的完整历史，必须配置基于相同后端存储的 `FilesystemMiddleware`，以便它像读取虚拟文件一样翻阅历史。
3. **自动触发 vs 工具调用（两套中间件）：**
   - **自动静默压缩：** 由后台摘要中间件处理（本段核心讨论的对象）。
   - **由 Agent 主动调用工具压缩：** 属于 `SummarizationToolMiddleware`，它向 Agent 暴露一个名为 `compact_conversation` 的 Tool。两者是分开配置的，不要混为一谈。

#### 为什么手动组装时摘要“失效

**手动实例化时要设置 `trigger`。** 两个同名类的默认值都是 `None`，不会按阈值主动摘要。`create_deep_agent()` 会为默认摘要中间件选择触发条件；直接用 `create_agent(middleware=[...])` 组装时，不会自动获得这组默认配置。图中是否有摘要节点，也不能证明摘要已经触发。

**开箱即用 vs 手动组装：**

- 使用高级封装函数 `create_deep_agent()` 时，框架会自动为你填入合理的阈值。
- 如果使用底层函数 `create_agent(middleware=[...])` 手动装配，摘要中间件的 `trigger` 默认值是 `None`。**如果不显式传参数，它就永远不会主动触发。**

**图不可代表运行状态：** 图上有没有节点只说明“代码结构连线”，不能代表实际执行时满足了阈值并真正执行了摘要。



####  如何进行可观测性验证（测试 Demo 策略）

想观察摘要，可以给两者都设置 `trigger=("messages", 4)`、`keep=("messages", 2)`，传入几轮消息后调用 Agent，再比较模型实际收到的消息。这里的两个数字只用于小规模演示；实际任务应按上下文大小和信息保留需求设置。

**参数含义：**

- `trigger=("messages", 4)`：当累积消息达到 4 条时，触发摘要机制。
- `keep=("messages", 2)`：压缩后，只保留最近的 2 条最新消息，其余旧消息浓缩成一段摘要内容。

**观察方式：** 在本地小步调试时，设置较小的数值（如 4 和 2），然后查看真正发送给 LLM API 的 Request Payload（而不是仅仅看用户输入端），就能验证消息是否被成功压缩。生产环境中再根据模型的 Context Window 大小调大阈值。

以上按 [Deep Agents 0.7.10 的实现](https://github.com/langchain-ai/deepagents/blob/deepagents%3D%3D0.7.10/libs/deepagents/deepagents/middleware/summarization.py)核对。Deep Agents 还会在模型调用抛出 `ContextOverflowError` 时尝试摘要后重试；这种溢出处理和按 `trigger` 主动摘要是两条触发路径。

## 任务规划与上下文管理的协同

在长时间运行的任务中，任务规划和上下文管理需要**协同工作**。

假设 Agent 正在执行一个包含 10 个步骤的研究任务。执行到第 6 步时，对话历史已经非常长了——前面 5 步的搜索结果、文件读写操作、中间思考过程全部堆在上下文里。

此时，Deep Agents 的上下文管理机制（第 3 章）会自动介入：

1. **大结果卸载**：前面步骤产生的大量搜索结果已经被卸载到文件系统
2. **对话总结**：如果上下文仍然超过配置的触发阈值，旧消息会被总结，本次模型输入改为摘要加近期消息。`create_deep_agent()` 在已知模型窗口大小时默认使用 85%；缺少该信息时使用固定 token 阈值，也可以通过 `trigger` 自定义

### 任务清单的锚定作用

关键点在于：**默认的对话总结压缩发给模型的消息，不会删除单独保存的 `todos` 字段**。

这份清单可以帮助 Agent 在总结后继续追踪：

- 总共有哪些步骤
- 哪些已经完成，哪些还在进行
- 下一步该做什么

清单是否及时更新、下一步是否合理，仍取决于模型的实际行为。调试时<span style="color: #b8860b">要把 `todos` 和工具调用记录、最终产物放在一起看</span>，不能只看进度标记。

### 在 LangChain 中手动组合

见课程示例代码

`trigger` 表示何时开始摘要，`keep` 表示保留多少近期消息；如果没有可被总结的旧消息，就不会生成摘要。这里的 4,000 tokens 和 20 条消息是示例配置。比例阈值写作 `("fraction", 0.85)`，需要模型 profile 提供 `max_input_tokens`；模型窗口未知时使用 `("tokens", N)` 或 `("messages", N)`

## 代码实战：让 Agent 规划并执行研究任务

代码见课程

## LangChain 中间件速查

现在你已经理解了中间件的配置方式，可以用下面的表格查找具体能力：哪些默认提供，哪些通过 `skills=` 等专用参数配置，哪些通过 `middleware=[...]` 加入或替换。

**默认中间件**

| 中间件                          | 用途                               |
| ------------------------------- | ---------------------------------- |
| FilesystemMiddleware            | 7 个文件工具 + 权限控制            |
| SummarizationMiddleware         | 对话历史自动总结（触发阈值可配置） |
| PatchToolCallsMiddleware        | 补齐消息历史中缺失的工具响应       |
| 模型 / Provider 相关 Middleware | 由 Harness profile 和实际模型决定  |

**通过专用参数配置**

| 参数                               | 中间件                   | 用途                        |
| ---------------------------------- | ------------------------ | --------------------------- |
| `subagents=`（默认含通用子 Agent） | SubAgentMiddleware       | `task` 工具 + 子 Agent 委派 |
| `skills=`                          | SkillsMiddleware         | 技能包注入                  |
| `subagents=`                       | AsyncSubAgentMiddleware  | 异步子 Agent 任务管理       |
| `memory=`                          | MemoryMiddleware         | AGENTS.md 记忆注入          |
| `interrupt_on=`                    | HumanInTheLoopMiddleware | 人工审批拦截                |

**通过 `middleware` 配置**（以下为 LangChain 预构建的常用可选中间件，需应用显式加入）

| 类别       | 中间件                   | 用途                                               |
| ---------- | ------------------------ | -------------------------------------------------- |
| **规划**   | TodoListMiddleware       | 任务规划与追踪，注入 `write_todos` 和 `todos` 状态 |
| **安全**   | PIIMiddleware            | 个人信息检测和脱敏                                 |
| **弹性**   | ToolRetryMiddleware      | 工具调用失败自动重试                               |
|            | ModelRetryMiddleware     | 模型调用失败自动重试                               |
|            | ModelFallbackMiddleware  | 主模型失败自动切换备用模型                         |
| **限制**   | ToolCallLimitMiddleware  | 限制工具调用次数                                   |
|            | ModelCallLimitMiddleware | 限制模型调用次数                                   |
| **上下文** | ContextEditingMiddleware | 清理旧的工具调用结果                               |

> [!NOTE]
>
> Checkpointer属于runtime：保存和恢复状态
>
> tools的执行结果是否正确仍然需要校验
>
> TodoList需要显式启用

#### 总结

1. **为什么需要规划**：任务清单帮助 Agent 拆解复杂任务、追踪进度；是否减少遗漏和重复劳动，需要通过实际任务验证
2. **`write_todos` 工具**：启用 `TodoListMiddleware` 后，任务以<span style="color: #b8860b"> pending、in_progress、completed </span>三种状态保存在 Agent State 中；完成标记仍需结合产物检查
3. **LangChain 中间件**：Agent 能力的插件机制。`create_deep_agent()` 的本质就是<span style="color: #b8860b">把一组中间件自动组装到 Agent 上</span>
4. **由表及里**：`write_todos` 的真身是 `TodoListMiddleware`，上下文压缩的真身是 `SummarizationMiddleware`——理解底层，才能自由扩展
5. **配置方式**：分清默认中间件、通过专用参数配置的能力，以及通过 `middleware=[...]` 加入或替换的能力；v0.7 的同名实例替换不会自动合并新旧配置
6. **职责边界**：PatchToolCalls 补齐缺失的工具响应，Retry 按策略重试异常，Checkpointer 保存和恢复状态；结果是否正确，需要另外校验