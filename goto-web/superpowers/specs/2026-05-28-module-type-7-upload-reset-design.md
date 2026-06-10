# Module Type 7 上传重置设计

## 背景

`src/constant/associationParamModify.js` 会在上传、删除、文件保存、标记保存等事件后调用 `reSetParamSettings(commonParams)`。上传成功时，当前调用链如下：

1. `uploadFileModify`
2. `reSetParamSettings`
3. `PublicFunctions.uploadFileModify`
4. 图表分组自己的上传后置处理器
5. 分面和分类图例刷新

`reSetParamSettings` 目前处理 `moduleType === 7` 分类参数时，会在上传后的文件表头里查找 `classifyItem.by`。如果同名列存在，并且该列有任意非零 `status`，就刷新 `valueList`；否则会把分类项重置为默认值，并清空 `by`。

这个行为能保留一部分配置，但它没有把判断绑定到分类参数配置的标记列上。当新上传文件缺少原标记列时，它也会丢弃模板里的旧值。

## 目标

对于 `moduleType === 7`，文件上传和标记保存后的处理应尽可能保留现有模板数据。

当旧标记列在新文件中不存在，并且 `byFileMarkColumnList` 已经被清空时，不允许重置 `classifyItem.by` 和 `classifyItem.valueList`。用户会重新标记一个列并保存；保存后，组件应复用旧值，同时同步到新标记列的内容。

## 范围

范围内：

- `reSetParamSettings` 中 `moduleType === 7` 分类参数的重置行为。
- 使用 `setting.byFileMarkColumnList` 做标记列感知的查找。
- `CLASSIFY_MANUAL` 的 `valueList` 刷新和值保留。
- `CLASSIFY_CONTINUES` 最小化同步 `by`、标题、文件索引。
- 针对重置辅助逻辑补充测试。

范围外：

- `moduleType === 22` 分类参数列表行为。
- 标记列选择控件的 UI 改动。
- 后端上传行为。
- 重置逻辑之外的大范围重构；仅允许为测试抽取少量必要 helper。

## 设计

### 状态模型

对每个 `moduleType === 7` 分类参数，从 `setting.byFileMarkColumnList` 解析实际绑定的标记列表头，而不是用 `classifyItem.by` 去匹配任意 `status` 非零的列。

重置逻辑分为三个状态：

- `BOUND_MATCHED`：绑定的标记列存在，并且实际 `columnName` 等于 `classifyItem.by`。
- `BOUND_CHANGED`：绑定的标记列存在，但实际 `columnName` 与 `classifyItem.by` 不一致。
- `BINDING_MISSING`：`byFileMarkColumnList` 为空、配置的标记项不存在，或无法解析出实际选中的标记列。

`BOUND_MATCHED` 使用当前文件内容刷新列表，并保留旧值。

`BOUND_CHANGED` 将 `classifyItem.by`、`legend.title.value` 和 `byFileSettingIndex` 更新为新标记列，然后使用当前文件内容刷新列表，并保留旧值。

`BINDING_MISSING` 保持 `classifyItem.by` 和 `classifyItem.valueList` 不变。它不会调用现有默认重置路径。这样可以在用户重新标记并保存替代列之前保留模板值。

如果配置了多个绑定标记列，解析器应优先选择 `columnName` 等于旧 `classifyItem.by` 的表头。没有匹配项时，按 `byFileMarkColumnList` 顺序使用第一个成功解析的绑定表头。

### 标记列解析

解析器使用 `setting.byFileMarkColumnList`，其中每一项形如 `{ filedName, fileSettingIndex }`。

对每个配置的标记项：

1. 读取目标 `fileSetting`。
2. 在 `fileMarkContainerList` 中查找 `field === filedName` 且 `type === "SELECT_COLUMN"` 的项。
3. 使用该项的 `positionParam` 查找满足 `BitwiseUtils.getBit(headerInfo.status, positionParam) === 1` 的表头。
4. 按现有 helper 行为确保 `chooseColumnUniqueList` 可用。
5. 给解析出的表头附加 `fileSettingIndex`。

如果找不到标记容器，或该标记位没有选中的表头，则这个标记项视为未解析。

### 手动分类值保留

对于 `CLASSIFY_MANUAL`，刷新已解析的绑定列时，先用当前 `chooseColumnUniqueList` 重建 key，再恢复 value：

1. 优先按 `key` 匹配新旧 `valueList` 项。
2. 对没有旧 key 匹配的新 key，按旧列表顺序使用下一个旧 value。
3. 如果新列表比旧值更多：
   - 对 `COLOR`，保留现有颜色策略：先保留旧颜色，再使用模板色，最后使用随机唯一颜色。
   - 对非颜色值类型，重复使用最后一个旧的有效 `value`。
4. 如果新列表更短，只保留新 key。被移除的尾部项不额外保留。
5. 如果 `chooseColumnUniqueList` 为空，保持旧 `valueList` 不变。上传后或解析暂态下，相比替换为空列表，保留模板更重要。

这种 key 优先策略可以避免同一列数据顺序变化时 value 错位。

### 连续分类项

对于 `CLASSIFY_CONTINUES`，不重建手动离散的 `valueList` 项。

如果状态是 `BOUND_MATCHED` 或 `BOUND_CHANGED`，用解析出的表头同步 `by`、`legend.title.value` 和 `byFileSettingIndex`。如果状态是 `BINDING_MISSING`，保持分类项不变。

### 保留现有跳过行为

保留当前“分类项 `by` 为空但已有有效配置时跳过处理”的行为。这样可以避免对尚未绑定列、但有意保留模板配置的分类项做误重置。

## 错误处理

缺少 `topParamSettingsList`、缺少 `classifyItemList`、文件索引无效、缺少 `fileMarkContainerList`、标记列无法解析，都应作为非致命情况处理。

对于 `BINDING_MISSING`，保留当前分类项。不抛错，不清空 `by`，也不重置 `valueList`。

## 测试

围绕重置 helper 或当前测试体系中最小可行单元补充聚焦测试。

必测用例：

- `BOUND_MATCHED`：同名标记列刷新唯一值，并按 key 保留旧值。
- `BOUND_CHANGED`：新标记列更新 `by`、标题和文件索引，同时保留旧值。
- `BINDING_MISSING`：`byFileMarkColumnList` 为空或无法解析时，不清空 `by`，也不重置 `valueList`。
- 唯一值顺序变化：按 key 匹配后仍保留正确 value。
- 唯一值增加：颜色值使用现有颜色补齐行为；非颜色值重复最后一个旧 value。
- 唯一值减少：列表收缩到新 key。
- `chooseColumnUniqueList` 为空：旧 `valueList` 保持不变。

手工验证：

- 上传同标记列名但数据变化的新文件，确认值被复用。
- 上传缺少旧标记列的新文件，确认 `by` 和 `valueList` 保持不变。
- 重新标记到另一个列并保存，确认组件切换到新列，同时复用旧值。

