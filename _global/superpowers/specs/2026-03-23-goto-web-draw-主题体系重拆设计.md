# goto-web draw 主题体系重拆设计

## 1. 背景

当前 `goto-web` 的 `draw` 样式体系同时存在以下问题：

- `src/main.js` 全局引入了 `src/assets/styles/draw/index.scss`
- `src/views/goto/components/form/draw` 下多个组件又在各自的 `<style lang="scss" scoped>` 中重复 `@import "@/assets/styles/draw/index.scss"`
- 部分非 `draw` 页面或非 `draw` 组件目录下的页面，也通过局部引入 `draw/index.scss` 借用基础控件样式
- `draw` 子系统内部同时混合了 token 定义、基础表单类、Element UI 覆盖、页面布局、深色模式覆盖和“现代实验室”视觉皮肤
- `draw` 自身的深浅主题判断与全局 `theme-variables.scss`、`dark-mode-global.scss`、`element-ui-dark.scss` 存在重叠和冲突

这导致当前样式依赖没有清晰边界：

1. 组件依赖了整包 `draw/index.scss`，但其中既有组件需要的基础能力，也有页面专属布局样式
2. 删除任何一个入口时，都可能同时丢失 token、基础类或真实页面样式
3. 深色模式来源不单一，问题排查需要在多个文件和多个选择器之间来回猜测
4. `draw` 基础控件已经扩散到统计页、`plotSetting` 页和部分通用页面，但它们并不应该自动继承绘图主页面的重型布局样式

本次设计的目标不是“小修 import 顺序”，而是一次性把 `draw` 主题体系重拆成可维护、可定位、可复用的分层结构。

## 2. 目标

### 2.1 业务目标

- 让 `draw` 子系统在深色 / 浅色主题下拥有稳定、可预测的视觉表现
- 允许 `draw` 基础控件继续在非绘图页面中复用
- 让绘图主页面、工具页和基础控件消费页面拥有明确的样式边界
- 解决当前“移除任一入口都会出问题”的耦合状态

### 2.2 工程目标

- 移除 `draw/index.scss` 作为“全局总入口 + 组件依赖入口 + 页面布局入口”的混合角色
- 取消 `src/main.js` 对 `draw` 样式的全局注入
- 移除 `form/draw` 组件内对 `draw/index.scss` 的整包引入
- 建立 `draw` 的分层样式结构：token 层、foundation 层、component skin 层、page/layout 层
- 将 `draw` 的深浅主题判定统一收口到 `html[data-theme]`
- 让 `draw` 页面布局样式只在显式页面作用域下生效，不再污染全局

## 3. 范围

### 3.1 本轮范围

- `goto-web/src/assets/styles/draw/*`
- `goto-web/src/views/goto/components/form/draw/*`
- `goto-web/src/views/goto/components/drawGroupOptional.vue`
- `goto-web/src/views/goto/plot/type/draw.vue`
- `goto-web/src/views/goto/toolsBox/type/tool.vue`
- `goto-web/src/views/goto/components/form/plotSetting/*` 中直接借用了 `draw` 基础控件或 `draw` 样式入口的页面
- `goto-web/src/views/goto/statistics/*` 中直接借用了 `draw` 基础控件的页面
- `goto-web/src/assets/styles/theme-variables.scss`
- `goto-web/src/assets/styles/dark-mode-global.scss`
- 与 `draw` 冲突或重复的 `Element UI` 深色模式覆盖

### 3.2 明确不在本轮范围

- 不重做整个网站的通用设计语言
- 不把所有非 `draw` 业务页面都迁移到新的样式架构
- 不重新设计 `ThemeManager` 的业务接口
- 不引入新的主题状态源，不新增“局部独立主题开关”
- 不保留新旧 `draw` 样式入口的长期兼容层

## 4. 已确认约束

本次设计基于以下已确认约束：

1. 本轮允许对 `draw` 样式做一定幅度调整，并在发现问题时顺手修复
2. 范围采用扩展收口模式，除 `draw` 子系统本身外，也要一并处理与其冲突的全局主题覆盖
3. `draw` 样式不再通过 `src/main.js` 全局加载
4. `draw` 组件允许在别处复用，但调用方必须显式启用 `draw` 主题作用域
5. `draw` 的主题判定只跟随应用主题，唯一来源是 `html[data-theme]`
6. 迁移策略采用一次性切换，不保留新旧双入口并行状态
7. 最终消费模型采用混合模式：
   - 组件基础样式由组件所属页面或容器显式接入
   - 页面级布局样式只由绘图页面或局部重型容器显式接入

## 5. 当前现状

### 5.1 入口现状

当前 `draw/index.scss` 同时承担：

- 设计 token 入口
- 基础类入口
- 组件公共覆盖入口
- 绘图页布局入口
- 深色模式布局入口
- 现代实验室皮肤入口

当前主要入口包括：

- `src/main.js` 中的全局 `import "./assets/styles/draw/index.scss"`
- `src/views/goto/components/form/draw/*.vue` 中的大量 `@import "@/assets/styles/draw/index.scss"`
- 若干非 `draw` 页面中的局部 `@import "@/assets/styles/draw/index.scss"`

### 5.2 使用边界现状

已确认以下事实：

- `draw` 基础控件不只在绘图主页面中使用
- 统计分析页面直接使用了 `GUploaderBtn`、`GSelectObject`、`GInput`
- `plotSetting` 相关页面直接使用了 `GCheckbox`、`GInput`、`GSelectDictImage`
- 部分页面为了拿到 `draw` 基础类或公共覆盖，直接引入了整包 `draw/index.scss`
- `#sub-module` 等重型布局只属于少数绘图页面，不适合作为全局样式存在

### 5.3 主题现状

当前主题实现同时存在：

- `theme-variables.scss` 中的应用级深浅主题变量
- `draw/_variables.scss` 中的 `draw` token 和自有 dark 逻辑
- `draw/_layout-dark.scss` 中的页面级深色覆盖
- `dark-mode-global.scss` 中的全局 dark 补丁
- `element-ui-dark.scss` 中的另一套 Element UI dark 覆盖
- `ThemeToggle.vue` 中与 `theme.js` 命名不完全一致的主题样式残留

这意味着 `draw` 的 dark 行为不是单一路径，存在重复覆盖与规则重叠。

## 6. 方案对比

### 6.1 方案一：组件完全自带样式

做法：

- 每个 `draw` 组件各自引入 token、基础类和控件覆盖
- 页面只保留极少量布局样式

优点：

- 单个组件移植成本最低
- 组件在任何地方使用都更像“自包含”

缺点：

- CSS 重复输出明显增加
- Element UI 覆盖继续分散在各组件中
- 后续维护成本高，问题定位路径仍然不清晰

### 6.2 方案二：分层重拆，双入口加载，显式作用域

做法：

- 将 `draw` 拆成 token、foundation、component skin、page/layout 四层
- 非绘图页面只接入基础入口
- 绘图页接入基础入口和页面布局入口
- 所有消费方通过显式 `draw` 主题作用域启用样式

优点：

- 组件依赖和页面依赖边界清晰
- 可以保留 `draw` 基础控件的跨页复用能力
- 可以彻底移除 `main.js` 全局注入
- 主题问题的排查路径最清晰

缺点：

- 需要一次性调整多个消费页面和样式入口
- 需要重构现有 `draw` 样式目录与若干页面模板

### 6.3 方案三：保留全局 token，只抽离页面布局

做法：

- 保留部分 `draw` 样式在全局
- 仅把 `#sub-module` 等绘图布局样式抽出局部入口

优点：

- 改动量相对最小

缺点：

- 根因没有消失，仍然保留全局耦合
- “哪些样式应该全局，哪些不该全局”会继续模糊
- 后续仍然容易回到“删哪都坏”的状态

## 7. 方案决策

采用方案二：**分层重拆，双入口加载，显式作用域**。

原因：

- 当前问题的根因不是单一 import 放置错误，而是职责层级混杂
- 只有分层重拆才能同时解决组件复用、页面布局隔离和主题判定统一三个问题
- 该方案既满足“draw 不再全局加载”，又满足“基础控件仍可跨页复用”
- 该方案与已确认约束完全一致，不需要保留长期兼容层

## 8. 设计概览

### 8.1 分层模型

新的 `draw` 样式体系分为四层：

1. `theme tokens`
   - 只负责定义和映射 `--draw-*` 变量
   - 不直接产出页面布局
   - 不做独立主题判定

2. `foundation`
   - 只负责基础表单语义类、通用控件尺寸、必要的局部 `Element UI` 覆盖
   - 是基础控件正常显示所需的最低依赖
   - 不承载 `#sub-module`、操作面板、画布等页面结构

3. `component skins`
   - 每个 `draw` 组件只保留自身专属样式
   - 不再整包引入 `draw/index.scss`
   - 默认依赖外层已经启用 `draw` foundation

4. `page/layout`
   - 只负责绘图页面和工具页的重型布局、装饰背景、操作区和布局深色覆盖
   - 只在明确页面作用域下生效

### 8.2 作用域模型

最终所有 `draw` 样式都建立在显式作用域上：

- 基础控件页面或局部容器通过 `.draw-theme-scope` 启用 `foundation`
- 绘图页或重型绘图容器通过 `.draw-page.draw-theme-scope` 启用页面布局样式

该模型确保：

- `draw` 基础控件可以跨页复用
- 页面级布局不会污染非绘图页面
- 调用方的样式依赖关系可见、可审计

## 9. 主题与 token 设计

### 9.1 单一主题来源

`draw` 的主题判定统一为：

- 唯一来源：`ThemeManager`
- 唯一 DOM 状态：`html[data-theme="light" | "dark"]`
- `auto` 逻辑只存在于 `ThemeManager` 中
- `draw` SCSS 不再自行使用 `prefers-color-scheme` 判断主题

### 9.2 应用级 token 与 draw token 的关系

应用级主题变量继续由 `theme-variables.scss` 负责，例如：

- `--primary-color`
- `--card-bg`
- `--input-bg`
- `--text-main`
- `--text-secondary`
- `--border-color`

`draw` token 通过映射方式从应用级 token 派生，例如：

- `--draw-bg-primary` 映射到 `--card-bg`
- `--draw-bg-secondary` 映射到 `--card-bg` 或 `--input-bg`
- `--draw-text-primary` 映射到 `--text-main`
- `--draw-border-normal` 映射到 `--border-color`

这样 `draw` 不再自建第二套主题判断，而是只做语义映射。

为避免 `Element UI` body 级弹层无法继承局部作用域变量，`draw token` 继续以**全局命名空间变量**形式存在于应用主题层之上：

- `--draw-*` 变量定义可以保留在 `:root` / `html[data-theme]` 维度
- 真正产出控件与页面视觉的 CSS 规则仍然必须受 `.draw-theme-scope` 或页面作用域约束
- 这样既避免 token 污染真实样式边界，又保证 body 级弹层可获取 `draw` 变量

### 9.3 现代实验室皮肤定位

`modern-lab-theme` 从“并行主题系统”降级为“draw 皮肤层”：

- 可以保留其视觉表达
- 但不再定义新的根级主题真相
- 必须建立在 `draw token` 和应用级 token 之上
- 不能再与基础 token 或全局主题竞争控制权

## 10. 样式入口设计

### 10.1 新入口职责

本轮重构后，`draw` 样式入口拆为两类：

1. 基础入口
   - 提供 `token + foundation + 必要公共覆盖`
   - 供复用 `draw` 控件的页面或容器接入

2. 页面布局入口
   - 在基础入口之上叠加绘图页专属布局样式
   - 仅供真正的绘图页或重型绘图容器接入

### 10.2 旧入口退场

以下旧用法在重构后不再允许存在：

- `src/main.js` 全局引入 `draw/index.scss`
- `form/draw` 组件内整包 `@import "@/assets/styles/draw/index.scss"`
- 非绘图页面为了拿基础控件样式而整包引入 `draw/index.scss`

### 10.3 文件结构目标

目标结构为显式职责化结构，例如：

```text
src/assets/styles/draw/
├── tokens.scss
├── foundation.scss
├── layout-plot.scss
├── layout-toolbox.scss
├── skins/
│   └── modern-lab.scss
├── partials/
│   ├── _mixins.scss
│   ├── _form-base.scss
│   ├── _form-components.scss
│   ├── _element-overrides.scss
│   └── ...
```

具体文件名可在实现时微调，但职责边界不得变化。

## 11. 组件与页面消费模型

### 11.1 `form/draw` 组件

`src/views/goto/components/form/draw/*` 下的组件遵循以下规则：

- 删除对 `draw/index.scss` 的整包引入
- 组件只保留自己专属的局部样式
- 组件不再直接引入页面布局层
- 组件默认依赖调用方已启用 `draw` foundation

### 11.2 复用基础控件的非绘图页面

统计页、`plotSetting` 页以及其他复用 `GInput`、`GSelectObject`、`GCheckbox`、`GUploaderBtn` 的页面，遵循以下规则：

- 显式接入 `draw` 基础入口
- 在页面或局部容器根节点包裹 `.draw-theme-scope`
- 不引入绘图页面布局入口

### 11.3 绘图主页面与工具页

绘图主页面与工具页遵循以下规则：

- 页面根节点显式带 `.draw-page.draw-theme-scope`
- 显式接入 `draw` 基础入口和对应页面布局入口
- 页面布局样式只在该作用域内生效

### 11.4 body 级弹层与 popper 契约

`Element UI` 的下拉菜单、tooltip、popover、选择器面板等组件经常挂载到 `body`，它们不会天然继承页面局部容器的 class 作用域。

因此本轮需要建立统一契约：

- 由 `draw` 基础控件发出的 body 级弹层必须带可识别的类名，例如 `draw-theme-popper`
- `draw` 专属弹层样式以该类名作为选择器，而不是依赖页面 DOM 层级
- 只有真正属于 `draw` 体系的弹层才挂该类名，避免影响全局普通 `Element UI` 组件

该契约是一次性切换成功的关键部分，不能在实施阶段省略。

## 12. 现有文件收口策略

### 12.1 `draw` 目录内部

以下内容需要从现有 `draw` 目录中拆分和重组：

- `draw/_variables.scss`
  - 拆出为 token 映射层
  - 删除 `prefers-color-scheme` 的主题判定

- `draw/_form-base.scss`
  - 进入 foundation 层

- `draw/_form-components.scss`
  - 进入 foundation 层

- `draw/_element-overrides.scss`
  - 拆分为：
    - 仅 `draw` 容器内生效的基础控件覆盖
    - 必要的专用组件覆盖
  - 不再作为全局 `Element UI` 美化入口

- `draw/_layout.scss`
  - 拆入绘图页或工具页页面布局入口

- `draw/_layout-dark.scss`
  - 与页面布局入口合并，只保留 `html[data-theme]` 分支

- `draw/_modern-lab-theme.scss`
  - 保留为皮肤层，但建立在新 token 之上

- `draw/index.scss`
  - 退出“总入口”角色
  - 本轮重构后不再作为业务消费入口继续使用

### 12.2 全局主题文件

以下全局文件需要一并收口：

- `theme-variables.scss`
  - 保留应用级 token 责任
  - 不承载 `draw` 子系统专属视觉逻辑

- `dark-mode-global.scss`
  - 仅保留真正的全站 dark 补丁
  - 删除 `draw` 专属控件或 `draw` 页面专属覆盖

- `element-ui-dark.scss`
  - 若与 `dark-mode-global.scss` 或 `draw` foundation 存在重复内容，则合并或删除重复源
  - 最终不能保留多份互相竞争的 `Element UI` dark 规则

### 12.3 主题切换残留

`ThemeToggle.vue` 中与 `ThemeManager` 命名不一致的主题样式残留应一并修正，确保：

- 主题 class / data attribute 使用单一命名体系
- `draw` 皮肤选择器不依赖无效或未实际写入的类名

## 13. 深色模式收口策略

### 13.1 基本原则

深色模式收口遵循以下原则：

1. 不在 `draw` SCSS 中保留独立主题判定逻辑
2. 不允许一个 `Element UI` 控件同时被三份 dark 规则覆盖
3. `draw` 的 dark 行为必须通过 token 映射或作用域内覆盖推导出来
4. 页面布局深色覆盖只存在于页面布局入口，不散落到组件文件中

### 13.2 覆盖优先级

新的优先级模型为：

1. `html[data-theme]` 设置应用级 token
2. `draw token` 由应用级 token 映射得到
3. `draw foundation` 消费 `draw token`
4. `draw page/layout` 在显式页面作用域内叠加页面视觉
5. 组件局部样式只补自身特例，不再承担系统级 dark 修复

### 13.3 允许的修复范围

本轮可以顺手修复以下问题：

- 深色模式下文字、边框、背景对比度异常
- 输入框、选择器、下拉菜单、按钮、卡片在 `draw` 作用域中的 dark 表现不一致
- 页面布局背景、阴影、操作面板在 dark 模式下层次混乱
- 主题切换后因错误选择器导致的局部不生效问题

## 14. 验收标准

重构完成后，必须满足以下验收标准：

1. `src/main.js` 不再全局引入 `draw` 总样式入口
2. `src/views/goto/components/form/draw/*` 中不再存在对 `draw/index.scss` 的整包引入
3. `draw` 页面布局样式只在显式页面作用域下生效
4. 非绘图页面只接入基础入口时，不会被 `#sub-module`、操作面板等页面样式污染
5. 绘图页面在显式页面入口下，深色 / 浅色模式均能正常显示
6. `draw` 基础控件在统计页、`plotSetting` 页等复用页面中可正常显示
7. `draw` SCSS 中不再自行使用 `prefers-color-scheme` 判定主题
8. `draw` 主题问题的排查路径清晰，可按 `ThemeManager -> html[data-theme] -> app token -> draw token -> scope 内样式` 逐层定位

## 15. 验证场景

### 15.1 绘图主页面

重点验证：

- `#sub-module` 主布局
- 左右面板
- 操作区按钮
- Tab、画布控制区、拖拽手柄
- 深色 / 浅色模式切换后的布局与视觉层次

### 15.2 工具页

重点验证：

- 工具页属性面板
- 弹窗、选择器、输入框、下载设置区
- 非绘图主页面场景下的 `draw` 页面布局入口是否独立生效

### 15.3 复用基础控件的非绘图页面

重点验证：

- 统计页中的上传器、输入框、选择器
- `plotSetting` 页中的列表项、复选框、输入控件
- 仅接入 foundation 时页面无额外布局污染

### 15.4 主题切换链路

重点验证：

- `auto / light / dark` 三种设置
- `ThemeToggle -> Vuex settings -> ThemeManager -> html[data-theme]` 的一致性
- `draw` token 是否随应用级 token 正确切换

## 16. 风险与控制

### 16.1 主要风险

- `draw` 基础控件已经跨多个页面复用，一次性切换时容易漏掉某些消费页面
- 某些页面当前依赖“偷吃整包入口”的副作用，拆分后会暴露出缺失的显式作用域
- `Element UI` dark 覆盖存在历史重复来源，合并时若判断不清可能造成局部回归

### 16.2 控制策略

- 以“消费方是否使用 `draw` 基础控件”为维度梳理接入点，而不是只盯 `form/draw` 目录
- 所有复用页面统一补显式 `draw-theme-scope`
- 全局 dark 覆盖与 `draw` 覆盖在职责上先分层，再做删除或合并
- 一次性切换完成后，以场景验收而非单文件验收为准

## 17. 设计结论

本次 `goto-web draw` 主题重构采用以下最终结论：

- `draw` 不再全局加载
- `draw` 基础控件继续保留跨页复用能力，但必须由调用方显式接入 `draw` 作用域
- `draw` 页面布局与基础控件样式彻底拆层
- `draw` 主题判定只跟随 `html[data-theme]`
- `draw` 的深色模式、Element UI 覆盖和皮肤层统一收口为单一路径
- 本轮采用一次性切换，不保留旧入口长期并行

该设计完成后，`draw` 主题体系将从“入口混杂、规则重叠、全局污染”转变为“分层明确、作用域清晰、主题来源单一”的可维护结构。
