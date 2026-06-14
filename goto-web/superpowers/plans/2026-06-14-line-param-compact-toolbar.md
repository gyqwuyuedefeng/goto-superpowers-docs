# LineParam Compact Toolbar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `lineParam.vue` from a vertical labeled list into a compact icon-first toolbar for color, line type, line width, and length.

**Architecture:** Keep the existing component boundary and data flow. Update only `lineParam.vue` plus a focused static regression test in `tests/unit/styles`, using scoped CSS to adapt the existing child controls into compact toolbar items.

**Tech Stack:** Vue 2 single-file components, Element UI `el-tooltip`, Jest static source tests, SCSS scoped styles.

---

## File Structure

- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
  - Owns the compact toolbar markup and scoped CSS for line parameter controls.
  - Continues to reuse `SettingForm`, `ColorPicker`, `GSelectDictImage`, and `GInput`.
- Create: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`
  - Static regression contract for compact classes, tooltip labels, minimum widths, wrapping, and exclusion of `lineEnd`.

## Task 1: Add Failing Compact Toolbar Regression Test

**Files:**
- Create: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`

- [ ] **Step 1: Create the regression test**

Create `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js` with this content:

```javascript
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

  test('keeps formLabel separate and removes visible per-field text labels', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<div class="form-label">\s*\{\{ formLabel \}\}\s*<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">颜色<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">线型<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">宽度<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">长度<\/div>/)
  })

  test('uses wrap-friendly minimum widths for compact controls', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*display:\s*flex\b/)
    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*flex-wrap:\s*wrap\b/)
    expect(source).toMatch(/\.line-param-field\s*\{[\s\S]*min-width:\s*var\(--line-param-field-min-width\)/)
    expect(source).toMatch(/\.line-param-field--color\s*\{[\s\S]*--line-param-field-min-width:\s*36px/)
    expect(source).toMatch(/\.line-param-field--type\s*\{[\s\S]*--line-param-field-min-width:\s*56px/)
    expect(source).toMatch(/\.line-param-field--number\s*\{[\s\S]*--line-param-field-min-width:\s*92px/)
    expect(source).toMatch(/\.line-param-number-icon\s*\{[\s\S]*width:\s*16px/)
  })
})
```

- [ ] **Step 2: Run the new test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: FAIL because the current `lineParam.vue` does not contain `line-param-toolbar`, tooltip wrappers, or compact field classes.

## Task 2: Implement Compact `lineParam.vue` Markup and Scoped Styles

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`

- [ ] **Step 1: Replace the template with compact tooltip-backed controls**

In `goto-web/src/views/goto/components/form/draw/lineParam.vue`, replace the full `<template>` block with:

```vue
<template>
  <div class="form-group line-param-box">
    <div class="form-label">
      {{ formLabel }}
    </div>
    <div class="form-content line-param-toolbar">
      <el-tooltip content="颜色" placement="top">
        <div class="line-param-field line-param-field--color">
          <SettingForm :container="setting.color">
            <ColorPicker
              :parent="setting.color"
              :field-name="'value'"
              @modify="handleModify"
            />
          </SettingForm>
        </div>
      </el-tooltip>

      <el-tooltip content="线型" placement="top">
        <div class="line-param-field line-param-field--type">
          <SettingForm :container="setting.lineType">
            <GSelectDictImage
              :form-label-show="false"
              :parent="setting.lineType.value"
              :field-name="'value'"
              :value-key="'id'"
              :option-name="'label'"
              :option-key="'id'"
              :list="setting.lineType.value.selectList"
              :image-url-part="setting.lineType.value.dictName"
              :suffix="'svg'"
              @modify="handleModify"
            />
          </SettingForm>
        </div>
      </el-tooltip>

      <el-tooltip content="宽度" placement="top">
        <div class="line-param-field line-param-field--number line-param-field--width">
          <span class="line-param-number-icon" data-icon="line-width" aria-hidden="true">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <line x1="4" y1="7" x2="20" y2="7" />
              <line x1="4" y1="12" x2="20" y2="12" />
              <line x1="4" y1="17" x2="20" y2="17" />
            </svg>
          </span>
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
      </el-tooltip>

      <el-tooltip content="长度" placement="top">
        <div class="line-param-field line-param-field--number line-param-field--length">
          <span class="line-param-number-icon" data-icon="line-length" aria-hidden="true">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <line x1="5" y1="12" x2="19" y2="12" />
              <polyline points="8 9 5 12 8 15" />
              <polyline points="16 9 19 12 16 15" />
            </svg>
          </span>
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
      </el-tooltip>
    </div>
  </div>
</template>
```

- [ ] **Step 2: Replace the scoped style block**

In the same file, replace the full `<style scoped lang="scss">` block with:

```scss
<style scoped lang="scss">
.line-param-box {
  padding: 10px;
  border: 1px solid var(--border-color);
  border-radius: var(--border-radius);
  background-color: var(--card-bg);
  transition: border-color 0.2s ease, background-color 0.2s ease, box-shadow 0.2s ease;

  [data-theme="dark"] & {
    box-shadow: 0 0 0 1px rgba(255, 255, 255, 0.02);
  }
}

.line-param-toolbar {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  row-gap: 8px;
  width: 100%;
  min-width: 0;
}

.line-param-field {
  --line-param-field-min-width: 56px;

  display: flex;
  align-items: center;
  min-width: var(--line-param-field-min-width);
  min-height: 32px;
  padding: 3px 6px;
  border: 1px solid var(--draw-border-normal, var(--border-color));
  border-radius: 4px;
  background: var(--draw-bg-primary, var(--card-bg));
  box-sizing: border-box;
  transition: border-color 0.2s ease, background-color 0.2s ease;

  &:hover {
    border-color: var(--primary-color);
    background: var(--primary-color-light);
  }

  > .container {
    width: 100%;
    min-width: 0;
  }
}

.line-param-field--color {
  --line-param-field-min-width: 36px;

  width: 36px;
  justify-content: center;
  padding: 3px;
}

.line-param-field--type {
  --line-param-field-min-width: 56px;

  width: 56px;
  justify-content: center;
}

.line-param-field--number {
  --line-param-field-min-width: 92px;

  flex: 0 1 104px;
  gap: 4px;
}

.line-param-number-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 16px;
  width: 16px;
  height: 16px;
  color: var(--draw-text-secondary, var(--text-secondary));
  line-height: 0;

  svg {
    display: block;
    width: 100%;
    height: 100%;
  }
}

::v-deep .line-param-field {
  .form-group {
    margin: 0;
    width: 100%;
    min-width: 0;
  }

  .form-content {
    width: 100%;
    min-width: 0;
  }

  .el-input-number {
    width: 100%;
  }

  .el-input__inner {
    height: 24px;
    line-height: 24px;
    padding: 0 4px;
    border: none;
    background: transparent;
    text-align: center;
    font-size: 13px;
    font-weight: 600;
    color: var(--draw-text-primary, var(--text-main));
  }

  .el-input-number__decrease,
  .el-input-number__increase {
    display: none;
  }

  .select-image-option {
    height: 22px;
  }

  .select-image-option img {
    height: 22px;
    max-width: 42px;
  }
}
</style>
```

- [ ] **Step 3: Run the focused test and confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: PASS.

## Task 3: Verify and Commit the UI Change

**Files:**
- Verify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
- Verify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`

- [ ] **Step 1: Run the focused style test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: PASS.

- [ ] **Step 2: Run ESLint for the changed files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/views/goto/components/form/draw/lineParam.vue tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: no ESLint errors.

- [ ] **Step 3: Manually inspect the changed component**

Check these items in the draw UI:

- `formLabel` appears on its own line.
- The controls for `颜色`, `线型`, `宽度`, and `长度` prefer one row.
- Each compact control shows the correct tooltip title on hover.
- The row wraps by control block when the panel is too narrow.
- Changing each value still triggers the existing downstream update behavior.
- `lineEnd` is not rendered.

- [ ] **Step 4: Commit the implementation**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
git status --short
git add src/views/goto/components/form/draw/lineParam.vue tests/unit/styles/lineParamCompactToolbar.spec.js
git commit -m "Improve line param compact layout"
```

Expected: a commit containing only the `lineParam.vue` implementation and `lineParamCompactToolbar.spec.js` test.
