# Agent Config Hub Codex 平台专属配置 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `agent-config-hub` 建立 Codex 专属配置层，明确 `shared` 边界，补齐 Codex 的规则入口、技能目录、配置模板、安装器和测试，同时不再实现 Claude 到 Codex 的跨平台同步。

**Architecture:** 保留 `shared/` 作为薄共享内核，只承载真正平台无关的脚本、文档和共享技能；将 Claude 与 Codex 的入口、规则和模板彻底拆分。Codex 安装器继续管理现有 `superpowers` 外部依赖，同时新增受控路径来托管 `AGENTS.md`、规则文档、配置模板和 repo 自带技能目录，但绝不覆盖用户现有 `~/.codex/config.toml`。

**Tech Stack:** Bash, Markdown, Python 3, Git, Codex CLI 路径约定

---

### Task 1: 收口共享边界并迁移 Claude 专属模板

**Files:**
- Modify: `agent-config-hub/shared/scripts/agent-link`
- Modify: `agent-config-hub/shared/docs/migration-inventory.md`
- Modify: `agent-config-hub/tests/claude/layout_test.sh`
- Modify: `agent-config-hub/tests/claude/skills_inventory_test.sh`
- Modify: `agent-config-hub/tests/claude/agent_link_smoke_test.sh`
- Create: `agent-config-hub/claude/templates/project-CLAUDE.md`
- Delete: `agent-config-hub/shared/templates/project-CLAUDE.md`

- [ ] **Step 1: 先把 Claude 模板迁移的失败测试写出来**

在 `agent-config-hub/tests/claude/layout_test.sh` 和 `agent-config-hub/tests/claude/skills_inventory_test.sh` 里，把模板路径从 `agent-config-hub/shared/templates/project-CLAUDE.md` 改成 `agent-config-hub/claude/templates/project-CLAUDE.md`，并让 `agent_link_smoke_test.sh` 断言 `shared/scripts/agent-link` 使用新的模板路径。

关键断言应类似：

```bash
test -f "$root/agent-config-hub/claude/templates/project-CLAUDE.md"
test ! -f "$root/agent-config-hub/shared/templates/project-CLAUDE.md"
rg -n 'claude/templates/project-CLAUDE.md' "$root/agent-config-hub/shared/scripts/agent-link" > /dev/null
```

- [ ] **Step 2: 运行测试，确认迁移前确实失败**

Run:

```bash
bash agent-config-hub/tests/claude/layout_test.sh
bash agent-config-hub/tests/claude/skills_inventory_test.sh
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
```

Expected:

- `layout_test.sh` 因新模板路径缺失而失败
- `skills_inventory_test.sh` 因旧模板路径断言失效而失败
- `agent_link_smoke_test.sh` 因脚本仍指向旧路径而失败

- [ ] **Step 3: 做最小迁移实现**

执行以下修改：

- 将 `agent-config-hub/shared/templates/project-CLAUDE.md` 移动到 `agent-config-hub/claude/templates/project-CLAUDE.md`
- 更新 `agent-config-hub/shared/scripts/agent-link` 中的 `TEMPLATE_PATH`
- 更新 `agent-config-hub/shared/docs/migration-inventory.md`，移除把 Claude 模板视为共享资产的表述

`agent-link` 中的路径应变成：

```bash
TEMPLATE_PATH="${AGENT_CONFIG_HUB}/claude/templates/project-CLAUDE.md"
```

- [ ] **Step 4: 重新运行测试，确认迁移后通过**

Run:

```bash
bash agent-config-hub/tests/claude/layout_test.sh
bash agent-config-hub/tests/claude/skills_inventory_test.sh
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
```

Expected:

- 三个测试全部 PASS

- [ ] **Step 5: 提交这一组边界收口改动**

```bash
git add agent-config-hub/claude/templates/project-CLAUDE.md \
  agent-config-hub/shared/scripts/agent-link \
  agent-config-hub/shared/docs/migration-inventory.md \
  agent-config-hub/tests/claude/layout_test.sh \
  agent-config-hub/tests/claude/skills_inventory_test.sh \
  agent-config-hub/tests/claude/agent_link_smoke_test.sh
git add -A agent-config-hub/shared/templates/project-CLAUDE.md
git commit -m "refactor: 收口 Claude 专属模板边界"
```

### Task 2: 建立 Codex 规则入口、模板和技能骨架

**Files:**
- Modify: `agent-config-hub/codex/README.md`
- Create: `agent-config-hub/codex/AGENTS.md`
- Create: `agent-config-hub/codex/rules/gitnexus-usage.md`
- Create: `agent-config-hub/codex/rules/git-commit-zh.md`
- Create: `agent-config-hub/codex/rules/plan-naming-zh.md`
- Create: `agent-config-hub/codex/config/config.toml.example`
- Create: `agent-config-hub/codex/skills/README.md`
- Create: `agent-config-hub/tests/codex/layout_test.sh`

- [ ] **Step 1: 先写 Codex 结构测试锁定目标骨架**

新增 `agent-config-hub/tests/codex/layout_test.sh`，断言以下文件必须存在：

```bash
test -f "$root/agent-config-hub/codex/AGENTS.md"
test -f "$root/agent-config-hub/codex/rules/gitnexus-usage.md"
test -f "$root/agent-config-hub/codex/rules/git-commit-zh.md"
test -f "$root/agent-config-hub/codex/rules/plan-naming-zh.md"
test -f "$root/agent-config-hub/codex/config/config.toml.example"
test -f "$root/agent-config-hub/codex/skills/README.md"
```

同时在测试中检查 `codex/README.md` 不再把 Codex 层描述成“只负责 superpowers skills 链接”。

- [ ] **Step 2: 运行新测试，确认 Codex 骨架还不存在**

Run:

```bash
bash agent-config-hub/tests/codex/layout_test.sh
```

Expected:

- 因 `AGENTS.md`、`rules/`、`config/`、`skills/README.md` 缺失而失败

- [ ] **Step 3: 按最小可用集合创建 Codex 资产**

创建以下内容：

1. `agent-config-hub/codex/AGENTS.md`
   - 说明全局 Codex 配置由 `agent-config-hub/codex/` 托管
   - 指向 `~/.codex/agent-config-hub/rules/*.md`
   - 指向 `~/.agents/skills/agent-config-hub-shared` 与 `~/.agents/skills/agent-config-hub-codex`

建议最小结构：

```md
# Codex Global Entry

- Global Codex configuration is managed via `~/.codex`, sourced from `agent-config-hub/codex/`.
- Shared Codex-discoverable skills live under `~/.agents/skills/agent-config-hub-shared`.
- Codex-specific skills live under `~/.agents/skills/agent-config-hub-codex`.

## Core Rules

- Use Chinese for replies, comments, and commit messages. Read `~/.codex/agent-config-hub/rules/git-commit-zh.md`.
- Follow GitNexus guidance when changing indexed code. Read `~/.codex/agent-config-hub/rules/gitnexus-usage.md`.
- Follow plan naming guidance when writing implementation plans. Read `~/.codex/agent-config-hub/rules/plan-naming-zh.md`.
```

2. `agent-config-hub/codex/rules/*.md`
   - 不复制 Claude 文案，而是改成 Codex 专属写法
   - `gitnexus-usage.md` 明确使用 Codex 会话里的 GitNexus 工作流
   - `git-commit-zh.md` 明确回复、注释、提交用中文
   - `plan-naming-zh.md` 明确 `docs/superpowers/plans/` 的命名约束

3. `agent-config-hub/codex/config/config.toml.example`
   - 只放注释和占位示例，不放任何真实地址或密钥

建议最小内容：

```toml
# Copy to ~/.codex/config.toml and edit locally.
model_provider = "OpenAI"
model = "gpt-5.4"
review_model = "gpt-5.4"
model_reasoning_effort = "high"
network_access = "enabled"

[notice]
hide_full_access_warning = true
```

4. `agent-config-hub/codex/skills/README.md`
   - 说明此目录仅承载 Codex 专属技能
   - 共享技能继续留在 `agent-config-hub/shared/skills/`

5. 重写 `agent-config-hub/codex/README.md`
   - 说明这是 Codex 平台专属配置层
   - 说明会管理 `AGENTS.md`、规则、模板和技能入口
   - 说明不做 Claude 到 Codex 的自动同步

- [ ] **Step 4: 重新运行结构测试，确认骨架通过**

Run:

```bash
bash agent-config-hub/tests/codex/layout_test.sh
```

Expected:

- PASS

- [ ] **Step 5: 提交 Codex 骨架**

```bash
git add agent-config-hub/codex/README.md \
  agent-config-hub/codex/AGENTS.md \
  agent-config-hub/codex/rules \
  agent-config-hub/codex/config/config.toml.example \
  agent-config-hub/codex/skills/README.md \
  agent-config-hub/tests/codex/layout_test.sh
git commit -m "feat: 补齐 Codex 配置层骨架"
```

### Task 3: 扩展 Codex 安装器管理受控路径与技能入口

**Files:**
- Modify: `agent-config-hub/codex/install.sh`
- Modify: `agent-config-hub/tests/codex/install_smoke_test.sh`
- Create: `agent-config-hub/tests/codex/managed_paths_test.sh`

- [ ] **Step 1: 先写安装器失败测试锁定受控路径**

扩展 `agent-config-hub/tests/codex/install_smoke_test.sh`。先在临时 repo fixture 里补出最小的 `codex/` 与 `shared/skills/` 树，再在现有 `superpowers` 链接断言之外增加以下断言：

```bash
assert_symlink_target "${tmp_home}/.codex/AGENTS.md" "${tmp_repo}/codex/AGENTS.md"
assert_symlink_target "${tmp_home}/.codex/agent-config-hub/rules" "${tmp_repo}/codex/rules"
assert_symlink_target "${tmp_home}/.codex/agent-config-hub/config" "${tmp_repo}/codex/config"
assert_symlink_target "${tmp_home}/.agents/skills/agent-config-hub-shared" "${tmp_repo}/shared/skills"
assert_symlink_target "${tmp_home}/.agents/skills/agent-config-hub-codex" "${tmp_repo}/codex/skills"
```

新增 `agent-config-hub/tests/codex/managed_paths_test.sh`，覆盖两件事：

- 预先存在的 `~/.codex/config.toml` 在 install/uninstall 后仍保留原内容
- `status` 能输出共享技能、Codex 技能、AGENTS、rules、config 模板各自的状态

- [ ] **Step 2: 运行测试，确认旧安装器无法满足新要求**

Run:

```bash
bash agent-config-hub/tests/codex/install_smoke_test.sh
bash agent-config-hub/tests/codex/managed_paths_test.sh
```

Expected:

- `install_smoke_test.sh` 因缺少新增 symlink 而失败
- `managed_paths_test.sh` 因 `status` 未覆盖新受控路径而失败

- [ ] **Step 3: 用最小受控面扩展 `codex/install.sh`**

在 `agent-config-hub/codex/install.sh` 中新增以下设计：

1. **受控本地路径常量**

```bash
CODEX_HOME="${HOME}/.codex"
CODEX_MANAGED_HOME="${CODEX_HOME}/agent-config-hub"
CODEX_AGENTS_TARGET="${CODEX_HOME}/AGENTS.md"
CODEX_RULES_TARGET="${CODEX_MANAGED_HOME}/rules"
CODEX_CONFIG_TARGET="${CODEX_MANAGED_HOME}/config"
CODEX_SHARED_SKILLS_TARGET="${HOME}/.agents/skills/agent-config-hub-shared"
CODEX_PLATFORM_SKILLS_TARGET="${HOME}/.agents/skills/agent-config-hub-codex"
```

2. **受控 repo 源路径**

```bash
CODEX_AGENTS_SOURCE="${REPO_ROOT}/codex/AGENTS.md"
CODEX_RULES_SOURCE="${REPO_ROOT}/codex/rules"
CODEX_CONFIG_SOURCE="${REPO_ROOT}/codex/config"
CODEX_SHARED_SKILLS_SOURCE="${REPO_ROOT}/shared/skills"
CODEX_PLATFORM_SKILLS_SOURCE="${REPO_ROOT}/codex/skills"
```

3. **安装行为**
- 保留现有 `superpowers` clone/link 逻辑
- 追加创建 `~/.codex/agent-config-hub/`
- 以 symlink 方式挂载 `AGENTS.md`、`rules`、`config`
- 以 symlink 方式挂载 `~/.agents/skills/agent-config-hub-shared`
- 以 symlink 方式挂载 `~/.agents/skills/agent-config-hub-codex`
- 若 `~/.codex/config.toml` 不存在，只打印提示，不自动覆盖或生成

4. **status 行为**
- 对每个受控路径分别输出 `OK` / `FAIL`
- 对本地 `~/.codex/config.toml` 输出 `INFO file missing` 或 `OK file`

5. **uninstall 行为**
- 只删除上述受控 symlink
- 若 `~/.codex/agent-config-hub` 为空再删除
- 不碰 `~/.codex/config.toml`、认证、历史、缓存

- [ ] **Step 4: 重新运行 Codex 安装测试，确认通过**

Run:

```bash
bash agent-config-hub/tests/codex/install_smoke_test.sh
bash agent-config-hub/tests/codex/managed_paths_test.sh
```

Expected:

- 两个测试都 PASS
- `managed_paths_test.sh` 中的 `config.toml` 哨兵内容保持不变

- [ ] **Step 5: 提交安装器升级**

```bash
git add agent-config-hub/codex/install.sh \
  agent-config-hub/tests/codex/install_smoke_test.sh \
  agent-config-hub/tests/codex/managed_paths_test.sh
git commit -m "feat: 扩展 Codex 安装器受控路径"
```

### Task 4: 更新根文档并完成回归验证

**Files:**
- Modify: `agent-config-hub/README.md`
- Modify: `agent-config-hub/codex/README.md`

- [ ] **Step 1: 更新根 README 的平台边界说明**

将 `agent-config-hub/README.md` 中关于 Codex 的描述从“只负责 superpowers skills 安装”更新为：

- `claude/` 只服务 Claude Code
- `codex/` 只服务 Codex
- `shared/` 只保留真正平台无关资产
- 不做 Claude 到 Codex 的自动同步

- [ ] **Step 2: 更新 Codex README 的安装与验证章节**

补充以下内容：

- 安装后管理的本地路径列表
- `~/.codex/config.toml` 不会被自动覆盖
- `~/.agents/skills/agent-config-hub-shared` 和 `~/.agents/skills/agent-config-hub-codex` 的用途
- 现有 `superpowers` 外部依赖仍然保留

- [ ] **Step 3: 复核迁移清单是否已在 Task 1 覆盖，无需重复改动**

确认 `agent-config-hub/shared/docs/migration-inventory.md` 已经在 Task 1 中完成模板归位说明；这里不再追加第二次无关改动。

- [ ] **Step 4: 运行最终回归**

Run:

```bash
bash agent-config-hub/tests/claude/layout_test.sh
bash agent-config-hub/tests/claude/skills_inventory_test.sh
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
bash agent-config-hub/tests/codex/layout_test.sh
bash agent-config-hub/tests/codex/install_smoke_test.sh
bash agent-config-hub/tests/codex/managed_paths_test.sh
```

Expected:

- 全部 PASS

- [ ] **Step 5: 提交文档与测试收尾**

```bash
git add agent-config-hub/README.md \
  agent-config-hub/codex/README.md \
  agent-config-hub/tests/claude/layout_test.sh \
  agent-config-hub/tests/claude/skills_inventory_test.sh \
  agent-config-hub/tests/claude/agent_link_smoke_test.sh \
  agent-config-hub/tests/codex/layout_test.sh \
  agent-config-hub/tests/codex/install_smoke_test.sh \
  agent-config-hub/tests/codex/managed_paths_test.sh
git commit -m "docs: 更新 Codex 平台专属配置说明"
```
