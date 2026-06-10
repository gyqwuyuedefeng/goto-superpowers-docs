# Personal Engineering Standards Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将已确认的个人工程规范设计落成一套正式、可长期维护、可供 AI 直接执行的 11 份规范文档。

**Architecture:** 过程性文档继续保留在 `docs/superpowers/specs/` 与 `docs/superpowers/plans/`，正式规范文档单独落在 `personal-engineering-standards/`，避免设计稿与长期规范混放。执行顺序固定为“先总纲，再 Python 基础，再 Python 模板，再前端与 Java，再跨栈契约，最后 AI 可执行启动清单”，并通过显式继承关系与文档边界规则消除重复。

**Tech Stack:** Markdown, Git, existing repository documentation style, GitNexus change-scope check before commits

---

## Repository Notes

- 当前设计事实源是 `docs/superpowers/specs/2026-03-27-personal-engineering-standards-design.md`，执行计划不得重新打开已确认的结构与 Python 方向讨论。
- 本轮实施目标是产出正式规范文档，不是写业务代码，也不是扩展新的模板体系。
- 正式规范文档应落在 `personal-engineering-standards/`；`docs/superpowers/` 只保留设计稿、计划稿、执行过程文档。
- 当前执行范围已收敛为 Python、前端（Vue 2）与跨栈协作；`30/31` 保留编号但当前只作为 Java 预留位，不作为已启用默认模板执行。
- 本计划中关于 `30/31` 的完整章节骨架保留为未来扩展参考，不代表当前这一版规范已经启用 Java 默认模板。
- 文档正文延续当前仓库风格：中文说明为主，文件名使用编号英文，关键约束使用 `MUST / SHOULD / MAY`。
- 每份正式规范文档都要显式写出“适用范围 / 继承来源 / 本文增量 / 例外条件 / AI 执行提示”，避免模板之间相互污染。
- 这是 Markdown-only 工作。执行落地时如未编辑代码符号，可不触发 GitNexus impact 分析；但在每次提交前仍应运行 `gitnexus_detect_changes()`，确认改动只涉及预期文档文件。

## File Map

**Create under `personal-engineering-standards/`:**

- `00-governance.md`
- `10-python-base.md`
- `11-python-api-template.md`
- `12-python-ai-agent-template.md`
- `13-python-script-template.md`
- `20-frontend-base.md`
- `21-frontend-vue-template.md`
- `30-java-base.md`
- `31-java-spring-template.md`
- `40-cross-stack-contract.md`
- `90-project-bootstrap-checklist.md`

## Phase Overview

| Phase | Deliverables | Why this order | Acceptance |
|---|---|---|---|
| Phase 0 | `personal-engineering-standards/` skeleton | 先固定长期目录与文件边界，避免正式规范继续堆进 `superpowers` 过程目录 | 11 个目标文件全部存在，目录用途说明清晰 |
| Phase 1 | `00-governance.md` | 所有后续文档都继承总纲；没有它就无法统一 `MUST/SHOULD/MAY`、模板选择与例外机制 | 总纲定义优先级、模板识别、例外规则、AI 规则、继承模型 |
| Phase 2 | `10-python-base.md` | Python 是当前信息最完整、最优先可落地的主线 | 12 个基础模块齐全，且无 FastAPI/LangGraph/TaskIQ 等模板专属内容 |
| Phase 3 | `11/12/13` 三份 Python 模板 | 在基础规范之上补模板增量，避免重复叙述共性规则 | 三份模板只写增量与场景差异，能直接指导不同 Python 项目启动 |
| Phase 4 | `20/21` 与 `30/31` | 结构已确认，但具体默认栈细节少于 Python，适合放在第二梯队归纳成文 | 前端与 Java 文档骨架完整，模板文档明确继承 base 文档且不混写跨栈规则 |
| Phase 5 | `40-cross-stack-contract.md` 与 `90-project-bootstrap-checklist.md` | 只有前面单栈边界写清楚，跨栈契约和启动清单才能稳定收束 | 跨栈文档只包含跨边界契约；启动清单可被 AI 逐步执行 |
| Phase 6 | 全套文档一致性检查与发布整理 | 最后统一清理重复、断链、遗漏与越界内容 | 所有文档边界清晰、条款体例一致、改动范围受控 |

## Boundary Decisions To Lock Before Writing

### 1. `00-governance.md` 必须先定义的内容

- 规范体系的目标、适用范围、非目标与术语。
- `MUST / SHOULD / MAY` 的强度定义与使用规则。
- 条款固定模板：`规则 / 原因 / 执行标准 / 例外 / AI 执行提示`。
- 文档继承关系与冲突优先级：`00` 高于单栈 base，高于模板，高于项目例外说明。
- 新项目启动时的模板识别流程与默认模板选择流程。
- “默认优先，例外显式”的偏离规则，包括必须记录的原因、影响、替代方案。
- AI 执行基本规则：先识别模板，再生成结构；默认遵循默认栈；新增代码同步补类型、测试、配置样例、最小运行说明；不得混入多套风格。
- 文档边界规则：哪些内容必须进入 base/template/cross-stack/checklist，哪些内容不得越界。
- 规范维护方式，包括版本更新、变更记录和新增模板时的归档原则。

### 2. `10-python-base.md` 与 `11/12/13` 的去重规则

- `10-python-base.md` 只写所有 Python 项目都适用的长期稳定底线，不写 FastAPI、LangGraph、TaskIQ、Typer、Langfuse、Nacos 等模板或跨栈专属细节。
- Python 模板文档统一采用“继承自 `00 + 10`，只补充增量规则”的写法，不重复重述类型注解、配置、日志、测试基线等共性条款。
- 如果某条规则同时被三个 Python 模板共享，应回收到 `10-python-base.md`；如果只被一个模板使用，则留在对应模板。
- 如果某条规则涉及 Python 与其他语言/前端之间的契约，不写入 Python 模板，转入 `40-cross-stack-contract.md`。
- `11-python-api-template.md` 只处理 API 服务默认栈与 FastAPI 生命周期、SSE 接口等服务模板增量。
- `12-python-ai-agent-template.md` 继承 `11`，只处理 LangGraph、TaskIQ、Supervisor Pattern、human-in-the-loop、Langfuse、状态恢复等 Agent 增量。
- `13-python-script-template.md` 只处理 CLI/脚本/批处理场景，不得复制 API/Agent 服务生命周期规则。

### 3. `40-cross-stack-contract.md` 应收与不应收的内容

**必须收：**

- Python / Java / Vue 之间共享的接口契约与字段语义。
- 统一错误码、任务状态码、状态闭环语义。
- `trace_id / request_id` 透传规则。
- SSE 事件格式与字段约束。
- Redis Stream 事件命名、消息 envelope、消费者侧语义约束。
- 服务发现与注册约束，例如 Nacos 接入边界。
- 跨栈版本兼容、破坏性变更与契约升级方式。

**不得收：**

- FastAPI 生命周期与依赖注入写法。
- LangGraph 节点拆分、Supervisor Pattern 实现细节。
- Java 包结构、Spring Bean 组织、事务实现细节。
- Vue 组件拆分、路由组织、样式系统细节。
- 任一语言内部的 lint、typing、测试、目录布局等单栈工程规范。

### 4. `90-project-bootstrap-checklist.md` 作为 AI 可执行清单的设计规则

- 清单必须按顺序执行，不能写成开放式建议列表。
- 每一步必须同时包含：`输入 / 动作 / 输出 / 例外处理 / AI 执行提示`。
- 每一步都应产出一个可检查结果，例如模板选择记录、目录骨架、配置样例、测试文件、运行命令。
- 清单的前置步骤必须先完成项目类型识别、模板选择、例外登记，后续步骤才允许初始化代码与依赖。
- 清单不能把“根据经验补充”或“按需完善”作为关键步骤描述；必须改写为确定性动作。
- 清单最后必须定义完成标准：最小可运行、最小可测试、最小可观测、最小 README。

## Document Outline Targets

### `00-governance.md`

```md
# 00 Governance
## 1. 文档目的与适用范围
## 2. 规范体系结构与继承关系
## 3. 术语、规则等级与条款模板
## 4. 新项目模板识别与选择流程
## 5. 默认技术栈与例外管理规则
## 6. 全项目统一工程底线
## 7. AI 协作执行规则
## 8. 文档边界与冲突处理
## 9. 维护、更新与版本记录规则
```

### `10-python-base.md`

```md
# 10 Python Base
## 1. 项目定位与适用范围
## 2. Python 版本与依赖管理
## 3. 目录结构与模块边界
## 4. 类型系统与静态检查
## 5. 配置与密钥管理
## 6. 异步、并发与任务执行
## 7. 数据建模与持久化边界
## 8. API 与接口契约基础规则
## 9. 错误处理与可靠性
## 10. 日志、监控与追踪
## 11. 测试策略
## 12. 安全基线与 AI 协作执行要求
```

### `11-python-api-template.md`

```md
# 11 Python API Template
## 1. 模板定位与启用条件
## 2. 继承关系与默认技术栈
## 3. 推荐目录结构与模块职责
## 4. 应用生命周期与资源管理
## 5. API 契约、OpenAPI 与校验规则
## 6. 数据访问、外部调用与缓存默认方案
## 7. 长耗时任务接口与 SSE 约束
## 8. 错误处理、观测与测试增量规则
## 9. 例外条件与升级/降级边界
## 10. AI 初始化输出要求
```

### `12-python-ai-agent-template.md`

```md
# 12 Python AI Agent Template
## 1. 模板定位与启用条件
## 2. 继承关系与默认技术栈增量
## 3. Agent 工作流组织与 Supervisor Pattern
## 4. 异步任务、队列与状态中枢
## 5. 状态持久化、重试与恢复
## 6. Human-in-the-loop 与人工审批节点
## 7. LLM tracing、Prompt 重试与观测
## 8. SSE 进度推送与任务编排责任
## 9. 例外条件、风险点与演练要求
## 10. AI 初始化输出要求
```

### `13-python-script-template.md`

```md
# 13 Python Script Template
## 1. 模板定位与适用场景
## 2. 继承关系与默认技术栈
## 3. 项目结构、入口与运行方式
## 4. 配置、密钥与运行参数
## 5. I/O、数据处理与外部调用边界
## 6. 失败处理、退出码与幂等性
## 7. 日志、监控与测试要求
## 8. 升级为 API / AI Agent 模板的触发条件
## 9. AI 初始化输出要求
```

### `20-frontend-base.md`

```md
# 20 Frontend Base
## 1. 项目定位与适用范围
## 2. 运行时、包管理与版本策略
## 3. 目录结构与模块边界
## 4. 类型系统与静态检查
## 5. 配置与环境变量管理
## 6. 状态管理、数据获取与缓存基线
## 7. 接口契约消费与错误/加载态基线
## 8. 日志、监控与追踪
## 9. 测试策略
## 10. 安全基线与 AI 协作要求
```

### `21-frontend-vue-template.md`

```md
# 21 Frontend Vue Template
## 1. 模板定位与启用条件
## 2. 继承关系与默认技术栈
## 3. 页面、组件、helper-mixin/store 边界
## 4. 路由、布局与状态组织
## 5. API、SSE 与 WebSocket 使用规则
## 6. 表单、校验与交互反馈
## 7. 样式主题与设计系统接入
## 8. 测试、构建与发布要求
## 9. AI 初始化输出要求
```

### `30-java-base.md`

```md
# 30 Java Base
## 1. 项目定位与适用范围
## 2. JDK、构建工具与依赖管理
## 3. 模块/包结构与分层边界
## 4. 类型、DTO、领域模型与映射边界
## 5. 配置与密钥管理
## 6. 并发、调度与任务执行基线
## 7. 数据访问与事务边界
## 8. API 与接口契约基础规则
## 9. 错误处理与可靠性
## 10. 日志、监控与追踪
## 11. 测试策略
## 12. 安全基线与 AI 协作要求
```

### `31-java-spring-template.md`

```md
# 31 Java Spring Template
## 1. 模板定位与启用条件
## 2. 继承关系与默认技术栈
## 3. Spring Boot 应用结构与装配边界
## 4. Web/API 层规范
## 5. 持久化、事务与外部集成规范
## 6. 任务执行、事件与调度规范
## 7. 配置运行与可观测性规范
## 8. 测试、构建与发布要求
## 9. AI 初始化输出要求
```

### `40-cross-stack-contract.md`

```md
# 40 Cross Stack Contract
## 1. 文档定位与参与方
## 2. 契约优先级、版本与变更原则
## 3. 统一标识与 trace 透传
## 4. 统一错误码与任务状态语义
## 5. Python / Java / Vue HTTP 接口契约
## 6. SSE 事件格式与字段规范
## 7. Redis Stream 事件命名与消息封装
## 8. Python-Java-Vue 状态闭环
## 9. 服务发现与 Nacos 接入约束
## 10. 明确不属于本文件的内容
```

### `90-project-bootstrap-checklist.md`

```md
# 90 Project Bootstrap Checklist
## 1. 使用方式与输入项
## 2. 清单步骤模板
## 3. Step 01: 识别项目类型
## 4. Step 02: 选择默认模板
## 5. Step 03: 登记例外与替代方案
## 6. Step 04: 初始化目录与仓库骨架
## 7. Step 05: 初始化依赖、运行时与配置样例
## 8. Step 06: 初始化核心入口、接口或任务骨架
## 9. Step 07: 初始化类型、测试与质量门禁
## 10. Step 08: 初始化契约、事件或状态流转骨架
## 11. Step 09: 生成 README 与最小运行说明
## 12. Step 10: 运行启动验证与完成检查
## 13. 完成定义与交付物清单
```

## Task 1: Scaffold The Permanent Standards Directory

**Files:**

- Create: `personal-engineering-standards/00-governance.md`
- Create: `personal-engineering-standards/10-python-base.md`
- Create: `personal-engineering-standards/11-python-api-template.md`
- Create: `personal-engineering-standards/12-python-ai-agent-template.md`
- Create: `personal-engineering-standards/13-python-script-template.md`
- Create: `personal-engineering-standards/20-frontend-base.md`
- Create: `personal-engineering-standards/21-frontend-vue-template.md`
- Create: `personal-engineering-standards/30-java-base.md`
- Create: `personal-engineering-standards/31-java-spring-template.md`
- Create: `personal-engineering-standards/40-cross-stack-contract.md`
- Create: `personal-engineering-standards/90-project-bootstrap-checklist.md`

- [ ] **Step 1: Create the permanent standards directory**

Run:

```bash
mkdir -p personal-engineering-standards
```

Expected: 正式规范开始拥有独立长期目录，不再继续放在 `docs/superpowers/` 下。

- [ ] **Step 2: Create the 11 target markdown files**

Run:

```bash
touch \
  personal-engineering-standards/00-governance.md \
  personal-engineering-standards/10-python-base.md \
  personal-engineering-standards/11-python-api-template.md \
  personal-engineering-standards/12-python-ai-agent-template.md \
  personal-engineering-standards/13-python-script-template.md \
  personal-engineering-standards/20-frontend-base.md \
  personal-engineering-standards/21-frontend-vue-template.md \
  personal-engineering-standards/30-java-base.md \
  personal-engineering-standards/31-java-spring-template.md \
  personal-engineering-standards/40-cross-stack-contract.md \
  personal-engineering-standards/90-project-bootstrap-checklist.md
```

Expected: 11 个目标文件全部存在，后续任务只往这些固定文件中写内容。

- [ ] **Step 3: Verify the target file set**

Run:

```bash
rg --files personal-engineering-standards
```

Expected: 输出只包含上述 11 个文件。

- [ ] **Step 4: Commit the directory skeleton**

```bash
git add personal-engineering-standards
git commit -m "docs: scaffold personal engineering standards structure"
```

## Task 2: Draft `00-governance.md` First

**Files:**

- Modify: `personal-engineering-standards/00-governance.md`
- Reference: `docs/superpowers/specs/2026-03-27-personal-engineering-standards-design.md`

- [ ] **Step 1: Add the exact section skeleton from the outline target**

Expected: 文档先具备 9 个一级职责章节，不先写散乱条款。

- [ ] **Step 2: Write sections 1-5 to lock the normative model**

必须覆盖：

- 文档目的与适用范围
- 规范体系结构与继承关系
- `MUST / SHOULD / MAY`
- `规则 / 原因 / 执行标准 / 例外 / AI 执行提示`
- 模板识别流程
- 默认模板选择流程
- 例外登记规则

Expected: 后续所有文档都可以直接引用这一套元规则，不再各自发明条款格式。

- [ ] **Step 3: Write sections 6-9 to lock the cross-document operating model**

必须覆盖：

- 全项目统一工程底线的高层范围
- AI 协作执行规则
- 文档边界与冲突处理
- 规范更新、版本记录与维护方式

Expected: `00` 成为所有文档的上位约束，而不是另一篇普通说明文。

- [ ] **Step 4: Verify that governance contains the required anchor terms**

Run:

```bash
rg -n "MUST|SHOULD|MAY|模板|例外|AI 执行提示|继承" personal-engineering-standards/00-governance.md
```

Expected: 可以找到规则等级、模板选择、例外管理、AI 执行与继承关系内容。

- [ ] **Step 5: Commit governance**

```bash
git add personal-engineering-standards/00-governance.md
git commit -m "docs: add governance for personal engineering standards"
```

## Task 3: Draft `10-python-base.md` As The Shared Python Floor

**Files:**

- Modify: `personal-engineering-standards/10-python-base.md`
- Reference: `personal-engineering-standards/00-governance.md`
- Reference: `docs/superpowers/specs/2026-03-27-personal-engineering-standards-design.md`

- [ ] **Step 1: Add the 12 confirmed modules as section headings**

Expected: 与 spec 中确认的 12 模块完全一致，不新增“杂项”章节去稀释边界。

- [ ] **Step 2: Draft modules 1-6 using only Python-wide baseline rules**

必须落入这里的内容：

- Python `3.11`
- Poetry
- 类型注解
- `Pydantic v2`
- `pydantic-settings`
- 结构化日志
- 异步原则
- 目录布局与模块边界

Expected: 这一半章节不出现 FastAPI、LangGraph、TaskIQ、Typer、Langfuse 等模板专属技术。

- [ ] **Step 3: Draft modules 7-12 using only cross-template Python engineering rules**

必须覆盖：

- 数据建模与持久化边界
- API 与接口契约基础规则
- 错误处理与可靠性
- 日志、监控与追踪
- 测试策略
- 安全基线与 AI 协作执行要求

Expected: 这些章节为 Python 模板提供统一底板，而不是再次写模板栈。

- [ ] **Step 4: Add an explicit anti-duplication note near the top**

明确写出：

- 本文不定义 FastAPI 生命周期
- 不定义 LangGraph / TaskIQ / Langfuse
- 不定义 Typer CLI 模板细节
- 不定义 Nacos、Redis Stream、跨语言契约

Expected: 后续模板文档的边界被提前锁定。

- [ ] **Step 5: Run the boundary check**

Run:

```bash
rg -n "FastAPI|LangGraph|TaskIQ|Langfuse|Typer|Nacos|Redis Stream" personal-engineering-standards/10-python-base.md
```

Expected: 无匹配，或仅出现在“本文不负责”边界说明中。

- [ ] **Step 6: Commit Python base**

```bash
git add personal-engineering-standards/10-python-base.md
git commit -m "docs: add python base engineering standard"
```

## Task 4: Draft `11-python-api-template.md`

**Files:**

- Modify: `personal-engineering-standards/11-python-api-template.md`
- Reference: `personal-engineering-standards/00-governance.md`
- Reference: `personal-engineering-standards/10-python-base.md`

- [ ] **Step 1: Add the 10 API template sections from the outline target**

Expected: 文档开头就写清“这是 API 服务模板，不是所有 Python 项目默认全文”。

- [ ] **Step 2: Write sections 1-3 to fix applicability and the default stack**

必须固定：

- `Python 3.11 + Poetry + FastAPI + Pydantic v2 + pydantic-settings + SQLAlchemy 2.x + Alembic + httpx + Redis + pytest + ruff + mypy + uvicorn + Docker`
- 继承自 `00-governance.md` 与 `10-python-base.md`
- 启用条件与升级/降级边界

Expected: API 服务模板不再需要重新讨论底座栈。

- [ ] **Step 3: Write sections 4-7 with API-specific implementation defaults**

必须覆盖：

- `FastAPI lifespan`
- `Depends + yield`
- OpenAPI 与响应模型
- 数据访问、外部调用与缓存默认方案
- 长耗时任务接口返回策略
- SSE 作为默认实时状态推送方式

Expected: 所有 API 服务特有规则都集中在模板内，不漂回 base 文档。

- [ ] **Step 4: Write sections 8-10 with observability, exceptions, and AI outputs**

必须覆盖：

- API 模板下的测试增量
- 例外条件
- AI 初始化输出要求

Expected: AI 能直接根据该模板生成 API 服务骨架，而不是只得到技术栈名单。

- [ ] **Step 5: Run the duplication boundary check**

Run:

```bash
rg -n "LangGraph|TaskIQ|Langfuse|Supervisor Pattern|human-in-the-loop" personal-engineering-standards/11-python-api-template.md
```

Expected: 无匹配。

- [ ] **Step 6: Commit the API template**

```bash
git add personal-engineering-standards/11-python-api-template.md
git commit -m "docs: add python api service template"
```

## Task 5: Draft `12-python-ai-agent-template.md`

**Files:**

- Modify: `personal-engineering-standards/12-python-ai-agent-template.md`
- Reference: `personal-engineering-standards/11-python-api-template.md`
- Reference: `personal-engineering-standards/40-cross-stack-contract.md`

- [ ] **Step 1: Add the 10 AI Agent template sections from the outline target**

Expected: 文档开头明确写出“继承 API 模板，再追加 Agent 增量”。

- [ ] **Step 2: Write sections 1-4 to fix the template delta**

必须固定：

- `LangGraph`
- `TaskIQ`
- `Langfuse`
- `Redis / Redis Stream`
- `SSE`
- `Supervisor Pattern`

Expected: Agent 模板和普通 API 模板的差异被一次性说清。

- [ ] **Step 3: Write sections 5-8 to lock production-grade agent behavior**

必须覆盖：

- 状态持久化
- 重试与恢复
- `human-in-the-loop`
- 人工审批节点
- LLM tracing
- prompt 重试与回放边界
- SSE 进度推送责任

Expected: 文档可直接约束 AI Agent 服务的工作流组织方式，而不是停留在工具清单。

- [ ] **Step 4: Write sections 9-10 for risk handling and AI outputs**

必须覆盖：

- 演练与恢复要求
- 偏离条件
- AI 初始化输出要求

Expected: 模板可直接支撑长期维护与故障恢复设计。

- [ ] **Step 5: Run the cross-stack leakage check**

Run:

```bash
rg -n "Nacos|统一错误码|trace_id|request_id|Python-Java-Vue|事件命名" personal-engineering-standards/12-python-ai-agent-template.md
```

Expected: 无匹配，或仅出现在“跨栈契约见 `40-cross-stack-contract.md`”的引用语句中。

- [ ] **Step 6: Commit the AI Agent template**

```bash
git add personal-engineering-standards/12-python-ai-agent-template.md
git commit -m "docs: add python ai agent template"
```

## Task 6: Draft `13-python-script-template.md`

**Files:**

- Modify: `personal-engineering-standards/13-python-script-template.md`
- Reference: `personal-engineering-standards/10-python-base.md`

- [ ] **Step 1: Add the 9 script template sections from the outline target**

Expected: 文档结构首先体现“脚本/数据任务”是独立模板，而不是简化版 API 服务。

- [ ] **Step 2: Write sections 1-5 to fix the script baseline**

必须固定：

- `Python 3.11 + Poetry + pydantic-settings + Pydantic v2 + structured logging + Typer + httpx + pytest + ruff + mypy`
- `pandas` 按需使用
- 入口、参数、I/O 与外部调用边界

Expected: 脚本模板保持轻量，但仍有工程底线。

- [ ] **Step 3: Write sections 6-9 to define reliability and promotion rules**

必须覆盖：

- 失败处理
- 退出码
- 幂等性
- 测试要求
- 升级为 API 模板的触发条件
- 升级为 AI Agent 模板的触发条件
- AI 初始化输出要求

Expected: 脚本不会长期停留在“临时脚本”灰区。

- [ ] **Step 4: Run the service-template leakage check**

Run:

```bash
rg -n "FastAPI lifespan|Depends \\+ yield|LangGraph|TaskIQ|Langfuse|Supervisor Pattern" personal-engineering-standards/13-python-script-template.md
```

Expected: 无匹配，或仅在“升级触发条件”章节中出现。

- [ ] **Step 5: Commit the script template**

```bash
git add personal-engineering-standards/13-python-script-template.md
git commit -m "docs: add python script template"
```

## Task 7: Draft `20-frontend-base.md` And `21-frontend-vue-template.md`

**Files:**

- Modify: `personal-engineering-standards/20-frontend-base.md`
- Modify: `personal-engineering-standards/21-frontend-vue-template.md`
- Reference: `personal-engineering-standards/00-governance.md`

- [ ] **Step 1: Write `20-frontend-base.md` as a framework-neutral baseline**

必须覆盖：

- 运行时、包管理与版本策略
- 目录结构与模块边界
- 类型系统与静态检查
- 配置与环境变量
- 状态管理、数据获取与缓存基线
- 错误/加载态
- 监控、测试、安全、AI 协作

Expected: base 文档不直接绑定 Vue 具体实现。

- [ ] **Step 2: Freeze the default Vue stack at the top of `21-frontend-vue-template.md`**

执行要求：

- 先在 section 2 写出唯一默认 Vue 模板栈
- 如果此时仍有未文档化细节，只允许在该 section 内补成一段明确的默认栈声明，不得在后文混入多套可选框架

Expected: `21` 一开始就表现为默认模板，而不是备选方案列表。

- [ ] **Step 3: Write the Vue template delta**

必须覆盖：

- 页面、组件、helper-mixin/store 边界
- 路由、布局与状态组织
- API、SSE、WebSocket 使用规则
- 表单与交互反馈
- 样式主题与设计系统接入
- 测试、构建与发布要求
- AI 初始化输出要求

Expected: `21` 只写 Vue 模板增量，不重讲 base 基线。

- [ ] **Step 4: Run the template/base duplication check**

Run:

```bash
rg -n "Vue|component|composable|store|WebSocket|SSE" personal-engineering-standards/20-frontend-base.md
```

Expected: 这些模板细节不应大面积出现在 `20-frontend-base.md` 中。

- [ ] **Step 5: Commit the frontend docs**

```bash
git add \
  personal-engineering-standards/20-frontend-base.md \
  personal-engineering-standards/21-frontend-vue-template.md
git commit -m "docs: add frontend base and vue template"
```

## Task 8: Draft `30-java-base.md` And `31-java-spring-template.md`

**Files:**

- Modify: `personal-engineering-standards/30-java-base.md`
- Modify: `personal-engineering-standards/31-java-spring-template.md`
- Reference: `personal-engineering-standards/00-governance.md`

- [ ] **Step 1: Write `30-java-base.md` as the framework-neutral Java baseline**

必须覆盖：

- JDK、构建工具与依赖管理
- 模块/包结构与分层边界
- DTO/领域模型/映射边界
- 配置与密钥
- 并发、调度与任务执行基线
- 数据访问与事务边界
- 接口契约基础规则
- 错误处理、日志、测试、安全、AI 协作

Expected: base 文档不直接锁死 Spring Boot 细节。

- [ ] **Step 2: Freeze the default Spring template stack at the top of `31-java-spring-template.md`**

执行要求：

- 在 section 2 明确唯一默认 Spring 模板栈
- 不在后文混入“Spring 或其他框架”并列写法

Expected: `31` 从开头就是默认模板，而不是二选一指南。

- [ ] **Step 3: Write the Spring template delta**

必须覆盖：

- Spring Boot 应用结构与装配边界
- Web/API 层规范
- 持久化、事务与外部集成规范
- 任务执行、事件与调度规范
- 配置运行与可观测性规范
- 测试、构建与发布要求
- AI 初始化输出要求

Expected: `31` 只处理 Spring 模板增量，不反复重讲 Java 基础。

- [ ] **Step 4: Run the template/base duplication check**

Run:

```bash
rg -n "Spring|Bean|Controller|Configuration|Transactional" personal-engineering-standards/30-java-base.md
```

Expected: 这些 Spring 模板细节不应大面积出现在 `30-java-base.md` 中。

- [ ] **Step 5: Commit the Java docs**

```bash
git add \
  personal-engineering-standards/30-java-base.md \
  personal-engineering-standards/31-java-spring-template.md
git commit -m "docs: add java base and spring template"
```

## Task 9: Draft `40-cross-stack-contract.md`

**Files:**

- Modify: `personal-engineering-standards/40-cross-stack-contract.md`
- Reference: `personal-engineering-standards/00-governance.md`
- Reference: `personal-engineering-standards/11-python-api-template.md`
- Reference: `personal-engineering-standards/12-python-ai-agent-template.md`
- Reference: `personal-engineering-standards/21-frontend-vue-template.md`
- Reference: `personal-engineering-standards/31-java-spring-template.md`

- [ ] **Step 1: Add the 10 cross-stack sections from the outline target**

Expected: 文档起手就写明“这是跨边界契约，不是单栈实现细节文档”。

- [ ] **Step 2: Write sections 1-5 to define shared contract vocabulary**

必须覆盖：

- 参与方与优先级
- 契约版本与变更原则
- `trace_id / request_id`
- 统一错误码
- 任务状态语义
- HTTP 接口字段约束

Expected: Python / Java / Vue 使用同一套外部契约语言。

- [ ] **Step 3: Write sections 6-10 to define event and integration rules**

必须覆盖：

- SSE 事件格式
- Redis Stream 事件命名
- 消息 envelope
- Python-Java-Vue 状态闭环
- Nacos 接入约束
- 明确不属于本文件的内容

Expected: `40` 只承担跨栈协作契约，不吞并语言内部规范。

- [ ] **Step 4: Run the single-stack leakage check**

Run:

```bash
rg -n "FastAPI lifespan|Depends \\+ yield|LangGraph|TaskIQ|Typer|Spring Bean|Vue 组件" personal-engineering-standards/40-cross-stack-contract.md
```

Expected: 无匹配。

- [ ] **Step 5: Commit the cross-stack contract**

```bash
git add personal-engineering-standards/40-cross-stack-contract.md
git commit -m "docs: add cross stack contract standard"
```

## Task 10: Draft `90-project-bootstrap-checklist.md` As An AI-Executable Checklist

**Files:**

- Modify: `personal-engineering-standards/90-project-bootstrap-checklist.md`
- Reference: `personal-engineering-standards/00-governance.md`
- Reference: `personal-engineering-standards/10-python-base.md`
- Reference: `personal-engineering-standards/11-python-api-template.md`
- Reference: `personal-engineering-standards/12-python-ai-agent-template.md`
- Reference: `personal-engineering-standards/13-python-script-template.md`
- Reference: `personal-engineering-standards/20-frontend-base.md`
- Reference: `personal-engineering-standards/21-frontend-vue-template.md`
- Reference: `personal-engineering-standards/30-java-base.md`
- Reference: `personal-engineering-standards/31-java-spring-template.md`
- Reference: `personal-engineering-standards/40-cross-stack-contract.md`

- [ ] **Step 1: Write the checklist item template before writing the actual steps**

必须固定每一步字段：

- 输入
- 动作
- 输出
- 例外处理
- AI 执行提示

Expected: 清单本身具备机器可执行结构，而不是散文式建议。

- [ ] **Step 2: Write Steps 01-03 for project identification and template selection**

必须覆盖：

- 识别项目类型
- 选择默认模板
- 登记例外与替代方案

Expected: 在真正生成项目骨架前，模板与例外已经被记录。

- [ ] **Step 3: Write Steps 04-07 for skeleton, dependencies, entrypoints, and quality gates**

必须覆盖：

- 初始化目录与仓库骨架
- 初始化依赖、运行时、配置样例
- 初始化核心入口、接口或任务骨架
- 初始化类型、测试与质量门禁

Expected: AI 能按顺序直接产出可运行的最小项目。

- [ ] **Step 4: Write Steps 08-10 and the definition of done**

必须覆盖：

- 初始化契约、事件或状态流转骨架
- 生成 README 与最小运行说明
- 运行启动验证与完成检查
- 最终交付物清单

Expected: 清单最后能给出完成标准，而不是只列“开始动作”。

- [ ] **Step 5: Run the executability check**

Run:

```bash
rg -n "输入|动作|输出|例外处理|AI 执行提示|完成定义" personal-engineering-standards/90-project-bootstrap-checklist.md
```

Expected: 每个关键步骤都能找到固定字段，且包含完成定义。

- [ ] **Step 6: Commit the bootstrap checklist**

```bash
git add personal-engineering-standards/90-project-bootstrap-checklist.md
git commit -m "docs: add project bootstrap checklist"
```

## Task 11: Run The Final Consistency Pass Across All Standards

**Files:**

- Modify: `personal-engineering-standards/*.md`

- [ ] **Step 1: Verify all 11 files exist and are non-empty**

Run:

```bash
rg --files personal-engineering-standards
find personal-engineering-standards -maxdepth 1 -type f -name '*.md' -size +0c | sort
```

Expected: 11 个文件全部存在，且没有空文档。

- [ ] **Step 2: Verify the governance and clause grammar is discoverable**

Run:

```bash
rg -n "MUST|SHOULD|MAY|规则|原因|执行标准|例外|AI 执行提示" personal-engineering-standards/*.md
```

Expected: 关键文档可以检索到统一条款体例。

- [ ] **Step 3: Re-run the boundary checks**

Run:

```bash
rg -n "FastAPI|LangGraph|TaskIQ|Langfuse|Typer|Nacos|Redis Stream" personal-engineering-standards/10-python-base.md
rg -n "LangGraph|TaskIQ|Langfuse|human-in-the-loop" personal-engineering-standards/11-python-api-template.md
rg -n "Spring|Controller|Bean|Transactional" personal-engineering-standards/30-java-base.md
rg -n "FastAPI lifespan|Depends \\+ yield|LangGraph|TaskIQ|Typer|Spring Bean|Vue 组件" personal-engineering-standards/40-cross-stack-contract.md
```

Expected: 越界项只出现在“排除项说明”中，不出现在正文规则中。

- [ ] **Step 4: Run the doc-only scope check before commit**

Run:

```text
gitnexus_detect_changes({scope: "all"})
```

Expected: GitNexus 只报告文档文件改动；若没有索引到符号变化，应明确记录“本轮为 docs-only changes”。

- [ ] **Step 5: Commit the complete standards set**

```bash
git add personal-engineering-standards
git commit -m "docs: add personal engineering standards set"
```

## Acceptance Checklist For The Whole Effort

- `00-governance.md` 已先于其他文档完成，并定义规则等级、条款模板、模板识别流程、例外管理和 AI 执行基本法。
- `10-python-base.md` 精确保留 12 个模块，且不被 FastAPI/LangGraph/TaskIQ 等模板技术污染。
- `11/12/13` 三份 Python 模板采用“继承 + 增量”写法，没有大段重复 `10` 的共性条款。
- `40-cross-stack-contract.md` 只收录跨栈契约，不包含任一语言内部实现细节。
- `90-project-bootstrap-checklist.md` 具备确定性步骤、输入输出、例外处理和完成定义，可被 AI 直接执行。
- 正式规范文档全部位于 `personal-engineering-standards/`，与过程性 `specs/plans` 分离。
- 每轮提交前都运行了 `gitnexus_detect_changes()`，确认改动范围符合预期。
