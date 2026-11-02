













# Content Builder — 完整 Agent 应用展示报告

把 agentseek `content-builder` 模版改造成一个把四种能力串起来的完整应用：
**子 agent 分工 → 成果落文件 → 本地工具人工审批 → 计划驱动、证据收口。**

## 1. 在原来的模版上作出的改动

| 能力 | 模版原状 | 现在 |
|---|---|---|
| 交给子 agent | 只有 `researcher`，且仅在联网搜索开启时存在；纯文本模式下没有任何子 agent | 新增 `source-analyst`（读资料 → 写 `notes/<slug>.md`）和 `reviewer`（审稿 → 写 `reviews/<slug>.md`，末行 `VERDICT: PASS/REVISE`）。纯文本模式即可用；`researcher` 用 `requires: search` 门控（`subagents.yaml`、`agent.py:_load_subagents`） |
| 成果留在文件里 | 报告落文件，但中间产物只存在于对话 | 子 agent 把结果写文件、只回一句路径；主 agent 必须读文件而不是信回复。产物链：`notes/` → `blogs/<slug>/post.md` → `reviews/` → `reports/publications.jsonl` |
| 本地工具审批 | 审批只存在于 SDK 实验脚本，浏览器没有；默认图不注册 `publish_report` | 服务端图默认开启审批（`CONTENT_REQUIRE_APPROVAL=true`）：`publish_report`（approve/reject）与 `save_preference`（approve/edit/reject）执行前暂停。前端新增 `ApprovalCard`，用 `stream.interrupt` + `command.resume` 回传决定 |
| 按计划完成 | `write_todos` 只是意图记录 | 新增 `check_deliverables(slug)`，列出各产物是否存在与评审结论；`publish_report` 增加硬门槛：没有 `VERDICT: PASS` 的评审文件直接报错。系统提示规定“先 check_deliverables 再把 todo 标 completed” |

文件清单：`subagents.yaml`、`src/.../agent.py`、`src/.../lesson_tools.py`（`check_deliverables`、发布门槛）、
`src/.../demo.py`（新，可复现 demo）、`frontend/src/ApprovalCard.tsx`（新）、`frontend/src/App.tsx`、`.env.example`、`tests/test_course.py`。

## 2. 可复现 demo

```bash
cd content_builder_deepagent
uv sync --frozen --dev
uv run --frozen python -m content_builder_deepagent.demo          # 脚本模型，无需任何密钥
uv run --frozen python -m content_builder_deepagent.demo --live --decision approve   # 真实模型（读 .env）
uv run --frozen pytest tests -q                                   # 25 个后端测试
cd frontend && npm test                                           # 12 个前端测试
```

脚本模式只替换聊天模型；graph、子 agent、工具、文件、审批中断都是真实的。输出：`demo_output/trace.json` 与 `demo_output/workspace/`。
浏览器版：`agentseek dev`，提示词见 §5。

## 3. 一次运行的完整轨迹（脚本模式，`demo_output/trace.json`）

| # | 角色 | 动作 |
|---|---|---|
| 0 | main | `write_todos` 4 步，第 1 步 in_progress |
| 1 | main | `task(source-analyst)` 委派 |
| 2–6 | source-analyst | `read_source` ×4 → `write_file /notes/agent-files.md` |
| 7 | source-analyst | 回复一句路径摘要 |
| 8 | main | `read_file /notes/agent-files.md`（读文件，不信回复） |
| 10–11 | main | `save_report` 初稿 → `read_report` 回读 |
| 12–16 | reviewer | 读稿、读 notes、写评审：**VERDICT: REVISE**（缺“合成资料”声明） |
| 17–19 | main | 读评审 → 修订稿 `save_report` → 回读 |
| 20–23 | reviewer | 再审：**VERDICT: PASS** |
| 24 | main | `check_deliverables` |
| 25 | main | `write_todos`：前三步 completed，发布步 in_progress |
| 27 | main | `publish_report` → **暂停**；人工 **reject** → 无记录 |
| 29 | main | 再次 `publish_report` → **暂停**；人工 **approve** → 写入 1 条记录 |
| 30 | main | `write_todos` 全部 completed |

脚本断言（全部 PASS）：

- notes 由子 agent 写出 ✔　草稿已保存 ✔　最终评审 PASS ✔　修订后含 synthetic 声明 ✔
- 暂停时尚无任何发布记录 ✔　reject → 记录数 0 ✔　approve → 恰好 1 条 ✔　最终 todos 全部 completed ✔

## 4. 真实模型验证

同一 demo 用 `.env` 中的 `deepseek-flash`（`--live --decision approve`）跑通：
todo 逐步推进到 5 步全部 completed；在 `publish_report` 处暂停；approve 后 `reports/publications.jsonl` 恰有 1 条；
评审文件以 `VERDICT: PASS` 结尾，草稿 7.6 KB 且带合成资料声明。产物在 `demo_output_live/`。
注意：该文件是一次非确定性运行的实录；实际子 agent 调用以其 `trace.json` 与文件为准，需自行复核内容（见 COURSE.md 的提醒）。
live 模式只记录 todo 进度与中断，不记录子 agent 内部调用。

## 5. 浏览器里复现

`agentseek dev` 后打开前端，发送：

```text
Slug agent-files. 按 content-project 流程：先 write_todos；把 planning.md、files.md、memory.md、approval.md
交给 source-analyst 写 notes；据此写报告并读回；交给 reviewer 审稿（REVISE 就修改再审）；
运行 check_deliverables；最后调用 publish_report 一次。
```

预期：Content plan 面板推进；出现 `task` 卡片（source-analyst / reviewer）；黄色 **Approval required** 卡片显示
`publish_report {"slug": "agent-files"}` 与 Approve/Reject；点 Reject 后 `reports/publications.jsonl` 不变，点 Approve 后新增一行。
（前端审批卡由组件测试覆盖；本次未做浏览器端人工点击验收。）

## 6. 测试覆盖

后端 25 项（原 20 + 新增 5）：发布门槛（无评审 / REVISE 均拒绝）、`save_preference` 审批且 reject 不写文件、
纯文本模式只加载本地子 agent（有搜索才加载 researcher）、整条 demo 流水线端到端。前端 12 项（含 `ApprovalCard`）。

## 7. 已知限制

- 审批 UI 只做 approve/reject；`save_preference` 的 edit 决定后端支持，前端未提供编辑界面。
- 服务端图的审批依赖 agentseek-api 注入 checkpointer（`served=True`）；`agentseek dev` 路径未在本次会话中实跑，仅验证了图构建与 SDK 路径。
- `check_deliverables` 与发布门槛只检查文件是否存在及 `VERDICT` 行，不判断评审质量。
- `demo_output*/` 是示例产物，`reports/` 目录仍被 .gitignore 忽略。
