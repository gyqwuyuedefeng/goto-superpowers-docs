# Personal Engineering Standards README And EXTENDING Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `personal-engineering-standards/` 目录补齐 `README.md` 与 `EXTENDING.md`，形成可直接阅读的入口层与后续扩展规则文档。

**Architecture:** `README.md` 只承担目录入口、当前生效范围说明、文档地图与使用顺序；`EXTENDING.md` 只承担未来扩展这套规范本身的方法与边界，不新增正式规范正文。两份文档都使用中文与目录内相对链接 `./<file>.md`。

**Tech Stack:** Markdown, repo-relative links, existing personal-engineering-standards document style

---

## File Map

**Create**

- `personal-engineering-standards/README.md`
- `personal-engineering-standards/EXTENDING.md`

**Reference**

- `docs/superpowers/specs/2026-03-27-personal-engineering-standards-readme-extension-design.md`
- `personal-engineering-standards/00-governance.md`
- `personal-engineering-standards/90-project-bootstrap-checklist.md`

## Task 1: Create `README.md`

**Files:**

- Create: `personal-engineering-standards/README.md`
- Reference: `personal-engineering-standards/00-governance.md`

- [ ] **Step 1: Write the fixed top-level structure**

Use exactly these headings:

- `# personal-engineering-standards`
- `## 1. 项目定位`
- `## 2. 当前生效范围`
- `## 3. 文档地图`
- `## 4. 推荐阅读顺序`
- `## 5. 如何使用这套规范`
- `## 6. 后续扩展入口`

- [ ] **Step 2: Write the scope summary**

Expected content:

- 当前正式生效范围为 Python、前端（Vue 2）与跨栈协作
- `30-java-base.md` 与 `31-java-spring-template.md` 为 `RESERVED`
- Vue 3 为后续补充，不是当前默认模板

- [ ] **Step 3: Write the document map**

Each line must use:

- `./<file>.md`
- one-line responsibility
- status tag `ACTIVE` or `RESERVED`

- [ ] **Step 4: Write the reading order and usage section**

Expected content:

- 先读 `./00-governance.md`
- 再按项目类型选择模板
- 涉及跨栈时补读 `./40-cross-stack-contract.md`
- 启动项目时执行 `./90-project-bootstrap-checklist.md`

## Task 2: Create `EXTENDING.md`

**Files:**

- Create: `personal-engineering-standards/EXTENDING.md`
- Reference: `personal-engineering-standards/00-governance.md`

- [ ] **Step 1: Write the fixed top-level structure**

Use exactly these headings:

- `# EXTENDING`
- `## 1. 本文定位`
- `## 2. 何时允许扩展或修改`
- `## 3. 内容归位规则`
- `## 4. 编号与命名规则`
- `## 5. 新增模板的判断标准`
- `## 6. Vue 3 的补充方式`
- `## 7. Java 的启用方式`
- `## 8. 修改默认栈时的同步检查清单`
- `## 9. AI 扩展执行要求`

- [ ] **Step 2: Write policy-only extension rules**

Expected content:

- 说明何时应修改规范本身，何时只记项目例外
- 说明 `00 / base / template / 40 / 90 / README / EXTENDING` 的归位规则
- 不新增正式规范条款
- 不写项目启动步骤

- [ ] **Step 3: Write Vue 3 and Java sections as gating rules only**

Expected content:

- Vue 3 section only defines when it may be added and which docs must be updated
- Java section only defines when `30/31` may move from `RESERVED` to `ACTIVE`
- No template正文 or setup steps in these sections

## Task 3: Verify The Two Docs

**Files:**

- Verify: `personal-engineering-standards/README.md`
- Verify: `personal-engineering-standards/EXTENDING.md`

- [ ] **Step 1: Verify section presence**

Run:

```bash
rg -n "^## " personal-engineering-standards/README.md
rg -n "^## " personal-engineering-standards/EXTENDING.md
```

Expected: the required headings are present in the correct order.

- [ ] **Step 2: Verify link style and scope markers**

Run:

```bash
rg -n "/mnt/|docs/personal-engineering-standards|ACTIVE|RESERVED|Vue 3|Java" personal-engineering-standards/README.md personal-engineering-standards/EXTENDING.md
```

Expected:

- no absolute machine paths
- README includes `ACTIVE` / `RESERVED`
- both docs mention Vue 3 and Java only with the intended scope

- [ ] **Step 3: Verify whitespace and staged diff cleanliness**

Run:

```bash
git diff --check -- personal-engineering-standards/README.md personal-engineering-standards/EXTENDING.md
```

Expected: no whitespace or conflict-marker issues.
