# gotoBase 与 gotoTools 批量重装设计

## 背景

`goto-deploy-backend/.bash/install_gotoTools.sh` 和 `remove_gotoTools.sh` 当前虽然使用 `.sh` 扩展名和 Bash shebang，文件主体却是 R 代码，只能由 `Rscript` 解释。现有安装脚本还会在部署时执行 `roxygen2::roxygenize()` 并主动移除旧包，职责混合且可能修改 `NAMESPACE`、`man/` 等源码文件。

WSL 开发容器启动脚本已将以下源码目录挂载到容器：

- `/root/software/gotoBase`
- `/root/software/gotoTools`

`gotoTools` 通过 `Imports` 和 `NAMESPACE` 依赖 `gotoBase`，因此批量更新必须按依赖关系反向移除、正向安装。

## 目标

- 保留并完善 `gotoTools` 的独立移除、安装脚本。
- 新增 `gotoBase` 的独立移除、安装脚本。
- 新增一个从 WSL 宿主机执行的批量入口，通过 `docker exec` 操作目标容器。
- 固定执行顺序为：移除 `gotoTools`、移除 `gotoBase`、安装 `gotoBase`、安装 `gotoTools`。
- 在开始移除前完成全部必要预检，任一步真实失败时立即停止并返回非零退出码。
- 安装完成后在新的 R 进程中验证两个包均可加载，并输出安装版本。

## 非目标

- 不修改 `gotoBase` 或 `gotoTools` 的业务代码。
- 不修改容器挂载方式或 `wsl_rbase_start.sh`。
- 不在部署阶段生成 Roxygen 文档。
- 不安装或升级 R 包依赖。
- 不重启容器或容器内的 R 服务。
- 不实现旧包版本回滚；当前没有可靠的旧包快照可供恢复。

## 方案选择

批量入口通过标准输入，把四个独立 R 脚本逐个送入容器中的 `Rscript`：

```bash
docker exec -i "$container_name" Rscript --vanilla /dev/stdin < "$script_path"
```

选择该方案的原因：

- 不要求将 `.bash` 目录额外挂载到容器。
- 不需要重建或重启容器。
- 不需要使用 `docker cp` 创建和清理临时文件。
- 四个单包脚本保持独立，可单独维护和排查。
- 每一步都运行在新的 R 进程中，避免前一步残留的已加载命名空间干扰后续操作。

## 文件结构

所有脚本位于 `goto-deploy-backend/.bash/`：

| 文件 | 类型 | 职责 |
| --- | --- | --- |
| `remove_gotoTools.sh` | Rscript 脚本 | 幂等移除 `gotoTools` |
| `install_gotoTools.sh` | Rscript 脚本 | 从固定容器路径安装 `gotoTools` |
| `remove_gotoBase.sh` | Rscript 脚本 | 幂等移除 `gotoBase` |
| `install_gotoBase.sh` | Rscript 脚本 | 从固定容器路径安装 `gotoBase` |
| `reinstall_gotoBase_gotoTools.sh` | Bash 脚本 | 预检、编排四个步骤并执行最终验证 |

四个单包脚本保留 `.sh` 扩展名以延续现有命名，但将 shebang 明确为 Rscript；批量入口使用 Bash shebang。所有新增或修改的说明、状态和错误消息使用中文。

## 批量入口接口

默认目标容器为 `goto_rbase_wsl`：

```bash
./reinstall_gotoBase_gotoTools.sh
```

允许通过第一个位置参数覆盖容器名：

```bash
./reinstall_gotoBase_gotoTools.sh other_container
```

脚本只接受零个或一个参数。容器名始终作为独立参数传给 Docker，不使用 `eval`，也不拼接成待解释的命令字符串。批量入口通过自身所在目录定位四个单包脚本，因此可以从任意当前工作目录启动。

## 预检

批量入口必须在移除任何已安装包之前完成以下检查：

1. WSL 宿主机存在可调用的 `docker` 命令。
2. 目标容器存在且状态为运行中。
3. 四个单包脚本存在并可读。
4. 容器内存在可执行的 `Rscript`。
5. 容器内已安装 `remotes`。
6. `/root/software/gotoBase/DESCRIPTION` 存在，且其 `Package` 字段等于 `gotoBase`。
7. `/root/software/gotoTools/DESCRIPTION` 存在，且其 `Package` 字段等于 `gotoTools`。

任一预检失败时，脚本输出明确原因、返回非零退出码，并且不执行任何移除操作。

## 单包脚本行为

### 移除脚本

移除脚本检查 `.libPaths()` 中是否存在目标包：

- 完全未安装时输出跳过信息并成功退出。
- 已安装时移除找到的目标包实例，并验证目标包不再可发现。
- 移除失败或仍能发现目标包时抛出错误，使批量入口立即停止。

该行为使“包原本不存在”成为可重复执行的正常状态，同时避免残留副本在安装前继续生效。

### 安装脚本

安装脚本只负责安装，不再内置移除操作，也不执行 `roxygen2::roxygenize()`。两个包分别使用固定路径，并统一调用：

```r
package_path <- "/root/software/gotoBase" # gotoTools 脚本使用 /root/software/gotoTools
remotes::install_local(
  path = package_path,
  force = TRUE,
  upgrade = "never",
  dependencies = FALSE
)
```

`gotoBase` 的路径为 `/root/software/gotoBase`，`gotoTools` 的路径为 `/root/software/gotoTools`。`dependencies = FALSE` 保证部署脚本不会隐式安装或升级依赖；`gotoTools` 仅在 `gotoBase` 成功安装后执行。

## 执行流程

批量入口使用 `set -Eeuo pipefail`，并按以下顺序运行：

1. 完成全部预检。
2. 执行 `remove_gotoTools.sh`。
3. 执行 `remove_gotoBase.sh`。
4. 执行 `install_gotoBase.sh`。
5. 执行 `install_gotoTools.sh`。
6. 启动一个新的 R 进程，使用 `requireNamespace()` 加载两个包并输出 `packageVersion()`。

每一步开始前输出步骤名称，成功后输出确认信息。任何命令返回非零退出码时立即停止，并输出失败步骤。失败后不尝试继续执行，也不自动回滚；修复源码、依赖或容器环境后，可重新运行同一批量入口。

## 最终验证

最终验证必须在四个安装/移除进程之外的新 R 进程中执行，至少确认：

- `requireNamespace("gotoBase", quietly = TRUE)` 返回真。
- `requireNamespace("gotoTools", quietly = TRUE)` 返回真。
- 能读取并输出两个包的实际 `packageVersion()`。

验证失败视为整个批量任务失败，批量入口返回非零退出码。验证过程不重启容器。

## 实施后测试

1. 对 `reinstall_gotoBase_gotoTools.sh` 执行 `bash -n`，验证 Bash 语法。
2. 在容器中解析四个 Rscript 文件，验证 R 语法。
3. 使用一个不存在的容器名执行批量入口，确认其在预检阶段失败、返回非零退出码且未执行移除。
4. 对默认容器 `goto_rbase_wsl` 完整执行一次批量重装。
5. 核对日志中的执行顺序，并确认最终输出两个包的版本。

## 验收标准

- 五个脚本职责清晰，四个单包脚本可由批量入口独立调用。
- 默认容器和容器覆盖参数均按接口设计生效。
- 所有预检均发生在首次移除之前。
- `gotoTools` 始终先于 `gotoBase` 移除，`gotoBase` 始终先于 `gotoTools` 安装。
- 包不存在不会导致移除步骤失败。
- 任一步真实失败都会停止后续步骤并传递非零退出码。
- 安装阶段不修改 Roxygen 生成文件、不升级依赖。
- 完整执行后，新的 R 进程能够加载两个包并输出版本。
- 整个流程不会重启目标容器。
