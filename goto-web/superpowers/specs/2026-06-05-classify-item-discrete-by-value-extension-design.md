# Classify Item 离散 by 值扩展设计

## 背景

`src/views/goto/components/form/draw/classifyItem.vue` 是分类参数的展示组件，实际 by 选择和 `valueList` 重建逻辑在 `src/views/goto/components/form/classifyItemMixins.js` 与 `src/constant/modify/publicFunction.js`。

当前用户选择新的 by 后，`PublicFunctions.doClassifyItemChooseColumnFinish` 会调用 `fillChooseColumnUniqueList` 按新列的唯一值重建离散 `valueList`。颜色类型会重新按模板颜色填充，非颜色类型不会继承旧 by 已配置的值。因此当新 by 的离散值数量超过旧 by 时，用户原先配置的大小、粗细、形状、线型等值会丢失或回到默认值。

上传/重置路径 `src/constant/paramSettingsReset.js` 已经有更接近目标的恢复策略：

- 颜色：`restoreClassifyItemValues` 调用 `PublicFunctions.reserveColorForClassifyItem`，保留旧颜色，不足时按模板色/随机色补齐。
- 非颜色：`syncValueListByKeyAndFallback` 先按 key/name 匹配，未匹配时按旧列表顺序继承，旧列表耗尽后使用旧列表最后一个有效值。

本需求需要让手动选择 by 的离散映射路径也遵循同一类扩展规则。

## 目标

- 用户选择 by 且映射模式为离散映射时，新 by 的离散内容数量超过旧 by 内容数量后自动补齐缺失配置。
- 非颜色类型缺失项使用旧 `valueList` 最后一项的 `value` 补齐。
- 颜色类型缺失项使用现有颜色逻辑补齐：保留旧颜色，不足时从模板色取值，再不足时随机生成。
- 已有名称/key 能匹配的项优先保留原值，避免仅按索引导致同名类别配置丢失。
- 不改变连续映射行为。
- 不改变 by 选择弹窗 UI 和事件流。

## 推荐方案

在 `PublicFunctions.doClassifyItemChooseColumnFinish` 内复用现有恢复策略，保持 by 选择路径与上传/重置路径一致。为避免 `publicFunction.js` 反向 import `paramSettingsReset.js` 形成循环依赖，非颜色同步逻辑抽到新的中立 helper 文件，由两个调用方共同使用。

### 数据流

1. `classifyItemMixins.chooseColumnFinish` 调用 `PublicFunctions.doClassifyItemChooseColumnFinish`。
2. `doClassifyItemChooseColumnFinish` 从弹窗深拷贝列头中找到用户选择的 `headerInfo`。
3. 在重建前深拷贝当前 `classifyItem` 为 `classifyItemBackup`。
4. 调用 `fillChooseColumnUniqueList`，按新 by 的 `chooseColumnUniqueList` 重建 `classifyItem.valueList`。
5. 如果当前是离散映射，执行值恢复：
   - `valueType === COLOR`：调用 `reserveColorForClassifyItem(commonParams, classifyItem, classifyItemBackup)`。
   - `valueType` 是 `SIZE`、`THICKNESS`、`SHAPE`、`LINE_TYPE`：调用中立 helper 中的非颜色恢复函数。
6. 继续设置 legend title、`by`、`byFileSettingIndex`，并返回 `headerInfo`。

### 非颜色恢复规则

非颜色恢复函数应与 `paramSettingsReset.js` 中现有 `syncValueListByKeyAndFallback` 语义保持一致：

1. 新项先按 key/name 匹配旧项，匹配到则继承该旧项 `value`。
2. 匹配不到时，如果还有未消费的旧项，按旧项顺序继承。
3. 未消费旧项耗尽后，使用旧 `valueList` 中最后一个 `value !== undefined` 的值。
4. 旧列表为空或没有有效值时，不额外补齐，保留 `fillChooseColumnUniqueList` 生成的默认值。

示例：

旧 by: `A/B/C`，旧值: `1/2/3`
新 by: `X/Y/Z/W/V`
结果值: `1/2/3/3/3`

### 颜色恢复规则

颜色恢复继续使用 `reserveColorForClassifyItem`：

1. 先收集旧 `valueList` 的颜色值。
2. 如果颜色数量不足新 `valueList` 长度，调用 `obtainNeedSizeColorListExcludeExistingColors`。
3. 该函数优先使用 `ManageParamTemplateColor` 中未被占用的模板颜色。
4. 模板颜色不足时，使用现有随机颜色生成逻辑。

示例：

旧 by 3 项，旧颜色 3 个；新 by 5 项，则前 3 个保留旧颜色，后 2 个按模板色/随机色补齐。

## 改动范围

| 文件 | 改动 |
| --- | --- |
| `src/constant/classifyValueListSync.js` | 新增中立 helper，承载 `syncValueListByKeyAndFallback` 及其私有辅助函数 |
| `src/constant/modify/publicFunction.js` | 在 `doClassifyItemChooseColumnFinish` 中备份旧 `classifyItem`，重建后按类型恢复值；非颜色调用 `classifyValueListSync.js` |
| `src/constant/paramSettingsReset.js` | 删除本文件内重复的非颜色同步 helper，改为从 `classifyValueListSync.js` 导入并继续导出兼容测试 |
| `tests/unit/constant/modify/publicFunction.spec.js` 或相邻现有测试文件 | 新增 by 选择后离散值扩展的单元测试 |

## 边界情况

| 场景 | 行为 |
| --- | --- |
| 新 by 项数少于旧 by | 只保留新 `valueList` 对应项，多余旧值丢弃 |
| 新旧类别名/key 相同 | 优先按 key/name 继承原值 |
| 新类别名不同但数量未超过旧列表 | 按旧列表顺序继承 |
| 新类别数超过旧列表 | 非颜色使用最后一个旧值补齐 |
| 旧列表为空 | 保留新列表默认值 |
| 颜色类型数量不足 | 走现有模板色/随机色补齐 |
| 连续映射 | 不执行本恢复逻辑 |
| 用户清空 by | 不改变当前 `resetClassifyItemByDefault` 行为 |

## 测试设计

新增 focused 单元测试覆盖 `doClassifyItemChooseColumnFinish`：

1. 离散大小：旧 by 3 项，新 by 5 项，结果值为旧值前三项加最后旧值补齐。
2. 离散形状或线型：验证非数字类型同样按最后旧值补齐。
3. 离散颜色：旧 by 2 项，新 by 4 项，前 2 项保留旧颜色，后 2 项不为空且来自现有颜色补齐逻辑。
4. key/name 匹配优先：新旧列表存在同名项时继承同名项值，不被纯索引覆盖。
5. 连续映射：选择 by 后不触发离散扩展逻辑。

测试中的方法注释需要完整描述被测试对象和测试目的，符合项目测试规范。

## 不涉及

- 不修改 `classifyItem.vue` 模板和样式。
- 不修改选择 by 弹窗交互。
- 不修改后端接口。
- 不调整 `moduleType === 22` 多分类逻辑。
- 不清理已有 console 日志或做无关重构。
