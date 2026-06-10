# Agent Config Hub Superpowers 安装整合 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `agent-config-hub` 增加统一的 Superpowers 安装入口，覆盖 Claude 与 Codex，并区分本地接入、平台安装与网络回退。

**Architecture:** 保留现有 `claude/install.sh` 作为 Claude 配置托管入口，在其上增加 Superpowers 安装逻辑；新增顶层 `install.sh` 做目标分发，新增 `codex/install.sh` 管理 Codex 的 skills 接入；使用 `vendor/manifest.json` 声明 Claude 与 Codex 的安装方式，并补一个单独的更新脚本。

**Tech Stack:** Bash, Python 3, Claude CLI, Codex CLI, Git

---

### Task 1: 先补失败测试锁定预期行为

**Files:**
- Modify: `agent-config-hub/tests/claude/layout_test.sh`
- Create: `agent-config-hub/tests/claude/plugin_install_test.sh`
- Create: `agent-config-hub/tests/codex/install_smoke_test.sh`

- [ ] **Step 1: 写 Claude 插件安装测试**
- [ ] **Step 2: 写 Codex 安装脚本测试**
- [ ] **Step 3: 更新 layout 测试要求**
- [ ] **Step 4: 运行新增测试并确认失败**

### Task 2: 增加统一安装与依赖声明

**Files:**
- Create: `agent-config-hub/install.sh`
- Create: `agent-config-hub/vendor/manifest.json`
- Create: `agent-config-hub/codex/install.sh`

- [ ] **Step 1: 新增 manifest 并声明 superpowers-claude / superpowers-codex**
- [ ] **Step 2: 新增顶层 install.sh 分发 claude/codex/all**
- [ ] **Step 3: 新增 codex/install.sh 支持 install/status/update/uninstall 的最小集合**
- [ ] **Step 4: 运行 Codex 测试并确认通过**

### Task 3: 扩展 Claude 安装脚本接入 superpowers

**Files:**
- Modify: `agent-config-hub/claude/install.sh`

- [ ] **Step 1: 为 Claude 安装脚本增加 superpowers 安装辅助函数**
- [ ] **Step 2: 实现本地 marketplace 优先、官方 marketplace / GitHub 回退**
- [ ] **Step 3: 扩展 status 输出以暴露 plugin 状态**
- [ ] **Step 4: 运行 Claude 相关测试并确认通过**

### Task 4: 增加更新脚本与文档

**Files:**
- Create: `agent-config-hub/update-deps.sh`
- Modify: `agent-config-hub/README.md`
- Modify: `agent-config-hub/claude/README.md`
- Modify: `agent-config-hub/codex/README.md`

- [ ] **Step 1: 新增更新脚本，优先更新 superpowers 相关依赖**
- [ ] **Step 2: 更新根 README，说明统一安装与更新入口**
- [ ] **Step 3: 更新 Claude README，补插件安装与回退路径**
- [ ] **Step 4: 重写 Codex README，补 skills 安装与验证方式**
- [ ] **Step 5: 运行最终测试与状态核对**
