# Module Type 7 by Mark Column Index 设计

> 前置计划：[2026-05-28-module-type-7-upload-reset](../plans/2026-05-28-module-type-7-upload-reset.md)

## 背景

`moduleType === 7` 的分类参数在上传文件、保存标记后调用 `reSetParamSettings` 刷新内容。当前存在两个问题：

1. **by 为空时不应进入重置流程** — 初始默认状态下 `by` 为空，不应尝试解析标记列。
2. **多标记列时 fallback 错误** — 当 `byFileMarkColumnList` 有多个标记列且用户选中的是第 N 个的列名时，`resolveBoundMarkHeaderInfo` 用 `candidates[0]` 做 fallback，导致错误地用第一个标记列的内容填充。

## 目标

- `by` 为空时直接跳过重置，不解析标记列
- 记住 `classifyItem.by` 对应 `byFileMarkColumnList` 中的索引，保存标记后用索引精确定位刷新

## 方案

### 纯前端实现，不持久化，不依赖后端

通过 `classifyItem.byMarkColumnIndex` 字段在运行时追踪索引：
- **首次加载**：在 `obtainPlotData` 中调用 `computeByMarkColumnIndex` 全局计算（传入 fileSettings 解析每个标记列的当前 columnName）
- **用户标记列时**：UI 组件同步写入 `byMarkColumnIndex`（组件内已知当前操作的索引）
- **重置时**：`resolveBoundMarkHeaderInfo` 用索引精确定位，无 fallback
- **兼容旧数据**：`byMarkColumnIndex` 为 null/undefined 时回退 columnName 全局匹配

### 改动范围

| 文件 | 改动 |
|------|------|
| `src/constant/paramSettingsReset.js` | 改 3 个函数 + 新增 1 个 + 注释 1 个 |
| `src/views/goto/components/form/plotSetting/classifyParam.vue` | 写 `classifyItem` 时同步 `byMarkColumnIndex` |
| `src/views/goto/plot/type/draw.vue` | `obtainPlotData` 中调用 `computeByMarkColumnIndex` |

### 详细设计

#### 1. `shouldProcessClassifyItem` — by 为空直接跳过

```js
function shouldProcessClassifyItem(classifyItem) {
  if (!classifyItem.by) {
    console.log(`跳过处理：by 为空 ${classifyItem.paramName || ''}`)
    return false
  }
  return true
}
```

`hasValidConfiguration` 整体注释保留。

#### 2. 新增 `computeByMarkColumnIndex(topParamSettingsList, fileSettings)`

遍历 `topParamSettingsList`，对 `moduleType === 7`、`byFileMarkColumnList` 非空、`classifyItemList` 非空的 setting：

对每个 `classifyItem`：
- `by` 为空 → `byMarkColumnIndex = null`
- `by` 非空 → 遍历 `byFileMarkColumnList`，用 `resolveHeaderInfoForMarkColumn` 解析每个标记列的当前 `headerInfo`，找 `headerInfo.columnName === classifyItem.by` 的那个索引写入

导出供 `draw.vue` 调用。

#### 3. `resolveBoundMarkHeaderInfo` — 索引用定位

```
有 byMarkColumnIndex → 直接用索引定位 markColumn → resolveHeaderInfoForMarkColumn
  columnName 匹配 → BOUND_MATCHED
  列名不匹配或被清空 → BINDING_MISSING
无 byMarkColumnIndex → 回退 columnName 全局匹配（兼容旧数据）
```

不再使用 `candidates[0]` fallback。原有的扁平匹配逻辑提取为内部函数 `resolveByColumnName`。

#### 4. `classifyParam.vue` — 标记时同步索引

在设置 `classifyItem.by` 的代码位置，同步写入 `classifyItem.byMarkColumnIndex`（组件内已知当前操作的 `byFileMarkColumnList` 索引）。重置 `by` 时设为 `null`。

#### 5. `draw.vue` — 首次加载计算

在 `obtainPlotData` 的 `this.plot = res.data` 之后调用：

```js
computeByMarkColumnIndex(
  this.plot.userDraw.paramSettingsList,
  this.plot.userDraw.fileSettings
)
```

### 边界情况

| 场景 | 行为 |
|------|------|
| by 为空（初始状态） | shouldProcessClassifyItem 返回 false，跳过 |
| 单标记列，by 有值 | 现有逻辑不受影响 |
| 多标记列，选中第 2 个 | 索引定位到第 2 个，不再 fallback 到第 1 个 |
| 对应索引的标记列未标记 | BINDING_MISSING，跳过不刷新 |
| byFileMarkColumnList 被删除/重排 | byMarkColumnIndex 越界 → BINDING_MISSING |
| 旧数据无 byMarkColumnIndex | 回退 columnName 全局匹配 |
| 用户重设 by（清空） | byMarkColumnIndex 设为 null |

## 不涉及

- 不修改后端接口
- 不修改 `associationParamModify.js` 调用链
- 不影响 `moduleType === 22` 多分类逻辑
