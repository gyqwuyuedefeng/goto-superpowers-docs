# 分类图例 by 增量同步设计

## 背景

`goto-web/src/constant/associationParamModify.js` 在上传文件成功、保存文件标记成功、保存文件成功等回调中会先调用 `reSetParamSettings(commonParams)`，再调用 `classifyLegendRefresh(commonParams)` 刷新分类图例参数。

当前 `reSetParamSettings` 只处理分类参数本身，主要覆盖 `moduleType === 7` 和 `moduleType === 22`。它会基于当前文件列信息校验分类项中的 `by` 是否仍然有效，并在无效时清空或重置分类项。

`moduleType === 28` 对应 `GClassifyLegendParam`，实际数据来自 `paramName === "legendList"` 的参数。现有 `classifyLegendRefresh` 会根据所有分类参数中的 `by` 全量重建 `legendList.value`。这种方式可以补齐图例项，但会覆盖仍然有效的图例配置，例如用户调整过的标题、方向、列数、刻度数和是否显示图例。

## 目标

调整分类图例刷新逻辑，使 `legendList.value` 按当前有效分类 `by` 增量同步：

- 当前图例项的 `legend.by.value` 仍存在于有效 `by` 集合中时，保留原对象，不改动 `legend` 内任何配置。
- 当前图例项的 `legend.by.value` 不存在于有效 `by` 集合中时，直接从 `legendList.value` 删除该项。
- 当前有效 `by` 没有对应图例项时，使用 `window.defaultClassifyLegend` 创建默认图例项并补齐。
- 保留现有类型归并规则：同一 `by` 同时被离散和连续分类使用时 `type = 3`，仅离散为 `1`，仅连续为 `2`。

## 非目标

- 不修改 `GClassifyLegendParam` 的展示布局和交互。
- 不调整 `moduleRegistry.js` 的 `28: 'GClassifyLegendParam'` 注册关系。
- 不改变 `reSetParamSettings` 对分类参数 `by` 的清理规则。
- 不额外扩展 `classifyLegendRefresh` 的图例来源范围。当前刷新逻辑只从 `moduleType === 7` 收集分类 `by`，本次改动保持该范围不变。

## 方案

改造 `goto-web/src/constant/classifyLegendRefresh.js`，将现有“全量重建”改为“增量同步”。

核心流程：

1. 查找 `paramName === "legendList"` 的分类图例参数。如果不存在，直接返回。
2. 遍历 `topParamSettingsList` 中的分组参数，收集 `moduleType === 7` 分类参数第一个 `classifyItem` 的非空 `by` 和 `type`，生成 `byTypeMap`。
3. 将现有 `classifyLegendSetting.value` 视为旧图例列表。遍历旧列表：
   - 读取 `item.legend.by.value`。
   - 如果该 `by` 在 `byTypeMap` 中，保留当前 `item` 对象，并更新外层 `item.type` 为归并后的类型。
   - 如果该 `by` 不在 `byTypeMap` 中，不放入新列表，相当于删除该项。
   - 如果旧列表存在重复 `by`，只保留第一项，避免同一分类图例重复显示。
4. 遍历 `byTypeMap`，对尚未保留的 `by` 创建新图例项：
   - 深拷贝 `window.defaultClassifyLegend`。
   - 设置 `legend.title.value = by`。
   - 设置 `legend.by.value = by`。
   - 设置外层 `type` 为归并后的类型。
5. 将同步后的数组赋值回 `classifyLegendSetting.value`。

已存在图例项只允许更新外层 `type`，不改 `legend` 内部字段。这样既能让 UI 根据类型显示列数或刻度数，又不会覆盖用户对图例配置的修改。

## 边界处理

- `classifyLegendSetting.value` 为空：只为当前有效 `by` 创建默认项。
- `window.defaultClassifyLegend` 不存在：保留有效旧项并删除失效旧项；无法为新增 `by` 创建默认项时跳过新增，避免生成结构不完整的数据。
- 旧项缺少 `legend`、`legend.by` 或 `legend.by.value`：视为无效项并删除。
- 当前没有任何有效 `by`：`legendList.value` 被同步为空数组。
- 重复旧项：同一 `by` 只保留第一项，后续重复项删除。

## 测试设计

新增或补充 `classifyLegendRefresh` 的单元测试，测试数据只覆盖当前函数既有支持的 `moduleType === 7` 分类参数，覆盖以下场景：

- 有效 `by` 已存在时，图例项对象引用和 `legend` 内配置保持不变。
- 失效 `by` 对应图例项被删除。
- 新增有效 `by` 会基于 `window.defaultClassifyLegend` 补齐默认图例项。
- 同一 `by` 同时出现离散和连续类型时，外层 `type` 更新为 `3`。
- 默认模板缺失时，不创建结构不完整的新图例项。

验证命令优先使用前端现有 Jest 入口：

```bash
cd goto-web
node scripts/run-unit-tests.js tests/unit/constants/classifyLegendRefresh.spec.js
```

如果依赖环境不足导致无法运行，需要记录失败原因和已完成的静态验证。

## 验收标准

- 上传文件成功或保存文件标记成功后，`legendList.value` 中失效 `by` 的图例项会被删除。
- 仍然有效的 `by` 对应图例项不会被重建，用户已配置的图例字段保持不变。
- 新出现的有效 `by` 会自动出现默认图例项。
- `moduleType === 28` 仍由 `GClassifyLegendParam` 渲染，无需组件额外承担数据清理职责。
