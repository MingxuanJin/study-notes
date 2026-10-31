# 打卡

## 一、实验问题

Agent 写销售报告，明细有了却**漏了 `total`**。能否用独立检查链路：发现缺项 → 把差距交回 Agent → 检查修订稿 → 决定交付与否？

## 二、做了什么

1. **定标准**：输入茶 30、咖啡 50，总额应为 80；验收三项：JSON 对象、`items` 一致、`total` 正确。
2. **先不用模型验证证据工具**：`assess_report()` 返回 `{ok, criteria[name, passed, gap]}`，评分工具和断言共用它；缺总额→失败，完整报告→通过，非法 JSON→不通过。
3. **脚本化两个模型**：工作模型依次吐出“缺 total 稿 → 完整稿”；评分模型先调 `check_report`，拿到真实工具结果后再返回 `GraderResponse`。
4. **挂中间件 + 护栏**：`RubricMiddleware(tools=[check_report])`；`StopRepeatedEvidence` 限制每候选最多 6 次模型调用，取证成功后收窄工具，防止反复取证。
5. **统一交付门 `accepted()`**：最后一轮必须 `satisfied`，且最后取证的报告 = 最新候选、工具 `ok=true`、业务检查再过一遍。
6. **跑三个场景**。

| 场景 | 设置        | 现象                                         | 交付 |
| ---- | ----------- | -------------------------------------------- | ---- |
| 5.1  | 预算 2      | 第 0 轮缺总额 → 反馈 → 第 1 轮补齐并重新检查 | ✅    |
| 5.2  | 预算 1      | 缺总额，预算用完 → `max_iterations_reached`  | ❌    |
| 5.3  | 不传 Rubric | 评分 0 次、取证 0 次，但有报告               | ❌    |

## 三、验证了什么

- 证据工具不靠模型也能区分好坏报告。
- `needs_revision` 让工作模型**真的再生成**一份（两份候选来自两次实际调用），反馈里含 `total` 和 `80`。
- 修订后**重新取证**：`records` 两条，第 0 条 `ok=False`，第 1 条对应新报告 `ok=True`。
- 预算耗尽 ≠ 通过：报告还在，但结论是 `max_iterations_reached`。
- 挂了中间件 ≠ 启动评分：没 Rubric 就不评分（Fail-Closed）。
- “候选生成完”“检查工具跑完（`status=success`）”“报告通过（`ok=true`）”是三个不同事件。

# 笔记

以下内容来自对于 https://datawhalechina.github.io/deepagents-in-action/chapters/ch13-grading-rubrics/ 的学习

## 核心问题：为什么需要运行时验收

案例：让 Agent 实现 `find_duplicates(values)`。

- 用 `set` 的版本能通过 `[1,2,2,3,1] -> [2,1]`
- 但遇到 `[[1],[1]]` 就 `TypeError: unhashable type: 'list'`（`set` 要求元素**可哈希**，list 不行）
- Agent 已经输出完整函数、也不再调工具，但**其实不能交付**

| 现象           | 实际说明         | 风险                          |
| -------------- | ---------------- | ----------------------------- |
| 模型不再调工具 | 生成循环自然停止 | 不能证明要求全满足            |
| 让模型自检     | 又一次语言判断   | 生成者和评审者共享盲点        |
| 笼统"再试一次" | 另一份候选       | 不知道哪项失败、证据是什么    |
| 拿到最终消息   | 只是有个候选答案 | 达到上限/评审异常时消息也还在 |

**关键区分**：「生成结束」是运行事件，「通过验收」才是业务结论。

### 可靠验收链路的 5 种能力

1. 用明确标准定义"完成"
2. **独立角色**评审（生成者不能自己放行）
3. 可确定的事实用**工具**取证（测试、Schema、文件）
4. 把未通过的 Criterion + Gap 送回生成环节
5. 设迭代预算，**Fail-Closed**（没明确通过就拒绝）

## 运行流程（状态机）

```
用户任务 + rubric
   ↓
Working Model 生成 → 自然停止（只是候选，没结论）
   ↓
Grader Model 读 Rubric + 对话 → 调 Evidence Tool 取证 → 给 Verdict
   ├─ needs_revision → 把 Gap 注入对话 → 回到 Working Model（有预算时）
   ├─ satisfied      → ✅ 放行
   └─ max_iterations_reached / failed / grader_error → 终止，❌ 不算通过
```

### 四个角色

| 角色          | 负责                     | 不负责             |
| ------------- | ------------------------ | ------------------ |
| Working Model | 写/修订代码              | 不决定自己是否通过 |
| Rubric        | 声明验收标准             | 不运行测试         |
| Grader Model  | 检查标准、调工具、给结论 | 不能替代确定性测试 |
| Evidence Tool | 跑测试、返回结构化事实   | 不直接放行         |

⚠️ **易错点**：`RubricMiddleware(tools=[...])` 的工具**只给评分模型用**，不会自动给工作模型。工作模型的工具要走 `create_deep_agent(tools=[...])`。

## 专业名词表

| 名词                             | 解释                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| **Rubric（评分量规）**           | 把"完成"写成一组可检查的标准清单                             |
| **LLM-as-a-Judge**               | 让一个模型按明确标准给另一个输出打分。离线用于批量评测，运行时可驱动当前任务修订 |
| **Working Model / Grader Model** | 干活的模型 / 评审的模型，可同可异；评审模型必须支持结构化输出 + 工具调用 |
| **Working Agent / Grader Agent** | 前者=工作模型+工具+Middleware；后者=`RubricMiddleware` 管理的评审子智能体 |
| **Middleware（中间件）**         | 插在 Agent 运行流程中的拦截层；这里用来在"自然停止"之后插入评分 |
| **Evidence Tool（证据工具）**    | 返回客观事实的工具（如 `run_test_suite`），供评分模型引用    |
| **Verdict（评审结论）**          | 一轮评分的结果：`satisfied` / `needs_revision` / `failed` 等 |
| **Criterion / Gap**              | 单条标准项 / 该项未通过时的差距说明（要能直接指导修改）      |
| **Fail-Closed（失败关闭）**      | 未知/异常状态默认**拒绝**，只有明确通过才放行（反义：Fail-Open） |
| **Structured Output**            | 模型按给定 Schema 返回结构化数据，而非自由文本               |
| **Tool Calling**                 | 模型输出"调用某工具"的结构化请求，由框架执行                 |
| **Checkpointer / `thread_id`**   | 保存运行状态的组件 / 会话标识；同一 thread 延续状态          |
| **`grading_run_id`**             | **一次评分尝试**的 ID，与 `thread_id` 不是一回事             |
| **Callback（`on_evaluation`）**  | 每轮评分后被调用的函数，用于观察，**不是控制钩子**           |
| **Sandbox（沙箱）**              | 隔离的代码执行环境，限制文件/网络/CPU/内存/时间              |
| **Offline Evaluation**           | 离线批量评测，衡量长期质量；不能在当前请求里反馈修订         |
| **Idempotent（幂等）**           | 重复执行结果相同；这里指 `(grading_run_id, iteration)` 去重  |
| **Docstring**                    | 函数文档字符串；`@tool` 靠它生成工具说明，影响评分模型是否调用 |

## 五种结果状态

| `result`                 | 来源                       | 后续               | 是否接收 |
| ------------------------ | -------------------------- | ------------------ | -------- |
| `satisfied`              | 全部标准通过且未被降级     | 结束               | ✅        |
| `needs_revision`         | 评分模型发现问题           | 有预算则反馈并重试 | ❌        |
| `failed`                 | 认为 Rubric 无法评估       | 结束               | ❌        |
| `grader_error`           | 评分调用链异常             | 结束               | ❌        |
| `max_iterations_reached` | 最后一轮仍需修订且预算用完 | 结束               | ❌        |

## 代码要点

### 1. 写好 Rubric（三个要求）

- **可判定**：每条只描述一个可检查行为（"代码要健壮"就不行）
- **可取证**：明确写"必须调用 `run_test_suite`，且结果 `ok=true`"，防止评分模型只读代码主观放行
- **可修订**：失败证据要包含输入、实际结果、异常，让工作模型知道改哪里

### 2. Evidence Tool 返回字典而非日志

```python
return {"ok": not failures, "failures": failures}
```

`ok` 给明确结论，`failures` 保留具体失败用例。测试同时检查**返回值**和**是否修改了输入**（`deepcopy` 对比）。

### 3. 装配三步

```python
rubric_middleware = RubricMiddleware(model=grader_model, tools=[run_test_suite],
                                     max_iterations=3, on_evaluation=record_evaluation)
agent = create_deep_agent(model=working_model, middleware=[rubric_middleware],
                          checkpointer=InMemorySaver())
agent.invoke({"messages": [...], "rubric": rubric}, config=...)   # rubric 必须非空
```

- `rubric` 是**调用状态**，不是构造参数 → 同一个 Agent 可处理不同任务/标准
- `max_iterations=3` = **最多评分 3 轮**，不是"至少修订 3 次"

### 4. 验收门（最重要的一段）

```python
accepted = final_evaluation is not None and final_evaluation["result"] == "satisfied"
```

❌ 错误写法：`accepted = bool(result["messages"])`（有消息 ≠ 通过）

### 5. 修订思路

`set` 判重 → 改为 `any(item == target for item in items)` 的相等性比较，不再要求可哈希（代价是 O(n²)，教学够用）。

## Q & A

### Q1：为什么不能让模型自检？

自检时「生成」和「评审」往往**共享同一套理解和盲点**——它没想到不可哈希，自检时同样想不到。所以需要：独立角色 + **确定性工具**（测试真的会抛 `TypeError`，不受模型"想没想到"影响）。

### Q2：既然有测试工具，为什么还要 Grader Model？

- 工具只给**事实**（`ok=false` + 失败信息），不会判断"这是否满足全部标准"
- Grader 负责：对照 Rubric 逐条判断 → 综合证据 → 写出可执行的 Gap
- 反过来，Grader 也**不能替代**测试：能用测试确定的，就别靠模型判断

### Q3：`unverified=True` 是什么？

Grader 说 `satisfied`，但没给够逐项评审（没覆盖全部标准），中间件重试后仍不完整 → **降级**为 `needs_revision` 并标 `unverified=True`。意思是"**没完整验证**"，不是"确认有缺陷"。

### Q4：`grading_run_id` vs `thread_id`

- `thread_id`：Checkpointer 的**会话**（消息、状态延续）
- `grading_run_id`：一次**评分尝试**（一个 Rubric 从开始到终止的所有迭代）
- 同一 thread 换新 Rubric / 终止后再调用 → 新 `grading_run_id`

### Q5：Callback 为什么不能做权限检查/放行？

`on_evaluation` 抛的普通异常会被**记录并吞掉**，循环照常继续。所以它只适合日志、指标、审计。**放行决定必须在调用返回后的应用验收门里做。**

### Q6：为什么并发时不能用全局 `evaluations` 列表？

多个请求同时写会串数据。应按 `grading_run_id` 分组存，再关联自己的请求 ID / `thread_id`，并对 `(grading_run_id, iteration)` 做幂等约束防重复写入。

### Q7：`exec` 为什么危险？

`exec(code, namespace)` 在**当前进程**里执行候选代码，能访问进程所有资源。`safe_builtins` 只是减少误触，**不是安全隔离**。生产必须放沙箱（限制文件、网络、凭据、CPU、内存、时间、进程创建、输出大小）。

### Q8：为什么不一直加大 `max_iterations`？

上限只管成本和延迟，**修不了**矛盾标准、错误测试、不可执行的反馈。只有每轮都有新证据、且工作模型真的在改，加上限才有意义。

### Q9：证据会不会"过期"？

会。若应用缓存了上一轮测试结果，Grader 可能拿旧证据放行新代码。做法：每次修订后**重跑**工具，或在输入/记录里加**候选版本/内容摘要**，证明证据对应当前版本。

### Q10：Offline Evaluation 和 Runtime Acceptance 区别？

|              | 运行时验收       | 离线评测                   |
| ------------ | ---------------- | -------------------------- |
| 作用范围     | 当前这一次请求   | 大量样本的长期统计         |
| 能否反馈修订 | ✅                | ❌                          |
| 回答的问题   | 这个答案能放行吗 | 版本/Prompt/模型整体好不好 |

两者要结合：看首轮通过率、最终通过率、平均评分轮数、失败状态占比、工具错误率、人工推翻 `satisfied` 的比例。

## 观察与排错速查

| 入口                                             | 时机       | 用途                          |
| ------------------------------------------------ | ---------- | ----------------------------- |
| `on_evaluation`                                  | 每轮结论后 | 日志、指标、评测集、审计      |
| `stream.custom`（`rubric_evaluation_start/end`） | 评分前后   | 实时 UI 进度（不是 token 流） |
| 应用验收门                                       | 调用返回后 | 只有 `satisfied` 才放行       |

| 现象                      | 先查                                          |
| ------------------------- | --------------------------------------------- |
| 评分从未运行              | 是否传了**非空** `rubric`                     |
| 测试工具没被调用          | Rubric 是否写明工具名与通过条件；看 Trace     |
| 报结构化输出错误          | 模型是否支持 Structured Output / Tool Calling |
| 一直 `needs_revision`     | 标准是否太宽泛、`gap` 是否可执行              |
| 有代码但 `accepted=False` | 最后一轮是不是 `satisfied`，**别绕过验收门**  |

排错顺序：**先单独验证 Evidence Tool（不用模型）→ 再调模型**，把执行问题和模型选择问题分开。

