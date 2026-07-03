# superpowers 分支收口 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `hybrid-subagent-driven-development` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 按已批准 spec 收口 `/mnt/g/Obsidian` 中的 `feat/002-deepseek-document-review-loop` 和 `refactor/001-agent-rules-lazy-loading`，并保持文档仓库、worktree、最终报告边界清晰。

**Architecture:** 当前 Codex 会话是 controller，负责用户确认、分支切换、最终合并、审查和最终回复。DeepSeek 只通过 `/home/gyq/.local/bin/codex-dp-exec` 在临时集成 worktree 中执行 `refactor/001` 冲突解决任务；DeepSeek 不依赖会话历史，每个 prompt 都必须携带完整上下文。

**Tech Stack:** Git/worktree、Bash、ripgrep、`codex-dp-exec`、DeepSeek reviewer/implementer、superpowers skill 文档与 Agent 规则 Markdown。

---

## 执行前上下文

- Approved spec: `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs/_global/superpowers/specs/2026-07-03-superpowers分支收口设计.md`
- Plan path: `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs/_global/superpowers/plans/2026-07-03-superpowers分支收口实施计划.md`
- `/mnt/g/Obsidian` 是实际 Git 顶层仓库，`skill/superpowers` 只是子目录。
- 当前未合并分支：
  - `feat/002-deepseek-document-review-loop`
  - `refactor/001-agent-rules-lazy-loading`
- 当前 `/mnt/g/Obsidian` worktree 位于 `feat/002-deepseek-document-review-loop`，存在未跟踪文件：
  - `backup/chrome-tab-manager-2026-07-03.json`
  - `skill/superpowers/final-review-handoff.md`
- `main` 设计时基准提交：`d0edf8b229d7a5b43d9b266d92ca8fee368f5412`
- `feat/002` 设计时 HEAD：`2a042746fa61fe2c0e77d316afd798c8ff878318`
- `refactor/001` 设计时 HEAD：`6af30db15c782f97a320effa396b028540090934`
- `refactor/001` 与 `main` 的 merge-base：`05a61e7695d9aef96cc16643e5c36db9d5c6f881`
- 全程不执行 `git push`，不修改 Git remote，不整理历史未跟踪文件。
- `goto-superpowers-docs` 当前有既有历史未跟踪文件；本计划只允许提交本 plan 文件和后续由本计划明确修改的文档文件。

## Hybrid Subagent-Driven 执行约定

Controller 在首次 DeepSeek 实施任务前必须运行：

```bash
/home/gyq/.local/bin/codex-dp-exec doctor --cd <task-worktree>
```

DeepSeek 实施任务命令模板：

```bash
/home/gyq/.local/bin/codex-dp-exec \
  --cd <task-worktree> \
  --prompt-file <prompt-file> \
  --task-id <task-id> \
  --controller-id codex1 \
  --output-last-message <report-file>

/home/gyq/.local/bin/codex-dp-exec status \
  --cd <task-worktree> \
  --task-id <task-id>
```

恢复或审计多任务时必须运行：

```bash
/home/gyq/.local/bin/codex-dp-exec list --cd <task-worktree>
/home/gyq/.local/bin/codex-dp-exec status --cd <task-worktree> --task-id <task-id>
```

Controller 每个 DeepSeek run 后必须检查：

- `codex-dp-exec status`
- implementer report
- `git diff`
- changed files
- test output
- spec compliance first；只有 spec compliance 通过后，才进入 code quality review

任务完成定义：spec compliance review 与 code quality review 均通过，并且本任务要求的验证命令已实际运行。

Recovery rules:

- 如果 run 是 `interrupted`，重试前先检查当前 `git diff`。
- 恢复时先运行 `codex-dp-exec list --cd <worktree>` 和 `codex-dp-exec status --cd <worktree> --task-id <task-id>`。
- 恢复同一 `task_id` 时创建新 run，不覆盖中断 run。
- 如果 DeepSeek 返回 `BLOCKED` 或 `NEEDS_CONTEXT`，controller 决定补充上下文、拆分任务、重试或升级给用户。
- 临时 prompt、report、run 目录不得提交，除非用户明确要求。
- 不得执行 `mvn deploy`。
- 不得通过数据库客户端执行写操作。
- 不得回退用户或其他 agent 的无关改动。

## 最终审核策略与交接包

```text
final_review_policy: current-controller
final_review_status_on_completion: final-review-completed
reviewer_required: none
handoff_package: /tmp/superpowers-branch-closeout-final-review-handoff.md
```

说明：

- 最终报告是 controller 任务完成时发给用户的最终回复，不默认落盘为仓库文件。
- `/tmp/superpowers-branch-closeout-final-review-handoff.md` 是运行期证据包，不是最终报告，不提交到任何仓库。
- 不使用当前未跟踪的 `/mnt/g/Obsidian/skill/superpowers/final-review-handoff.md` 作为验收证据。
- 如果 `codex-dp-exec handoff` 仍无法过滤历史 run，controller 手工生成 `/tmp` 证据包，内容必须包含 spec path、plan path、各仓库 HEAD/status、任务运行状态、验证命令结果、worktree 清理状态和 open concerns。

## 文件结构

预计涉及文件与目录：

```text
/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs/_global/superpowers/specs/2026-07-03-superpowers分支收口设计.md
/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs/_global/superpowers/plans/2026-07-03-superpowers分支收口实施计划.md

/mnt/g/Obsidian/backup/chrome-tab-manager-2026-07-03.json
/mnt/g/Obsidian/skill/superpowers/final-review-handoff.md
/mnt/g/Obsidian/skill/superpowers/scripts/codex-dp-exec
/mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-spec
/mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-plan
/mnt/g/Obsidian/code/01_System_Core/global_rules.md
/mnt/g/Obsidian/code/01_System_Core/rules/
/mnt/g/Obsidian/code/01_Project_Specs/hsmap/
/mnt/g/Obsidian/code/01_Project_Specs/goto-software/
/mnt/g/Obsidian/code/tools/validate-agent-rules.sh
/mnt/g/Obsidian/skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md

/mnt/f/IdeaProjects/goto-software/tmp/superpowers-branch-closeout/
/mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout/
```

## Task 0: 工作区核查

- [ ] task_id: `task-000-workspace-audit`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/g/Obsidian`
- [ ] Branch provenance:

| project | main_project_path | base_branch | base_commit | feature_branch | worktree_path | creation_status |
| --- | --- | --- | --- | --- | --- | --- |
| Obsidian/superpowers feat | `/mnt/g/Obsidian` | `main` | `d0edf8b229d7a5b43d9b266d92ca8fee368f5412` | `feat/002-deepseek-document-review-loop` | `/mnt/g/Obsidian` | `reused-dirty-needs-controller` |
| Obsidian/agent-rules refactor | `/mnt/g/Obsidian` | `main` | `d0edf8b229d7a5b43d9b266d92ca8fee368f5412` | `refactor/001-agent-rules-lazy-loading` | `/mnt/g/Obsidian-worktrees/refactor/001-agent-rules-lazy-loading` | `reused-clean` |
| Obsidian/refactor integration | `/mnt/g/Obsidian` | `main` | `由 Task 3 在 feat/002 合并后的 main HEAD 记录` | `integration/003-refactor-001-closeout` | `/mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout` | `created` |
| goto-superpowers-docs | `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs` | `main` | `f4822a6` | `main` | `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs` | `reused-dirty-needs-controller` |

Required steps:

1. 记录当前状态：

```bash
git -C /mnt/g/Obsidian status --short --branch
git -C /mnt/g/Obsidian branch --no-merged main
git -C /mnt/g/Obsidian worktree list --porcelain
git -C /mnt/g/Obsidian rev-parse main
git -C /mnt/g/Obsidian rev-parse feat/002-deepseek-document-review-loop
git -C /mnt/g/Obsidian rev-parse refactor/001-agent-rules-lazy-loading
git -C /mnt/g/Obsidian merge-base main refactor/001-agent-rules-lazy-loading
```

2. 验证 `goto-superpowers-docs` 中本 spec 与本 plan 已提交；历史未跟踪文件只记录，不整理：

```bash
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs status --short --branch
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs log --oneline -5
```

3. 验证 DeepSeek 脚本软链接存在、目标路径和可执行权限：

```bash
ls -la /home/gyq/.local/bin/codex-dp-exec /home/gyq/.local/bin/deepseek-review-spec /home/gyq/.local/bin/deepseek-review-plan
readlink -f /home/gyq/.local/bin/codex-dp-exec
readlink -f /home/gyq/.local/bin/deepseek-review-spec
readlink -f /home/gyq/.local/bin/deepseek-review-plan
```

4. 如果分支 HEAD、未合并分支列表或 worktree 状态与计划不一致，停止并向用户报告差异；不要猜测继续。

Expected result:

- 已记录所有分支、worktree、dirty/untracked 状态。
- 确认本 plan 已提交，或先提交本 plan 后再进入 `/mnt/g/Obsidian` 分支操作。

## Task 1: 提交计划并保护文档仓库边界

- [ ] task_id: `task-001-docs-plan-commit`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs`
- [ ] Editable files: this plan only, unless plan review requires edits.
- [ ] Forbidden changes: 不整理历史未跟踪文件；不 push；不修改 remote。

Required steps:

1. 如果本 plan 还未提交，只暂存并提交本文件：

```bash
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs add _global/superpowers/plans/2026-07-03-superpowers分支收口实施计划.md
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs commit -m "docs: 增加superpowers分支收口实施计划"
```

2. 记录状态：

```bash
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs status --short --branch
git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs show --stat --oneline --no-renames HEAD
```

Expected result:

- 本 plan 文件已提交。
- `goto-superpowers-docs` 可能仍显示既有未跟踪文件；最终报告必须明确它们不是本任务新增整理对象。

## Task 2: 收口 feat/002 并暂停等待用户确认

- [ ] task_id: `task-002-close-feat-002`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/g/Obsidian`
- [ ] Editable files:
  - `/mnt/g/Obsidian/backup/chrome-tab-manager-2026-07-03.json` only for moving out of Git worktree
  - `/mnt/g/Obsidian/skill/superpowers/final-review-handoff.md` only for moving out of Git worktree
  - Git branch state for local merge
- [ ] Forbidden changes:
  - 不提交 `backup/chrome-tab-manager-2026-07-03.json`
  - 不提交 `/mnt/g/Obsidian/skill/superpowers/final-review-handoff.md`
  - 不修复 `codex-dp-exec handoff`
  - 不 push
  - 不修改 `feat/002` 已提交内容，除非 verification 发现真实问题并先获得用户确认

Required steps:

1. 记录未跟踪文件详情：

```bash
git -C /mnt/g/Obsidian status --porcelain=v1 --untracked-files=all
ls -la /mnt/g/Obsidian/backup/chrome-tab-manager-2026-07-03.json /mnt/g/Obsidian/skill/superpowers/final-review-handoff.md
```

2. 使用保留数据的方式移出 Git 工作区，不直接删除：

```bash
mkdir -p /mnt/f/IdeaProjects/goto-software/tmp/superpowers-branch-closeout
mv /mnt/g/Obsidian/backup/chrome-tab-manager-2026-07-03.json /mnt/f/IdeaProjects/goto-software/tmp/superpowers-branch-closeout/
mv /mnt/g/Obsidian/skill/superpowers/final-review-handoff.md /mnt/f/IdeaProjects/goto-software/tmp/superpowers-branch-closeout/
```

3. 验证 `feat/002`：

```bash
git -C /mnt/g/Obsidian status --porcelain=v1 --untracked-files=all
git -C /mnt/g/Obsidian log --oneline main..feat/002-deepseek-document-review-loop
git -C /mnt/g/Obsidian diff --stat main..feat/002-deepseek-document-review-loop
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/codex-dp-exec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-spec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-plan
/home/gyq/.local/bin/deepseek-review-spec --help
/home/gyq/.local/bin/deepseek-review-plan --help
```

4. 本地合并 `feat/002`：

```bash
git -C /mnt/g/Obsidian switch main
git -C /mnt/g/Obsidian merge --no-ff feat/002-deepseek-document-review-loop
git -C /mnt/g/Obsidian status --short --branch
git -C /mnt/g/Obsidian branch --no-merged main
```

5. 暂停并向用户报告：
   - `feat/002` merge commit 或 fast-forward 结果。
   - 当前 `main` HEAD。
   - 剩余未合并分支。
   - `/mnt/g/Obsidian` status。
   - 两个未跟踪文件被移动到的路径。
   - 明确询问是否继续进入 `refactor/001` 冲突评估与临时集成。

Acceptance criteria:

- `feat/002-deepseek-document-review-loop` 已本地合并到 `main`。
- `/mnt/g/Obsidian` 位于 `main`。
- 没有未提交的 `feat/002` 相关文件。
- 用户确认后才能继续 Task 3。

## Task 3: 创建 refactor/001 临时集成 worktree

- [ ] task_id: `task-003-create-refactor-integration-worktree`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/g/Obsidian`
- [ ] Editable files: Git branch/worktree metadata only.
- [ ] Forbidden changes:
  - 不直接在 `/mnt/g/Obsidian` 的 `main` 上解决冲突。
  - 不 push。
  - 不删除既有 `refactor/001` worktree。

Required steps:

1. 重新盘点分支和 worktree：

```bash
git -C /mnt/g/Obsidian status --short --branch
git -C /mnt/g/Obsidian branch --list
git -C /mnt/g/Obsidian branch --no-merged main
git -C /mnt/g/Obsidian worktree list --porcelain
```

2. 只读冲突预检：

```bash
git -C /mnt/g/Obsidian merge-tree main refactor/001-agent-rules-lazy-loading
```

3. 重新扫描本地分支、远程分支和 worktree 目录的最大三位序号。如果最大序号仍小于 `003`，创建：

```bash
git -C /mnt/g/Obsidian switch main
git -C /mnt/g/Obsidian worktree add -b integration/003-refactor-001-closeout /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout main
```

如果已有更高序号，停止并向用户报告序号冲突，请用户确认新的临时分支名；不要复用已存在或有历史含义的序号。

4. 在临时 worktree 中确认环境：

```bash
git -C /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout status --short --branch
/home/gyq/.local/bin/codex-dp-exec doctor --cd /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout
```

Acceptance criteria:

- 临时集成 worktree clean。
- 临时集成分支从合并 `feat/002` 后的 `main` 创建。
- `codex-dp-exec doctor` 可用；如果失败，先解决工具环境问题或停止。

## Task 4: 解决 refactor/001 冲突并验证规则完整性

- [ ] task_id: `task-004-resolve-refactor-001-conflicts`
- [ ] Execution mode: `DeepSeek implementer + controller重点复核`
- [ ] `--cd`: `/mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout`
- [ ] Editable files:
  - `code/01_System_Core/global_rules.md`
  - `code/01_System_Core/rules/**`
  - `code/01_Project_Specs/hsmap/**`
  - `code/01_Project_Specs/goto-software/**`
  - `code/tools/validate-agent-rules.sh`
  - `skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md`
- [ ] Forbidden changes:
  - 不修改 `skill/superpowers/scripts/codex-dp-exec`
  - 不修改 `skill/superpowers/scripts/deepseek-review-spec`
  - 不修改 `skill/superpowers/scripts/deepseek-review-plan`
  - 不修改 `/home/gyq/.local/bin/*`
  - 不删除全局硬约束
  - 不 push
  - 不整理与本 task 无关的未跟踪文件

Why This Task Exists:

`refactor/001` 拆分 Agent 规则并增加校验脚本，但基于旧 `main`，合并后会在 `code/01_System_Core/global_rules.md` 冲突。该任务在临时集成 worktree 中保留规则拆分收益，同时确保当前硬约束和 `feat/002` 工具链不丢失。

Existing Context:

- Approved spec 要求先合 `feat/002`，再处理 `refactor/001`。
- `global_rules.md` 必须保留中文输出、WSL 预览地址、数据库只读、中文计划文件名、中文 commit、Git 远端用 `git.exe`、worktree 序号、Maven 禁止 deploy、Windows Maven 调用、Java 验证流程、Java 测试注释、生成代码中文注释。
- `hybrid-subagent-driven-development/SKILL.md` 不可盲目接受删除 finishing/本地合并约束的改动；必须保留“本地合并、不自动 push”的约束。
- DeepSeek 必须在报告中列出具体冲突文件、解决策略、验证结果和 commit。

Required implementation steps:

1. 确认 worktree 位于临时集成分支且 clean：

```bash
git status --short --branch
git rev-parse --abbrev-ref HEAD
```

2. 合并 `refactor/001` 到临时集成分支：

```bash
git merge --no-ff refactor/001-agent-rules-lazy-loading
```

3. 如果发生冲突，只解决允许文件范围内的冲突。

   `code/01_System_Core/global_rules.md` 是规则源文件冲突，禁止简单使用 `ours` 或 `theirs` 整文件覆盖。必须逐条核对当前 `main` 硬约束和 `refactor/001` 规则拆分结构，确认两边有效内容都被保留或明确改写。

4. 专门复核 `hybrid-subagent-driven-development/SKILL.md` 差异：

```bash
git diff main...refactor/001-agent-rules-lazy-loading -- skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md
```

5. 验证规则完整性：

```bash
bash -n code/tools/validate-agent-rules.sh
code/tools/validate-agent-rules.sh
rg -n "git.exe|mvn.cmd|deploy|数据库|127.0.0.1|中文|worktree|push" code/01_System_Core code/01_Project_Specs
rg -n "本地合并|git push|远端|finishing-a-development-branch" skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md
```

6. 提交结果：

```bash
git status --short
git add code/01_System_Core code/01_Project_Specs code/tools/validate-agent-rules.sh skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md
git commit -m "docs: 收口Agent规则拆分冲突"
```

Self Review Before Final Report:

- 确认没有修改 forbidden files。
- 确认全局硬约束仍可被 `rg` 找到。
- 确认 `hybrid-subagent-driven-development` 仍包含本地合并/不自动 push 约束。
- 确认 `git status --short` clean。

最终报告必须包含：

- status: `DONE` | `DONE_WITH_CONCERNS` | `BLOCKED` | `NEEDS_CONTEXT`
- changed_files: 本任务改动的文件列表
- commits: 本任务产生的 commit hash；没有提交时说明原因
- tests: 实际运行的测试命令和结果
- concerns: 风险、未验证项或需要 controller 判断的问题

执行要求：

- 只能修改本 Task 声明的仓库和文件范围。
- 不得回退用户或其他 agent 的无关改动。
- 不得执行 `mvn deploy`。
- 不得通过数据库客户端执行写操作。
- 如果计划与代码现状冲突，停止并返回 `NEEDS_CONTEXT`，不要猜测。

## Task 5: Controller 复核、用户确认并合并临时集成分支

- [ ] task_id: `task-005-review-and-merge-integration`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/g/Obsidian`
- [ ] Editable files: Git branch/worktree metadata only, unless review fix is required.
- [ ] Forbidden changes:
  - 不在未获得用户确认前合并临时集成分支回 `main`。
  - 不 push。
  - 不直接修改 DeepSeek task 结果；如需修复，发 self-contained fix prompt 或 controller 说明后小修并验证。

Required steps:

1. Controller review spec compliance:

```bash
git -C /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout status --short --branch
git -C /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout show --stat --oneline --no-renames HEAD
git -C /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout diff --stat main..HEAD
```

2. 复跑 Task 4 验证命令。

3. 如果发现问题，构造 fix prompt，必须包含：
   - Original Task: `task-004-resolve-refactor-001-conflicts`
   - Current State: diff summary、changed files、latest commit、report status、tests run
   - Review Findings To Fix: spec compliance findings 和 code quality findings 分开列出
   - Allowed Fix Scope: Task 4 editable files
   - Required Tests: Task 4 验证命令
   - Self Review Before Final Report

4. 临时集成 worktree 验证通过后，暂停并向用户报告：
   - 临时集成分支名与 HEAD。
   - 变更摘要。
   - 验证命令结果。
   - 是否建议合并回 `main`。
   - 明确询问是否合并回 `main`。

5. 只有用户确认后，执行本地合并：

```bash
git -C /mnt/g/Obsidian switch main
git -C /mnt/g/Obsidian merge --no-ff integration/003-refactor-001-closeout
git -C /mnt/g/Obsidian status --short --branch
```

Acceptance criteria:

- 用户确认后才将集成分支合并回 `main`。
- `main` 包含 `feat/002` 和已批准的 `refactor/001` 收口结果。
- 不执行 `git push`。

## Task 6: 最终验证、worktree 清理与最终回复

- [ ] task_id: `task-006-final-verification`
- [ ] Execution mode: `controller inline`
- [ ] `--cd`: `/mnt/g/Obsidian`
- [ ] Editable files:
  - Git worktree metadata for removing clean temporary integration worktree
  - `/tmp/superpowers-branch-closeout-final-review-handoff.md`
- [ ] Forbidden changes:
  - 不把 `/tmp` handoff package 提交到仓库。
  - 不生成仓库内最终报告文件，除非用户明确要求。
  - 不 push。

Required verification:

```bash
git -C /mnt/g/Obsidian status --short --branch
git -C /mnt/g/Obsidian status --porcelain=v1 --untracked-files=all
git -C /mnt/g/Obsidian branch --no-merged main
git -C /mnt/g/Obsidian worktree list --porcelain
git -C /mnt/g/Obsidian rev-parse HEAD

bash -n /mnt/g/Obsidian/skill/superpowers/scripts/codex-dp-exec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-spec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-plan
/home/gyq/.local/bin/deepseek-review-spec --help
/home/gyq/.local/bin/deepseek-review-plan --help

ls -la /home/gyq/.local/bin/codex-dp-exec /home/gyq/.local/bin/deepseek-review-spec /home/gyq/.local/bin/deepseek-review-plan
readlink -f /home/gyq/.local/bin/codex-dp-exec
readlink -f /home/gyq/.local/bin/deepseek-review-spec
readlink -f /home/gyq/.local/bin/deepseek-review-plan

bash -n /mnt/g/Obsidian/code/tools/validate-agent-rules.sh
/mnt/g/Obsidian/code/tools/validate-agent-rules.sh
rg -n "git.exe|mvn.cmd|deploy|数据库|127.0.0.1|中文|worktree|push" /mnt/g/Obsidian/code/01_System_Core /mnt/g/Obsidian/code/01_Project_Specs
rg -n "本地合并|git push|远端|finishing-a-development-branch" /mnt/g/Obsidian/skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md

git -C /mnt/f/IdeaProjects/goto-software/goto-superpowers-docs status --short --branch
```

Worktree cleanup:

1. 检查旧 refactor worktree：

```bash
git -C /mnt/g/Obsidian-worktrees/refactor/001-agent-rules-lazy-loading status --short --branch
```

2. 检查临时集成 worktree：

```bash
git -C /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout status --short --branch
```

3. 如果临时集成 worktree clean 且合并结果已进入 `main`，删除临时 worktree：

```bash
git -C /mnt/g/Obsidian worktree remove /mnt/g/Obsidian-worktrees/integration/003-refactor-001-closeout
```

旧 `refactor/001` worktree 是否删除由用户决定；如果保留，最终回复说明原因。

Handoff package:

- 推荐手工写入 `/tmp/superpowers-branch-closeout-final-review-handoff.md`，不得提交。
- 必须包含：controller id/model、final review policy/status、spec path、plan path、每个 affected repository 的 path/base branch/base commit/feature branch/worktree/current HEAD/clean 状态、每个 task 的 status/report/commits/tests、最终验证结果、worktree 清理状态、open concerns。

Final response must include:

- 合并结果。
- `/mnt/g/Obsidian` 最终 `HEAD`。
- 仍未合并或保留的分支及原因。
- worktree 清理结果。
- 关键验证命令结果。
- 未跟踪文件处理结果。
- `goto-superpowers-docs` 本次文件提交状态和既有未跟踪文件说明。
- 明确确认未执行 `git push`。

## Hybrid readiness checklist

- [ ] Approved spec path 已写入计划。
- [ ] Plan path 符合 `goto-superpowers-docs/_global/superpowers/plans` 规范。
- [ ] 每个 task 有稳定 ASCII `task_id`。
- [ ] 每个 task 有执行模式和 `--cd` 路径。
- [ ] DeepSeek task 自包含上下文、editable files、forbidden changes、required tests、final report format。
- [ ] Task 0 有 branch provenance table。
- [ ] 恢复规则、controller review gates、fix prompt 要求已写入。
- [ ] `feat/002` 合并后有用户确认门。
- [ ] `refactor/001` 临时集成验证通过后、合并回 `main` 前有用户确认门。
- [ ] 最终报告定义为 controller 最终回复，不默认落盘。
- [ ] 运行期 handoff package 位于 `/tmp`，不提交。
- [ ] 所有 `git status` 使用 `--porcelain=v1 --untracked-files=all` 长参数形式。
- [ ] 全程不 push、不改 remote、不整理历史未跟踪文件。
