# superpowers 分支收口设计

## 背景

当前 `/mnt/g/Obsidian` 是实际 Git 顶层仓库，`skill/superpowers` 是其中的一个子目录。刚完成的 `feat/002-deepseek-document-review-loop` 分支实现了 DeepSeek 文档审核循环，但验收时发现最终状态报告不准确：工作区仍有未跟踪文件，且 `final-review-handoff.md` 内容混入了历史 run，不适合作为可靠交接证据。

同时，仓库中还存在一个更早的未合并分支 `refactor/001-agent-rules-lazy-loading`。该分支不是基于当前 `main`，而是基于 `05a61e7695d9aef96cc16643e5c36db9d5c6f881`，合并到当前 `main` 时会在 `code/01_System_Core/global_rules.md` 产生内容冲突。该分支还修改了 `skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md`，删除了此前关于 finishing/本地合并约束的段落，需要人工判断是否保留或调整。

本设计目标是给出完整分支收口策略：先修复并合并低风险的 `feat/002`，再单独评估和合并存在冲突的 `refactor/001`，最后确认没有遗漏未合并分支和脏工作区。

## 当前分支盘点

### `/mnt/g/Obsidian`

`main` 当前提交：

```text
d0edf8b229d7a5b43d9b266d92ca8fee368f5412 auto sync: 2026-07-03 01:00:27
```

未合并到 `main` 的本地分支：

```text
feat/002-deepseek-document-review-loop
refactor/001-agent-rules-lazy-loading
```

当前 worktree：

```text
/mnt/g/Obsidian
  branch: feat/002-deepseek-document-review-loop
  dirty: yes
  untracked:
    backup/chrome-tab-manager-2026-07-03.json
    skill/superpowers/final-review-handoff.md

/mnt/g/Obsidian-worktrees/refactor/001-agent-rules-lazy-loading
  branch: refactor/001-agent-rules-lazy-loading
  dirty: no
```

### `/mnt/f/IdeaProjects/goto-software/goto-superpowers-docs`

该文档仓库没有发现其它本地分支未合并到 `main`。`main` 领先 `origin/main` 2 个提交，且存在一批历史未跟踪文件。分支收口实现不得整理这些历史未跟踪文件；只允许新增本设计和后续计划相关文件。

## 收口目标

- 修复 `feat/002-deepseek-document-review-loop` 的验收问题，使其工作区状态、交接证据和最终报告一致。
- 将 `feat/002-deepseek-document-review-loop` 本地合并回 `main`。
- 对 `refactor/001-agent-rules-lazy-loading` 做冲突评估和冲突解决设计。
- 在 `feat/002` 已合并后的 `main` 上收口 `refactor/001`，保留有价值的规则拆分成果。
- 最终确认 `/mnt/g/Obsidian` 没有需要合并而遗漏的本地分支。
- 全程不执行远端 push，不修改 Git remote，不整理与本任务无关的历史未跟踪文件。

## 非目标

- 不直接删除任何不明来源的未跟踪文件。需要先分类、确认归属，再移动、忽略、提交或保留。
- 不把 `goto-superpowers-docs` 的历史未跟踪文件纳入本次提交。
- 不重写已存在分支历史，除非用户后续明确要求。
- 不自动执行远端同步。远端 `push` 由用户手动决定。
- 不修改业务项目代码；本次只处理分支、文档、规则和 superpowers skill 收口。

## 推荐方案

采用“先低风险收口，再处理冲突分支”的顺序。

1. **先收口 `feat/002-deepseek-document-review-loop`**
   - 先处理 `/mnt/g/Obsidian` 当前未跟踪文件。
   - `skill/superpowers/final-review-handoff.md` 当前内容不可靠，默认不提交。可重新生成到仓库外临时位置，或删除后在最终报告中说明不保留运行时交接包。
   - `backup/chrome-tab-manager-2026-07-03.json` 不属于 `feat/002` 变更范围，默认移出 Git 工作区或保留但明确阻止 clean 验收。若用户确认其无用，可删除；若需要保留，应移动到 Git 仓库外。
   - 验证 `git status --porcelain=v1 -uall` clean 后，本地合并到 `main`。

2. **再收口 `refactor/001-agent-rules-lazy-loading`**
   - 在 `feat/002` 已合入的 `main` 上处理。
   - 先用 `git merge-tree main refactor/001-agent-rules-lazy-loading` 或临时集成分支预检冲突。
   - 已知冲突点是 `code/01_System_Core/global_rules.md`。
   - 冲突解决原则：保留当前 `main` 的最新 auto sync 内容，同时保留 `refactor/001` 的规则拆分结构和 `rules/` 细则目录。
   - 对 `skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md` 的删除段落单独复核：如果该段仍是用户规则或当前流程需要的本地合并约束，应保留或改写；不能因为 refactor 分支删除了它就直接接受。

3. **最后做统一验证**
   - 检查 `git branch --no-merged main`。
   - 检查 `git status --porcelain=v1 -uall`。
   - 检查 `git worktree list --porcelain`。
   - 检查 DeepSeek 审核脚本软链接、`bash -n`、`--help` 和 dry-run。
   - 检查规则拆分后的入口文件和 trigger-index 是否能定位到各细则。

## 合并顺序

推荐顺序：

```text
1. feat/002-deepseek-document-review-loop
2. refactor/001-agent-rules-lazy-loading
```

理由：

- `feat/002` 基于当前 `main`，merge-tree 未显示冲突，且它的验收问题集中在未跟踪产物和交接包质量，适合先收口。
- `refactor/001` 基于旧提交，存在 `global_rules.md` 冲突，还涉及规则架构调整。先合它会放大不确定性，并可能影响 `feat/002` 的已验证路径。
- `refactor/001` 修改了 `hybrid-subagent-driven-development`，而 `feat/002` 依赖当前 Hybrid 工作流和 `codex-dp-exec`；先合 `feat/002` 更容易保护新审核工具链。

## `feat/002` 修复设计

### 未跟踪文件处理

必须先分类处理：

```text
backup/chrome-tab-manager-2026-07-03.json
skill/superpowers/final-review-handoff.md
```

默认策略：

- `backup/chrome-tab-manager-2026-07-03.json`
  - 不属于 `feat/002` scope。
  - 不提交。
  - 若用户确认无用，删除。
  - 若需要保留，移动到 Git 仓库外，例如 `/mnt/f/IdeaProjects/goto-software/tmp/` 或用户指定位置。

- `skill/superpowers/final-review-handoff.md`
  - 当前内容混入历史 run，且 `Evidence To Inspect` 指向不相关任务。
  - 不提交当前版本。
  - 若仍需要交接包，应重新生成到仓库外临时路径，或者修正 `codex-dp-exec handoff` 过滤能力后再生成。
  - 如果本次只要求合并分支，删除该未跟踪文件并在最终报告中说明“不保留运行时交接包”。

### 验证要求

`feat/002` 合并前必须满足：

```bash
git -C /mnt/g/Obsidian status --porcelain=v1 -uall
git -C /mnt/g/Obsidian log --oneline main..feat/002-deepseek-document-review-loop
git -C /mnt/g/Obsidian diff --stat main..feat/002-deepseek-document-review-loop
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/codex-dp-exec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-spec
bash -n /mnt/g/Obsidian/skill/superpowers/scripts/deepseek-review-plan
/home/gyq/.local/bin/deepseek-review-spec --help
/home/gyq/.local/bin/deepseek-review-plan --help
```

`status --porcelain` 必须为空，除非用户明确接受某个未跟踪文件继续存在并放弃 clean 验收。

### 合并方式

本地合并，不 push：

```bash
git -C /mnt/g/Obsidian switch main
git -C /mnt/g/Obsidian merge --no-ff feat/002-deepseek-document-review-loop
```

合并后再次验证：

```bash
git -C /mnt/g/Obsidian status --short --branch
git -C /mnt/g/Obsidian branch --no-merged main
```

## `refactor/001` 冲突评估设计

### 分支内容

`refactor/001-agent-rules-lazy-loading` 包含 5 个提交：

```text
ebc955a docs: 拆分全局 Agent 规范为短入口与细则文件
21cd9cf docs: 拆分 hsmap 项目 Agent 规范
dee692e docs: 拆分 goto-software 项目 Agent 规范
373f5f7 docs: 清理 Agent 规范项目边界
6af30db chore: 增加 Agent 规范结构校验
```

主要文件类型：

- `code/01_System_Core/global_rules.md`
- `code/01_System_Core/rules/*.md`
- `code/01_Project_Specs/hsmap/*`
- `code/01_Project_Specs/goto-software/*`
- `code/tools/validate-agent-rules.sh`
- `skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md`

### 已知冲突

只读预检显示：

```text
CONFLICT (content): Merge conflict in code/01_System_Core/global_rules.md
```

原因：当前 `main` 的 `d0edf8b` 修改了 `global_rules.md` 和 `hybrid-subagent-driven-development/SKILL.md`，而 `refactor/001` 基于更早的 `05a61e7` 做规则拆分。

### 冲突解决原则

`global_rules.md`：

- 保留短入口和触发条件结构。
- 保留当前用户规则中的硬约束，不允许因为拆分而丢失：
  - 中文输出。
  - HTML/WSL 预览地址规则。
  - 数据库只读。
  - 中文计划文件名。
  - 中文 commit 信息。
  - Git 远端用 `git.exe`。
  - worktree 分支序号规则。
  - Maven 禁止 deploy 和 Windows Maven 调用方式。
  - Java 验证流程和测试注释规范。
  - 生成代码中文注释规范。
- 将长规则放到 `code/01_System_Core/rules/*.md`，但 `global_rules.md` 必须保留足够的硬约束摘要和触发条件，避免 agent 不读取细则时误操作。

`hybrid-subagent-driven-development/SKILL.md`：

- 不盲目接受 `refactor/001` 删除 finishing/本地合并约束的改动。
- 当前用户规则明确远端操作不能自动 push；如果 skill 中存在合并选项，必须继续保留“只本地合并、不自动 push”的约束。
- 若需要简化，应改写而不是删除。

### 验证要求

合并 `refactor/001` 前后必须验证：

```bash
bash -n /mnt/g/Obsidian/code/tools/validate-agent-rules.sh
/mnt/g/Obsidian/code/tools/validate-agent-rules.sh
rg -n "git.exe|mvn.cmd|deploy|数据库|127.0.0.1|中文|worktree|final-review|push" /mnt/g/Obsidian/code/01_System_Core /mnt/g/Obsidian/code/01_Project_Specs
rg -n "本地合并|git push|远端|finishing-a-development-branch" /mnt/g/Obsidian/skill/superpowers/skills/hybrid-subagent-driven-development/SKILL.md
```

如果 `validate-agent-rules.sh` 的设计不支持当前环境，应记录失败原因并用 `rg` 检查替代，但不能忽略规则完整性。

## 最终验收标准

- `feat/002-deepseek-document-review-loop` 已本地合并到 `main`。
- `refactor/001-agent-rules-lazy-loading` 已完成冲突评估；若用户批准合并，则也本地合并到 `main`。
- `git branch --no-merged main` 不再包含已决定收口的分支；如保留某分支，最终报告必须说明保留原因。
- `/mnt/g/Obsidian` 目标工作区 `git status --porcelain=v1 -uall` 为空，或所有剩余未跟踪文件都有用户确认的保留理由。
- `feat/002` 新增脚本和 skill 接入点仍通过验证。
- `refactor/001` 的规则拆分没有丢失全局硬约束。
- 不执行 `git push`。
- 不修改 Git remote。
- 不整理 `goto-superpowers-docs` 的历史未跟踪文件。

## 风险与约束

- `refactor/001` 的 `global_rules.md` 冲突属于规则源文件冲突，不能简单使用 ours/theirs。必须逐条核对硬约束是否保留。
- `hybrid-subagent-driven-development` 中合并约束的删改会影响后续执行分支的行为，需要人工复核。
- `/mnt/g/Obsidian` 是 Git 顶层，`skill/superpowers` 只是子目录；所有 `git status` 和合并判断必须以 `/mnt/g/Obsidian` 为准。
- `final-review-handoff.md` 当前内容不可靠，不能作为验收依据。
- 如果用户希望保留 `backup/chrome-tab-manager-2026-07-03.json`，应先移出 Git 工作区再验收 clean 状态。
