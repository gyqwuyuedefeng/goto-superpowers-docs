# goto-web 全局复选框样式统一设计

## 1. 背景

`goto-web` 当前的复选框样式存在明显分裂，至少包括以下三类实现：

1. `Element UI el-checkbox`
2. 原生 `input[type='checkbox']` + 自绘 `checkmark`
3. `Handsontable` 内部和表头手写的原生 checkbox

用户最先指出的两处问题是：

- `goto-web/src/views/components/handsontable.vue` 中的表头全选框和单元格勾选框
- `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue` 中的紧凑型 `el-checkbox`

实际扫描 `goto-web/src/views/goto` 后，确认复选框分布比预期更广，典型入口还包括：

- `goto-web/src/views/goto/components/form/draw/checkbox.vue`
- `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- `goto-web/src/views/goto/components/form/plotSetting/classifyParam.vue`
- `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- `goto-web/src/views/goto/process/components/Step1Upload.vue`
- `goto-web/src/views/goto/process/components/Step5Transform.vue`
- `goto-web/src/views/goto/components/SyncIcon.vue`

这些复选框当前在尺寸、圆角、边框、选中态、半选态、深色模式表现上并不统一。结果是：

1. 同一业务域内的控件看起来不像同一套系统
2. 深色主题下部分复选框对比度不足或风格跳脱
3. 新页面继续沿用各自写法，会持续放大视觉漂移

本轮目标不是重构复选框实现，而是在不改行为逻辑的前提下，为 `goto-web` 尤其是 `src/views/goto` 建立统一的复选框视觉基线。

## 2. 已确认约束

本次设计基于以下已确认约束：

1. 本轮只统一视觉样式和深浅主题，不主动改交互实现
2. 保留现有控件实现方式，不强制统一为同一个组件
3. 统一范围以 `goto-web/src/views/goto` 为主，同时覆盖 `goto-web/src/views/components/handsontable.vue`
4. 所有共享样式资产放在 `goto-web/src/assets/styles`
5. 视觉方向采用 `B. Soft Panel`
6. `Handsontable` 等高密度场景允许采用更紧凑的同家族变体

## 3. 目标

### 3.1 视觉目标

- 让 `goto` 目录里的复选框看起来属于同一套设计语言
- 让浅色和深色主题下的复选框都具备稳定的层次和可读性
- 让配置面板、分组卡片、列表项中的复选框具备更贴近当前绘图面板的“轻面板感”
- 让半选态、禁用态、hover 和 focus 表现具备一致语义

### 3.2 工程目标

- 建立一层全局复选框共享样式，避免继续在组件内重复硬编码
- 覆盖 `Element UI`、原生 checkbox、自绘 checkbox 三种主要家族
- 允许通过少量尺寸变体支持常规表单和高密度表格
- 控制改造范围，避免引入业务行为回归

### 3.3 维护目标

- 后续页面新增复选框时，优先复用全局基线而不是从零写样式
- 页面级样式只允许调整布局和密度，不再各自定义颜色、圆角和选中态
- 降低局部 `scoped` 样式和深色模式补丁长期分叉的风险

## 4. 非目标

本轮不做以下事情：

1. 不重构 checkbox 组件体系
2. 不把所有复选框统一替换为同一种 DOM 实现
3. 不修改 `v-model`、`@input`、`@change` 等业务行为
4. 不处理 radio、switch 等其他选择控件
5. 不顺带重做页面布局或表单结构
6. 不在本设计阶段直接修改业务代码

## 5. 现状分层与覆盖范围

### 5.1 现有复选框家族

#### A. Element UI 家族

典型文件：

- `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
- `goto-web/src/views/goto/components/form/plotSetting/classifyParam.vue`
- `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- `goto-web/src/views/goto/process/components/Step1Upload.vue`
- `goto-web/src/views/goto/process/components/Step5Transform.vue`

特点：

- 有现成的 checked / indeterminate / disabled 状态
- 当前不少页面通过 `::v-deep` 做局部覆盖，导致视觉差异较大

#### B. 原生自绘家族

典型文件：

- `goto-web/src/views/goto/components/form/draw/checkbox.vue`
- `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`

特点：

- 通过隐藏原生 input，再自绘 `checkmark`
- 当前多处样式接近但不完全一致，且深色模式适配不足

#### C. 表格 / 特殊场景家族

典型文件：

- `goto-web/src/views/components/handsontable.vue`
- `goto-web/src/views/goto/components/form/draw/list.vue`

特点：

- 以紧凑密度为主
- 表头全选框、浮动操作条伪 checkbox 有独立视觉
- 需要与全局风格统一，但不能牺牲高密度场景可用性

### 5.2 本轮纳入统一范围

本轮统一分为三层：

1. 全局统一层
2. 重点适配层
3. 允许保留的局部例外

#### 全局统一层

全局统一层负责建立复选框的共享视觉规范，包括：

- 尺寸
- 圆角
- 边框
- 背景
- 勾形状
- 半选态
- 禁用态
- focus ring
- label 间距
- 深浅主题 token 映射

#### 重点适配层

以下入口需要显式核对和收口：

- `goto-web/src/views/components/handsontable.vue`
- `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
- `goto-web/src/views/goto/components/form/draw/checkbox.vue`
- `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- `goto-web/src/views/goto/components/form/plotSetting/classifyParam.vue`
- `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- `goto-web/src/views/goto/components/form/draw/list.vue`

#### 允许保留的局部例外

只允许保留布局和密度差异，不允许保留视觉语言差异：

- `classifyLegendParam.vue` 可保留无 label 的紧凑尺寸
- `handsontable.vue` 可保留表格紧凑密度
- `list.vue` 的悬浮勾选入口可保留位置和交互结构

以上例外仍必须回收到统一的颜色、边框、圆角和勾形状体系中。

## 6. 方案决策

### 6.1 采用 B. Soft Panel 方向

最终采用 `B. Soft Panel` 作为统一基线。

原因：

1. 最贴近 `goto` 目录中当前的面板、卡片和绘图配置区视觉语言
2. 深色主题下比传统默认方框更稳定，更不容易发灰或发空
3. 既能服务配置面板，又能通过紧凑变体兼容表格场景

### 6.2 不统一 DOM，实现视觉统一

本轮明确不追求“所有复选框都换成同一组件”，而是采用以下策略：

- `el-checkbox` 继续使用 `el-checkbox`
- 自绘 checkbox 继续保留原有 DOM 结构
- `Handsontable` 和原生 checkbox 继续保留现有交互

统一的是视觉结果，而不是内部实现。

### 6.3 默认 + 紧凑双密度

为避免一套尺寸同时压进配置面板和表格导致不协调，本轮采用双密度：

1. 默认密度：面板、分组卡片、表单场景
2. 紧凑密度：高密度列表、表头全选框、表格单元格勾选框

## 7. 视觉规范

### 7.1 基础尺寸

推荐尺寸如下：

- 常规表单 / 配置区：`18px`
- 紧凑场景：`16px`
- `Handsontable` / 高密度表格：`15px` 到 `16px`

### 7.2 圆角

- 默认圆角：`6px`
- 紧凑变体：`4px`

### 7.3 未选中态

- 背景：使用 `surface-elevated`
- 边框：使用 `border-default`
- hover：边框提到 `border-strong`
- 浅色模式保持轻面板感，不使用纯黑边框
- 深色模式保持清晰边界，不使用过低对比度的灰边

### 7.4 选中态

- 背景：`accent-solid`
- 边框：`accent-solid`
- 勾颜色：`text-on-accent`
- 勾线宽略强于当前旧样式，避免深色模式下发虚

### 7.5 半选态

- 半选态与选中态共享底色语义
- 中间短横为白色
- 不单独使用“灰色半选”视觉，避免语义不清

### 7.6 禁用态

- 通过透明度和对比下降表达禁用
- 但必须保证选中 / 未选中的语义仍可区分
- 禁止出现禁用后几乎不可见的情况

### 7.7 Focus 与无障碍

- focus ring 使用 `focus-ring`
- 不依赖 hover 才能识别控件边界
- 主要状态差异同时依赖颜色和形状，不仅依赖透明度

### 7.8 Label 间距

- checkbox 与文字间距统一
- 无 label 场景只允许调整尺寸，不允许重新定义色值和圆角

## 8. 主题映射

本轮不引入新的主题状态源，继续沿用现有：

- `html[data-theme='light']`
- `html[data-theme='dark']`

复选框样式应尽量直接建立在现有语义 token 上，包括：

- `--surface-elevated`
- `--border-default`
- `--border-strong`
- `--accent-solid`
- `--text-on-accent`
- `--focus-ring`
- `--text-secondary`
- `--text-muted`

这样可以减少专门为 checkbox 再造一套平行主题变量的必要性。

如确有必要补充 checkbox 专用 token，应保持数量最少，并由现有语义 token 派生。

## 9. 样式架构

### 9.1 文件组织

建议在 `goto-web/src/assets/styles` 下新增一份复选框共享样式文件，例如：

- `goto-web/src/assets/styles/checkbox.scss`

再由全局入口引入：

- `goto-web/src/assets/styles/index.scss`

这样可以确保：

1. checkbox 基线样式集中维护
2. 所有页面默认共享一套视觉语言
3. 局部组件不再需要反复写颜色和状态规则

### 9.2 全局职责

全局共享样式负责：

- `Element UI el-checkbox` 的视觉覆盖
- 原生 checkbox / 自绘 checkbox 的 token 化
- 紧凑变体的通用类
- `Handsontable` 相关 checkbox 的公共基线

### 9.3 局部职责

局部组件样式只负责：

- 容器布局
- 与相邻文案的间距
- 特定场景的密度切换

局部组件不应再负责：

- 自定义主色
- 自定义边框色
- 自定义选中背景
- 自定义勾形态
- 自定义深色模式色值

### 9.4 推荐变体

推荐保留两类通用变体：

1. 默认变体
2. `compact` 变体

现有局部类名可以继续存在，但语义上应映射到共享变体，例如：

- `compact-checkbox` -> `compact`
- `select-all-checkbox` -> `compact`

## 10. 重点适配说明

### 10.1 classifyLegendParam

`goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`

要求：

- 保留当前无 label 的紧凑型 checkbox 布局
- 回收当前局部硬编码的尺寸、边框、选中色值
- 与全局 `compact` 变体对齐

### 10.2 Handsontable

`goto-web/src/views/components/handsontable.vue`

要求：

- 表头全选框与单元格勾选框视觉统一
- 保持高密度，不引入过大的点击区视觉
- 支持深色主题时的边框、底色和选中态对比

### 10.3 原生自绘 checkbox

涉及：

- `goto-web/src/views/goto/components/form/draw/checkbox.vue`
- `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`

要求：

- 保留现有 input + checkmark 结构
- 统一到同一套颜色、尺寸、圆角和勾形状
- 深色模式下不再依赖浅色写死色值

### 10.4 列表和悬浮操作条

涉及：

- `goto-web/src/views/goto/components/form/draw/list.vue`

要求：

- 保留悬浮勾选入口的定位和交互节奏
- 让伪 checkbox 的视觉与主体系一致
- 避免它继续成为一套独立按钮风格

## 11. 风险与控制

### 11.1 scoped 样式优先级冲突

风险：

- 当前多个组件使用 `scoped` + `::v-deep` 覆盖 `el-checkbox`
- 新的全局基线可能被旧规则反向覆盖

控制方式：

- 改造时同步清理已冗余的局部视觉覆盖
- 仅保留布局相关局部规则

### 11.2 表格密度风险

风险：

- 默认 `Soft Panel` 尺寸直接用于表格时可能显得拥挤

控制方式：

- 为表格场景使用 `compact` 变体
- 单独核对 `Handsontable` 表头与单元格

### 11.3 自绘 checkbox 焦点与状态不一致

风险：

- 自绘实现容易出现 focus 态缺失、禁用态不统一、深色模式漏覆盖

控制方式：

- 统一状态定义
- 把关键色值切到主题 token

### 11.4 漏网页面

风险：

- `goto` 目录里仍可能有分散的复选框局部样式未被收口

控制方式：

- 实施前后分别做一次全局搜索
- 以三类关键词核对：`el-checkbox`、`type="checkbox"`、自绘容器类名

## 12. 验证标准

实施完成后，至少满足以下验收条件：

1. `goto-web/src/views/goto` 中主要复选框家族在视觉上属于同一系统
2. `Handsontable` 表头全选框和单元格勾选框在浅色 / 深色下均可读
3. 以下状态表现一致：
   - 默认
   - hover
   - checked
   - indeterminate
   - disabled
   - focus
4. `classifyLegendParam`、`MarkColumnSelector`、`classifyParam`、`draw/checkbox`、`checkboxBasicStatisticsMethod` 的局部硬编码样式得到收口
5. 深色模式下不再出现明显发灰、发空、对比不足的问题

## 13. 实施顺序建议

为降低回归风险，建议按以下顺序执行：

1. 建立全局 checkbox 共享样式文件并接入入口
2. 先覆盖 `Element UI` 复选框基线
3. 再统一原生 / 自绘 checkbox
4. 最后单独处理 `Handsontable` 和高密度列表变体
5. 通过全局搜索和页面人工核对做收尾

## 14. 结论

本轮采用 `B. Soft Panel` 作为 `goto-web` 复选框统一基线。

策略不是重构实现，而是建立一层共享视觉规则，统一 `Element UI`、原生、自绘和表格场景中的复选框外观，并通过默认 / 紧凑双密度兼顾面板和高密度表格。

这样可以在较低行为风险下，显著提升 `goto-web/src/views/goto` 的视觉一致性和深浅主题稳定性，并为后续复选框新增场景提供清晰的复用基线。
