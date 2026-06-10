# agent-config-hub Codex 平台专属配置设计

## 背景

`agent-config-hub` 已经完成了 Claude Code 侧的统一配置管理，并预留了 `codex/` 目录作为未来适配层。但经过重新梳理，当前真正需要解决的问题不是“把 Claude Code 的规则与技能一键翻译到 Codex”，而是：

- 新电脑上可以快速恢复 Claude Code
- 新电脑上可以快速恢复 Codex
- 两个平台各自维护各自的配置，不强行对齐

此前“Claude 规则与技能同步到 Codex”的方向存在两个问题：

- 技能部分虽然可以共享，但规则并不天然等价。Claude 的 `rules/*.md` 是自然语言行为规范，而 Codex 的 `~/.codex/rules/*.rules` 更偏向命令权限 DSL。
- 强行做跨平台翻译会引入额外抽象层、生成逻辑和回归负担，反而弱化 `agent-config-hub` 作为“新机器快速恢复配置”的核心目标。

因此，本轮设计将 Codex 适配层重新定义为“Codex 平台专属配置管理层”，而不是 Claude 到 Codex 的同步层。

## 目标

### 1. 业务目标

- 让 `agent-config-hub/codex/` 成为 Codex 专属配置的版本化事实源。
- 支持新电脑通过 `git clone/pull + install.sh` 快速恢复 Codex 环境。
- 允许 Claude Code 与 Codex 维护不同的规则、技能和入口文件，不要求平台间对齐。
- 将真正平台无关的资产沉淀到 `shared/`，避免重复维护。

### 2. 工程目标

- 明确 `shared/`、`claude/`、`codex/` 三层职责边界。
- 补齐 Codex 层的入口文件、规则说明、技能目录、配置模板和安装器语义。
- 保证安装器只管理受控路径，不覆盖用户已有的私有运行态配置。
- 为后续新增 Codex 规则或技能提供稳定的目录和测试边界。

## 非目标

- 不做 Claude Code 规则自动翻译到 Codex。
- 不做 Claude Code 技能自动翻译到 Codex。
- 不追求 Claude 与 Codex 目录结构完全镜像。
- 不自动改写用户已有的 `~/.codex/config.toml`、认证信息、历史数据库和缓存。
- 不将本轮设计扩展为跨平台统一抽象框架。

## 核心设计决策

### 1. 采用“薄共享内核”，而不是跨平台同步层

保留三层结构：

- `agent-config-hub/shared`
- `agent-config-hub/claude`
- `agent-config-hub/codex`

但重新定义边界：

- `shared/` 只放真正平台无关的资产
- `claude/` 只服务 Claude Code
- `codex/` 只服务 Codex

这样做的原因是：

- 新电脑恢复配置的真实需求是“同平台恢复”，不是“跨平台翻译”
- 平台专属能力差异较大，强行同步会放大维护成本
- 允许人工决定某个平台安装哪些规则和技能，比强制对齐更符合实际使用方式

### 2. 共享资产只因“平台无关”而共享，不因“减少重复”而共享

是否放入 `shared/` 的判断标准不是“两个平台都能勉强用”，而是“资产本身不携带平台语义、可被两个平台直接消费”。

这意味着：

- 真正平台无关的通用脚本、文档、依赖声明可以共享
- 只要内容写死 Claude 或 Codex 的工具、目录、流程，就不应继续放在 `shared/`
- 同一概念若在两个平台的落地方式不同，允许各自维护独立实现

### 3. Codex 的规则说明以自然语言入口承载，而不是强行映射到 `.rules`

Codex 的行为规则说明应优先通过以下方式表达：

- `codex/AGENTS.md` 作为主入口
- `codex/rules/*.md` 作为细则文档

`~/.codex/rules/*.rules` 只在确实需要命令许可或白名单控制时使用，不作为自然语言规则说明的主要载体。

这样可以避免：

- 把 Claude 的 Markdown 规则硬翻译成不等价的 DSL
- 把“说明文档”和“权限规则”混为一谈
- 未来每加一条规则都必须设计翻译器

## 目录职责

### 1. `shared/`

`shared/` 只保留真正平台无关的内容，例如：

- `shared/docs/`：配置分层、安全边界、迁移清单等通用文档
- `shared/scripts/`：跨平台都可复用的辅助脚本
- `vendor/manifest.json`：统一的外部依赖声明
- 可被多个平台直接消费、且不带平台语义的模板或素材

不应再放入 `shared/` 的内容：

- Claude 专属入口模板
- Codex 专属规则说明
- 写死 Claude 或 Codex 工具语义的技能

### 2. `claude/`

`claude/` 继续作为 Claude Code 专属配置层，负责：

- `CLAUDE.md`
- `rules/`
- `commands/`
- `agents/`
- `hooks/`
- `settings.json`
- `settings.local.example.json`
- `install.sh`

Claude 的规则、命令、代理和 hooks 不要求为 Codex 提供同步版本。

### 3. `codex/`

`codex/` 应从当前的最小安装器扩展为完整的 Codex 专属管理层，至少包含：

- `README.md`
- `install.sh`
- `AGENTS.md`
- `rules/`
- `skills/`
- `config/`

其中：

- `AGENTS.md` 是 Codex 的主入口说明
- `rules/` 存放 Codex 专属 Markdown 规则
- `skills/` 存放 Codex 专属技能
- `config/` 存放模板，不存放机器私密配置

## 现有资产归位建议

### 保留在 `shared/` 的内容

- `shared/docs/*`
- `shared/scripts/*` 中真正平台无关的脚本
- `vendor/manifest.json`

### 移出 `shared/` 的内容

以下内容不应继续视为共享资产：

- `shared/templates/project-CLAUDE.md`

它应移动到 `claude/templates/` 或其他 Claude 专属位置，因为名称和用途都明确绑定 Claude。

### `shared/skills/` 的处理原则

`shared/skills/` 不再默认等于“两个平台通用技能仓库”，而需要重新按以下标准筛选：

- 若技能不依赖平台专属工具、目录或工作流，可继续留在 `shared/skills/`
- 若技能内嵌 Claude 专属语义，应迁移到 `claude/skills/`
- 若技能是为 Codex 单独适配的，应放入 `codex/skills/`

共享不是强约束；如果同一能力在 Claude 与 Codex 的最佳写法不同，允许两边各自保有一份实现。

## Codex 层目标结构

建议的 Codex 目录结构如下：

```text
agent-config-hub/
└── codex/
    ├── README.md
    ├── install.sh
    ├── AGENTS.md
    ├── rules/
    │   ├── gitnexus-usage.md
    │   ├── git-commit-zh.md
    │   ├── plan-naming-zh.md
    │   └── ...
    ├── skills/
    │   └── ...
    ├── config/
    │   └── config.toml.example
    └── placeholders/
```

说明：

- `rules/` 与 `skills/` 是否首轮就补齐全部内容，不在本设计中强制要求，但目录边界应先建立。
- `config.toml.example` 只提供模板，不承载任何真实私密值。
- `placeholders/` 只在确有必要时保留；若无实际用途，可以在后续实现时裁撤。

## Codex 安装模型

Codex 安装器不再只负责 `superpowers` 链接，而应承担三类职责：

### 1. 安装共享可消费资产

例如：

- 将真正平台无关、且 Codex 可直接消费的技能挂接到 Codex 的技能发现路径

### 2. 安装 Codex 专属资产

例如：

- `codex/AGENTS.md`
- `codex/rules/`
- `codex/skills/`

这些资产的安装方式应与 Codex 的原生消费方式兼容，但不强求与 Claude 的安装行为一致。

### 3. 初始化本地模板

例如：

- 在本机缺少对应模板文件时，提供 `config.toml.example` 的初始化或提示

安装器必须满足：

- 不覆盖用户已有的 `~/.codex/config.toml`
- 不删除认证信息、历史数据库、缓存和运行时目录
- 只管理明确声明的受管路径

## 测试与验收边界

本轮 Codex 层的完成标准，不是“功能尽量多”，而是“可安全安装、可稳定恢复、边界清楚”。

### 1. 结构测试

校验：

- `codex/` 的核心文件和目录存在
- `README.md`、`install.sh`、`AGENTS.md`、`rules/`、`skills/`、`config/` 结构完整

### 2. 安装冒烟测试

在临时 `HOME` 下验证：

- `codex/install.sh install` 可执行
- 受管路径被正确创建或链接
- 不会误覆盖已有私有配置

### 3. 状态测试

验证 `codex/install.sh status` 能明确区分：

- 共享技能缺失
- 规则入口缺失
- 技能链接失效
- 模板未初始化

### 4. 幂等性测试

验证连续执行两次安装后：

- 不会重复制造脏状态
- 不会破坏已有受管路径

### 5. 卸载边界测试

验证卸载时：

- 只移除受管资产
- 不删除用户私有 `config.toml`、历史、认证、缓存等运行态文件

## 第一阶段实施顺序

建议按以下顺序落地：

1. 收口目录职责
2. 迁移或重分类现有资产
3. 建立 Codex 资产骨架
4. 升级 `codex/install.sh`
5. 补齐测试

更具体地说：

### 阶段 1：收口边界

- 确认 `shared/` 中哪些内容是真正通用资产
- 将明显平台专属的模板和说明迁回各平台目录

### 阶段 2：建立 Codex 骨架

- 增加 `codex/AGENTS.md`
- 增加 `codex/rules/`
- 增加 `codex/skills/`
- 增加 `codex/config/`
- 更新 `codex/README.md`

### 阶段 3：升级安装器

- 扩展 `codex/install.sh` 的安装范围和状态检查范围
- 保持与现有 `install/status/update/uninstall` 语义一致
- 明确本地私有配置的保护策略

### 阶段 4：测试兜底

- 补齐结构测试
- 补齐安装冒烟测试
- 补齐状态、幂等、卸载边界测试

## 风险与取舍

### 1. 风险：重新分类 `shared/skills/` 需要人工判断

风险说明：

- 一些现有技能可能表面上是“共享技能”，但内部已经隐含平台假设

取舍：

- 本轮先建立边界和目录，不要求一次性完成全部技能重分类
- 允许先保留现状，再逐步将明显平台专属技能迁出

### 2. 风险：Codex 原生能力与 Claude 不一致

风险说明：

- 同名规则或同类流程在两个平台可能需要不同写法

取舍：

- 明确放弃“平台间自动同步”
- 允许平台专属规则和技能各自演化

### 3. 风险：安装器扩展过快，破坏已有 Codex 本机配置

风险说明：

- 当前本机 `~/.codex/` 已包含配置、历史、rules、数据库等真实运行态

取舍：

- 第一阶段只管理明确受控路径
- 对用户已有配置默认采取保守策略，不自动覆盖

## 最终结论

`agent-config-hub` 后续应采用“薄共享内核 + 平台专属配置层”的演进方式：

- Claude Code 配置同步到 Claude Code
- Codex 配置同步到 Codex
- `shared/` 只保留真正平台无关的资产
- 不再设计 Claude 到 Codex 的自动同步或翻译机制

对 Codex 而言，本轮工作的目标不是复制 Claude 层，而是建立一个独立、可维护、适合新机器恢复的专属管理层。
