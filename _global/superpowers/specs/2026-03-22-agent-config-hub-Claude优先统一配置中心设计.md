# agent-config-hub Claude 优先统一配置中心设计

## 1. 背景

当前 `goto-software` 仓库与用户主目录中的 Claude Code 配置同时存在，且配置来源分散：

- 项目内配置：`CLAUDE.md`、`AGENTS.md`、`.claude/CLAUDE.md`、`.claude/rules/*`、`.claude/settings*.json`、`.claude/skills/*`
- 用户级配置：`~/.claude/settings.json`、`~/.claude/commands/*`、`~/.claude/agents/*`、`~/.claude/hooks/*`
- 运行时数据：`~/.claude/debug/*`、`~/.claude/todos/*`、`~/.claude/cache/*`、`~/.claude/backups/*`

这导致以下问题：

- 规则、技能、命令、代理分散在多个目录中，难以形成单一事实源
- 项目级 `.claude` 同时承担配置与命令输出目录，职责混杂
- 用户级 `settings.json` 已包含敏感字段，不适合直接纳入版本控制
- 一些命令直接写死 `~/.claude/...` 或 `.claude/...` 路径，迁移后易失效
- 跨电脑同步依赖手工复制，无法通过 Git 形成稳定的配置分发链路

本设计的目标是建立仓库内的 `agent-config-hub`，先完成 Claude Code 侧的统一配置中心建设，再以此为基础派生未来的 Codex 适配层。

## 2. 目标

### 2.1 业务目标

- 让当前仓库中的 `agent-config-hub` 成为整机 Claude 配置的唯一事实源
- 支持任意电脑通过 `git clone/pull + install.sh` 完成接入
- 保持技能、规则、命令、代理、hooks 的统一托管与可版本化演进
- 为未来 Codex 适配层预留清晰边界，但本轮不实现 Codex 细节

### 2.2 工程目标

- 建立共享层与工具适配层的分层架构
- 彻底消除项目内 `.claude` 作为配置事实源的角色
- 将密钥、缓存、调试产物、待办等运行时数据隔离到本机私有层
- 为项目级命令输出定义新的本地状态目录，避免继续写入项目 `.claude`

## 3. 范围

### 3.1 本轮范围

- 在仓库内创建 `agent-config-hub/shared`、`agent-config-hub/claude`、`agent-config-hub/codex`
- 迁移当前项目中的：
  - `.claude/CLAUDE.md`
  - `.claude/rules/*`
  - `.claude/settings.json`
  - `.claude/settings.local.json` 中可公开的部分
  - `.claude/skills/*`
- 迁移当前用户目录中的：
  - `~/.claude/commands/*`
  - `~/.claude/agents/*`
  - `~/.claude/hooks/*`
  - `~/.claude/settings.json` 中可公开的部分
- 提供 Claude 安装器 `install.sh`
- 提供项目接入辅助命令 `agent-link`

### 3.2 明确不在本轮范围

- 不实现 Codex 的 `AGENTS.md`、规则、安装器和运行时接入
- 不收集 Codex 当前本机配置状态
- 不将 `~/.claude/debug`、`~/.claude/todos`、`~/.claude/cache`、`~/.claude/backups` 纳入版本控制
- 不直接迁移或提交任何真实密钥、私有代理地址、机器专属路径

## 4. 关键设计决策

### 4.1 采用三层结构，而不是单目录或双平级复制

选择结构：

- `agent-config-hub/shared`
- `agent-config-hub/claude`
- `agent-config-hub/codex`

原因：

- `shared` 承担可复用事实源，避免 Claude/Codex 各自复制技能
- `claude` 仅保留 Claude 专属入口和平台能力
- `codex` 先保留骨架，避免未来再拆仓或二次大重构

### 4.2 采用“受管覆盖层”，而不是将整个 `agent-config-hub/claude` 直接映射到 `~/.claude`

不采用整目录直链的原因：

- Claude 会持续向 `~/.claude` 写入运行时数据
- 直接映射会让运行时垃圾文件进入仓库目录
- 无法同时满足“整机统一配置”和“本机私有态不入仓”

采用的方案是：

- 仓库内保存版本化源配置
- 安装器创建真实的 `~/.claude`
- 受管项按目录或文件逐项软链接到 Hub
- 本机私有目录保留在真实 `~/.claude` 内

### 4.3 定义新的项目本地状态目录

本轮新增项目本地状态目录：

```text
.agent-hub/
└── claude/
    ├── checkpoints.log
    ├── evals/
    ├── package-manager.json
    └── runtime/
```

原因：

- 当前一些命令会把项目级产物写入仓库内 `.claude/...`
- 迁移后项目内 `.claude` 不再作为事实源，继续复用会让职责再次混乱
- 使用 `.agent-hub/claude` 可明确表达“项目本地状态”，同时为未来 `codex` 留出平级空间

项目级命令、日志、评估文件、包管理器本地决策等都应迁移到该目录。

## 5. 目标目录结构

```text
agent-config-hub/
├── shared/
│   ├── docs/
│   │   ├── migration-inventory.md
│   │   ├── local-only-assets.md
│   │   ├── config-layering.md
│   │   └── security-boundary.md
│   ├── metadata/
│   ├── scripts/
│   ├── skills/
│   │   ├── agent-builder/
│   │   ├── agent-requirement-analyzer/
│   │   └── qiuzhi-skill-creator/
│   └── templates/
├── claude/
│   ├── CLAUDE.md
│   ├── README.md
│   ├── install.sh
│   ├── rules/
│   │   ├── git-commit-zh.md
│   │   ├── gitnexus-usage.md
│   │   ├── plan-naming-zh.md
│   │   └── tdd-guideline-zh.md
│   ├── commands/
│   ├── agents/
│   ├── hooks/
│   ├── settings.json
│   └── settings.local.example.json
└── codex/
    ├── README.md
    └── placeholders/
```

## 6. 分层职责

### 6.1 `shared/`

职责：

- 承载跨工具共享的技能事实源
- 存放模板、迁移文档、配置边界说明
- 不包含任何 Claude 或 Codex 私有运行配置

适合放入 `shared` 的内容：

- `skills/*`
- 模板
- 迁移映射文档
- 配置边界说明

### 6.2 `claude/`

职责：

- 承载 Claude 专属的入口和平台适配能力
- 表达 Claude 的规则、命令、代理、hooks 和共享设置模板

适合放入 `claude` 的内容：

- `CLAUDE.md`
- `rules/*`
- `commands/*`
- `agents/*`
- `hooks/*`
- `settings.json`
- `settings.local.example.json`
- `install.sh`

### 6.3 `codex/`

职责：

- 仅作为未来 Codex 适配层骨架
- 本轮不提供真实安装逻辑，不生成 `AGENTS.md`

## 7. Claude 运行目录模型

安装完成后的 `~/.claude` 结构应为：

```text
~/.claude/
├── CLAUDE.md                  -> link to agent-config-hub/claude/CLAUDE.md
├── rules/                     -> link to agent-config-hub/claude/rules
├── skills/                    -> link to agent-config-hub/shared/skills
├── commands/                  -> link to agent-config-hub/claude/commands
├── agents/                    -> link to agent-config-hub/claude/agents
├── hooks/                     -> link to agent-config-hub/claude/hooks
├── settings.json              -> generated or linked managed config
├── settings.local.json        -> local-only file
├── debug/                     -> local runtime directory
├── todos/                     -> local runtime directory
├── cache/                     -> local runtime directory
└── backups/                   -> local backup directory
```

其中：

- 受管项由安装器维护
- 本机私有项由 Claude 与本机环境维护
- `settings.local.json` 永远不进仓库

## 8. `CLAUDE.md` 精简策略

`claude/CLAUDE.md` 应控制为短入口文档，职责是导航和总纲，不再承载全部细则。

建议内容仅保留：

- 工作区地图
- 核心准则总纲
- GitNexus 使用入口
- 文档与计划的高优先级约束
- skill 使用入口
- 配置与运行态边界说明

详细规则必须拆分到 `claude/rules/*.md`，避免入口文件继续膨胀。

## 9. settings 分层设计

### 9.1 仓库内共享配置

文件：`agent-config-hub/claude/settings.json`

仅允许放入：

- 可共享的默认行为开关
- 通用插件启用状态
- 可公开的 hooks 框架配置

禁止放入：

- `ANTHROPIC_AUTH_TOKEN`
- 私有 `ANTHROPIC_BASE_URL`
- 仅适用于单台机器的绝对路径
- 任何敏感令牌

### 9.2 本机模板

文件：`agent-config-hub/claude/settings.local.example.json`

用途：

- 为新机器初始化提供结构模板
- 仅包含占位符，不包含任何真实密钥

### 9.3 本机实际覆盖层

文件：`~/.claude/settings.local.json`

职责：

- 持有 token、私有 base URL、机器特有路径
- 永远不入仓
- 安装器在文件缺失时创建模板，不覆盖已存在内容

### 9.4 合并策略

优先策略：

- 如果 Claude 原生支持 `settings.json + settings.local.json` 覆盖，则直接利用

回退策略：

- 由 `install.sh` 将共享配置与本地覆盖合成为最终运行态 `~/.claude/settings.json`

## 10. 迁移映射

### 10.1 当前项目配置迁移

| 当前位置 | 目标位置 | 说明 |
| --- | --- | --- |
| `.claude/CLAUDE.md` | `agent-config-hub/claude/CLAUDE.md` | 重写为短入口 |
| `.claude/rules/*.md` | `agent-config-hub/claude/rules/*.md` | 保留规则粒度 |
| `.claude/skills/*` | `agent-config-hub/shared/skills/*` | 作为共享 skill 事实源 |
| `.claude/settings.json` | `agent-config-hub/claude/settings.json` | 仅保留共享字段 |
| `.claude/settings.local.json` | `agent-config-hub/claude/settings.local.example.json` | 删除真实敏感内容后模板化 |

### 10.2 用户级配置迁移

| 当前位置 | 目标位置 | 说明 |
| --- | --- | --- |
| `~/.claude/commands/*` | `agent-config-hub/claude/commands/*` | Claude 专属命令 |
| `~/.claude/agents/*` | `agent-config-hub/claude/agents/*` | Claude 专属代理 |
| `~/.claude/hooks/*` | `agent-config-hub/claude/hooks/*` | Claude hooks |
| `~/.claude/settings.json` | 拆分为共享层和本地层 | 敏感字段必须剥离 |

### 10.3 保留为本机私有层

以下内容不迁入仓库：

- `~/.claude/debug/*`
- `~/.claude/todos/*`
- `~/.claude/cache/*`
- `~/.claude/backups/*`
- `~/.claude/settings.local.json`
- 任何历史会话、调试文本、私钥、访问令牌

## 11. 对现有 commands / agents / hooks 的适配要求

### 11.1 路径分类

当前路径引用大致分为三类：

1. 指向全局配置：
   - `~/.claude/agents/...`
   - `~/.claude/skills/...`
   - `~/.claude/hooks/...`

2. 指向项目内旧目录：
   - `.claude/checkpoints.log`
   - `.claude/evals/...`
   - `.claude/package-manager.json`

3. 指向私有或缓存目录：
   - `~/.claude/homunculus/...`
   - `~/.claude/plugins/cache/...`

### 11.2 迁移原则

- 继续保留指向 `~/.claude/{agents,skills,hooks}` 的引用是可接受的，因为最终这些路径由受管软链接提供
- 所有项目内 `.claude/...` 输出必须迁移到 `.agent-hub/claude/...`
- 引用真实用户目录绝对路径 `/home/gyq/.claude/...` 的地方必须全部清除
- hooks 中若存在基于脚本相对路径推导的逻辑，应保持“相对脚本自身定位”，而不是依赖固定家目录

### 11.3 已知需要处理的命令类型

从当前扫描结果看，至少以下命令需要改写：

- `checkpoint.md`：输出改为 `.agent-hub/claude/checkpoints.log`
- `eval.md`：定义与日志改为 `.agent-hub/claude/evals/...`
- `setup-pm.md`：项目级配置改为 `.agent-hub/claude/package-manager.json`
- `instinct-*`、`learn`、`evolve`：继续以 `~/.claude/...` 为全局工作目录，但需移除写死用户绝对路径
- `plan.md`、`tdd.md`、`e2e.md`：文档中的引用保留 `~/.claude/agents/...` 和 `~/.claude/skills/...` 即可

### 11.4 GitNexus hook

当前 GitNexus hook 使用脚本相对路径寻找 CLI，可沿用此模式：

- `hooks/gitnexus/gitnexus-hook.cjs` 放入 `agent-config-hub/claude/hooks/gitnexus/`
- `settings.json` 中的 hook 命令改为 `node "~/.claude/hooks/gitnexus/gitnexus-hook.cjs"` 或通过安装器生成的绝对路径
- 脚本内部继续以 `__dirname` 计算相对路径，避免绑定某个家目录

## 12. 安装器设计

`agent-config-hub/claude/install.sh` 是 Hub 的核心能力，不是附属脚本。

### 12.1 职责

- 校验 Hub 目录完整性
- 备份现有 `~/.claude`
- 创建真实 `~/.claude`
- 建立受管链接
- 初始化本地私有文件与运行态目录
- 写入 shell 环境变量
- 提供状态检查、重建链接与卸载能力

### 12.2 建议命令模型

```bash
./install.sh install
./install.sh install --force
./install.sh install --no-backup
./install.sh install --dry-run
./install.sh status
./install.sh relink
./install.sh uninstall
```

### 12.3 参数语义

- `install`：默认备份并安装
- `--force`：允许覆盖受管项
- `--no-backup`：跳过备份
- `--dry-run`：仅打印计划操作
- `status`：检查当前接入状态
- `relink`：修复或重建受管软链接
- `uninstall`：移除受管项并尝试恢复备份

### 12.4 默认备份策略

用户已确认默认策略为：

- 默认备份后再切换
- 可通过参数显式跳过备份直接替换

### 12.5 shell 集成

安装器应向 `~/.bashrc` 或 `~/.zshrc` 写入：

- `AGENT_CONFIG_HUB`
- `AGENT_CONFIG_HUB_CLAUDE`

本轮不写入 `CODEX_HOME`，该变量在 Codex 适配层启用时再补。

## 13. `agent-link` 设计

`agent-link` 负责项目接入，不负责整机安装。

### 13.1 主要能力

- 在当前项目生成极薄 `CLAUDE.md`
- 为项目建立 `.agent-hub/claude/` 本地状态目录
- 将 `.agent-hub/` 加入项目级忽略规则
- 检查项目是否已接入全局 Hub

### 13.2 建议命令

```bash
agent-link init-project
agent-link doctor
```

### 13.3 输出策略

项目根的 `CLAUDE.md` 仅用于：

- 引导 Claude 使用全局 `~/.claude`
- 补充极少量项目特有说明

不再复制整套规则或技能。

## 14. 多电脑同步模型

接入流程：

1. `git clone` 仓库
2. 进入 `agent-config-hub/claude`
3. 执行 `./install.sh install`
4. 生成或保留 `~/.claude/settings.local.json`
5. 手动填入本机私密信息

后续同步流程：

1. `git pull`
2. 如有结构变化，执行 `./install.sh relink`

该模型满足：

- 仓库是事实源
- 配置可通过 Git 分发
- 本机私有态留在本地

## 15. 风险与缓解

### 15.1 绝对路径残留

风险：

- 命令、规则、settings 中仍残留 `/home/gyq/.claude/...`

缓解：

- 使用全文搜索完成迁移清理
- 安装器增加诊断检查，发现绝对路径即报警

### 15.2 项目级 `.claude/...` 输出残留

风险：

- 命令继续写入项目 `.claude`，导致旧职责回流

缓解：

- 在迁移时统一替换为 `.agent-hub/claude/...`
- 将 `.agent-hub/` 加入忽略规则

### 15.3 敏感配置误提交

风险：

- token、私有 base URL 被提交到仓库

缓解：

- 强制使用 `settings.local.example.json` 模板
- `.gitignore` 明确排除本机私有文件
- 在设计与实施阶段增加 secrets 自检

### 15.4 整机切换失败

风险：

- 立即切换模式下，`~/.claude` 接管失败会影响 Claude 可用性

缓解：

- 默认备份
- `install.sh status` 与 `relink` 提供修复入口
- `uninstall` 提供回滚入口

## 16. 验收标准

迁移完成后，应同时满足以下条件：

1. 仓库内存在稳定三层结构：
   - `shared`
   - `claude`
   - `codex`
2. 当前项目原有 `.claude` 中的版本化资产已完成收敛
3. Claude 的规则、skills、commands、agents、hooks 均可由 `~/.claude` 正常发现
4. 项目级命令输出不再写入项目 `.claude`，而是写入 `.agent-hub/claude`
5. `settings` 中的敏感字段全部留在本机私有层
6. 新电脑可通过 `git clone/pull + install.sh` 完成接入

## 17. 实施阶段建议

### Phase 1：建立骨架

- 创建 `agent-config-hub` 三层目录
- 建立文档、模板与 `.gitignore`

### Phase 2：迁移共享与 Claude 专属资产

- 迁移 skills
- 迁移 rules
- 迁移 commands / agents / hooks
- 生成精简版 `CLAUDE.md`

### Phase 3：配置分层与安装器

- 拆分 `settings`
- 实现 `install.sh`
- 实现 `agent-link`

### Phase 4：路径改写与验证

- 清理绝对路径
- 迁移项目输出路径到 `.agent-hub/claude`
- 验证整机切换与新机器接入链路

## 18. 设计结论

本方案采用“`shared + claude + codex` 三层结构 + Claude 受管覆盖层 + 项目本地状态目录”的组合策略。

这是当前唯一同时满足以下要求的方案：

- 先完成 Claude 统一化
- 保持未来 Codex 扩展空间
- 保证技能只维护一份
- 保证整机统一与本机私有边界并存
- 允许通过 Git 实现跨电脑同步

后续实施应严格以本设计为准，不再让项目 `.claude` 重新承担配置事实源角色。
