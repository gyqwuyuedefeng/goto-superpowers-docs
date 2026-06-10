# goto-web draw 表单窄宽度布局 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `goto-web` 的 `draw` 表单在窄容器下按字段组换行，避免 `label + select/input` 拆裂，并稳定复合坐标字段的 `x/y` 行内布局。

**Architecture:** 先补一层 Jest 样式回归测试，锁定“整组换行、组内不拆”的公共契约。实现分三层推进：外层/嵌套网格负责给复合字段更合理的列跨度，共享表单样式负责 label 与控件的收缩链路，具体字段组件负责接入这套契约并清理局部冲突。最后用现有 draw 样式测试和手工窄宽度回归一起兜底。

**Tech Stack:** Vue 2 SFC, SCSS, Element UI, Jest (`vue-cli-service test:unit`), ESLint

---

## File Map

- Create: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
  - 窄宽度布局回归测试，覆盖外层网格、嵌套 `list-param` 网格、共享字段契约、普通字段组件与复合坐标字段。
- Modify: `goto-web/src/assets/styles/draw/_grid.scss`
  - `drawGroupOptional` 外层网格规则；给复合字段预留更宽的列跨度类。
- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`
  - `list-param` 和 `text-param-other` 这类嵌套网格的最小列宽与跨列规则。
- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`
  - `form-group / form-label / form-label-wrapper / form-content / form-content-item` 的共享窄宽度契约。
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
  - `modern-lab` 输入/下拉在共享契约上的收缩链与焦点状态兼容。
- Modify: `goto-web/src/views/goto/components/drawGroupOptional.vue`
  - 动态字段容器类名分配；让 `moduleType 25` 走更宽的栅格跨度。
- Modify: `goto-web/src/views/goto/components/form/draw/moduleRegistry.js`
  - 提供窄宽度布局所需的模块类型到栅格类映射辅助函数。
- Modify: `goto-web/src/views/goto/components/form/draw/legendSettingsParam.vue`
  - 图例设置内部的 `legendInsideCoord` 使用更宽的 `SettingForm` 跨列类。
- Modify: `goto-web/src/views/goto/components/form/draw/input.vue`
  - 普通输入字段接入共享字段组类，补齐 `min-width`/收缩链。
- Modify: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
  - 单选下拉接入共享字段组类，清理局部 label wrapper 与收缩链冲突。
- Modify: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
  - 多选下拉接入共享字段组类并保持 tag 溢出链路。
- Modify: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
  - 简单下拉接入共享字段组类与统一的 `form-content` 收缩链。
- Modify: `goto-web/src/views/goto/components/form/draw/selectDictImage.vue`
  - 图片选择字段接入共享字段组类，保持隐藏 `el-select` 不挤占布局。
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue`
  - 复合坐标字段保持 `x/y` 行内稳定，不在组内拆裂。

## Task 1: Lock the Narrow-Width Regression Contract

**Files:**
- Create: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`

- [ ] **Step 1: Write the failing regression test file**

```js
const fs = require('fs')
const path = require('path')
const os = require('os')
const { execFileSync } = require('child_process')

function readSource(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

function compileScss(relativePath) {
  const projectRoot = path.resolve(__dirname, '../../../')
  const inputPath = path.resolve(projectRoot, relativePath)
  const outputPath = path.join(os.tmpdir(), `draw-narrow-layout-${Date.now()}-${process.pid}.css`)

  execFileSync('npx', ['sass', inputPath, outputPath], {
    cwd: projectRoot,
    stdio: 'pipe'
  })

  const css = fs.readFileSync(outputPath, 'utf8')
  fs.unlinkSync(outputPath)
  return css
}

describe('draw narrow-width layout contract', () => {
  test('drawGroupOptional and legendSettingsParam reserve a wider slot for single coordinate fields', () => {
    const drawGroupSource = readSource('src/views/goto/components/drawGroupOptional.vue')
    const legendSettingsSource = readSource('src/views/goto/components/form/draw/legendSettingsParam.vue')
    const gridCss = compileScss('src/assets/styles/draw/foundation.scss')

    expect(drawGroupSource).toContain('width-2-columns')
    expect(legendSettingsSource).toContain('width-2-columns')
    expect(gridCss).toContain('.param.container.width-2-columns')
  })
})
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL because the new regression file references selectors/classes that do not exist yet.

- [ ] **Step 3: Expand the test to cover all planned contract points before implementation**

Add assertions for:

```js
expect(compileScss('src/assets/styles/draw/_form-base.scss')).toContain('.draw-field-group')
expect(compileScss('src/assets/styles/draw/_form-components.scss')).toContain('.list-param > .container.width-2-columns')
expect(readSource('src/views/goto/components/form/draw/input.vue')).toContain('draw-field-group')
expect(readSource('src/views/goto/components/form/draw/selectObject.vue')).toContain('draw-field-group')
expect(readSource('src/views/goto/components/form/draw/coordinatePointSingleParam.vue')).toContain('draw-field-group--composite')
```

- [ ] **Step 4: Re-run the focused test and confirm the expanded assertions still fail**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL with missing `draw-field-group`, missing `width-2-columns`, and missing composite field markers.

- [ ] **Step 5: Commit the failing-test harness**

```bash
git add goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js
git commit -m "test: add draw narrow-width layout regression harness"
```

## Task 2: Reserve Wider Grid Slots for Coordinate-Style Fields

**Files:**
- Modify: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Modify: `goto-web/src/assets/styles/draw/_grid.scss`
- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`
- Modify: `goto-web/src/views/goto/components/drawGroupOptional.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/moduleRegistry.js`
- Modify: `goto-web/src/views/goto/components/form/draw/legendSettingsParam.vue`
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`

- [ ] **Step 1: Add failing assertions for outer-grid and nested-grid span classes**

Extend the test with exact expectations for both grid layers:

```js
test('grid layers expose a reusable two-column span contract', () => {
  const foundationCss = compileScss('src/assets/styles/draw/foundation.scss')
  const formComponentsCss = compileScss('src/assets/styles/draw/_form-components.scss')

  expect(foundationCss).toContain('.param.container.width-2-columns')
  expect(foundationCss).toContain('grid-column: span 2')
  expect(formComponentsCss).toContain('.list-param > .container.width-2-columns')
})

test('module registry and dynamic wrapper map coordinate single fields to width-2-columns', () => {
  const registrySource = readSource('src/views/goto/components/form/draw/moduleRegistry.js')
  const drawGroupSource = readSource('src/views/goto/components/drawGroupOptional.vue')

  expect(registrySource).toContain('getGridSpanClass')
  expect(registrySource).toContain('25')
  expect(drawGroupSource).toContain('getGridSpanClass')
})
```

- [ ] **Step 2: Run the focused test and capture the failure**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL because `_grid.scss`, `_form-components.scss`, `moduleRegistry.js`, and `drawGroupOptional.vue` do not yet define the span contract.

- [ ] **Step 3: Implement the minimal grid-span wiring**

Use these code shapes:

```scss
// src/assets/styles/draw/_grid.scss
.param.container.width-2-columns {
  grid-column: span 2;
}
```

```scss
// src/assets/styles/draw/_form-components.scss
.list-param > .container.width-2-columns {
  grid-column: span 2 !important;
}
```

```js
// src/views/goto/components/form/draw/moduleRegistry.js
export function getGridSpanClass(moduleType) {
  if (moduleType === 25) {
    return 'width-2-columns'
  }
  return ''
}
```

```js
// src/views/goto/components/drawGroupOptional.vue
clazz += ` ${getGridSpanClass(subSetting.moduleType)}`
```

```vue
<!-- src/views/goto/components/form/draw/legendSettingsParam.vue -->
<SettingForm class="width-2-columns" :container="setting.legendInsideCoord">
```

- [ ] **Step 4: Run the focused regression test until it passes**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: PASS for the grid-span and mapping assertions.

- [ ] **Step 5: Commit the grid-span changes**

```bash
git add goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js \
        goto-web/src/assets/styles/draw/_grid.scss \
        goto-web/src/assets/styles/draw/_form-components.scss \
        goto-web/src/views/goto/components/drawGroupOptional.vue \
        goto-web/src/views/goto/components/form/draw/moduleRegistry.js \
        goto-web/src/views/goto/components/form/draw/legendSettingsParam.vue
git commit -m "style: reserve wider draw grid slots for coordinate fields"
```

## Task 3: Add the Shared Field-Group Narrow-Width Contract

**Files:**
- Modify: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`

- [ ] **Step 1: Add failing assertions for the shared field-group contract**

Add a test that locks the common selectors and behaviors:

```js
test('shared form styles define a non-breaking field-group contract', () => {
  const baseCss = compileScss('src/assets/styles/draw/_form-base.scss')
  const modernLabCss = compileScss('src/assets/styles/draw/_modern-lab-theme.scss')

  expect(baseCss).toContain('.form-group.draw-field-group')
  expect(baseCss).toContain('min-width: 0')
  expect(baseCss).toContain('text-overflow: ellipsis')
  expect(baseCss).toContain('.form-group.draw-field-group .form-content')
  expect(modernLabCss).toContain('.modern-lab-form-group.draw-field-group')
})
```

- [ ] **Step 2: Run the focused test and verify the contract is missing**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL because `.draw-field-group` has not been introduced in the shared SCSS yet.

- [ ] **Step 3: Implement the shared contract in the base and modern-lab layers**

Use concrete selectors like:

```scss
// src/assets/styles/draw/_form-base.scss
.form-group.draw-field-group {
  min-width: 0;
}

.form-group.draw-field-group .form-label,
.form-group.draw-field-group .form-label-wrapper {
  min-width: 0;
  max-width: 100%;
}

.form-group.draw-field-group .form-content,
.form-group.draw-field-group .form-content-item {
  min-width: 0;
  width: 100%;
}
```

```scss
// src/assets/styles/draw/_modern-lab-theme.scss
.modern-lab-form-group.draw-field-group {
  .form-content {
    min-width: 0;
  }
}
```

- [ ] **Step 4: Re-run the focused regression test and confirm the shared-style assertions pass**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: PASS for the shared contract selectors while later component-source assertions may still fail.

- [ ] **Step 5: Commit the shared-style contract**

```bash
git add goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js \
        goto-web/src/assets/styles/draw/_form-base.scss \
        goto-web/src/assets/styles/draw/_modern-lab-theme.scss
git commit -m "style: add shared draw field-group narrow-width contract"
```

## Task 4: Align the Common Select/Input Components to the Contract

**Files:**
- Modify: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Modify: `goto-web/src/views/goto/components/form/draw/input.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectDictImage.vue`
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`

- [ ] **Step 1: Add failing source assertions for the five common field components**

Add explicit checks:

```js
test('common draw field components opt into draw-field-group and min-width shrink chains', () => {
  expect(readSource('src/views/goto/components/form/draw/input.vue')).toContain('draw-field-group')
  expect(readSource('src/views/goto/components/form/draw/selectObject.vue')).toContain('draw-field-group')
  expect(readSource('src/views/goto/components/form/draw/selectMultipleObject.vue')).toContain('draw-field-group')
  expect(readSource('src/views/goto/components/form/draw/selectSimple.vue')).toContain('draw-field-group')
  expect(readSource('src/views/goto/components/form/draw/selectDictImage.vue')).toContain('draw-field-group')
  expect(readSource('src/views/goto/components/form/draw/selectObject.vue')).toMatch(/\.form-content[\s\S]*min-width:\s*0;/)
  expect(readSource('src/views/goto/components/form/draw/selectDictImage.vue')).toMatch(/\.form-content[\s\S]*min-width:\s*0;/)
})
```

- [ ] **Step 2: Run the focused regression test and confirm these source checks fail**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL because the components do not yet share a common root class or a consistent `min-width` chain.

- [ ] **Step 3: Apply the minimal component-level changes**

Use these code shapes:

```vue
<!-- input.vue -->
<div class="form-group draw-field-group modern-lab-form-group modern-lab-input">
```

```vue
<!-- selectObject.vue -->
<div class="form-group draw-field-group modern-lab-form-group modern-lab-select">
```

```vue
<!-- selectSimple.vue / selectDictImage.vue -->
<div class="form-group draw-field-group" ...>
```

```scss
.form-content {
  min-width: 0;
  width: 100%;
}

.form-content-item,
.form-label-wrapper {
  min-width: 0;
}
```

Do not rewrite business logic; keep the change scoped to root classes, label wrappers, and shrink-chain CSS.

- [ ] **Step 4: Re-run the focused regression test and lint the touched Vue files**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
cd goto-web && npx eslint src/views/goto/components/form/draw/input.vue \
  src/views/goto/components/form/draw/selectObject.vue \
  src/views/goto/components/form/draw/selectMultipleObject.vue \
  src/views/goto/components/form/draw/selectSimple.vue \
  src/views/goto/components/form/draw/selectDictImage.vue
```

Expected: targeted Jest file PASS; ESLint exits 0.

- [ ] **Step 5: Commit the common field-component alignment**

```bash
git add goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js \
        goto-web/src/views/goto/components/form/draw/input.vue \
        goto-web/src/views/goto/components/form/draw/selectObject.vue \
        goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue \
        goto-web/src/views/goto/components/form/draw/selectSimple.vue \
        goto-web/src/views/goto/components/form/draw/selectDictImage.vue
git commit -m "style: align draw select and input fields to narrow-width contract"
```

## Task 5: Stabilize the Composite Coordinate Field

**Files:**
- Modify: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue`
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`

- [ ] **Step 1: Add failing assertions for the composite coordinate layout**

Lock the intended source contract:

```js
test('coordinatePointSingleParam keeps x/y pairs inline inside a composite field group', () => {
  const source = readSource('src/views/goto/components/form/draw/coordinatePointSingleParam.vue')

  expect(source).toContain('draw-field-group--composite')
  expect(source).toMatch(/\.coordinate-point-single[\s\S]*flex-wrap:\s*nowrap;/)
  expect(source).toMatch(/\.coordinate-point-single[\s\S]*min-width:\s*0;/)
  expect(source).toMatch(/\.coordinate-point-single[\s\S]*\.container[\s\S]*flex:\s*1/)
})
```

- [ ] **Step 2: Run the focused regression test and confirm the composite-field checks fail**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles/drawNarrowWidthLayout.spec.js --runInBand
```

Expected: FAIL because the component does not yet advertise a composite field-group marker or a fully locked inline pair rule.

- [ ] **Step 3: Implement the composite-field layout guardrails**

Use these concrete changes:

```vue
<div class="form-group draw-field-group draw-field-group--composite">
```

```scss
.coordinate-point-single {
  display: flex;
  flex-wrap: nowrap;
  min-width: 0;
}

.coordinate-point-single > .container,
.coordinate-input-wrapper {
  min-width: 0;
  flex: 1 1 0;
}
```

Keep `x` and `y` inline; do not convert the component to stacked mobile rows.

- [ ] **Step 4: Re-run the focused regression test and the existing coordinate theme test**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit \
  tests/unit/styles/drawNarrowWidthLayout.spec.js \
  tests/unit/styles/coordinatePointTheme.spec.js \
  --runInBand
```

Expected: both test files PASS.

- [ ] **Step 5: Commit the composite-field stabilization**

```bash
git add goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js \
        goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue
git commit -m "style: stabilize draw coordinate field narrow-width layout"
```

## Task 6: Run the Full Regression Slice and Manual Narrow-Width Verification

**Files:**
- Test: `goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js`
- Test: `goto-web/tests/unit/styles/drawArchitecture.spec.js`
- Test: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
- Test: `goto-web/tests/unit/styles/drawRegressionFixes.spec.js`
- Test: `goto-web/tests/unit/styles/coordinatePointTheme.spec.js`

- [ ] **Step 1: Run the draw style regression slice**

Run:

```bash
cd goto-web && npx vue-cli-service test:unit tests/unit/styles --runInBand
```

Expected: PASS for the new narrow-width suite and the existing draw style suites.

- [ ] **Step 2: Run ESLint across the touched Vue/JS sources**

Run:

```bash
cd goto-web && npx eslint src/views/goto/components/drawGroupOptional.vue \
  src/views/goto/components/form/draw/moduleRegistry.js \
  src/views/goto/components/form/draw/legendSettingsParam.vue \
  src/views/goto/components/form/draw/input.vue \
  src/views/goto/components/form/draw/selectObject.vue \
  src/views/goto/components/form/draw/selectMultipleObject.vue \
  src/views/goto/components/form/draw/selectSimple.vue \
  src/views/goto/components/form/draw/selectDictImage.vue \
  src/views/goto/components/form/draw/coordinatePointSingleParam.vue
```

Expected: exit code 0.

- [ ] **Step 3: Run the app locally and manually verify the narrow-width behavior**

Run:

```bash
cd goto-web && NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service serve
```

Manual checks:

- Open the draw page containing legend settings.
- Narrow the form container until the old screenshot used to break.
- Verify `图例位置 / 内部相对坐标 / 排列方式` now wrap by field group.
- Verify `内部相对坐标` keeps `x` and `y` on one row inside the group.
- Spot-check one normal input field and one normal select field elsewhere in `draw` for the same grouped-wrap behavior.

- [ ] **Step 4: Review the diff to confirm only expected layout files changed**

Run:

```bash
git diff --stat
git diff -- goto-web/src/assets/styles/draw/_grid.scss \
  goto-web/src/assets/styles/draw/_form-components.scss \
  goto-web/src/assets/styles/draw/_form-base.scss \
  goto-web/src/assets/styles/draw/_modern-lab-theme.scss \
  goto-web/src/views/goto/components/drawGroupOptional.vue \
  goto-web/src/views/goto/components/form/draw/moduleRegistry.js \
  goto-web/src/views/goto/components/form/draw/legendSettingsParam.vue \
  goto-web/src/views/goto/components/form/draw/input.vue \
  goto-web/src/views/goto/components/form/draw/selectObject.vue \
  goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue \
  goto-web/src/views/goto/components/form/draw/selectSimple.vue \
  goto-web/src/views/goto/components/form/draw/selectDictImage.vue \
  goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue \
  goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js
```

Expected: only the planned SCSS, Vue, JS, and test files appear.

- [ ] **Step 5: Commit the verified implementation slice**

```bash
git add goto-web/src/assets/styles/draw/_grid.scss \
        goto-web/src/assets/styles/draw/_form-components.scss \
        goto-web/src/assets/styles/draw/_form-base.scss \
        goto-web/src/assets/styles/draw/_modern-lab-theme.scss \
        goto-web/src/views/goto/components/drawGroupOptional.vue \
        goto-web/src/views/goto/components/form/draw/moduleRegistry.js \
        goto-web/src/views/goto/components/form/draw/legendSettingsParam.vue \
        goto-web/src/views/goto/components/form/draw/input.vue \
        goto-web/src/views/goto/components/form/draw/selectObject.vue \
        goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue \
        goto-web/src/views/goto/components/form/draw/selectSimple.vue \
        goto-web/src/views/goto/components/form/draw/selectDictImage.vue \
        goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue \
        goto-web/tests/unit/styles/drawNarrowWidthLayout.spec.js
git commit -m "style: fix draw narrow-width grouped field layout"
```
