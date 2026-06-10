# goto-web draw 表单标签颜色统一设计

## 背景

`goto-web/src/views/goto/components/form/draw` 目录下存在一类集中问题：在深色主题中，同一块表单里的标题型文本颜色来源不一致，导致视觉层级混乱。

典型表现见 `textParam.vue`：

- “字体粗细”
- “字体颜色”
- “字体大小”
- “其他设置” 折叠标题

这些文本都属于“字段标题/分组标题/折叠标题”语义，但当前分别受到基础表单样式、modern-lab 样式、组件局部 `scoped` 样式和 Element 折叠头样式影响，因此在深色模式下会出现有的偏灰、有的偏亮的问题。

用户目标：

- 统一 `draw` 目录内这类标题型文本的颜色语义
- 深色主题下统一为纯白 `#fff`
- 浅色主题下继续保持可读，不使用白字
- 只处理标题型文本，不改输入值、placeholder、下拉值颜色
- 让后续颜色调整可以集中在一个变量上完成

## 范围

本次范围仅限：

- `goto-web/src/views/goto/components/form/draw`
- 与其直接关联的 draw 主题样式文件

本次明确不包含：

- 输入框内容颜色
- placeholder 颜色
- 下拉面板选项颜色
- `draw` 目录外页面和组件

## 问题归因

当前颜色不一致主要来自四类来源并存：

1. draw 基础表单样式中的 `.form-label`
2. draw modern-lab 风格中的 `.modern-lab-form-group .form-label`
3. 组件局部 `scoped` 样式对 `.form-label` / `.form-label-wrapper` 的局部增强
4. 折叠面板头部 `el-collapse-item__header` 的单独样式

其中 `modern-lab` 版本的标签还额外使用了文本渐变，深色主题下会进一步放大“同层级标题不统一”的观感。

## 设计目标

本次设计要达到四个结果：

1. 在 `draw` 范围内建立独立的“表单标签颜色”语义变量，而不是继续复用正文颜色变量
2. 深色主题下，字段标题、分组标题、折叠标题统一为纯白 `#fff`
3. 浅色主题下，标签颜色继续跟随高对比深色文本，避免浅底白字不可读
4. 后续如果调整这类标题颜色，只需要改变量映射，不再逐组件修补

## 方案选择

### 方案 A：新增 draw 目录公共标签语义变量

做法：

- 在 draw 变量层增加 `--draw-form-label-color`
- 在深浅主题下分别映射不同值
- 公共表单样式、modern-lab 标签样式、折叠头样式统一改为引用该变量
- 对少量局部样式组件做补漏

优点：

- 语义清晰，后续维护成本低
- 能覆盖 `draw` 目录中的大多数同类组件
- 同时兼容深浅主题切换

缺点：

- 需要同时修改变量层和公共样式层

### 方案 B：逐组件补样式

做法：

- 在 `textParam.vue` 及其他问题组件中逐个覆盖标题颜色

优点：

- 上手快

缺点：

- 易遗漏
- 复发概率高
- 后续仍需继续扫尾

### 方案 C：直接提升 draw 主文本变量

做法：

- 直接修改 `--draw-text-primary`

优点：

- 改动点少

缺点：

- 会误伤输入值、正文、辅助说明等非标题文本
- 风险超出本次需求

最终选择方案 A。

## 样式架构设计

### 0. draw 作用域锚点

本次设计明确采用现有稳定命名空间 `.draw-theme-scope` 作为 draw 样式的根作用域锚点。

代码现状表明该类已经在 draw 相关宿主容器中使用，例如：

- `goto-web/src/views/goto/plot/type/draw.vue`
- `goto-web/src/views/goto/toolsBox/type/tool.vue`
- `goto-web/src/views/goto/components/drawGroupOptional.vue`

因此本次公共样式调整的选择器策略固定为：

- 变量定义与主题映射的唯一权威出口固定为 `goto-web/src/assets/styles/draw/_variables.scss`
- 公共规则继续通过 `goto-web/src/assets/styles/draw/foundation.scss` 注入
- 标签和折叠头等实际样式选择器统一挂在 `.draw-theme-scope` 下

编译与导入锚点也固定为当前 `foundation.scss` 的结构：

1. 先 `@import './tokens'`
2. 再 `@import './_form-base'`
3. 再 `@import './_form-components'`
4. 再 `@import './_element-overrides'`
5. 最后 `@import './_modern-lab-theme'`
6. 然后在 `.draw-theme-scope { ... }` 中 `@include` 各个 mixin

本次 spec 依赖这一现有结构，不额外引入新的样式入口文件，也不重排 `foundation.scss` 的导入顺序。

责任边界固定如下：

- `_variables.scss` 负责声明并输出 `--draw-form-label-color` 的 light/default 和 dark 映射
- `tokens.scss` 只负责引入 `_variables.scss` 并保持现有输出顺序，不再重复定义该变量
- `foundation.scss` 负责在 `.draw-theme-scope` 下挂载统一样式规则

禁止做法：

- 直接在全局输出裸 `.form-label`
- 直接在全局输出裸 `.form-label-wrapper`
- 直接在全局输出裸 `.el-collapse-item__header`

如果实施时发现某个 `draw/form/draw` 组件会脱离 `.draw-theme-scope` 宿主独立渲染，不扩大全局覆盖范围，而是给该组件宿主补上 `.draw-theme-scope` 或在该组件内做最小局部补漏。

### 1. 新增语义变量

在 draw 变量层新增：

- `--draw-form-label-color`

主题映射规则：

- 继续沿用项目当前主题挂钩：`html[data-theme='light']` 与 `html[data-theme='dark']`
- 默认态沿用现有 draw 变量文件做法：`:root, html[data-theme='light']` 作为浅色/默认映射
- `html[data-theme='dark']` 下映射为 `#fff`
- 默认态与浅色主题下映射为可读的深色文本，建议对齐 `var(--draw-text-primary)`

输出顺序要求：

- 必须先输出 `:root, html[data-theme='light']`
- 再输出 `html[data-theme='dark']`
- 这一顺序由 `_variables.scss` 作为唯一权威定义保持，`tokens.scss` 仅透传现有顺序，不得在其他入口中重复定义同名变量后打乱覆盖关系

原因是两者特异性相同，深色块需要依赖后出现来覆盖默认/浅色映射；这一点要保持与当前 draw token 文件的输出顺序一致。

这样可以保证：

- 深色主题得到明确的纯白标题
- 浅色主题不出现白字不可读问题
- 标签色与正文色解耦

### 2. 公共标题选择器统一

以下标题语义统一改用 `--draw-form-label-color`：

- `.draw-theme-scope .form-label`
- `.draw-theme-scope .form-label-wrapper .form-label`
- `.draw-theme-scope .modern-lab-form-group .form-label`
- `.draw-theme-scope .el-collapse-item__header`
- `.draw-theme-scope .el-collapse-item__header .el-collapse-item__arrow`
- `.draw-theme-scope .el-collapse-item__header .el-icon-arrow-right`

折叠头策略固定为以下两类公共规则，不再依赖 `text-param-*` 之类局部类名做主路径：

- `.draw-theme-scope .el-collapse-item__header` 负责标题文本色
- `.draw-theme-scope .el-collapse-item__header .el-collapse-item__arrow`
- `.draw-theme-scope .el-collapse-item__header .el-icon-arrow-right`

上面两条箭头规则覆盖当前 Element UI 折叠头的常见 DOM 形态，避免把实现写成脆弱的裸 `i` 选择器。

如果具体组件使用了 Element 标题插槽并引入额外文本包装元素，也要求这些文本元素继承 header 的 `color`，不允许继续单独写死灰色。

这样可以同时保证：

- 不泄漏到 draw 目录外的 Element 折叠面板
- 只要组件运行在现有 draw 宿主容器下，就能一致命中
- 不要求每个具体组件都重复维护一套折叠标题颜色

### 3. modern-lab 渐变处理

`modern-lab-form-group .form-label` 当前使用由主文本色到次文本色的渐变文本。

这与“深色主题统一纯白”目标冲突，因此需要收口：

- 深色主题下取消标题渐变效果，直接呈现统一纯白
- 浅色主题可保留统一实色，不再使用会造成层级漂移的渐变

这里不建议保留渐变并仅修改起止色，因为仍会带来“同类标题颜色不完全一致”的观感问题。

实施时不能只补 `color`，还必须显式移除会拦截文字颜色的渐变相关属性，至少包括：

- `background` / `background-image`
- `-webkit-background-clip: text`
- `background-clip: text`
- `-webkit-text-fill-color`

然后必须显式写成：

- `color: var(--draw-form-label-color)`
- `-webkit-text-fill-color: currentColor`

否则部分 WebKit 浏览器下仍可能继承旧的透明填充，导致标题不可见。

## 组件落点

### 变量层

文件：

- `goto-web/src/assets/styles/draw/_variables.scss`

改动：

- 增加 `--draw-form-label-color`
- 在 `:root, html[data-theme='light']` 下提供默认/浅色映射
- 在 `html[data-theme='dark']` 下提供深色映射
- 保持深色块输出在默认/浅色块之后，确保覆盖稳定生效

### 公共样式层

文件：

- `goto-web/src/assets/styles/draw/_form-base.scss`
- `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`

改动：

- `.draw-theme-scope .form-label` 使用 `--draw-form-label-color`
- `.draw-theme-scope .form-label-wrapper .form-label` 使用 `--draw-form-label-color`
- `.draw-theme-scope .modern-lab-form-group .form-label` 去掉渐变文本，统一使用 `--draw-form-label-color`
- `.draw-theme-scope .el-collapse-item__header` 使用 `--draw-form-label-color`
- `.draw-theme-scope .el-collapse-item__header .el-collapse-item__arrow` 使用 `--draw-form-label-color`
- `.draw-theme-scope .el-collapse-item__header .el-icon-arrow-right` 使用 `--draw-form-label-color`

层叠策略固定如下：

- 第一层：公共统一规则放在 `foundation.scss` 对应的 draw mixin 中，作为主路径
- 第二层：若某个 Vue SFC 的 `scoped` 样式因 attribute selector 导致其优先级高于公共规则，则只允许在该组件内增加最小补漏
- 补漏写法必须继续引用 `var(--draw-form-label-color)`，不能写新的 `#fff` 或其他字面量颜色
- 补漏优先使用组件自身根类名加 `:deep(.form-label)` / `:deep(.form-label-wrapper .form-label)` / `:deep(.el-collapse-item__header)` / `:deep(.el-collapse-item__header .el-collapse-item__arrow)` 这种最小命中方式
- 不允许为了压过 `scoped` 样式而把全局选择器继续无限加深，避免形成新的维护债务

### 组件补漏层

重点验证组件：

- `goto-web/src/views/goto/components/form/draw/textParam.vue`
- `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- `goto-web/src/views/goto/components/form/draw/input.vue`
- `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`

这些组件里未必都需要显式修改，但需要逐个确认它们的 `scoped` 样式不会压过公共规则。

### 折叠标题统一

目标对象：

- `textParam.vue` 中“其他设置”
- `draw` 范围内其他使用 `el-collapse-item__header` 的同类折叠块

处理原则：

- 不在单个组件中硬编码白色
- 通过 `.draw-theme-scope .el-collapse-item__header` 统一折叠头文本色
- 通过 `.draw-theme-scope .el-collapse-item__header .el-collapse-item__arrow` 及兼容类统一折叠箭头图标色
- 不以 `textParam.vue` 的局部类名作为唯一依赖，避免覆盖不完整

## 风险与控制

### 风险 1：浅色主题出现白字不可读

控制：

- 不把变量全局固定为 `#fff`
- 仅在深色主题下映射为 `#fff`

### 风险 2：正文颜色被误伤

控制：

- 不修改 `--draw-text-primary`
- 仅新增并使用 `--draw-form-label-color`

### 风险 3：局部 `scoped` 样式覆盖公共规则

控制：

- 实施时检查 `draw` 目录内关键组件
- 仅对确实压过公共规则的组件做最小补漏

### 风险 4：默认主题态未命中导致变量不生效

控制：

- 延续项目现有主题变量文件的输出方式
- 在 `:root, html[data-theme='light']` 上提供默认映射
- 在 `html[data-theme='dark']` 上提供深色覆盖

### 风险 5：深色映射因输出顺序错误而未覆盖默认值

控制：

- 保持 `html[data-theme='dark']` 规则块输出在 `:root, html[data-theme='light']` 之后
- 不调整现有 draw token 文件的主题块先后顺序
- 不在其他 draw 样式入口重新定义 `--draw-form-label-color`

### 风险 6：折叠头和普通标签仍然不一致

控制：

- 将 `el-collapse-item__header` 纳入同一语义变量
- 将折叠箭头的 Element 约定类名一并纳入同一语义变量
- 验证 `textParam.vue` 和分类相关组件中的折叠标题
- 同时确认作用域没有泄漏到 draw 目录外

### 风险 7：Vue SFC `scoped` 样式压过公共统一规则

控制：

- 统一规则先落在 `foundation.scss` 中作为主路径
- 对确实被 `scoped` 样式压过的组件，只做组件内最小补漏
- 补漏必须继续复用 `--draw-form-label-color`
- 不新增新的颜色字面量和并行语义变量

### 风险 8：`form-label-wrapper` 的非标题子节点被误着色

控制：

- 不把颜色直接挂到 `.form-label-wrapper`
- 仅命中 `.form-label-wrapper .form-label`
- 保证同步图标等辅助节点继续走各自语义，不被标题白色误伤

## 验收标准

在深色主题下：

- `textParam.vue` 中“字体粗细”“字体颜色”“字体大小”“其他设置”标题视觉上统一为纯白 `#fff`
- `draw` 目录内其他同类字段标题、分组标题、折叠标题保持同一颜色语义
- 输入值、placeholder、下拉项颜色不发生连带变化

在浅色主题下：

- 上述标题保持高对比可读
- 不出现白字贴浅底的问题

维护性要求：

- 后续如需调整该类标题颜色，只改 `--draw-form-label-color` 的主题映射即可

## 测试与验证计划

实施后至少验证：

1. 深色主题下打开 `textParam.vue` 对应界面，核对标题统一
2. 浅色主题下切换同一界面，核对标签可读性
3. 验证 `draw` 目录中至少一个非 `textParam` 的表单组件，确认公共规则生效
4. 核对输入值、placeholder、下拉面板条目颜色未被误改

## 实施边界

本次只做颜色语义统一，不顺带处理：

- 字号统一
- 间距统一
- 边框和背景统一
- 下拉面板主题彻底重构

这些问题如果存在，应在后续独立需求中处理，避免本次范围扩散。
