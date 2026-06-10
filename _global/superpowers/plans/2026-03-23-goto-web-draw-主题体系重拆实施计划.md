# goto-web draw 主题体系重拆 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `goto-web` 的 `draw` 样式从“全局整包注入 + 组件重复导入 + 多源 dark override”重构为“显式作用域 + 双入口 + 单一 `html[data-theme]` 来源”的可维护体系。

**Architecture:** 保留应用级主题变量为上游真相，在 `draw` 层拆出 `tokens`、`foundation`、`page layout` 三类入口，并让所有真实样式规则受 `.draw-theme-scope` 或页面作用域控制。基础控件消费页只接入 foundation，绘图页额外接入页面布局入口，所有 body 级 popper 使用统一 `draw-theme-popper` 契约。

**Tech Stack:** Vue 2.6, Element UI 2.15, Vue CLI 3, SCSS (`sass-loader@10`), Jest (`@vue/cli-plugin-unit-jest`), Vuex `settings`, `ThemeManager`

---

## Implementation Notes

- 先按 `@test-driven-development` 写失败的源文件守卫测试，再做样式重构。
- 每个任务完成后都要跑该任务列出的 Jest / ESLint 命令，再提交。
- 若某一步与预期不符，先用 `@systematic-debugging` 定位，不要跳过失败验证。
- 在宣称“完成”之前，执行 `@verification-before-completion` 风格的验证：至少覆盖样式守卫测试、关键 lint、目标页面 smoke 检查。
- 如果当前会话具备 GitNexus MCP/CLI，可在开工前对以下入口做一次范围确认：`goto-web/src/main.js`、`goto-web/src/views/goto/plot/type/draw.vue`、`goto-web/src/views/goto/toolsBox/type/tool.vue`、`goto-web/src/views/goto/components/form/draw/selectObject.vue`。若不可用，使用本计划中的 `rg` 清单和 Jest 守卫测试代替。

## File Structure / Change Map

### New Style Entrypoints

- Create: `goto-web/src/assets/styles/draw/tokens.scss`
  - 负责 `--draw-*` 变量映射，只消费应用级主题状态
- Create: `goto-web/src/assets/styles/draw/foundation.scss`
  - 负责 `.draw-theme-scope` 下的基础表单类、控件覆盖、皮肤层
- Create: `goto-web/src/assets/styles/draw/layout-plot.scss`
  - 负责绘图主页面专属布局
- Create: `goto-web/src/assets/styles/draw/layout-toolbox.scss`
  - 负责工具页专属布局

### Existing Style Partials To Refactor

- Modify: `goto-web/src/assets/styles/draw/_variables.scss`
- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`
- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`
- Modify: `goto-web/src/assets/styles/draw/_element-overrides.scss`
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
- Modify: `goto-web/src/assets/styles/draw/_layout.scss`
- Modify: `goto-web/src/assets/styles/draw/_layout-dark.scss`
- Delete: `goto-web/src/assets/styles/draw/index.scss`

### Global Theme Cleanup

- Modify: `goto-web/src/assets/styles/theme-variables.scss`
- Modify: `goto-web/src/assets/styles/dark-mode-global.scss`
- Modify: `goto-web/src/assets/styles/element-ui-dark.scss`
- Modify: `goto-web/src/views/goto/components/ThemeToggle.vue`
- Modify: `goto-web/src/utils/theme.js` only if a naming mismatch cannot be fixed in CSS alone

### Draw Component Files To Migrate Off `draw/index.scss`

- Modify: `goto-web/src/views/goto/components/form/draw/input.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/inputList.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/radio.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/range.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectDictImage.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/colorPicker.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/pointParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/legendPosition.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesShow.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesSettings.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/by.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/list.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/slider.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/textParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyItem.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/settingChooseColumnNameMultiple.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`

### Consumer Entrypoints / Containers

- Modify: `goto-web/src/main.js`
- Modify: `goto-web/src/views/goto/plot/type/draw.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Modify: `goto-web/src/views/goto/components/drawGroupOptional.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`
- Modify: `goto-web/src/views/goto/statistics/type/operate.vue`
- Modify: `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue`

### Tests

- Create: `goto-web/tests/unit/styles/drawArchitecture.spec.js`
- Modify: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
- Modify: `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js`
- Modify: `goto-web/tests/unit/styles/coordinatePointTheme.spec.js` only if assertions need to follow the new contract

## Task 1: Add Guardrail Tests For The New Draw Architecture

**Files:**
- Create: `goto-web/tests/unit/styles/drawArchitecture.spec.js`
- Reference: `goto-web/jest.config.js`

- [ ] **Step 1: Write a failing architecture-guard test file for the new entrypoints**

```js
const fs = require('fs')
const path = require('path')

function read(relPath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relPath), 'utf8')
}

describe('draw style architecture', () => {
  test('new scoped entry files exist', () => {
    expect(fs.existsSync(path.resolve(__dirname, '../../../src/assets/styles/draw/tokens.scss'))).toBe(true)
    expect(fs.existsSync(path.resolve(__dirname, '../../../src/assets/styles/draw/foundation.scss'))).toBe(true)
    expect(fs.existsSync(path.resolve(__dirname, '../../../src/assets/styles/draw/layout-plot.scss'))).toBe(true)
    expect(fs.existsSync(path.resolve(__dirname, '../../../src/assets/styles/draw/layout-toolbox.scss'))).toBe(true)
  })
})
```

- [ ] **Step 2: Run the targeted Jest suite and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- FAIL because the new entry files do not exist yet

- [ ] **Step 3: Create empty-but-valid entrypoint skeletons so later tasks have stable targets**

Create:

```scss
/* src/assets/styles/draw/tokens.scss */
@import './_variables';

/* src/assets/styles/draw/foundation.scss */
@import './tokens';

/* src/assets/styles/draw/layout-plot.scss */
@import './tokens';

/* src/assets/styles/draw/layout-toolbox.scss */
@import './tokens';
```

Do not move behavior yet. This step only establishes the contract that later tasks will fill in.

- [ ] **Step 4: Run the targeted Jest suite again**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- PASS on the file-existence assertions for the new entry files

- [ ] **Step 5: Commit the test scaffolding and entrypoint skeletons**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/tests/unit/styles/drawArchitecture.spec.js \
  goto-web/src/assets/styles/draw/tokens.scss \
  goto-web/src/assets/styles/draw/foundation.scss \
  goto-web/src/assets/styles/draw/layout-plot.scss \
  goto-web/src/assets/styles/draw/layout-toolbox.scss
git commit -m "test: add draw style architecture guards"
```

## Task 2: Rebuild Draw Tokens And Foundation Around `html[data-theme]`

**Files:**
- Modify: `goto-web/src/assets/styles/draw/_variables.scss`
- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`
- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`
- Modify: `goto-web/src/assets/styles/draw/_element-overrides.scss`
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
- Modify: `goto-web/src/assets/styles/draw/tokens.scss`
- Modify: `goto-web/src/assets/styles/draw/foundation.scss`
- Modify: `goto-web/src/assets/styles/theme-variables.scss`
- Modify: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`

- [ ] **Step 1: Write or tighten failing assertions for token-source cleanup**

Add assertions such as:

```js
const source = readSource('src/assets/styles/draw/tokens.scss')
expect(source).toMatch(/data-theme="dark"/)
expect(source).not.toMatch(/prefers-color-scheme/)
```

Add source assertions for foundation:

```js
const foundation = readSource('src/assets/styles/draw/foundation.scss')
expect(foundation).toMatch(/\.draw-theme-scope/)
```

- [ ] **Step 2: Run the focused token/foundation tests and confirm they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawDarkTheme.spec.js tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- FAIL because `_variables.scss` still contains `prefers-color-scheme`
- FAIL because foundation rules are not yet scoped

- [ ] **Step 3: Refactor token mapping and foundation scoping**

Implement the following:

1. Keep `--draw-*` variables globally available through `tokens.scss`, anchored to the app theme state.
2. Remove `draw`-owned `prefers-color-scheme` logic.
3. Convert the existing draw base partials so they can be emitted under `.draw-theme-scope` in a deterministic way.
4. Keep `modern-lab` as a skin layer that consumes `--draw-*`, not as a second theme source.

Target pattern:

```scss
/* tokens.scss */
:root,
html[data-theme='light'] {
  --draw-bg-primary: var(--card-bg);
}

html[data-theme='dark'] {
  --draw-bg-primary: var(--card-bg);
}

/* foundation.scss */
.draw-theme-scope {
  @include draw-form-base;
  @include draw-form-components;
  @include draw-element-overrides;
  @include draw-modern-lab-skin;
}
```

Make the partials buildable by choosing one explicit strategy and using it consistently:

- preferred: turn `_form-base.scss`, `_form-components.scss`, `_element-overrides.scss`, `_modern-lab-theme.scss` into mixin-emitting partials
- acceptable fallback: duplicate only the minimal selectors into `foundation.scss` and delete the now-dead legacy partial output in Task 5

Do not rely on ambiguous nested Sass `@import` behavior.

- [ ] **Step 4: Re-run the focused tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawDarkTheme.spec.js tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- PASS for token-source and scope assertions

- [ ] **Step 5: Run grep sanity checks for removed theme branches**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
rg -n "prefers-color-scheme" src/assets/styles/draw
```

Expected:

- no matches

- [ ] **Step 6: Commit the token/foundation refactor**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/src/assets/styles/draw/_variables.scss \
  goto-web/src/assets/styles/draw/_form-base.scss \
  goto-web/src/assets/styles/draw/_form-components.scss \
  goto-web/src/assets/styles/draw/_element-overrides.scss \
  goto-web/src/assets/styles/draw/_modern-lab-theme.scss \
  goto-web/src/assets/styles/draw/tokens.scss \
  goto-web/src/assets/styles/draw/foundation.scss \
  goto-web/src/assets/styles/theme-variables.scss \
  goto-web/tests/unit/styles/drawDarkTheme.spec.js
git commit -m "refactor: rebuild draw tokens and foundation"
```

## Task 3: Migrate Draw Components Off `draw/index.scss` And Add Overlay Contracts

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/input.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/inputList.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/radio.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/range.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectDictImage.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/colorPicker.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/coordinatePointSingleParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/pointParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/legendPosition.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesShow.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesSettings.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/by.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/list.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/slider.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/textParam.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyItem.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/settingChooseColumnNameMultiple.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`
- Modify: `goto-web/tests/unit/styles/drawArchitecture.spec.js`
- Modify: `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js`
- Modify: `goto-web/tests/unit/styles/coordinatePointTheme.spec.js` if needed

- [ ] **Step 1: Add a failing assertion that no draw component imports `draw/index.scss`**

In `drawArchitecture.spec.js`, scan the entire `src/views/goto/components/form/draw` directory:

```js
const drawDir = path.resolve(__dirname, '../../../src/views/goto/components/form/draw')
const offenders = fs.readdirSync(drawDir)
  .filter(name => name.endsWith('.vue'))
  .filter(name => /draw\/index\.scss/.test(fs.readFileSync(path.join(drawDir, name), 'utf8')))

expect(offenders).toEqual([])
```

Also add assertions for popper classes:

```js
expect(read('src/views/goto/components/form/draw/selectObject.vue')).toMatch(/draw-theme-popper/)
expect(read('src/views/goto/components/form/draw/selectMultipleObject.vue')).toMatch(/draw-theme-popper/)
expect(read('src/views/goto/components/form/draw/selectSimple.vue')).toMatch(/draw-theme-popper/)
expect(read('src/views/goto/components/form/draw/selectDictImage.vue')).toMatch(/draw-theme-popper/)
```

- [ ] **Step 2: Run the component-architecture tests and confirm they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js tests/unit/styles/classifyLegendParamTheme.spec.js tests/unit/styles/coordinatePointTheme.spec.js --runInBand
```

Expected:

- FAIL because many components still import `draw/index.scss`
- FAIL because popper classes do not yet include the new contract

- [ ] **Step 3: Remove `draw/index.scss` imports from draw components**

For each file listed above:

- delete `@import "@/assets/styles/draw/index.scss";`
- keep local component styles that reference `var(--draw-*)`
- if a component was only importing `draw/index.scss` for base classes, rely on outer `foundation.scss` instead
- if a component was importing `theme-variables.scss` only to read colors that now have `--draw-*` equivalents, switch to `--draw-*`

- [ ] **Step 4: Standardize body-level overlay classes**

Update draw overlay components to append a shared contract class:

```vue
<el-select
  :popper-class="'g-select draw-theme-popper'"
  :popper-append-to-body="true"
/>
```

Use the same idea for:

- select dropdowns
- draw-specific tooltips that need theme-aware body overlays
- image/dict dropdowns

Do not widen global selectors like `.el-select-dropdown` without a `draw`-specific class.

- [ ] **Step 5: Re-run the targeted source tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js tests/unit/styles/classifyLegendParamTheme.spec.js tests/unit/styles/coordinatePointTheme.spec.js --runInBand
```

Expected:

- PASS on “no `draw/index.scss` import” checks
- PASS on `var(--draw-*)` assertions
- PASS on `draw-theme-popper` assertions

- [ ] **Step 6: Lint the migrated Vue components**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint \
  src/views/goto/components/form/draw/input.vue \
  src/views/goto/components/form/draw/selectObject.vue \
  src/views/goto/components/form/draw/selectMultipleObject.vue \
  src/views/goto/components/form/draw/selectSimple.vue \
  src/views/goto/components/form/draw/selectDictImage.vue \
  src/views/goto/components/form/draw/classifyLegendParam.vue
```

Expected:

- PASS or only pre-existing warnings unrelated to the refactor

- [ ] **Step 7: Commit the draw component migration**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/src/views/goto/components/form/draw \
  goto-web/tests/unit/styles/drawArchitecture.spec.js \
  goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js \
  goto-web/tests/unit/styles/coordinatePointTheme.spec.js
git commit -m "refactor: migrate draw components to scoped foundation"
```

## Task 4: Migrate Consumer Containers To Explicit `draw-theme-scope`

**Files:**
- Modify: `goto-web/src/views/goto/plot/type/draw.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Modify: `goto-web/src/views/goto/components/drawGroupOptional.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`
- Modify: `goto-web/src/views/goto/statistics/type/operate.vue`
- Modify: `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue`
- Modify: `goto-web/src/main.js`
- Modify: `goto-web/tests/unit/styles/drawArchitecture.spec.js`

- [ ] **Step 1: Add failing tests for consumer entrypoints**

Add assertions such as:

```js
expect(read('src/views/goto/plot/type/draw.vue')).toMatch(/draw-page[\s\S]*draw-theme-scope/)
expect(read('src/views/goto/toolsBox/type/tool.vue')).toMatch(/draw-page[\s\S]*draw-theme-scope/)
expect(read('src/views/goto/statistics/type/operate.vue')).toMatch(/draw-theme-scope/)
expect(read('src/main.js')).not.toMatch(/assets\/styles\/draw\/index\.scss/)
```

If you choose import-based assertions, look for:

- `@import "@/assets/styles/draw/foundation.scss"`
- `@import "@/assets/styles/draw/layout-plot.scss"`
- `@import "@/assets/styles/draw/layout-toolbox.scss"`

- [ ] **Step 2: Run the consumer-entrypoint tests and confirm they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- FAIL because the root containers do not yet expose the new scope classes
- FAIL because `main.js` still imports the global draw stylesheet

- [ ] **Step 3: Move style loading to the correct containers**

Implement the following:

1. Remove `import "./assets/styles/draw/index.scss"` from `src/main.js`.
2. In `draw.vue`, add `draw-page draw-theme-scope` to the root and import:

```scss
@import '@/assets/styles/draw/foundation.scss';
@import '@/assets/styles/draw/layout-plot.scss';
```

3. In `tool.vue`, add the same explicit page scope and import `layout-toolbox.scss`.
4. In `statistics/type/operate.vue`, wrap the active statistics module tree in `.draw-theme-scope` and import `foundation.scss`.
5. In `drawGroupOptional.vue` and `listMarkColumnContentParam.vue`, replace direct `draw/index.scss` usage with scoped foundation usage.
6. Update `PlotSidePanel.vue` and `PlotResultCanvas.vue` to import from the new stable entrypoints/partials rather than raw legacy files where appropriate.

Important implementation detail:

- `foundation.scss`, `layout-plot.scss`, and `layout-toolbox.scss` must be imported from **unscoped** `<style lang="scss">` blocks or another global style entry.
- If they are imported from `<style scoped>`, Vue will rewrite the selectors and the styles will not reach nested child-component DOM or body-level poppers.

- [ ] **Step 4: Re-run the entrypoint tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- PASS for root-scope and import-boundary assertions

- [ ] **Step 5: Lint the touched container files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint \
  src/main.js \
  src/views/goto/plot/type/draw.vue \
  src/views/goto/toolsBox/type/tool.vue \
  src/views/goto/components/drawGroupOptional.vue \
  src/views/goto/plot/type/components/PlotSidePanel.vue \
  src/views/goto/plot/type/components/PlotResultCanvas.vue \
  src/views/goto/statistics/type/operate.vue \
  src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue \
  src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue
```

Expected:

- PASS or only pre-existing warnings

- [ ] **Step 6: Commit the consumer-entrypoint migration**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/src/main.js \
  goto-web/src/views/goto/plot/type/draw.vue \
  goto-web/src/views/goto/toolsBox/type/tool.vue \
  goto-web/src/views/goto/components/drawGroupOptional.vue \
  goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue \
  goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue \
  goto-web/src/views/goto/statistics/type/operate.vue \
  goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue \
  goto-web/src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue \
  goto-web/tests/unit/styles/drawArchitecture.spec.js
git commit -m "refactor: scope draw style consumers"
```

## Task 5: Consolidate Global Dark Overrides And Remove Legacy Entrypoints

**Files:**
- Modify: `goto-web/src/assets/styles/dark-mode-global.scss`
- Modify: `goto-web/src/assets/styles/element-ui-dark.scss`
- Modify: `goto-web/src/views/goto/components/ThemeToggle.vue`
- Delete: `goto-web/src/assets/styles/draw/index.scss`
- Modify: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
- Modify: `goto-web/tests/unit/styles/drawArchitecture.spec.js`

- [ ] **Step 1: Add failing assertions for the final cleanup state**

Examples:

```js
expect(fs.existsSync(path.resolve(__dirname, '../../../src/assets/styles/draw/index.scss'))).toBe(false)
expect(read('src/views/goto/components/ThemeToggle.vue')).not.toMatch(/:global\(\.dark\)/)
expect(read('src/assets/styles/dark-mode-global.scss')).not.toMatch(/#sub-module/)
```

If you prefer softer guards for `dark-mode-global.scss`, assert that `draw` page selectors live in `layout-plot.scss` / `layout-toolbox.scss` instead.

- [ ] **Step 2: Run the final cleanup tests and confirm they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawDarkTheme.spec.js tests/unit/styles/drawArchitecture.spec.js --runInBand
```

Expected:

- FAIL because legacy files/selectors still exist
- FAIL because `ThemeToggle.vue` still references an outdated `.dark` selector

- [ ] **Step 3: Consolidate and delete legacy paths**

Implement the following:

1. Move any remaining `draw`-specific dark overrides out of `dark-mode-global.scss` into the scoped draw entrypoints.
2. Merge or delete duplicate `Element UI` dark rules so there is one authoritative source per control category.
3. Update `ThemeToggle.vue` to follow the real theme contract (`data-theme` / `theme-dark`) instead of dead `.dark` selectors.
4. Delete `src/assets/styles/draw/index.scss` only after `rg` confirms no remaining references.

Confirmation command before deletion:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
rg -n "draw/index.scss" src tests
```

Expected:

- no results

- [ ] **Step 4: Run the full style-guard suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/*.spec.js --runInBand
```

Expected:

- PASS for all style/source guard tests

- [ ] **Step 5: Run targeted lint and a production-style smoke build command**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/main.js src/views/goto/components/ThemeToggle.vue src/views/goto/plot/type/draw.vue src/views/goto/toolsBox/type/tool.vue
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service build --mode staging --dest dist-draw-theme-smoke
```

Expected:

- ESLint passes on touched JS/Vue files
- Vue build completes without missing-style-import errors

- [ ] **Step 6: Commit the final cleanup**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/src/assets/styles/dark-mode-global.scss \
  goto-web/src/assets/styles/element-ui-dark.scss \
  goto-web/src/views/goto/components/ThemeToggle.vue \
  goto-web/src/assets/styles/draw \
  goto-web/tests/unit/styles
git commit -m "refactor: unify draw dark theme pipeline"
```

## Final Verification Checklist

- [ ] Run `cd /mnt/f/IdeaProjects/goto-software/goto-web && npx jest tests/unit/styles/*.spec.js --runInBand`
- [ ] Run `cd /mnt/f/IdeaProjects/goto-software/goto-web && npx eslint src/main.js src/views/goto/components/ThemeToggle.vue src/views/goto/plot/type/draw.vue src/views/goto/toolsBox/type/tool.vue`
- [ ] Run `cd /mnt/f/IdeaProjects/goto-software/goto-web && NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service build --mode staging --dest dist-draw-theme-smoke`
- [ ] Manually smoke-check:
  - plot draw page root uses `.draw-page.draw-theme-scope`
  - toolbox page root uses `.draw-page.draw-theme-scope`
  - statistics operate page uses `.draw-theme-scope`
  - no page still depends on `draw/index.scss`
  - dark / light theme switching changes `draw` visuals without local `prefers-color-scheme`

## Handoff Notes

- Implement the tasks in order. Task 3 depends on Task 2’s `foundation.scss`. Task 4 depends on both Task 2 and Task 3. Task 5 is last because deleting `draw/index.scss` too early will strand consumers.
- If a task reveals more consumers of `draw` foundation than listed here, update Task 4’s file list before coding further. Do not silently expand scope without recording the new entrypoints in this plan.
- If `NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service build ...` fails for an environment-specific reason unrelated to this refactor, record the exact failure and continue only after the Jest + lint suites are green.
