# 打卡内容

## 1.明确机制

`content_builder_deepagent/src/content_builder_deepagent/agent.py` 的 `build_graph()`：

```python
backend = FilesystemBackend(root_dir=EXAMPLE_DIR, virtual_mode=True)  # EXAMPLE_DIR = content_builder_deepagent/
...
skills=["/skills/"],
backend=backend,
```

对应到`deepagents/middleware/skills.py`：

| 阶段             | 发生了什么                                                   | 观察证据                                |
| :--------------- | :----------------------------------------------------------- | :-------------------------------------- |
| 启动 / 新 thread | `before_agent` 扫描 `/skills/` 的**直接子目录**，只解析 `SKILL.md` 的 frontmatter | prompt 里出现 name + description + path |
| 每轮请求         | `modify_request` 把上面的**元数据**拼进 system prompt，正文不进去 | prompt 里搜不到 `### 1. Hook` 这类正文  |
| 模型判断匹配后   | 模型自己调 `read_file("/skills/<name>/SKILL.md", limit=1000)` | 工具卡片里的 `arguments`                |
| 需要时           | 再读 skill 目录下的附件 / 跑脚本                             |                                         |

1. **"匹配"没有任何检索代码**，完全是模型读 `description` 自己判断。description 写得含糊 → 不触发；写得太宽 → 误触发。
2. **`/skills/` 只扫一层**，`skills/email/newsletter/SKILL.md` 不会被发现，必须 `skills/newsletter/SKILL.md`。
3. **`name` 必须等于目录名**（不一致只 warn 但会加载）；**缺 `description` 或 frontmatter 非法则整个 skill 静默跳过**，prompt 里看不到，只有日志告警。

## 2.Live观察匹配与渐进式加载

开了三个新的thread，用了以下promt进行测试

| 输入                                           | 期望加载       | 不应加载       |
| :--------------------------------------------- | :------------- | :------------- |
| "写一篇关于 Agent Skills 渐进式加载的博客文章" | `blog-post`    | `social-media` |
| "把这篇内容改写成一条 LinkedIn 帖子"           | `social-media` | `blog-post`    |
| "你好，帮我看下现在有哪些工具"                 | 都不读         | 两个都不读     |

### 第一次输入

在第一次输入后，read_file的arguments：

```
{
  "file_path": "/skills/blog-post/SKILL.md",
  "limit": 1000
}
```

output：

文章已写好并保存到 `blogs/agent-skills-progressive-loading/post.md`，我也把文件读回来核对过，标题、结构、代码块和 Sources 都完整。

**文章主题：《Agent Skills 的渐进式加载：让上下文按需展开》**（元描述已按 SEO 要求放在文首）

结构：

1. **开头**——装了 20 个技能却变慢、选错技能，问题出在上下文预算上，不在模型本身。
2. **三层结构**——用表格对比「常驻元数据 → SKILL.md 正文 → 支持文件」各自何时进入上下文、代价多大，并用本机实际可查的技能索引和 `SKILL.md` 头部做对照。
3. **为什么划算**——把四处收益展开讲清楚：选择和执行分成两个阶段交替执行（上下文窗口是预算、注意力更稀缺）、描述质量决定路由质量、按需读取让正文可以写长、长正文中间段天然被忽略。
4. **最小实现**——用伪代码演示三层加载器（含 `limit=1000` 一次读全、绝对路径引用附录），外加三个常见坑：读不全、相对路径、正文里塞元数据。
5. **可验证流程**——计划不等于证据、写完就回读、偏好放 `memory/preferences.md`。
6. **落地清单**——盘点、重写描述、拆分正文（以 200 行为切分参考）、加日志、定期修剪，并给出四个衡量指标。
7. **结语 + CTA**——3 条要点，加一个「现在就切开你最大的 SKILL.md」的具体动作。

关于来源：

- 分层机制与 `SKILL.md` 结构的描述取自本机技能库实际文件 `/skills/blog-post/SKILL.md`；
- 文件产物、计划校验、跨会话记忆的论述取自 **合成课程笔记（synthetic course notes）** `/sources/planning.md`、`/sources/files.md`、`/sources/memory.md`，文中已标注「合成课程笔记」并逐条注明出处；
- 代码是示意性伪代码，文中已说明不代表任何具体 SDK 的 API；全文没有编造网络调研数据。

需要的话，我可以再改一版：比如缩短成 800 字左右的短文、加强技术深度（补上并行加载与缓存失效的讨论），或补一段英文版/社交平台改写稿。

### 第二次输入

tool: read_file的arguments：

```
{
  "file_path": "/skills/social-media/SKILL.md",
  "limit": 1000
}
```

输出：

帖子已写好，保存到 `linkedin/agent-skills-progressive-loading/post.md`，我也回读核对过，内容完整。

**帖子正文（可以直接复制发布）：**

> 我的 agent 装了 20 个 Skills，反而变笨了。
>
> 不是模型的问题，是上下文预算的问题。
>
> 把所有技能的完整说明一次性塞进系统提示，等于同时做错三件事：烧掉窗口、稀释注意力、让每个技能都失去“什么时候该用我”的区分度。
>
> 我现在的做法是渐进式加载，三层里只有第一层常驻：
>
> - 元数据（name + description）：始终在上下文里，每个技能 1-3 行，只负责路由
> - `SKILL.md` 正文：命中相关任务时才读
> - 支持文件：脚本、模板、资料，执行到那一步才读
>
> 一句话：技能常驻的是名片，不是简历。
>
> 三个我踩过的坑：默认分页只给前 100 行，长技能被截成半份规范；支持文件要用绝对路径；把“何时使用我”写进正文，等于让路由直接失效。
>
> 还有一个反直觉的收益：省 token 不是重点。重点是每次进入上下文的内容，都短到值得被完整读完。
>
> 验证方式也得跟着变——待办清单记录的是意图，不是产物的存在；写完要回读文件。
>
> 你现在的技能库，常驻部分占了多少 token？
>
> \#AIAgents #ContextEngineering #AgentSkills #LLM #DeveloperTools

**改写时的取舍：**

| 长文           | 帖子                               | 原因                                               |
| :------------- | :--------------------------------- | :------------------------------------------------- |
| 三段式开头铺垫 | 只留“装了 20 个 Skills 反而变笨了” | LinkedIn 前 2 行是折叠区，钩子必须第一句出现       |
| 三层结构表格   | 三条短清单                         | 信息流里表格难读，清单更容易扫视                   |
| Python 伪代码  | 删除                               | 代码在社交平台上吸引力低，改为“三个坑”更具体       |
| 五步落地清单   | 删除                               | 与后面的联系入口功能重复，帖子只保留一个行动点     |
| 结尾链接       | 改成提问                           | 提问比外链更利于评论互动；博客链接可以放第一条评论 |

正文约 700 字（含标点），在 1,300 字符限制内，5 个标签都是具体主题词。

内容全部来自你自己的博客长文，分层描述也以 `/skills/blog-post/SKILL.md` 里的实际可查内容为依据，没有编造数据。

需要的话我还可以再出：

- **英文版**（面向国际受众，钩子改成 “20 skills, and my agent got dumber.”）
- **配图**——需要你明确说一声，我再生成 4:5 的单焦点封面
- **Twitter/X 线程版**（把三层结构拆成 1/–5/ 的节奏）

## 3.在主项目新增了一种任务流程

改动了**`skills/` 目录**，新建了`newsletter/` 文件夹，name、allowed tools、description如下：

```b
name: newsletter
description: Drafts email newsletters, weekly digests and subscriber updates, with subject line, preview text and CTA. Use when the user asks to write a newsletter, email update, weekly digest, subscriber email, or repurpose a blog post into an email.
allowed-tools: read_source write_file read_file
```

### live验证

使用了以下promt：

帮我写一封这周的产品更新邮件，发给订阅用户

工具read_file的arguments如下，使用了新的newsletter skill

```
{
  "file_path": "/skills/newsletter/SKILL.md",
  "limit": 1000
}
```

# 笔记

以下内容来自于对于https://github.com/datawhalechina/deepagents-in-action/blob/main/content/ch07-skills.md 课程的学习，内容所有权归作者本人

### 第一项：Skills 的基本概念与目录规范

#### 1. 为什么需要 Skills？

- **工具（Tools）的局限**：工具是原子操作（如调一次 API、读一个文件、搜索一次）。   
- **Skills 的定位**：针对复杂任务，Agent 需要的是“**多步骤工作流程 + 领域专业知识 + 模板资源**”的组合能力包（类比 Node.js 的 npm 包）。   
- **行业通用规范**：遵循开放的 **Agent Skills 规范**，编写一套可在 Claude Code、OpenAI Codex、Cursor、Deep Agents 等 30+ 主流开发工具中直接复用。   

#### 2. 标准目录结构解析

一个标准的 Skill 是一个独立目录，内部结构如下：   

Plaintext

```
skills/
└── langgraph-docs/
    ├── SKILL.md              # 必须：核心元数据 + 执行指令
    ├── scripts/              # 可选：可执行脚本（Python/Bash/JS等）
    ├── references/           # 可选：参考文档（API 规约、风格指南）
    └── assets/               # 可选：静态资产（模板、JSON Schema 等）
```

- **`SKILL.md`**：核心入口文件。元数据在 Agent 启动时扫描注入提示词，正文则由 Agent 判断需要时才调工具读取。   
- **`scripts/`**：存放复杂处理脚本，需要时让 Agent 执行（而非让大模型自己重复手写代码）。   
- **`references/`**：深层长文本参考（如 API 手册），避免把大段规范塞在主指令里。   
- **`assets/`**：报告模板、配置骨架等按需提取的静态文件。   

#### 3. `SKILL.md` 的两部分结构

`SKILL.md` 由 **YAML Frontmatter** 和 **Markdown Body** 构成：   

1. **YAML Frontmatter（元数据）**：   
   - `name`（必填）：小写字母、数字和连字符，必须与目录名完全一致。   
   - `description`（必填）：**最重要的字段**。Agent 决定要不要用这个 Skill **完全只看这行描述**，不会预先读取正文。   
   - `license` / `compatibility` / `allowed-tools` 等可选配置。   
2. **Markdown Body（正文）**：   
   - 详细的流程指引、分步步骤（Step 1/2/3）、判断逻辑和示例，是 Agent 激活后执行的行动指南。   

### 第二项：Progressive Disclosure（渐进式加载机制）

Progressive Disclosure 是 Skills 设计中最核心的架构思想，旨在解决将所有指令一股脑塞入上下文导致的 **Token 浪费** 和 **提示词干扰（Attention 稀释）** 问题。   

#### 1. 三级加载体系（Three-tier Loading）

Agent 不会一次性读取所有内容，而是分为三个层级按需展开：   

| **层级**                  | **加载内容**                             | **加载时机**         | **驱动者 / 处理机制**                          |
| ------------------------- | ---------------------------------------- | -------------------- | ---------------------------------------------- |
| **Level 1: Metadata**     | `name` + `description` + 文件路径        | **Agent 启动时**     | 由 `SkillsMiddleware` 扫描提取并注入系统提示词 |
| **Level 2: Instructions** | `SKILL.md` 正文指令                      | **匹配到任务后**     | LLM 做出决策，主动调用 `read_file` 工具读取    |
| **Level 3: Resources**    | `scripts/`、`references/` 等目录下的文件 | **正文指令引用到时** | LLM 自行决定是否读取，无中间件强制介入         |

#### 2. 匹配与执行全流程示例

整个生命周期的运作逻辑如下：   

1. **启动阶段**：中间件扫描已配置的 Skills 目录，将每个 Skill 的 Frontmatter（名字和描述）提取出来，写入系统提示词列表。此时所有正文完全不占用上下文。   
2. **决策阶段**：
   - 用户发起请求，例如：“*帮我查一下 LangGraph 的 interrupt 机制*”。   
   - Agent 检索系统提示词中的 Skills 描述，发现 `langgraph-docs` 的 description 命中需求。   
3. **加载阶段**：Agent 调用文件读取工具（如 `read_file("/skills/langgraph-docs/SKILL.md")`），正文作为工具调用的返回结果进入上下文。   
4. **深入资源阶段**：正文指导 Agent 去查阅 `references/api-patterns.md` 或执行某脚本，Agent 根据实际需要再次决定是否调用工具读取详细资源。   

#### 3. 为什么这样设计？（设计优势）

- **极致节省 Token**：如果系统挂载了 20 个 Skill，启动阶段只加载 20 句描述（仅需数百 Token），而非 20 份完整流程文档（可能高达数万 Token）。   
- **减少幻觉与干扰**：无关的 Skill 细节不会进入上下文，避免无关规则干扰模型的逻辑判断。   
- **极高的扩展性（Scalability）**：Skill 数量由 5 个增加到 50 个时，启动阶段仅线性增加几十行描述，不会触碰上下文窗口的上限。   



### 第三项：存储后端（Backends）与优先级覆盖

Deep Agents 提供了三种存储后端，用以在不同运行环境下存放和读取 Skills 目录与文件：   

#### 1. 三种后端对比与选型

- **FilesystemBackend（本地磁盘存储）**：   
  - **机制**：直接挂载和扫描操作系统的本地磁盘目录。   
  - **适用场景**：本地开发、CLI 脚本及测试环境。   
  - **核心配置**：`skills` 传绝对虚拟路径（如 `["/skills/"]`），本地路径建议开启 `virtual_mode=True` 实施路径沙箱隔离。   
- **StateBackend（Agent State 状态存储）**：   
  - **机制**：完全不依赖物理磁盘，文件数据以字典结构保存在 LangGraph 的 state 中。   
  - **使用注意**：内容必须用 `create_file_data()` 包装格式化，并在每次调用 `agent.invoke()` 时通过 `files` 参数传入。   
  - **适用场景**：Serverless 无盘运行环境或需动态按需注入 Skill 的场景。   
- **StoreBackend（持久化存储）**：   
  - **机制**：基于 LangGraph Store 存储，通过 `store.put()` 提前写入。   
  - **适用场景**：生产环境、多用户/多线程共享公共或私有技能库。   
  - **特性**：只需写入一次，即可支持跨线程持续访问，支持通过 `namespace` 函数划分命名空间。   

#### 2. 多源加载与优先级（Last Wins 规则）

`skills` 参数接收路径列表，当多个目录出现**同名 Skill** 时，**列表中后面的路径覆盖前面的路径**（Last wins）：   

- **经典分层实践**：
  - `/skills/shared/`（组织/团队共享的通用规范）   
  - `/skills/project/`（当前项目特定定制，排在后面优先级更高）   
- **运行时动态加载**：
  - `skills` 是标准 Python 列表，可依据运行时上下文动态组装，例如根据**用户角色**（前端/运维）、**租户配置**、**意图预测**或**环境变量**动态注入特定列表，避免无用 Skill 占用和误召回。   

### 第四项：子 Agent 继承与权限控制

#### 1. 子 Agent（Subagent）的继承与隔离

在多 Agent 协作架构中，主 Agent 与子 Agent 的 Skill 权限关系如下：   

- **通用子 Agent（General-Purpose Subagent）**：自动继承主 Agent 的全部 Skills，无需重复配置。   
- **自定义子 Agent**：**不继承**主 Agent 的任何 Skills，必须在声明字典里用 `skills` 字段单独指定。   
- **完全状态隔离**：不同 Agent 间的 Skill 执行空间与文件修改严格隔离，杜绝并发执行下的竞态条件。   

#### 2. 权限控制模式（FilesystemPermission）

企业部署常结合 `CompositeBackend`（路由挂载）与 `FilesystemPermission` 实现精细权限管理：   

- **只读拦截（`mode="deny"`）**：
  - 设置针对 `/skills/**` 的 `write` 操作为 `deny`。   
  - Agent 仅能发现并执行 `read_file`，对 `write_file`、`edit_file`、`delete` 均直接拦截返回错误，防止公用企业库被篡改。   
- **人工审批（`mode="interrupt"`）**：
  - 引入人在回路（Human-in-the-loop）机制。   
  - 当 Agent 尝试改写 `/skills/**` 文件时触发流程暂停（interrupt），交由人类审查确认或驳回，必须配置 `checkpointer`（如 `MemorySaver`）保存挂起状态。   
- **共享 + 个人双层结构**：
  - 将 `/skills/shared/` 映射为只读（`mode="deny"`），`/skills/personal/` 映射为当前用户专属的可写命名空间。   
  - 用户既能使用公司统一技能，又能在个人目录下覆盖同名技能进行个性化修改。   

### 第五项：Skills 代码执行（沙箱与解释器）

Skills 支持嵌入真实的可执行逻辑，避免每次让 LLM 现写代码导致的幻觉与开销：   

#### 1. 沙箱脚本（Sandbox Scripts）

- **结构**：在 Skill 目录下设立 `scripts/`（如 `search.py`），并在 `SKILL.md` 正文中指示 Agent 调用命令行执行它。   
- **执行环境**：依靠隔离后端（如 DaytonaSandbox 容器）运行，防止外部代码污染宿主机。   
- **适用场景**：重型数据处理、Shell 指令、第三方工具拉取与数据格式化转换。   

#### 2. 解释器技能（Interpreter Skills）

- **机制**：
  1. 在 `SKILL.md` 的 YAML 元数据中声明 `metadata.entrypoint: scripts/index.ts`。   
  2. 结合 `CodeInterpreterMiddleware`（如基于 QuickJS 内存运行时）。   
  3. Agent 可直接在代码块执行中编写 `await import("@/skills/<name>")` 引入经测试的工具函数。   
- **优势**：
  - **高确定性**：关键逻辑交由已通过单测的真实代码执行，告别模型临时生成的逻辑误差。   
  - **省 Token**：单行 `import` 即可完成调用，无需模型重新打印多行实现代码。   

### 第六项：Skills、Memory 与 Tools 对比

三者服务于不同的注入目的，构成了互补的能力网络：   

| **评估维度**     | **Skills**                               | **Memory（AGENTS.md）**                | **Tools**                                |
| ---------------- | ---------------------------------------- | -------------------------------------- | ---------------------------------------- |
| **功能定位**     | 按需调度的多步骤工作流与领域规范         | 全局持久化的用户偏好与环境约束         | 执行特定外部操作的原子接口               |
| **上下文加载**   | **按需加载**（通过三级机制逐步按需读取） | **启动加载**（始终固定在初始提示词中） | **每轮全局可用**（定义在 Tool Schema）   |
| **存储组织**     | 包含 `SKILL.md` 的标准化能力目录         | 纯 Markdown 文件（`AGENTS.md`）        | 绑定的代码函数                           |
| **覆盖机制**     | 列表后来者居上（Last wins）              | 用户级与项目级合并叠加                 | Agent 初始创建时统一绑定                 |
| **适用分流规则** | **“特定复杂任务才需要的专业指南”**       | **“所有交互都需要知道的全局约束”**     | **“必须调用的原子行为”**（如检索、读写） |

### 第七项：高效编写 Skills 的最佳实践

1. **Description 务必精准具体**：   
   - Agent 挑选 Skill 仅依赖该字段。明确说明**何时触发**（触发意图）以及**负责解决什么**，杜绝“处理各种问题”等空泛描述，防止误触发或漏匹配。   
2. **正文保持精简（控制在 500 行 / 5000 tokens 以内）**：   
   - 主文件聚焦核心工作流骨干，长篇的协议定义、API 规约统统放进 `references/`，由指令引导模型深入查阅。   
3. **面向 Agent（而非面向人类）排版逻辑**：   
   - 采用分步骤序号（1-2-3）、条件分支（if-else 选择依据）、规范的入参与出参示例，避免冗长散文段落。   
4. **节制数量，定期审查收敛**：   
   - 候选集数量过多会导致描述冲突与模型判断犹豫，应定期清理过期技能，合并职责重叠项，拆分职责过载项。   