# 堆叠柱状图视觉属性联动设计

## 1. 背景

堆叠柱状图新增了环形样式能力，相关配置分散在“视觉属性”“角度轴设置”“径向轴设置”“X轴设置”“Y轴设置”“分面参数”“标签设置”等 tab 中。

当前需要让这些参数根据 X 轴标记列类型、视觉属性样式、分面类型和标签位置自动显隐或禁用，避免用户选择出不合法或无效的参数组合。

本设计只覆盖堆叠柱状图的参数联动，不调整后端绘图逻辑，不改动参数定义数据源。

## 2. 目标

1. X 轴为数值列时，“视觉属性 > 样式”禁用两个环形选项，并在当前值非法时回退到默认样式。
2. 样式为默认时隐藏环形专属参数和环形轴 tab；样式为环形时隐藏笛卡尔轴 tab。
3. 环形分面标签位置根据当前环形样式动态禁用不适用选项。
4. 标签设置根据“系列标签位置”和“数值标签位置”动态隐藏相关参数。
5. 规则集中在堆叠柱状图修改逻辑内，尽量复用现有公共函数和 Vue 响应式写法。

## 3. 非目标

1. 不新增公共业务规则到 `publicFunction.js`，除非实现时发现已有 API 无法覆盖。
2. 不修改图形参数原始 JSON 模板。
3. 不改变分面参数已有的 `facetParamsModify` 行为，只在堆叠柱状图层追加本图形的约束。
4. 不隐藏“标签设置”tab 本身，因为用户仍需通过两个标签位置参数重新开启标签。

## 4. 方案决策

采用“堆叠柱状图专属联动集中在 `stackBarModify.js`”方案。

原因：

1. 这些规则均绑定堆叠柱状图的参数语义，放在 `stackBarModify.js` 风险最低。
2. `publicFunction.js` 已提供标记列查询、参数查询、分面修改等基础能力，不需要把本需求提升到公共层。
3. 视图层只补齐下拉禁用索引的透传，业务规则不进入 Vue 组件。

## 5. 涉及文件

主要修改：

- [stackBarModify.js](/mnt/f/IdeaProjects/goto-software/goto-web/src/constant/modify/stackBarModify.js)
- [facetParams.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/goto/components/form/draw/facetParams.vue)

可能复用：

- [publicFunction.js](/mnt/f/IdeaProjects/goto-software/goto-web/src/constant/modify/publicFunction.js)
- [associationParamModify.js](/mnt/f/IdeaProjects/goto-software/goto-web/src/constant/associationParamModify.js)

## 6. 数据与入口

`stackBarModify.js` 已经接入以下入口：

1. `doStackBarModify(commonParams)`：用户修改参数时触发。
2. `stackBarSaveFileMarkSuccess(commonParams)`：保存文件列标记成功后触发。
3. `stackBarSaveFileSuccess(commonParams)`：保存文件成功后触发。

X 轴标记列判断使用现有数据：

1. 通过 `fileMarkContainerList` 中 `field === "x_column"` 找到 `positionParam`。
2. 通过 `headerInfoList.status` 对应位判断列是否被标记。
3. 通过标记列的 `numberColumn` 判断是否为数值列。

实现优先使用：

- `PublicFunctions.findFilteredHeaderInfoListByGroupIdAndKey(commonParams, "x_column")`
- `localFindFromTopParamSettingsList`
- `localFindGroupFromTopParamSettingsListByName`
- `setSettingsVisible`

## 7. 视觉属性样式联动

### 7.1 X 轴数值列限制

保存文件标记成功时执行：

1. 查询 `x_column` 标记列。
2. 如果任一有效 X 轴标记列 `numberColumn === true`，则将“视觉属性 > 样式”的 `circular_x`、`circular_y` 加入 `disabledIndexList`。
3. 如果当前样式值是 `circular_x` 或 `circular_y`，自动回退为 `default`。
4. 如果 X 轴不是数值列，清空样式的 `disabledIndexList`。

回退到 `default` 后继续执行样式显隐联动，保证 UI 状态与当前值一致。

### 7.2 样式默认值

当样式值为 `default`：

1. 显示 “X轴设置” tab。
2. 显示 “Y轴设置” tab。
3. 隐藏 “角度轴设置” tab。
4. 隐藏 “径向轴设置” tab。
5. 隐藏视觉属性中的 `start`、`end`、`innerRadius`。

### 7.3 样式为环形

当样式值为 `circular_x` 或 `circular_y`：

1. 隐藏 “X轴设置” tab。
2. 隐藏 “Y轴设置” tab。
3. 显示 “角度轴设置” tab。
4. 显示 “径向轴设置” tab。
5. 显示视觉属性中的 `start`、`end`、`innerRadius`。

## 8. 环形分面联动

触发时机：

1. “视觉属性 > 样式”变化。
2. `facetParams.type` 变化。
3. X 轴数值列导致样式自动回退。

规则：

1. 读取当前样式值。
2. 读取 `facetParams.type.value.value`。
3. 仅当分面类型为 `radial` 时设置 `radialStripPosition` 的禁用项。
4. 样式为 `circular_x` 时，禁用 `start`、`end`。
5. 样式为 `circular_y` 时，禁用 `inside`、`outside`。
6. 其他情况清空禁用项。
7. 如果当前 `radialStripPosition.value.value` 命中禁用项，回退到第一个未禁用选项。

`facetParams.vue` 需要给环形分面标签位置的 `GSelectObject` 透传 `disabledIndexList`，使 `stackBarModify.js` 设置的禁用项在 UI 中生效。

## 9. 标签设置联动

触发时机：

1. `classLabelPos` 变化。
2. `valueLabelPos` 变化。
3. `type` 变化后调用一次，保证样式切换后的界面状态一致。

规则：

1. `classLabelPos === "none"` 时隐藏 `classLabelOrientation`、`classLabels`。
2. `classLabelPos !== "none"` 时显示 `classLabelOrientation`、`classLabels`。
3. `valueLabelPos === "none"` 时隐藏 `valueLabelOrientation`、`valueLabelDigits`、`valueLabels`。
4. `valueLabelPos !== "none"` 时显示 `valueLabelOrientation`、`valueLabelDigits`、`valueLabels`。
5. 当 `classLabelPos` 与 `valueLabelPos` 都是 `none` 时，保留这两个位置参数可见，隐藏该 tab 下其它参数，包括 `autofitAxis`。
6. 当任一标签位置不是 `none` 时，显示 `autofitAxis`，并按前述规则显示对应标签参数。

## 10. 状态更新与持久化

实现需要同时保证当前页面响应式更新和后续刷新后状态一致。

建议：

1. 对新增或替换的 `disabledIndexList` 使用 `commonParams.vueContext.$set`。
2. 对普通 `visible` 和 `value` 变更沿用现有直接赋值风格。
3. 当自动回退样式或分面标签位置时，调用 `PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId` 保存被回退的参数。
4. 仅显隐变化不额外强制保存，保持与现有参数联动逻辑一致。

## 11. 测试策略

手工验证：

1. X 轴标记为非数值列时，样式三个选项都可选。
2. X 轴标记为数值列时，`circular_x`、`circular_y` 被禁用。
3. 当前样式为环形时，将 X 轴改标记为数值列，保存后样式回退到 `default`。
4. 样式为 `default` 时，视觉属性隐藏 `start`、`end`、`innerRadius`，隐藏角度轴和径向轴 tab，显示 X/Y 轴 tab。
5. 样式为 `circular_x` 或 `circular_y` 时，显示环形参数和环形轴 tab，隐藏 X/Y 轴 tab。
6. 分面类型为环形分面且样式为 `circular_x` 时，环形分面标签位置禁用“起始”“终止”。
7. 分面类型为环形分面且样式为 `circular_y` 时，环形分面标签位置禁用“内侧”“外侧”。
8. 被禁用的环形分面标签位置如果正好是当前值，会回退到第一个可用值。
9. 系列标签位置为不显示时，系列标签方向和字体隐藏。
10. 数值标签位置为不显示时，数值标签方向、小数位和字体隐藏。
11. 两个标签位置都为不显示时，标签设置 tab 下除两个位置参数以外的参数都隐藏。

自动验证：

1. 运行 `goto-web` 现有 lint 或构建命令，确认语法无误。
2. 如仓库没有针对 modify 常量逻辑的单元测试，本次不新增测试框架。

## 12. 风险与缓解

风险：`disabledIndexList` 挂载层级不一致导致 UI 不生效。

缓解：实现时确认 `GSelectObject` 实际 props 来源。普通参数使用参数本体的 `disabledIndexList`，`radialStripPosition` 如果数据嵌套在 `.value` 下，则组件透传时兼容两处。

风险：自动回退值后没有持久化，刷新又恢复非法值。

缓解：只要发生值回退，就调用现有保存设置 API 持久化该参数。

风险：tab 名称依赖中文名称精确匹配。

缓解：沿用现有 `localFindGroupFromTopParamSettingsListByName` 模式，按当前参数组名匹配；找不到 group 时函数应静默跳过，不影响其它联动。

