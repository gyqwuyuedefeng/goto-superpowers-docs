# agent-config-hub Claude 优先统一配置中心 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在当前仓库内落地 `agent-config-hub`，完成 Claude Code 侧的全局统一配置中心、共享 skills、受管安装器和项目接入工具，并把现有项目级/用户级 Claude 资产从分散状态收敛到新的 Hub。

**Architecture:** 仓库内创建 `agent-config-hub/shared`、`agent-config-hub/claude`、`agent-config-hub/codex` 三层结构；`shared` 只存可复用事实源，`claude` 存入口、rules、commands、agents、hooks 与共享 settings 模板，`codex` 只保留未来兼容骨架。运行时采用“受管覆盖层”而非整目录直链：`install.sh` 创建真实 `~/.claude` 并逐项软链接到 Hub，把 `debug/todos/cache/backups` 与 `settings.local.json` 留在本机私有层；项目本地状态统一迁移到 `.agent-hub/claude/`。

**Tech Stack:** Bash, POSIX shell utilities, Markdown, JSON, Node.js `node --check`, Bash smoke tests, Git, GitNexus CLI/MCP（如可用）

---

## Repository Notes

- 代码仓库根目录为 `/mnt/f/IdeaProjects/goto-software`，本次实现直接在父仓库完成，不涉及 `goto`、`goto-web`、`freedom-public` 子仓库代码。
- 当前项目的 `.claude/` 是本地 source asset，但已被 `.gitignore` 忽略；迁移时应以“复制到 `agent-config-hub`”为主，不要依赖 Git 跟踪 `.claude/` 本身。
- 当前用户级 Claude 资产来源于 `~/.claude/{commands,agents,hooks,settings.json}`；这些内容要迁入仓库，但 `~/.claude/debug`、`~/.claude/todos`、`~/.claude/cache`、`~/.claude/backups` 与 `~/.claude/settings.local.json` 必须继续保留在本机。
- 当前 `.gitignore` 已忽略 `.claude/`、`CLAUDE.md`、`AGENTS.md`；`CLAUDE.md` 与 `AGENTS.md` 已经是 tracked 文件，可直接修改；新增 `.agent-hub/` 忽略规则时不要误删现有忽略项。
- 这个实现主要是 Markdown / shell / JSON / 文件编排工作。GitNexus 对 Markdown 与用户家目录文件价值有限；如果实现时需要分析 repo 内的 JS hook 或未来的新脚本，优先尝试 GitNexus，若当前会话 MCP 仍不可用，再用 CLI 或本地文件检查。
- 仓库规则要求 TDD；本计划对 shell 安装器与迁移脚本使用 Bash smoke test，所有功能先写失败验证，再做最小实现。
- 本轮不改 root `AGENTS.md` 的 Codex 语义，不为 Codex 实现安装与运行逻辑；仅创建 `agent-config-hub/codex` 占位骨架。

## File Structure

### Create

- `agent-config-hub/shared/docs/migration-inventory.md`
  - 记录项目 `.claude/` 与 `~/.claude/` 到新 Hub 的迁移映射
- `agent-config-hub/shared/docs/local-only-assets.md`
  - 记录必须保留在本机、不入仓的目录与配置
- `agent-config-hub/shared/docs/config-layering.md`
  - 记录共享配置、模板、本机覆盖层的分层边界
- `agent-config-hub/shared/docs/security-boundary.md`
  - 记录 secrets、permissions、hook 边界
- `agent-config-hub/shared/templates/project-CLAUDE.md`
  - `agent-link init-project` 生成项目入口文件的模板
- `agent-config-hub/shared/scripts/agent-link`
  - 项目接入 CLI，支持 `init-project` 与 `doctor`
- `agent-config-hub/shared/skills/agent-builder/SKILL.md`
- `agent-config-hub/shared/skills/agent-requirement-analyzer/SKILL.md`
- `agent-config-hub/shared/skills/qiuzhi-skill-creator/SKILL.md`
- `agent-config-hub/shared/skills/qiuzhi-skill-creator/LICENSE.txt`
- `agent-config-hub/claude/README.md`
  - 说明 Claude 适配层结构、安装方式、本机私有文件边界
- `agent-config-hub/claude/CLAUDE.md`
  - 全局 Claude 总入口
- `agent-config-hub/claude/install.sh`
  - Claude 受管安装器，支持 `install/status/relink/uninstall`
- `agent-config-hub/claude/rules/git-commit-zh.md`
- `agent-config-hub/claude/rules/gitnexus-usage.md`
- `agent-config-hub/claude/rules/plan-naming-zh.md`
- `agent-config-hub/claude/rules/tdd-guideline-zh.md`
- `agent-config-hub/claude/settings.json`
  - 共享默认设置
- `agent-config-hub/claude/settings.local.example.json`
  - 本机模板，不含真实 secret
- `agent-config-hub/claude/commands/build-fix.md`
- `agent-config-hub/claude/commands/checkpoint.md`
- `agent-config-hub/claude/commands/code-review.md`
- `agent-config-hub/claude/commands/e2e.md`
- `agent-config-hub/claude/commands/eval.md`
- `agent-config-hub/claude/commands/evolve.md`
- `agent-config-hub/claude/commands/go-build.md`
- `agent-config-hub/claude/commands/go-review.md`
- `agent-config-hub/claude/commands/go-test.md`
- `agent-config-hub/claude/commands/instinct-export.md`
- `agent-config-hub/claude/commands/instinct-import.md`
- `agent-config-hub/claude/commands/instinct-status.md`
- `agent-config-hub/claude/commands/learn.md`
- `agent-config-hub/claude/commands/orchestrate.md`
- `agent-config-hub/claude/commands/plan.md`
- `agent-config-hub/claude/commands/refactor-clean.md`
- `agent-config-hub/claude/commands/setup-pm.md`
- `agent-config-hub/claude/commands/skill-create.md`
- `agent-config-hub/claude/commands/tdd.md`
- `agent-config-hub/claude/commands/test-coverage.md`
- `agent-config-hub/claude/commands/update-codemaps.md`
- `agent-config-hub/claude/commands/update-docs.md`
- `agent-config-hub/claude/commands/verify.md`
- `agent-config-hub/claude/agents/architect.md`
- `agent-config-hub/claude/agents/build-error-resolver.md`
- `agent-config-hub/claude/agents/code-reviewer.md`
- `agent-config-hub/claude/agents/database-reviewer.md`
- `agent-config-hub/claude/agents/doc-updater.md`
- `agent-config-hub/claude/agents/e2e-runner.md`
- `agent-config-hub/claude/agents/go-build-resolver.md`
- `agent-config-hub/claude/agents/go-reviewer.md`
- `agent-config-hub/claude/agents/planner.md`
- `agent-config-hub/claude/agents/refactor-cleaner.md`
- `agent-config-hub/claude/agents/security-reviewer.md`
- `agent-config-hub/claude/agents/tdd-guide.md`
- `agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs`
- `agent-config-hub/codex/README.md`
- `agent-config-hub/codex/placeholders/.gitkeep`
- `agent-config-hub/tests/claude/layout_test.sh`
- `agent-config-hub/tests/claude/skills_inventory_test.sh`
- `agent-config-hub/tests/claude/core_layer_test.sh`
- `agent-config-hub/tests/claude/managed_paths_test.sh`
- `agent-config-hub/tests/claude/install_smoke_test.sh`
- `agent-config-hub/tests/claude/agent_link_smoke_test.sh`

### Modify

- `.gitignore`
  - 新增 `.agent-hub/`
  - 新增 `agent-config-hub/claude/settings.local.json`
  - 保持 `.claude/` 忽略项，避免本地旧目录重新入仓
- `CLAUDE.md`
  - 改成薄项目入口，指向全局 `~/.claude` 与仓库内 `agent-config-hub`

### Local Runtime / No Commit

- `~/.claude/settings.local.json`
  - 安装时保留或生成
- `~/.claude/debug/`
- `~/.claude/todos/`
- `~/.claude/cache/`
- `~/.claude/backups/`
- `.agent-hub/claude/`
  - 项目本地状态目录，实施时会创建但必须被忽略

## Test Strategy

- 所有 Bash 工具与目录编排逻辑都先写 smoke test，再实现。
- 所有 smoke test 使用 `bash` + `mktemp -d` + 受控 `HOME`，避免污染真实环境。
- 使用 `rg` 做路径规则与 secrets 自检，不依赖额外 shell 测试框架。
- 每个任务完成后只运行受影响测试脚本，最终再跑一轮完整 smoke suite。
- 对 `gitnexus-hook.cjs` 使用 `node --check` 做语法验证。
- 对 `install.sh` 与 `agent-link` 使用 `bash -n` 做语法验证。

## Task 1: Scaffold The Managed Hub Layout

**Files:**
- Create: `agent-config-hub/shared/docs/migration-inventory.md`
- Create: `agent-config-hub/shared/docs/local-only-assets.md`
- Create: `agent-config-hub/shared/docs/config-layering.md`
- Create: `agent-config-hub/shared/docs/security-boundary.md`
- Create: `agent-config-hub/claude/README.md`
- Create: `agent-config-hub/codex/README.md`
- Create: `agent-config-hub/codex/placeholders/.gitkeep`
- Create: `agent-config-hub/tests/claude/layout_test.sh`
- Modify: `.gitignore`
- Test: `agent-config-hub/tests/claude/layout_test.sh`

- [ ] **Step 1: Write the failing layout smoke test**

Create `agent-config-hub/tests/claude/layout_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"

required_paths=(
  "$root/agent-config-hub/shared/docs/migration-inventory.md"
  "$root/agent-config-hub/shared/docs/local-only-assets.md"
  "$root/agent-config-hub/shared/docs/config-layering.md"
  "$root/agent-config-hub/shared/docs/security-boundary.md"
  "$root/agent-config-hub/claude/README.md"
  "$root/agent-config-hub/codex/README.md"
  "$root/agent-config-hub/codex/placeholders/.gitkeep"
)

for path in "${required_paths[@]}"; do
  [[ -e "$path" ]] || { echo "missing: $path" >&2; exit 1; }
done

grep -q '^\.agent-hub/$' "$root/.gitignore" || {
  echo ".gitignore missing .agent-hub/" >&2
  exit 1
}
```

- [ ] **Step 2: Run the layout test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/layout_test.sh
```

Expected: FAIL because `agent-config-hub/` and the required docs do not exist yet.

- [ ] **Step 3: Create the skeleton directories and baseline docs**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
mkdir -p \
  agent-config-hub/shared/docs \
  agent-config-hub/shared/templates \
  agent-config-hub/shared/scripts \
  agent-config-hub/shared/skills \
  agent-config-hub/claude/{rules,commands,agents,hooks} \
  agent-config-hub/codex/placeholders \
  agent-config-hub/tests/claude
```

Create the four shared docs as short, concrete inventories:

- `migration-inventory.md`: source path → target path table
- `local-only-assets.md`: `~/.claude` 私有态列表
- `config-layering.md`: `settings.json` / `settings.local.example.json` / `~/.claude/settings.local.json`
- `security-boundary.md`: secret、permissions、hooks 边界

Create `agent-config-hub/claude/README.md` with a short overview:

```md
# Claude Layer

This directory is the version-controlled Claude configuration source for the global agent-config-hub.

- `CLAUDE.md`: global Claude entry instructions
- `rules/`: stable behavior constraints
- `commands/`: Claude slash commands
- `agents/`: Claude agents
- `hooks/`: Claude hook scripts
- `settings.json`: shared default settings
- `settings.local.example.json`: local-only template
```

Create `agent-config-hub/codex/README.md`:

```md
# Codex Placeholder

Codex compatibility is intentionally out of scope for this phase.
This directory exists so the repository shape is stable before the Codex layer is implemented.
```

- [ ] **Step 4: Add the ignore rules for project-local runtime state**

Update `.gitignore` by appending:

```gitignore
.agent-hub/
agent-config-hub/claude/settings.local.json
```

- [ ] **Step 5: Re-run the layout test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/layout_test.sh
```

Expected: PASS.

- [ ] **Step 6: Commit the scaffold**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add .gitignore agent-config-hub docs/superpowers/plans/2026-03-22-agent-config-hub-Claude优先统一配置中心实施计划.md
git commit -m "chore: 初始化 agent-config-hub 骨架"
```

## Task 2: Migrate Shared Skills Into The Hub

**Files:**
- Create: `agent-config-hub/shared/skills/agent-builder/SKILL.md`
- Create: `agent-config-hub/shared/skills/agent-requirement-analyzer/SKILL.md`
- Create: `agent-config-hub/shared/skills/qiuzhi-skill-creator/SKILL.md`
- Create: `agent-config-hub/shared/skills/qiuzhi-skill-creator/LICENSE.txt`
- Create: `agent-config-hub/shared/templates/project-CLAUDE.md`
- Create: `agent-config-hub/tests/claude/skills_inventory_test.sh`
- Modify: `agent-config-hub/shared/docs/migration-inventory.md`
- Test: `agent-config-hub/tests/claude/skills_inventory_test.sh`

- [ ] **Step 1: Write the failing skills inventory test**

Create `agent-config-hub/tests/claude/skills_inventory_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"

required_files=(
  "$root/agent-config-hub/shared/skills/agent-builder/SKILL.md"
  "$root/agent-config-hub/shared/skills/agent-requirement-analyzer/SKILL.md"
  "$root/agent-config-hub/shared/skills/qiuzhi-skill-creator/SKILL.md"
  "$root/agent-config-hub/shared/skills/qiuzhi-skill-creator/LICENSE.txt"
  "$root/agent-config-hub/shared/templates/project-CLAUDE.md"
)

for path in "${required_files[@]}"; do
  [[ -f "$path" ]] || { echo "missing: $path" >&2; exit 1; }
done

find "$root/agent-config-hub/shared/skills" -mindepth 1 -maxdepth 2 -name SKILL.md | grep -q . || {
  echo "no SKILL.md found in shared skills" >&2
  exit 1
}
```

- [ ] **Step 2: Run the skills inventory test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/skills_inventory_test.sh
```

Expected: FAIL because the shared skill directories do not exist yet.

- [ ] **Step 3: Copy the existing repo-local Claude skills into `shared/skills`**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
mkdir -p agent-config-hub/shared/skills
cp -R .claude/skills/agent-builder agent-config-hub/shared/skills/
cp -R .claude/skills/agent-requirement-analyzer agent-config-hub/shared/skills/
cp -R .claude/skills/qiuzhi-skill-creator agent-config-hub/shared/skills/
```

- [ ] **Step 4: Add the project entry template used by `agent-link`**

Create `agent-config-hub/shared/templates/project-CLAUDE.md`:

```md
# Project Claude Entry

This project uses the globally managed Claude configuration from `~/.claude`.

Repository-managed source of truth:
- `agent-config-hub/claude/CLAUDE.md`
- `agent-config-hub/claude/rules/`
- `agent-config-hub/shared/skills/`

Project-local runtime state lives under `.agent-hub/claude/`.
Do not recreate `.claude/` as a tracked config source inside this repository.
```

- [ ] **Step 5: Update the migration inventory to record the skill moves**

Append rows like:

```md
| Source | Target | Notes |
| --- | --- | --- |
| `.claude/skills/agent-builder` | `agent-config-hub/shared/skills/agent-builder` | shared Claude skill |
| `.claude/skills/agent-requirement-analyzer` | `agent-config-hub/shared/skills/agent-requirement-analyzer` | shared Claude skill |
| `.claude/skills/qiuzhi-skill-creator` | `agent-config-hub/shared/skills/qiuzhi-skill-creator` | shared Claude skill |
```

- [ ] **Step 6: Re-run the skills inventory test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/skills_inventory_test.sh
```

Expected: PASS.

- [ ] **Step 7: Commit the shared skill migration**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add agent-config-hub/shared
git commit -m "feat: 迁移共享 Claude skills"
```

## Task 3: Build The Claude Core Layer

**Files:**
- Create: `agent-config-hub/claude/CLAUDE.md`
- Create: `agent-config-hub/claude/rules/git-commit-zh.md`
- Create: `agent-config-hub/claude/rules/gitnexus-usage.md`
- Create: `agent-config-hub/claude/rules/plan-naming-zh.md`
- Create: `agent-config-hub/claude/rules/tdd-guideline-zh.md`
- Create: `agent-config-hub/claude/settings.json`
- Create: `agent-config-hub/claude/settings.local.example.json`
- Create: `agent-config-hub/tests/claude/core_layer_test.sh`
- Modify: `CLAUDE.md`
- Test: `agent-config-hub/tests/claude/core_layer_test.sh`

- [ ] **Step 1: Write the failing core-layer test**

Create `agent-config-hub/tests/claude/core_layer_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"

test -f "$root/agent-config-hub/claude/CLAUDE.md"
test -f "$root/agent-config-hub/claude/settings.json"
test -f "$root/agent-config-hub/claude/settings.local.example.json"

for rule in git-commit-zh gitnexus-usage plan-naming-zh tdd-guideline-zh; do
  test -f "$root/agent-config-hub/claude/rules/${rule}.md"
done

! rg -n 'ANTHROPIC_AUTH_TOKEN|ANTHROPIC_BASE_URL' "$root/agent-config-hub/claude/settings.json"
rg -n 'agent-config-hub|~/.claude|\.agent-hub/claude' "$root/CLAUDE.md" > /dev/null
```

- [ ] **Step 2: Run the core-layer test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/core_layer_test.sh
```

Expected: FAIL because the Claude layer files do not exist yet.

- [ ] **Step 3: Copy the stable rules and rewrite the global Claude entry**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
cp .claude/rules/git-commit-zh.md agent-config-hub/claude/rules/
cp .claude/rules/gitnexus-usage.md agent-config-hub/claude/rules/
cp .claude/rules/plan-naming-zh.md agent-config-hub/claude/rules/
cp .claude/rules/tdd-guideline-zh.md agent-config-hub/claude/rules/
```

Create `agent-config-hub/claude/CLAUDE.md` as a short entry:

```md
# Claude Global Entry

## Start Here

- This directory is the version-controlled global Claude source managed by `agent-config-hub`.
- Shared reusable skills live in `~/.claude/skills` and map back to `agent-config-hub/shared/skills`.
- Project-local runtime state must live under `.agent-hub/claude/`, not project `.claude/`.

## Core Rules

- Follow TDD. Read `rules/tdd-guideline-zh.md`.
- Use Chinese for replies, comments, and commit messages. Read `rules/git-commit-zh.md`.
- Follow GitNexus guidance when understanding or changing indexed code. Read `rules/gitnexus-usage.md`.
- Follow plan naming guidance when writing implementation plans. Read `rules/plan-naming-zh.md`.
```

- [ ] **Step 4: Split shared settings from local-only secrets**

Create `agent-config-hub/claude/settings.json`:

```json
{
  "enabledPlugins": {
    "superpowers@superpowers-marketplace": true
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Grep|Glob|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/gitnexus/gitnexus-hook.cjs",
            "statusMessage": "Enriching with GitNexus graph context...",
            "timeout": 8000
          }
        ]
      }
    ]
  },
  "skipDangerousModePermissionPrompt": true
}
```

Create `agent-config-hub/claude/settings.local.example.json`:

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "replace-me",
    "ANTHROPIC_BASE_URL": "https://your-proxy.example.com"
  },
  "permissions": {
    "allow": []
  }
}
```

- [ ] **Step 5: Turn the repo root `CLAUDE.md` into a thin project entry**

Replace the current root `CLAUDE.md` with a short project-scoped note that keeps GitNexus context but points Claude to the global hub:

```md
# goto-software Claude Project Entry

- Global Claude configuration is managed via `~/.claude`, sourced from `agent-config-hub/claude/`.
- Shared skills are sourced from `agent-config-hub/shared/skills/`.
- Project-local runtime state must use `.agent-hub/claude/`.

## GitNexus

This repository is indexed by GitNexus as `goto-software`.
Use the existing GitNexus workflow when understanding indexed code or impact.
```

- [ ] **Step 6: Re-run the core-layer test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/core_layer_test.sh
```

Expected: PASS.

- [ ] **Step 7: Commit the Claude core layer**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add CLAUDE.md agent-config-hub/claude
git commit -m "feat: 建立 Claude 核心配置层"
```

## Task 4: Migrate Commands, Agents, And Hooks With Path Rewrites

**Files:**
- Create: `agent-config-hub/claude/commands/build-fix.md`
- Create: `agent-config-hub/claude/commands/checkpoint.md`
- Create: `agent-config-hub/claude/commands/code-review.md`
- Create: `agent-config-hub/claude/commands/e2e.md`
- Create: `agent-config-hub/claude/commands/eval.md`
- Create: `agent-config-hub/claude/commands/evolve.md`
- Create: `agent-config-hub/claude/commands/go-build.md`
- Create: `agent-config-hub/claude/commands/go-review.md`
- Create: `agent-config-hub/claude/commands/go-test.md`
- Create: `agent-config-hub/claude/commands/instinct-export.md`
- Create: `agent-config-hub/claude/commands/instinct-import.md`
- Create: `agent-config-hub/claude/commands/instinct-status.md`
- Create: `agent-config-hub/claude/commands/learn.md`
- Create: `agent-config-hub/claude/commands/orchestrate.md`
- Create: `agent-config-hub/claude/commands/plan.md`
- Create: `agent-config-hub/claude/commands/refactor-clean.md`
- Create: `agent-config-hub/claude/commands/setup-pm.md`
- Create: `agent-config-hub/claude/commands/skill-create.md`
- Create: `agent-config-hub/claude/commands/tdd.md`
- Create: `agent-config-hub/claude/commands/test-coverage.md`
- Create: `agent-config-hub/claude/commands/update-codemaps.md`
- Create: `agent-config-hub/claude/commands/update-docs.md`
- Create: `agent-config-hub/claude/commands/verify.md`
- Create: `agent-config-hub/claude/agents/architect.md`
- Create: `agent-config-hub/claude/agents/build-error-resolver.md`
- Create: `agent-config-hub/claude/agents/code-reviewer.md`
- Create: `agent-config-hub/claude/agents/database-reviewer.md`
- Create: `agent-config-hub/claude/agents/doc-updater.md`
- Create: `agent-config-hub/claude/agents/e2e-runner.md`
- Create: `agent-config-hub/claude/agents/go-build-resolver.md`
- Create: `agent-config-hub/claude/agents/go-reviewer.md`
- Create: `agent-config-hub/claude/agents/planner.md`
- Create: `agent-config-hub/claude/agents/refactor-cleaner.md`
- Create: `agent-config-hub/claude/agents/security-reviewer.md`
- Create: `agent-config-hub/claude/agents/tdd-guide.md`
- Create: `agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs`
- Create: `agent-config-hub/tests/claude/managed_paths_test.sh`
- Test: `agent-config-hub/tests/claude/managed_paths_test.sh`

- [ ] **Step 1: Write the failing managed-paths test**

Create `agent-config-hub/tests/claude/managed_paths_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"

test -f "$root/agent-config-hub/claude/commands/checkpoint.md"
test -f "$root/agent-config-hub/claude/commands/eval.md"
test -f "$root/agent-config-hub/claude/commands/setup-pm.md"
test -f "$root/agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs"

! rg -n '/home/gyq/.claude' "$root/agent-config-hub/claude"
! rg -n '\.claude/checkpoints\.log|\.claude/evals/|\.claude/package-manager\.json' \
  "$root/agent-config-hub/claude/commands"

rg -n '\.agent-hub/claude/checkpoints\.log' "$root/agent-config-hub/claude/commands/checkpoint.md" > /dev/null
rg -n '\.agent-hub/claude/evals/' "$root/agent-config-hub/claude/commands/eval.md" > /dev/null
rg -n '\.agent-hub/claude/package-manager\.json' "$root/agent-config-hub/claude/commands/setup-pm.md" > /dev/null
```

- [ ] **Step 2: Run the managed-paths test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/managed_paths_test.sh
```

Expected: FAIL because the command, agent, and hook files do not exist yet.

- [ ] **Step 3: Copy the current user-level commands, agents, and hook into the Claude layer**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
cp -R /home/gyq/.claude/commands/. agent-config-hub/claude/commands/
cp -R /home/gyq/.claude/agents/. agent-config-hub/claude/agents/
mkdir -p agent-config-hub/claude/hooks/gitnexus
cp /home/gyq/.claude/hooks/gitnexus/gitnexus-hook.cjs agent-config-hub/claude/hooks/gitnexus/
```

- [ ] **Step 4: Rewrite all project-local output paths from `.claude/` to `.agent-hub/claude/`**

Run the exact replacements below:

```bash
cd /mnt/f/IdeaProjects/goto-software
perl -0pi -e 's#\.claude/checkpoints\.log#\.agent-hub/claude/checkpoints.log#g' agent-config-hub/claude/commands/checkpoint.md
perl -0pi -e 's#\.claude/evals/#\.agent-hub/claude/evals/#g' agent-config-hub/claude/commands/eval.md
perl -0pi -e 's#\.claude/package-manager\.json#\.agent-hub/claude/package-manager.json#g' agent-config-hub/claude/commands/setup-pm.md
perl -0pi -e 's#/home/gyq/\.claude#~/.claude#g' \
  agent-config-hub/claude/commands/*.md \
  agent-config-hub/claude/agents/*.md \
  agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs
```

Do not rewrite runtime references like `~/.claude/skills/...`, `~/.claude/agents/...`, or `~/.claude/hooks/...`; those are valid post-install runtime paths.

- [ ] **Step 5: Validate the managed JS hook syntax**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
node --check agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs
```

Expected: no syntax errors.

- [ ] **Step 6: Re-run the managed-paths test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/managed_paths_test.sh
```

Expected: PASS.

- [ ] **Step 7: Commit the migrated commands, agents, and hooks**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add agent-config-hub/claude/commands agent-config-hub/claude/agents agent-config-hub/claude/hooks
git commit -m "refactor: 迁移 Claude commands agents hooks"
```

## Task 5: Implement The Claude Installer

**Files:**
- Create: `agent-config-hub/claude/install.sh`
- Create: `agent-config-hub/tests/claude/install_smoke_test.sh`
- Modify: `agent-config-hub/claude/README.md`
- Test: `agent-config-hub/tests/claude/install_smoke_test.sh`

- [ ] **Step 1: Write the failing installer smoke test**

Create `agent-config-hub/tests/claude/install_smoke_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"
tmp_home="$(mktemp -d)"
tmp_bashrc="$tmp_home/.bashrc"
mkdir -p "$tmp_home/.claude"
echo legacy > "$tmp_home/.claude/legacy.txt"
: > "$tmp_bashrc"

HOME="$tmp_home" bash "$root/agent-config-hub/claude/install.sh" install --repo "$root/agent-config-hub" --shell bash

test -L "$tmp_home/.claude/CLAUDE.md"
test -L "$tmp_home/.claude/rules"
test -L "$tmp_home/.claude/skills"
test -L "$tmp_home/.claude/commands"
test -L "$tmp_home/.claude/agents"
test -L "$tmp_home/.claude/hooks"
test -f "$tmp_home/.claude/settings.local.json"
test -d "$tmp_home/.claude/debug"
test -d "$tmp_home/.claude/todos"
test -d "$tmp_home/.claude/cache"
test -d "$tmp_home/.claude/backups"
test -d "$tmp_home/.claude.backups"
rg -n 'AGENT_CONFIG_HUB=' "$tmp_bashrc" > /dev/null
```

- [ ] **Step 2: Run the installer smoke test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/install_smoke_test.sh
```

Expected: FAIL because `install.sh` does not exist yet.

- [ ] **Step 3: Implement `install.sh` with install/status/relink/uninstall**

Create `agent-config-hub/claude/install.sh` with these behaviors:

- `install`
  - resolve repo root from `--repo` or script parent
  - default backup old `~/.claude` into `~/.claude.backups/<timestamp>/`
  - create real `~/.claude`
  - symlink managed files and directories from the repo
  - create `settings.local.json` from template if absent
  - create `debug/todos/cache/backups`
  - inject a marked shell block into `~/.bashrc` or `~/.zshrc`
- `status`
  - verify required symlinks and local files exist
- `relink`
  - rebuild only managed symlinks
- `uninstall`
  - remove managed symlinks and restore latest backup if present

Use this shell block format:

```bash
# >>> agent-config-hub >>>
export AGENT_CONFIG_HUB="/absolute/path/to/agent-config-hub"
export AGENT_CONFIG_HUB_CLAUDE="$AGENT_CONFIG_HUB/claude"
export PATH="$AGENT_CONFIG_HUB/shared/scripts:$PATH"
# <<< agent-config-hub <<<
```

- [ ] **Step 4: Validate the installer script syntax**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash -n agent-config-hub/claude/install.sh
```

Expected: no syntax errors.

- [ ] **Step 5: Re-run the installer smoke test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/install_smoke_test.sh
```

Expected: PASS.

- [ ] **Step 6: Document the installer usage in the Claude README**

Add a short usage section to `agent-config-hub/claude/README.md`:

```md
## Install

- `./install.sh install`
- `./install.sh status`
- `./install.sh relink`
- `./install.sh uninstall`

Backups are written to `~/.claude.backups/` before a managed install unless `--no-backup` is used.
```

- [ ] **Step 7: Commit the installer**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add agent-config-hub/claude/install.sh agent-config-hub/claude/README.md agent-config-hub/tests/claude/install_smoke_test.sh
git commit -m "feat: 添加 Claude 全局安装脚本"
```

## Task 6: Implement `agent-link` For Project Onboarding

**Files:**
- Create: `agent-config-hub/shared/scripts/agent-link`
- Create: `agent-config-hub/tests/claude/agent_link_smoke_test.sh`
- Modify: `agent-config-hub/shared/templates/project-CLAUDE.md`
- Test: `agent-config-hub/tests/claude/agent_link_smoke_test.sh`

- [ ] **Step 1: Write the failing `agent-link` smoke test**

Create `agent-config-hub/tests/claude/agent_link_smoke_test.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

root="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../.." && pwd)"
tmp_project="$(mktemp -d)"
: > "$tmp_project/.gitignore"

AGENT_CONFIG_HUB="$root/agent-config-hub" \
  bash "$root/agent-config-hub/shared/scripts/agent-link" init-project --project "$tmp_project"

test -f "$tmp_project/CLAUDE.md"
test -d "$tmp_project/.agent-hub/claude"
rg -n '^\.agent-hub/$' "$tmp_project/.gitignore" > /dev/null

AGENT_CONFIG_HUB="$root/agent-config-hub" \
  bash "$root/agent-config-hub/shared/scripts/agent-link" doctor --project "$tmp_project"
```

- [ ] **Step 2: Run the `agent-link` test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
```

Expected: FAIL because `agent-link` does not exist yet.

- [ ] **Step 3: Implement `agent-link` with `init-project` and `doctor`**

Create `agent-config-hub/shared/scripts/agent-link` with this behavior:

- `init-project`
  - resolve target project root from `--project` or current working directory
  - create `.agent-hub/claude/`
  - ensure `.gitignore` contains `.agent-hub/`
  - copy `shared/templates/project-CLAUDE.md` to `<project>/CLAUDE.md` if missing
- `doctor`
  - fail if `CLAUDE.md` missing
  - fail if `.agent-hub/claude/` missing
  - fail if `.gitignore` does not ignore `.agent-hub/`

Use this script header:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- [ ] **Step 4: Update the template so the generated project entry is explicit**

Make sure `shared/templates/project-CLAUDE.md` contains all three lines:

```md
- Global Claude configuration is managed via `~/.claude`.
- Source-of-truth files live in `agent-config-hub/`.
- Project-local runtime state belongs in `.agent-hub/claude/`.
```

- [ ] **Step 5: Validate the `agent-link` script syntax and rerun the smoke test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash -n agent-config-hub/shared/scripts/agent-link
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
```

Expected: PASS.

- [ ] **Step 6: Commit the project onboarding tool**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add agent-config-hub/shared/scripts/agent-link agent-config-hub/shared/templates/project-CLAUDE.md agent-config-hub/tests/claude/agent_link_smoke_test.sh
git commit -m "feat: 添加 agent-link 项目接入工具"
```

## Task 7: Perform Final Verification And Switch The Current Machine

**Files:**
- Modify: local runtime only `~/.claude/`
- Modify: local runtime only `.agent-hub/claude/`
- Test: `agent-config-hub/tests/claude/layout_test.sh`
- Test: `agent-config-hub/tests/claude/skills_inventory_test.sh`
- Test: `agent-config-hub/tests/claude/core_layer_test.sh`
- Test: `agent-config-hub/tests/claude/managed_paths_test.sh`
- Test: `agent-config-hub/tests/claude/install_smoke_test.sh`
- Test: `agent-config-hub/tests/claude/agent_link_smoke_test.sh`

- [ ] **Step 1: Run the full repo-managed verification suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
bash agent-config-hub/tests/claude/layout_test.sh
bash agent-config-hub/tests/claude/skills_inventory_test.sh
bash agent-config-hub/tests/claude/core_layer_test.sh
bash agent-config-hub/tests/claude/managed_paths_test.sh
bash agent-config-hub/tests/claude/install_smoke_test.sh
bash agent-config-hub/tests/claude/agent_link_smoke_test.sh
bash -n agent-config-hub/claude/install.sh
bash -n agent-config-hub/shared/scripts/agent-link
node --check agent-config-hub/claude/hooks/gitnexus/gitnexus-hook.cjs
```

Expected: all PASS, no shell syntax errors, no Node syntax errors.

- [ ] **Step 2: Confirm there are no banned hard-coded Claude home paths in the managed repo content**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
! rg -n '/home/gyq/.claude' agent-config-hub
```

Expected: no output, command exits success.

- [ ] **Step 3: Run a dry-run install against the real workspace**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/agent-config-hub/claude
./install.sh install --repo /mnt/f/IdeaProjects/goto-software/agent-config-hub --dry-run --shell bash
```

Expected: shows planned backup path, planned symlinks, and shell block insertion target, but does not modify `~/.claude`.

- [ ] **Step 4: Execute the real install with backup enabled**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/agent-config-hub/claude
./install.sh install --repo /mnt/f/IdeaProjects/goto-software/agent-config-hub --shell bash
```

Expected:

- old `~/.claude` is moved to `~/.claude.backups/<timestamp>/`
- new `~/.claude` contains managed symlinks plus local runtime directories
- `~/.claude/settings.local.json` exists

- [ ] **Step 5: Verify the installed global Claude directory**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/agent-config-hub/claude
ls -l ~/.claude
test -L ~/.claude/CLAUDE.md
test -L ~/.claude/rules
test -L ~/.claude/skills
test -L ~/.claude/commands
test -L ~/.claude/agents
test -L ~/.claude/hooks
test -f ~/.claude/settings.local.json
test -d ~/.claude/debug
test -d ~/.claude/todos
test -d ~/.claude/cache
test -d ~/.claude/backups
./install.sh status
```

Expected: all checks pass and `status` reports healthy managed links.

- [ ] **Step 6: Retire the old project-local `.claude/` source so it no longer competes with the hub**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
mkdir -p .claude.bak
mv .claude ".claude.bak/project-local-claude-$(date +%Y%m%d-%H%M%S)"
mkdir -p .agent-hub/claude
```

Expected: project root no longer has a live `.claude/` config source; local project runtime state now has a dedicated `.agent-hub/claude/` directory.

- [ ] **Step 7: Record the final repo state and stop before any extra feature work**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git status --short
```

Expected:

- repo-managed changes are already committed from Tasks 1-6
- only ignored local runtime paths may remain (`.agent-hub/`, `.claude.bak/`)

At this point the implementation is complete, the machine is switched to the new Hub, and no additional Codex work should be started in this plan.
