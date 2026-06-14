# LineParam Compact Toolbar Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the completed `lineParam.vue` compact toolbar so color renders as a square swatch and all four visible controls fit in one row at the current default panel width.

**Architecture:** Keep the existing outer `SettingForm` visibility gate for each compact field. Remove redundant inner wrappers from numeric controls and add strictly scoped CSS overrides in `lineParam.vue` to prevent global `.container .el-input` width rules from expanding this toolbar.

**Tech Stack:** Vue 2 single-file components, Element UI `el-tooltip`, globally registered `ColorPicker`, local `GInput` and `GSelectDictImage`, Jest static source tests, scoped SCSS.

---

## File Structure

- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
  - Fixes the actual DOM structure and local CSS sizing.
- Modify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`
  - Adds regression coverage for color sizing chain, compact numeric sizing, no nested numeric `SettingForm`, and local input width overrides.

## Task 1: Update Regression Test For The Follow-Up Fix

**Files:**
- Modify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`

- [ ] **Step 1: Replace the full test file**

Replace the full contents of `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js` with:

```javascript
/* eslint-env jest */
const fs = require('fs')
const path = require('path')

function readSource(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

describe('line param compact toolbar contract', () => {
  test('renders only the active line fields as tooltip-backed compact controls', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/class="form-content line-param-toolbar"/)
    expect(source).toMatch(/<el-tooltip[^>]*content="颜色"[^>]*>/)
    expect(source).toMatch(/<el-tooltip[^>]*content="线型"[^>]*>/)
    expect(source).toMatch(/<el-tooltip[^>]*content="宽度"[^>]*>/)
    expect(source).toMatch(/<el-tooltip[^>]*content="长度"[^>]*>/)
    expect(source).toMatch(/class="line-param-field line-param-field--color"/)
    expect(source).toMatch(/class="line-param-field line-param-field--type"/)
    expect(source).toMatch(/class="line-param-field line-param-field--number line-param-field--width"/)
    expect(source).toMatch(/class="line-param-field line-param-field--number line-param-field--length"/)
    expect(source).not.toMatch(/setting\.lineEnd/)
    expect(source).not.toMatch(/lineEnd/)
  })

  test('uses SettingForm only as the outer visibility gate for each compact field', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--color")(?=[^>]*:container="setting\.color")[^>]*>\s*<el-tooltip[^>]*content="颜色"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--type")(?=[^>]*:container="setting\.lineType")[^>]*>\s*<el-tooltip[^>]*content="线型"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--width")(?=[^>]*:container="setting\.lineWidth")[^>]*>\s*<el-tooltip[^>]*content="宽度"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--length")(?=[^>]*:container="setting\.length")[^>]*>\s*<el-tooltip[^>]*content="长度"/)

    const widthBlock = source.match(/<!-- 宽度 -->[\s\S]*?<\/SettingForm>\s*<!-- 长度 -->/)
    const lengthBlock = source.match(/<!-- 长度 -->[\s\S]*?<\/SettingForm>\s*<\/div>/)

    expect(widthBlock && widthBlock[0]).toBeTruthy()
    expect(lengthBlock && lengthBlock[0]).toBeTruthy()
    expect(widthBlock[0]).not.toMatch(/<SettingForm :container="setting\.lineWidth"/)
    expect(lengthBlock[0]).not.toMatch(/<SettingForm :container="setting\.length"/)
  })

  test('keeps formLabel separate and removes visible per-field text labels', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<div class="form-label">\s*\{\{ formLabel \}\}\s*<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">颜色<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">线型<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">宽度<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">长度<\/div>/)
  })

  test('uses W and L text badges instead of SVG icons for width and length', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<span class="line-param-badge">W<\/span>/)
    expect(source).toMatch(/<span class="line-param-badge">L<\/span>/)
    expect(source).not.toMatch(/line-param-number-icon/)
    expect(source).not.toMatch(/<svg/)
    expect(source).not.toMatch(/stroke="currentColor"/)
  })

  test('uses compact one-row sizing targets', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/--line-param-control-height:\s*36px/)
    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*display:\s*flex\b/)
    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*flex-wrap:\s*wrap\b/)
    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*gap:\s*6px/)
    expect(source).toMatch(/\.line-param-field\s*\{[\s\S]*height:\s*var\(--line-param-control-height\)/)
    expect(source).toMatch(/\.line-param-field--color\s*\{[\s\S]*width:\s*var\(--line-param-control-height\)/)
    expect(source).toMatch(/\.line-param-field--type\s*\{[\s\S]*width:\s*70px/)
    expect(source).toMatch(/\.line-param-field--number\s*\{[\s\S]*width:\s*90px/)
    expect(source).not.toMatch(/\.line-param-field--number\s*\{[\s\S]*width:\s*104px/)
  })

  test('forces color picker root and reference chain to fill the square', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/::v-deep \.line-param-field--color\s*\{[\s\S]*>\s*span\s*\{[\s\S]*width:\s*100%/)
    expect(source).toMatch(/::v-deep \.line-param-field--color\s*\{[\s\S]*>\s*span\s*\{[\s\S]*height:\s*100%/)
    expect(source).toMatch(/\.el-popover__reference-wrapper\s*\{[\s\S]*width:\s*100%/)
    expect(source).toMatch(/\.el-popover__reference-wrapper\s*\{[\s\S]*height:\s*100%/)
    expect(source).toMatch(/\.color-reference\s*\{[\s\S]*width:\s*100%\s*!important/)
    expect(source).toMatch(/\.color-reference\s*\{[\s\S]*height:\s*100%\s*!important/)
  })

  test('overrides inherited Element input widths inside numeric controls only', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/::v-deep \.line-param-field--number\s*\{[\s\S]*\.el-input\s*\{[\s\S]*width:\s*100%\s*!important/)
    expect(source).toMatch(/::v-deep \.line-param-field--number\s*\{[\s\S]*\.el-input-number\s*\{[\s\S]*width:\s*100%\s*!important/)
    expect(source).toMatch(/::v-deep \.line-param-field--number\s*\{[\s\S]*\.el-input__inner\s*\{[\s\S]*width:\s*100%\s*!important/)
  })

  test('keeps approved line type preview sizing', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/class="line-param-type-preview"/)
    expect(source).toMatch(/\.line-param-field--type\s*\{[\s\S]*padding:\s*0\s+5px/)
    expect(source).toMatch(/\.line-param-type-preview\s*\{[\s\S]*height:\s*20px/)
    expect(source).toMatch(/\.line-param-type-preview\s*\{[\s\S]*border-radius:\s*4px/)
  })
})
```

- [ ] **Step 2: Run the updated test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: FAIL because the current implementation still has nested numeric `SettingForm`, `gap: 8px`, numeric width `104px`, and incomplete color/input width overrides.

## Task 2: Fix The Toolbar Template Structure

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`

- [ ] **Step 1: Remove nested numeric `SettingForm` wrappers**

In `goto-web/src/views/goto/components/form/draw/lineParam.vue`, replace the width block:

```vue
<div class="line-param-field line-param-field--number line-param-field--width">
  <span class="line-param-badge">W</span>
  <SettingForm :container="setting.lineWidth">
    <GInput
      :parent="setting.lineWidth"
      :field-name="'value'"
      :type="'number'"
      :min="setting.lineWidthMin.value"
      :max="setting.lineWidthMax.value"
      :name-disabled="true"
      @modify="handleModify"
    />
  </SettingForm>
</div>
```

with:

```vue
<div class="line-param-field line-param-field--number line-param-field--width">
  <span class="line-param-badge">W</span>
  <GInput
    :parent="setting.lineWidth"
    :field-name="'value'"
    :type="'number'"
    :min="setting.lineWidthMin.value"
    :max="setting.lineWidthMax.value"
    :name-disabled="true"
    @modify="handleModify"
  />
</div>
```

- [ ] **Step 2: Remove the nested length `SettingForm` wrapper**

In the same file, replace the length block:

```vue
<div class="line-param-field line-param-field--number line-param-field--length">
  <span class="line-param-badge">L</span>
  <SettingForm :container="setting.length">
    <GInput
      :parent="setting.length"
      :field-name="'value'"
      :type="'number'"
      :min="setting.lengthMin.value"
      :max="setting.lengthMax.value"
      :name-disabled="true"
      @modify="handleModify"
    />
  </SettingForm>
</div>
```

with:

```vue
<div class="line-param-field line-param-field--number line-param-field--length">
  <span class="line-param-badge">L</span>
  <GInput
    :parent="setting.length"
    :field-name="'value'"
    :type="'number'"
    :min="setting.lengthMin.value"
    :max="setting.lengthMax.value"
    :name-disabled="true"
    @modify="handleModify"
  />
</div>
```

- [ ] **Step 3: Confirm outer visibility wrappers remain**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
rg -n "line-param-setting--(color|type|width|length)|<SettingForm :container=\"setting\\.(lineWidth|length)\"" src/views/goto/components/form/draw/lineParam.vue
```

Expected:

```text
... line-param-setting--color ...
... line-param-setting--type ...
... line-param-setting--width ...
... line-param-setting--length ...
```

There should be no matches for `<SettingForm :container="setting.lineWidth"` or `<SettingForm :container="setting.length"` inside the numeric fields.

## Task 3: Add Local CSS Overrides For Real DOM Widths

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`

- [ ] **Step 1: Tighten toolbar gap and numeric width**

In the `<style scoped lang="scss">` block, change `.line-param-toolbar`:

```scss
  gap: 8px;
  row-gap: 8px;
```

to:

```scss
  gap: 6px;
  row-gap: 6px;
```

Change `.line-param-field--number`:

```scss
  width: 104px;
```

to:

```scss
  width: 90px;
```

- [ ] **Step 2: Make the color picker root span fill the square**

In the existing `::v-deep .line-param-field--color` block, replace it with:

```scss
::v-deep .line-param-field--color {
  > span {
    display: block;
    width: 100%;
    height: 100%;
  }

  .el-popover__reference-wrapper {
    display: block;
    width: 100%;
    min-width: 0;
    height: 100%;
  }

  .color-reference {
    width: 100% !important;
    height: 100% !important;
    min-width: 0;
    border: 2px solid #fff !important;
    border-radius: 8px;
  }
}
```

- [ ] **Step 3: Add numeric input width overrides**

After the existing `::v-deep .line-param-field` block and before `::v-deep .line-param-field--color`, add:

```scss
::v-deep .line-param-field--number {
  .form-group,
  .form-content,
  .draw-field-group__content,
  .draw-field-group__item {
    width: 100% !important;
    min-width: 0;
  }

  .el-input,
  .el-input-number {
    width: 100% !important;
    min-width: 0;
  }

  .el-input__inner {
    width: 100% !important;
  }
}
```

- [ ] **Step 4: Run the focused test and confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: PASS.

## Task 4: Verify And Commit

**Files:**
- Verify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
- Verify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`

- [ ] **Step 1: Run focused test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: PASS.

- [ ] **Step 2: Run ESLint for changed files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/views/goto/components/form/draw/lineParam.vue tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: no ESLint errors.

- [ ] **Step 3: Manually inspect the draw UI**

Use the same default panel width as the screenshot and check:

- `刻度线` renders color, line type, width, and length in one row.
- Color renders as a square swatch, not a dot.
- `轴线` still hides the whole length control when length is invisible.
- Width and length still show `W` and `L`.
- Width and length values still update normally.
- `lineEnd` is still not rendered.

- [ ] **Step 4: Commit the implementation**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
git status --short
git add src/views/goto/components/form/draw/lineParam.vue tests/unit/styles/lineParamCompactToolbar.spec.js
git commit -m "Fix line param compact toolbar sizing"
```

Expected: a commit containing only `lineParam.vue` and `lineParamCompactToolbar.spec.js`.
