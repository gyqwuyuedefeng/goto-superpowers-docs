# 分面参数 radial 类型设计

## 背景

当前分面参数由后端 `FacetParams` 初始化参数结构和字典数据，前端 `facetParams.vue` 显式渲染各子参数，`publicFunction.js` 负责类型切换、字段显示、分面变量联动和标签颜色列表生成。

现有分面类型字典使用 `29L`，包含 `none`、`wrap`、`grid`。新字典 `73L` 在保留这三个值的基础上新增 `radial`，显示名称为“环形分面”。`radial` 需要复用 `nested` 和 `facets`，并新增径向分面专属参数。

## 目标

1. 分面类型改用字典 `73L`，让前端类型下拉出现 `radial`。
2. `radial` 类型显示独立参数：
   - `nested`
   - `facets`
   - `scaleA`
   - `scaleR`
   - `spaceA`
   - `spaceR`
   - `radial.strip.position`
3. `radial` 的 `facets` 勾选行为与 `wrap` 一致，能生成标签背景色和标签背景边框色列表。
4. `wrap` 原有 `strip.position` 不受影响；`radial.strip.position` 新增为独立字段。

## 非目标

1. 不重构整个分面参数组件为动态 schema 渲染。
2. 不修改字典数据本身，假设 `73L` 和 `66L` 已由数据库或后台配置提供。
3. 不改变 `none`、`grid`、`wrap` 现有 R 参数含义。

## 后端设计

修改 `goto/common/src/main/java/com/freedom/model/form/FacetParams.java`。

`type` 初始化从 `CacheService.getGotoDictById(29L)` 改为 `CacheService.getGotoDictById(73L)`，默认值继续为 `"none"`。

新增五个 `InternalParameterItem` 字段：

- `scaleA`: `@AutoParamField(name = "scaleA", quotes = false)`，默认 `"T"`。
- `scaleR`: `@AutoParamField(name = "scaleR", quotes = false)`，默认 `"F"`。
- `spaceA`: `@AutoParamField(name = "spaceA", quotes = false)`，默认 `"T"`。
- `spaceR`: `@AutoParamField(name = "spaceR", quotes = false)`，默认 `"T"`。
- `radialStripPosition`: `@AutoParamField(name = "radial.strip.position", quotes = true)`，默认 `"inside"`，字典 `66L`。

这些字段都使用 `@Syncable(nested = true)`，保持与现有 `scaleX`、`spaceX`、`stripPosition` 一致的同步和序列化行为。

保留现有 `stripPosition` 字段，继续生成 `strip.position`，用于 `wrap`。`radialStripPosition` 独立生成 `radial.strip.position`，避免一个字段在不同分面类型下对应两个 R 参数名。

## 前端组件设计

修改 `goto-web/src/views/goto/components/form/draw/facetParams.vue`。

在现有 `scaleX`、`scaleY`、`spaceX`、`spaceY` 控件附近新增四个布尔单选项：

- `scaleA`: 标签“角度轴刻度自适应”。
- `scaleR`: 标签“径向轴刻度自适应”。
- `spaceA`: 标签“角度轴面板自适应”。
- `spaceR`: 标签“径向轴面板自适应”。

在现有 `stripPosition` 控件后新增 `radialStripPosition` 下拉项，界面标签使用“分面标签位置”。它读取 `setting.radialStripPosition.value.selectList`，修改参数名为 `radialStripPosition`。

新增控件继续通过 `SettingForm :container="setting.xxx"` 判断显示，符合当前组件模式。

## 联动设计

修改 `goto-web/src/constant/modify/publicFunction.js`。

`facetTypeModify` 新增 `radial` 分支：

- 主题相关显示：
  - `stripTextX`、`stripTextY` 隐藏。
  - `stripText`、`stripMargin` 显示。
  - `panelSpace` 如存在则显示。
  - `nestLine` 按现有 `nestedModify` 规则处理；类型切换到 `radial` 时先显示 `nested`，后续由 `nested` 值控制 `nestLine`。
- 分面参数显示：
  - 显示 `nested`、`facets`、`scaleA`、`scaleR`、`spaceA`、`spaceR`、`radialStripPosition`、`fillColorList`、`borderColorList`。
  - 隐藏 `scaleX`、`scaleY`、`spaceX`、`spaceY`、`nrow`、`stripPosition`、`rows`、`cols`。

同时在 `none`、`grid`、`wrap` 分支里显式隐藏新增的 `scaleA`、`scaleR`、`spaceA`、`spaceR`、`radialStripPosition`，避免切换类型后残留显示。

`rowsOrColsOrFacetsModify` 将 `radial` 与 `wrap` 使用相同逻辑：从 `facets.value.list` 中收集已勾选项，去重后生成 `fillColorList` 和 `borderColorList`。

切换 `type` 时继续沿用当前行为：清空 `rows`、`cols`、`facets` 勾选状态，并清空颜色列表。

## 图组个性化逻辑

`pieModify.js` 存在对 `facetParams` 的额外隐藏规则：`grid` 隐藏 `scaleX`、`scaleY`、`spaceX`、`spaceY`，`wrap` 隐藏 `scaleX`、`scaleY`。

新增 `radial` 后，需要检查并补充该图组逻辑，确保饼图切换到 `radial` 时不会错误显示直角坐标分面字段。若该图组不支持 radial 的部分参数，应在该文件中显式隐藏；否则保持通用逻辑。

## 数据流

1. 后端初始化 `FacetParams`，`type.selectList` 来自字典 `73L`，`radialStripPosition.selectList` 来自字典 `66L`。
2. 前端渲染 `facetParams.vue`，下拉选到 `radial` 后触发 `modify`。
3. `associationParamModify` 按图组调用 `PublicFunctions.facetParamsModify`。
4. `facetTypeModify` 更新 `visible` 状态并清空旧类型勾选。
5. 用户在 `facets` 勾选列后，`rowsOrColsOrFacetsModify` 生成分面标签颜色配置。
6. 提交绘图时，`AutoParamCreateUtils` 根据 `@AutoParamField` 输出 `scaleA`、`scaleR`、`spaceA`、`spaceR`、`radial.strip.position`。

## 兼容性

旧数据没有新增字段时，Jackson 当前配置允许忽略未知字段，但缺少字段可能导致前端模板访问 `setting.scaleA` 时报错。因此新增字段应由后端默认初始化提供；如果存在旧的已保存参数配置，需要确认加载时是否会经过默认对象合并。如果不会，实施时需要在前端渲染或后端迁移层补齐缺失字段。

现有 `stripPosition` 不改名，不迁移，不影响 `wrap`。

## 测试方案

1. 后端单元或编译测试：确认 `FacetParams` 新字段编译通过，默认字典 id 和默认值正确。
2. 前端静态检查：确认 `facetParams.vue` 可编译，新字段访问路径存在。
3. 联动验证：
   - `none` 隐藏所有分面子参数。
   - `grid` 显示 rows/cols 和 X/Y 参数，不显示 radial 字段。
   - `wrap` 显示 facets、nrow、stripPosition，不显示 radial 字段。
   - `radial` 显示 facets、scaleA/scaleR/spaceA/spaceR、radialStripPosition，不显示 rows/cols、nrow、stripPosition。
4. 变量勾选验证：`radial` 勾选 `facets` 后生成标签背景色和标签背景边框色列表。
5. R 参数验证：radial 生成 `radial.strip.position="inside"`，不误生成 radial 专属值到 `strip.position`。

## 实施风险

主要风险是旧保存配置缺少新增字段导致前端模板访问 undefined。实施计划需要先确认参数加载路径是否会补齐默认 `FacetParams`，必要时增加缺省字段保护或迁移补齐逻辑。

另一个风险是部分图组个性化 modify 函数覆盖通用显示逻辑。实施时需要重点验证已调用 `PublicFunctions.facetParamsModify` 的图组，尤其是 `pieModify.js`。
