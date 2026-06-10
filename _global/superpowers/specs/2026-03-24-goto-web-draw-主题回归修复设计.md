# goto-web draw 主题回归修复设计

## 1. 背景

`goto-web` 已经按照此前的设计文档 [`2026-03-23-goto-web-draw-主题体系重拆设计.md`](/mnt/f/IdeaProjects/goto-software/docs/superpowers/specs/2026-03-23-goto-web-draw-主题体系重拆设计.md) 完成了 `draw` 主题体系重拆：

- `draw` 样式从全局入口改为显式 `draw-theme-scope`
- `draw` 基础控件不再直接依赖 `draw/index.scss`
- 页面布局入口与基础入口已开始拆分
- `html[data-theme]` 成为 `draw` 主题的唯一来源

重拆后的主流程已经可运行，但当前暴露出一批样式层回归问题。这些问题不是新的独立功能，而是重构后的第一轮收口修复，主要集中在以下三类：

1. `draw` 结果页 / 工具页的深色样式观感不统一
2. `draw` 基础控件在深色模式与溢出场景下仍有细节回归
3. 非 `draw` 页面中的深色样式补丁仍不完整

用户已经明确要求本轮不要把问题拆成孤立补丁，而是按共用根因分组修复，并在修复过程中把同类明显问题顺手收掉。

## 2. 问题清单

本轮确认的问题共 6 个：

### 2.1 `PlotResultCanvas` 结果页顶部区域深色观感错误

文件：

- `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`

现象：

- `el-tabs__header`、`el-tabs__nav`、`el-tabs__item` 虽然能跟随深浅主题切换，但深色主题下整体仍然更像浅色主题
- 结果页右上角放大 / 缩小 / 重置按钮区域只有按钮本身变暗，按钮容器和周围视觉层级仍偏浅

### 2.2 `PlotSidePanel` 上传文件区域没有占满宽度

文件：

- `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`

现象：

- 上传文件卡片所在区域没有拉满可用宽度
- 容器、上传器、文件卡片之间的宽度传递不完整，导致布局显得局促

### 2.3 `input.vue` 数字输入框加减按钮深色不对

文件：

- `goto-web/src/views/goto/components/form/draw/input.vue`

现象：

- 深色模式下，鼠标悬浮文本输入/数字输入时露出的加减按钮仍然是浅色/白色背景
- 按钮与深色输入框背景割裂

### 2.4 `selectMultipleObject.vue` 多选下拉框出现溢出

文件：

- `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`

现象：

- 已选项目较多时，多选标签区域溢出
- 当前行为相比重构前退化

### 2.5 `tool.vue` 右侧操作面板深色模式不完整

文件：

- `goto-web/src/views/goto/toolsBox/type/tool.vue`

现象：

- 多处面板背景在深色模式下仍然发白
- 部分 icon 在深色模式下仍是黑色
- 底部按钮风格没有统一到深色主题体系

### 2.6 `excelFormatConvert.vue` 深色模式补丁不完整

文件：

- `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`

现象：

- 两个说明 `el-alert` 在深色模式下仍显示白色背景
- 输入框 prepend 区域和 icon 没有随深色模式切换

## 3. 目标

本轮设计目标如下：

1. 修复以上 6 个明确回归问题
2. 按“页面壳层 / 基础控件层 / 容器布局层”三组收口，而不是做 6 个独立补丁
3. 在修复这些问题时，把同类明显样式缺口一并统一，但不扩大到整站重做
4. 保持此前 `draw` 主题重拆的主架构不回退：
   - 继续以 `html[data-theme]` 为唯一主题来源
   - 继续以 `draw-theme-scope` 为显式作用域
   - 不恢复 `draw/index.scss` 全局入口

## 4. 非目标

本轮不做以下事情：

1. 不重新设计整个 `draw` 主题体系
2. 不重写 `ThemeManager`
3. 不处理与本轮 6 类问题无关的广泛视觉重构
4. 不顺手清理所有历史 lint 债
5. 不修改业务逻辑、接口、交互流程或数据结构

## 5. 设计决策

### 5.1 采用“方案 B：按组件族 + 页面壳”修复

已确认本轮采用的不是“逐点单修”，而是按职责分组修复：

1. 页面壳层
2. 基础控件层
3. 容器布局层

原因：

- 当前 6 个问题明显不是 6 个完全独立的 bug
- 它们反映的是“页面容器 dark polish 未收齐”和“基础控件深色细节/溢出行为未收齐”
- 如果只做单点补丁，很容易在相邻组件里继续冒出同类问题

### 5.2 采用“中度统一”而不是“纯保守补丁”

已确认这轮修复的视觉策略是：

- 不重做视觉语言
- 但在修复当前问题时，把相关的深色面板、按钮、输入框、alert 层次统一到同一套深色观感

原因：

- 当前问题本质是同一页面内深色风格不一致
- 只修单点会继续留下半套深色面板、半套浅色残留

## 6. 修复分组设计

### 6.1 页面壳层

文件：

- `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`
- `goto-web/src/views/goto/toolsBox/type/tool.vue`
- `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`

这一组负责：

- 结果页 tab header / nav / active item 的深色观感
- 结果页 zoom controls 容器与按钮的一致性
- tools 右侧面板背景、icon、底部操作按钮的深色统一
- Excel 格式转换页的说明 alert、prepend icon 背景

设计原则：

- 页面壳问题优先在页面自己的样式里修
- 不把页面观感继续压到基础控件层
- 优先使用现有主题 token，例如 `--card-bg`、`--input-bg`、`--hover-bg`、`--text-main`、`--text-secondary`

### 6.2 基础控件层

文件：

- `goto-web/src/views/goto/components/form/draw/input.vue`
- `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`

这一组负责：

- 数字输入框加减按钮在深色下的背景、边框、icon 颜色
- 多选标签容器与输入区域的宽度计算、换行/裁剪、溢出控制

设计原则：

- 这是组件级问题，应该修到组件本身
- 修完后，其它复用该控件的页面应自动受益
- 优先恢复“重构前正常”的行为基线，不做无关交互改造

### 6.3 容器布局层

文件：

- `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`

这一组负责：

- 上传文件区域的 100% 宽度
- 文件卡片容器、上传组件外壳、内部按钮卡片之间的拉伸关系

设计原则：

- 通过容器布局修复，而不是靠按钮组件本身硬塞 `width: 100% !important`
- 优先修 `file-settings-container / file-settings / container / uploader` 这一层

## 7. 具体修复策略

### 7.1 `PlotResultCanvas.vue`

修复策略：

- 对 `el-tabs--border-card` 相关深色样式补充更明确的 header/nav/item/active/hover 层次
- 不只让按钮变暗，还要让 header 容器、tab item 背板、文字对比度一起统一
- 对 `zoom-controls` 容器本身补充深色玻璃/面板背景，避免出现“按钮深色但容器浅色”的割裂感

### 7.2 `PlotSidePanel.vue`

修复策略：

- 让上传文件区域的外层容器、文件卡片和 uploader 结构能够把宽度完整传递下去
- 补齐必要的 `width: 100%`、`flex: 1`、`min-width: 0` 等约束
- 确保 collapse content 里的上传文件卡片在深浅主题下都能自然占满行宽

### 7.3 `input.vue`

修复策略：

- 补齐深色下 `el-input-number__decrease` / `el-input-number__increase` 的背景与边框
- 同时覆盖默认态、hover 态、focus-within 态
- 保持按钮显隐逻辑不变，只修视觉层

### 7.4 `selectMultipleObject.vue`

修复策略：

- 修标签容器和输入区域的溢出、收缩、裁剪关系
- 优先恢复标签区域在宽度不足时的稳定表现
- 如需同步处理 `selectObject.vue` / `selectSimple.vue` 的同类 dropdown 样式，只限同类控件一致性，不扩大到其它表单组件

### 7.5 `tool.vue`

修复策略：

- 把右侧操作面板中的白底容器、黑色 icon、底部风格突兀按钮统一到同一套深色面板规则
- 重点关注：
  - collapse content 内部容器
  - `svg-btn`
  - select 输入框
  - 最底部下载按钮

### 7.6 `excelFormatConvert.vue`

修复策略：

- 为 `el-alert--info.is-light` 提供页面内的深色补丁
- 为 `el-input-group__prepend` 和内部 icon 补齐深色背景/边框/文字色
- 保持浅色模式现状不回归

## 8. 优先顺序

本轮建议按以下顺序实施：

1. `PlotResultCanvas.vue` + `PlotSidePanel.vue`
2. `input.vue` + `selectMultipleObject.vue`
3. `tool.vue` + `excelFormatConvert.vue`

原因：

- 第 1 组是 draw 主流程最直接可见的问题
- 第 2 组是基础控件回归，修完后收益会扩散到复用页面
- 第 3 组是工具页收尾，适合在主流程恢复后统一 polish

## 9. 验收方式

### 9.1 draw 主流程

文件：

- `PlotResultCanvas.vue`
- `PlotSidePanel.vue`

验收点：

- 深色下 tab header、tab item、active tab、zoom controls 视觉统一
- 上传文件区域可以占满可用宽度
- 浅色模式不回归

### 9.2 draw 基础控件

文件：

- `input.vue`
- `selectMultipleObject.vue`

验收点：

- 深色下数字输入 hover 后的加减按钮不再出现浅色块
- 多选下拉框已选项较多时不溢出、不挤坏布局
- 浅色模式保持可用

### 9.3 tools 页面

文件：

- `tool.vue`
- `excelFormatConvert.vue`

验收点：

- `tool.vue` 右侧面板深色下不再有明显白底和黑色 icon
- 底部按钮并入统一深色视觉
- Excel 格式转换页 alert 和 prepend icon 可以随深浅主题切换

## 10. 风险与控制

### 10.1 风险

- 页面壳层和基础控件层边界如果处理不清，容易再把页面观感问题塞回基础组件
- 多选下拉框和数字输入框修复时，容易影响浅色模式或已有交互行为
- tools 页面本身历史样式较多，深色补丁容易引入局部遗漏

### 10.2 控制策略

- 只在上述 3 组职责范围内修复，不扩大到无关文件
- 每一组修复后都做深色 / 浅色双向验收
- 优先修复现有回归，不引入新的风格实验

## 11. 设计结论

这轮不是新功能，而是 `draw` 主题重拆后的第一轮回归收口。

最终结论如下：

- 按“页面壳层 + 基础控件层 + 容器布局层”三组修复
- 采用中度统一策略，不做纯单点补丁
- 不回退此前的主题重拆架构
- 修复目标聚焦在当前确认的 6 类问题及其同类明显缺口

如果这轮修复完成，`draw` 主流程、基础控件和 `tools` 相关页面的深色样式将重新回到一致、可预测的状态，同时不会破坏此前完成的主题分层成果。
