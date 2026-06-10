# goto-web goto Cold Slate 主题设计

## 1. 背景

`goto-web` 当前已经具备一套可运行的浅色 / 深色主题变量体系，主要集中在以下位置：

- `goto-web/src/assets/styles/theme-variables.scss`
- `goto-web/src/assets/styles/dark-mode-global.scss`
- `goto-web/src/assets/styles/draw/_variables.scss`
- `goto-web/src/assets/styles/draw/tokens.scss`

现状存在两个问题：

1. 应用级主题变量已经能支撑基础深浅模式，但视觉语言仍偏“通用后台”，距离 `Linear` 式冷静、理性、偏中性蓝灰的气质还有差距
2. 变量层级还不够语义化，旧变量以 `body/card/input/border` 为主，足够维持运行，但不利于持续优化 `src/views/goto` 这类工具型页面的细腻层级

本轮目标不是重做全站设计语言，而是在保留现有变量兼容性的前提下，为 `src/views/goto` 的核心页面建立一套更稳定、更专业、更易维护的主题基线。

## 2. 已确认约束

本次设计基于以下已确认约束：

1. 视觉方向采用 `A. Cold Slate`
2. 整体气质以 `Linear` 式冷静蓝灰为目标，不走暖灰，也不走更强品牌感的靛蓝路线
3. 主题优化优先服务 `goto-web/src/views/goto` 这批核心页面，而不是全站同步重做
4. 保留现有变量命名，避免大面积破坏现有页面
5. 在兼容旧变量的同时，新增一套更完整的语义 token
6. 深色模式继续跟随现有 `html[data-theme='light' | 'dark']` 机制，不新增新的主题状态源

## 3. 目标

### 3.1 视觉目标

- 让浅色模式从“纯白后台”升级为更通透的冷灰蓝画布
- 让深色模式从“深色兜底”升级为更有纵深的深蓝灰层级
- 保留品牌主色的靛蓝识别，但在深色模式中通过去饱和和亮度控制减轻疲劳
- 让 `goto` 页面看起来更像“精密控制台”，而不是普通运营后台

### 3.2 工程目标

- 保持旧变量继续可用，降低对现有页面的改造成本
- 建立 `surface / text / accent / border / shadow` 五类语义 token
- 让后续组件可以基于语义 token 编写，而不是继续依赖经验性颜色拼接
- 为后续 `draw-theme-scope`、工具页、表单页、弹层页提供统一的主题基线

### 3.3 无障碍目标

- 正文文本与主要背景保持 AA 级对比度
- 次级文本在高密度表单和控制面板中仍需保持清晰可读
- focus、selected、disabled 的状态差异必须依赖对比而不是单纯透明度

## 4. 非目标

本轮不做以下事情：

1. 不重构 `ThemeManager` 或主题状态管理
2. 不重做全站所有业务页面的视觉语言
3. 不引入新的局部主题切换机制
4. 不把所有旧页面立即迁移到语义 token
5. 不在本设计阶段直接修改业务页面逻辑或组件行为

## 5. 方案决策

### 5.1 采用 Cold Slate 方向

最终采用 `A. Cold Slate` 作为新的主题方向。

原因：

- 相比更中性的 `Graphite Minimal`，它能保留更多产品品牌识别
- 相比更强调靛蓝的 `Indigo Rail`，它在深色模式下更稳，不容易出现发紫、发亮、视觉疲劳的问题
- 它更适合 `src/views/goto` 这种信息密度高、长时间使用的工具界面

### 5.2 颜色策略

核心策略如下：

1. 浅色模式不用纯白，主背景改为冷灰蓝画布
2. 深色模式不用纯黑，采用分层深蓝灰系统拉开结构
3. 品牌色继续保留靛蓝基因，但 light / dark 使用不同强度
4. 语义层级优先于视觉“装饰感”，边框和表面层次比高饱和装饰更重要

## 6. 主题原则

### 6.1 Light 模式原则

- `canvas` 采用轻蓝灰背景，避免廉价白底
- `panel` 与 `elevated` 维持极轻差异，让输入框、卡片、内容面之间有柔和区分
- 阴影偏空气感，透明度低、扩散大，不做重投影
- 边框用冷灰蓝，不用暖灰或纯灰

### 6.2 Dark 模式原则

- `canvas` 采用深蓝灰而非纯黑
- `panel / elevated / muted` 通过亮度递增拉开层级
- 主色做去饱和处理，保留可识别性但不过亮
- 阴影减少存在感，更多依赖边框、表面亮度差和极轻的外阴影

### 6.3 品牌色原则

- Light 主品牌色采用 `#4666e5`
- Dark 高亮品牌色采用 `#7c8cff`
- Dark 实心按钮品牌色采用更稳的 `#586ce0`
- hover / active 不依赖显著饱和度提升，而依赖适度明度调整
- 选中态、focus ring、浅色 tag、浅色背景统一从 `accent-soft` 派生
- 链接、高亮、focus 使用 `accent-primary`，实心按钮和高强调操作使用 `accent-solid`

## 7. Token 结构

### 7.1 兼容层：保留现有变量

以下变量继续保留并维持原有职责：

- `--primary-color`
- `--primary-color-light`
- `--primary-color-btn-light`
- `--body-color`
- `--sidebar-color`
- `--sidebar-text`
- `--text-main`
- `--text-secondary`
- `--text-tertiary`
- `--border-color`
- `--border-color-light`
- `--border-color-strong`
- `--card-bg`
- `--input-bg`
- `--hover-bg`
- `--shadow-sm`
- `--shadow-md`
- `--shadow-lg`
- `--border-radius`

### 7.2 语义层：新增变量

新增语义变量如下：

- `--surface-canvas`
- `--surface-panel`
- `--surface-elevated`
- `--surface-muted`
- `--surface-overlay`
- `--text-primary`
- `--text-secondary`
- `--text-muted`
- `--text-inverse`
- `--accent-primary`
- `--accent-solid`
- `--accent-soft`
- `--accent-strong`
- `--text-on-accent`
- `--border-subtle`
- `--border-default`
- `--border-strong`
- `--shadow-xs`
- `--shadow-sm`
- `--shadow-md`
- `--shadow-lg`
- `--focus-ring`
- `--success`
- `--warning`
- `--danger`

### 7.3 映射原则

旧变量尽量从语义变量派生，避免两套数值长期并行。

推荐映射如下：

- `--body-color <- --surface-canvas`
- `--card-bg <- --surface-panel`
- `--input-bg <- --surface-elevated`
- `--hover-bg <- --surface-muted`
- `--primary-color <- --accent-solid`
- `--primary-color-light <- --accent-soft`
- `--text-main <- --text-primary`
- `--text-secondary <- --text-secondary`
- `--text-tertiary <- --text-muted`
- `--border-color <- --border-default`
- `--border-color-light <- --border-subtle`
- `--border-color-strong <- --border-strong`

## 8. 建议色值

### 8.1 Light

```css
:root,
html[data-theme='light'] {
  color-scheme: light;

  --surface-canvas: #f5f7fb;
  --surface-panel: #fcfdff;
  --surface-elevated: #ffffff;
  --surface-muted: #eef2f8;
  --surface-overlay: rgba(252, 253, 255, 0.78);

  --text-primary: #111827;
  --text-secondary: #5f6b7a;
  --text-muted: #7d8a9b;
  --text-inverse: #f8fbff;

  --accent-primary: #4666e5;
  --accent-solid: #4666e5;
  --accent-soft: #e8efff;
  --accent-strong: #3d5add;
  --text-on-accent: #ffffff;

  --border-subtle: #e6ebf2;
  --border-default: #d7dfea;
  --border-strong: #b8c4d6;

  --shadow-xs: 0 1px 2px rgba(15, 23, 36, 0.04);
  --shadow-sm: 0 10px 24px rgba(15, 23, 36, 0.06);
  --shadow-md: 0 18px 48px rgba(15, 23, 36, 0.1);
  --shadow-lg: 0 24px 64px rgba(15, 23, 36, 0.14);

  --focus-ring: rgba(70, 102, 229, 0.18);
  --success: #0f9f6e;
  --warning: #c77b18;
  --danger: #d45563;

  --primary-color: var(--accent-solid);
  --primary-color-light: var(--accent-soft);
  --primary-color-btn-light: var(--accent-strong);

  --body-color: var(--surface-canvas);
  --sidebar-color: #f8fbff;
  --sidebar-text: var(--text-secondary);
  --text-main: var(--text-primary);
  --text-tertiary: var(--text-muted);

  --border-color: var(--border-default);
  --border-color-light: var(--border-subtle);
  --border-color-strong: var(--border-strong);

  --card-bg: var(--surface-panel);
  --input-bg: var(--surface-elevated);
  --hover-bg: var(--surface-muted);

  --border-radius: 12px;
}
```

### 8.2 Dark

```css
html[data-theme='dark'] {
  color-scheme: dark;

  --surface-canvas: #0f1724;
  --surface-panel: #161f2d;
  --surface-elevated: #1c2636;
  --surface-muted: #243044;
  --surface-overlay: rgba(19, 30, 45, 0.78);

  --text-primary: #e8edf5;
  --text-secondary: #a7b4c7;
  --text-muted: #8290a6;
  --text-inverse: #0f1724;

  --accent-primary: #7c8cff;
  --accent-solid: #586ce0;
  --accent-soft: #1d2b4d;
  --accent-strong: #4f63d8;
  --text-on-accent: #ffffff;

  --border-subtle: #243042;
  --border-default: #2b3646;
  --border-strong: #40506a;

  --shadow-xs: 0 1px 2px rgba(2, 6, 23, 0.26);
  --shadow-sm: 0 10px 28px rgba(2, 6, 23, 0.34);
  --shadow-md: 0 18px 48px rgba(2, 6, 23, 0.44);
  --shadow-lg: 0 28px 72px rgba(2, 6, 23, 0.52);

  --focus-ring: rgba(124, 140, 255, 0.3);
  --success: #32b38a;
  --warning: #e0a13b;
  --danger: #f07c8a;

  --primary-color: var(--accent-solid);
  --primary-color-light: var(--accent-soft);
  --primary-color-btn-light: var(--accent-strong);

  --body-color: var(--surface-canvas);
  --sidebar-color: #0c1420;
  --sidebar-text: var(--text-secondary);
  --text-main: var(--text-primary);
  --text-tertiary: var(--text-muted);

  --border-color: var(--border-default);
  --border-color-light: var(--border-subtle);
  --border-color-strong: var(--border-strong);

  --card-bg: var(--surface-panel);
  --input-bg: var(--surface-elevated);
  --hover-bg: var(--surface-muted);

  --border-radius: 12px;
}
```

## 9. 组件质感规范

### 9.1 Border Radius

建议统一为以下阶梯：

- 大卡片、右侧操作面板、主容器：`14px`
- 常规卡片、tabs、表单区块：`12px`
- 输入框、select、按钮：`10px`
- tag、状态胶囊：`999px`

原则：

- 不建议超过 `16px`
- 避免“圆过头”的消费级观感
- 保持工具型界面的克制和精密感

### 9.2 Backdrop Filter

仅用于浮起的局部组件，不用于承载主体内容的大面积卡片。

建议：

- Light：`blur(12px) saturate(140%)`
- Dark：`blur(16px) saturate(120%)`

推荐使用位置：

- 顶部悬浮工具条
- 浮动操作按钮组
- overlay 弹层或轻量 popup

不推荐使用位置：

- 长表单主区域
- 主结果面板
- 数据密集卡片主体

### 9.3 Shadow

Shadow 目标是“抬升感”，不是“黑块感”。

Light：

- 优先空气感阴影
- 透明度低，扩散大
- hover 和 elevated 状态用不同档位抬升

Dark：

- 降低传统黑影存在感
- 依赖边框、亮度层级和轻外阴影
- 必要时允许加入轻微 `inset` 顶部高光线

### 9.4 Border 优先级

此方案中边框的重要性高于阴影。

要求：

- 常规层级主要由 `surface` 差异和 `border` 差异完成
- hover/focus 优先提亮边框或更换 surface，不靠重阴影制造状态
- 深色模式下避免过高对比的亮边框，改为低对比蓝灰线条

## 10. 重点落地范围

本轮主题优化优先覆盖以下区域：

1. `src/views/goto/plot/type/*`
2. `src/views/goto/toolsBox/type/*`
3. `src/views/goto/components/form/draw/*`
4. 已经依赖 `draw-theme-scope` 的统计和配置类页面

优先感知点：

- 页面 canvas 与面板层级
- 右侧操作面板
- tabs、输入框、select、数字步进器
- dropdown、dialog、tooltip、alert
- hover、focus、selected、disabled 状态

## 11. 验证要求

### 11.1 视觉验证

- Light / Dark 下 canvas、panel、input、hover 的层级关系清晰
- 主题不发紫、不发脏、不出现“浅色残留”
- 同一页面中的按钮、面板、输入框语言一致

### 11.2 无障碍验证

- `text-primary` 对 `surface-canvas / surface-panel` 满足 AA
- `text-secondary` 对常用 surface 满足次级文本可读性要求
- 实心主按钮使用 `accent-solid + text-on-accent`，并满足 AA
- `text-muted` 仅用于辅助信息，不用于正文、表单标签或关键交互文案
- focus ring 在 light / dark 下都清晰可见

### 11.3 组件验证

重点检查以下组件：

- `draw` 右侧参数面板
- `PlotResultCanvas` tabs 与顶部操作区
- `input.vue` 与数字输入步进器
- `selectObject.vue` / `selectMultipleObject.vue`
- 各类 dropdown 和 body 级弹层
- `tool.vue`、`excelFormatConvert.vue` 等工具页说明卡片和输入区

## 12. 实施建议

实现阶段建议按以下顺序推进：

1. 先在 `theme-variables.scss` 中建立兼容层 + 语义层
2. 再让 `draw` token 继续通过映射消费应用级 token
3. 最后逐页检查 `src/views/goto` 中高频组件的具体观感和细节补丁

这样可以保证：

- 先统一基线
- 再做局部抛光
- 避免一开始就在页面里散落大量“临时深色补丁”

## 13. 设计结论

本设计采用 `Cold Slate` 方向，为 `goto-web` 的 `src/views/goto` 核心页面建立一套兼容旧变量、补全语义 token、强调冷静蓝灰层级的主题体系。

它的关键价值不只是“替换几个颜色值”，而是把主题系统从“能运行的变量表”升级为“可以长期演进的语义化设计基础”。
