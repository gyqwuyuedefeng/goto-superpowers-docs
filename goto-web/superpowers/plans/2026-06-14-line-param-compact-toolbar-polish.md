# LineParam Compact Toolbar Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish the existing `lineParam.vue` compact toolbar so hidden fields remove their whole shell, width/length use `W`/`L`, color is square, and all four controls share the approved visual height.

**Architecture:** Keep the existing `GLineParam` component and child control APIs. Move `SettingForm` outside each tooltip-backed compact field so existing visibility logic controls the full field wrapper, then use scoped CSS to size only this component's color, line type, and numeric controls.

**Tech Stack:** Vue 2 single-file components, Element UI `el-tooltip`, Jest static source tests, scoped SCSS.

---

## File Structure

- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`
  - Owns the template wrapper order and scoped visual rules for compact fields.
- Modify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`
  - Static regression contract for wrapper order, no empty shells, `W`/`L` badges, square color control, equal height, and approved line preview sizing.

## Task 1: Update Regression Test For Polish Requirements

**Files:**
- Modify: `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js`

- [ ] **Step 1: Replace the static regression test**

Replace the full contents of `goto-web/tests/unit/styles/lineParamCompactToolbar.spec.js` with:

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

  test('uses SettingForm as the outer visibility gate for each compact field', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--color")(?=[^>]*:container="setting\.color")[^>]*>\s*<el-tooltip[^>]*content="颜色"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--type")(?=[^>]*:container="setting\.lineType")[^>]*>\s*<el-tooltip[^>]*content="线型"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--width")(?=[^>]*:container="setting\.lineWidth")[^>]*>\s*<el-tooltip[^>]*content="宽度"/)
    expect(source).toMatch(/<SettingForm(?=[^>]*class="line-param-setting line-param-setting--length")(?=[^>]*:container="setting\.length")[^>]*>\s*<el-tooltip[^>]*content="长度"/)

    expect(source).not.toMatch(/<el-tooltip[^>]*content="颜色"[\s\S]*?<div class="line-param-field line-param-field--color"[\s\S]*?<SettingForm :container="setting\.color"/)
    expect(source).not.toMatch(/<el-tooltip[^>]*content="线型"[\s\S]*?<div class="line-param-field line-param-field--type"[\s\S]*?<SettingForm :container="setting\.lineType"/)
    expect(source).not.toMatch(/<el-tooltip[^>]*content="宽度"[\s\S]*?<div class="line-param-field line-param-field--number line-param-field--width"[\s\S]*?<SettingForm :container="setting\.lineWidth"/)
    expect(source).not.toMatch(/<el-tooltip[^>]*content="长度"[\s\S]*?<div class="line-param-field line-param-field--number line-param-field--length"[\s\S]*?<SettingForm :container="setting\.length"/)
  })

  test('keeps formLabel separate and removes visible per-field text labels', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/<div class="form-label">\s*\{\{ formLabel \}\}\s*<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">颜色<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">线型<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">宽度<\/div>/)
    expect(source).not.toMatch(/<div class="form-label">长度<\/div>/)
  })

  test('uses W and L text badges instead of duplicated arrow svg icons', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/class="line-param-badge line-param-badge--width"[^>]*>\s*W\s*<\/span>/)
    expect(source).toMatch(/class="line-param-badge line-param-badge--length"[^>]*>\s*L\s*<\/span>/)
    expect(source).not.toMatch(/class="line-param-number-icon"/)
    expect(source).not.toMatch(/data-icon="line-width"/)
    expect(source).not.toMatch(/data-icon="line-length"/)
  })

  test('uses wrap-friendly equal-height sizing for compact controls', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*display:\s*flex\b/)
    expect(source).toMatch(/\.line-param-toolbar\s*\{[\s\S]*flex-wrap:\s*wrap\b/)
    expect(source).toMatch(/--line-param-control-height:\s*36px/)
    expect(source).toMatch(/\.line-param-field\s*\{[\s\S]*height:\s*var\(--line-param-control-height\)/)
    expect(source).toMatch(/\.line-param-field--color\s*\{[\s\S]*width:\s*var\(--line-param-control-height\)/)
    expect(source).toMatch(/\.line-param-field--type\s*\{[\s\S]*width:\s*70px/)
    expect(source).toMatch(/\.line-param-field--type\s*\{[\s\S]*padding:\s*0 5px/)
    expect(source).toMatch(/\.line-param-field--number\s*\{[\s\S]*width:\s*104px/)
  })

  test('uses approved line type preview sizing', () => {
    const source = readSource('src/views/goto/components/form/draw/lineParam.vue')

    expect(source).toMatch(/class="line-param-type-preview"/)
    expect(source).toMatch(/\.line-param-type-preview\s*\{[\s\S]*height:\s*20px/)
    expect(source).toMatch(/\.line-param-type-preview\s*\{[\s\S]*border-radius:\s*4px/)
    expect(source).toMatch(/\.line-param-type-preview\s*\{[\s\S]*overflow:\s*hidden/)
  })
})
```

- [ ] **Step 2: Run the updated test and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npm run test:unit -- tests/unit/styles/lineParamCompactToolbar.spec.js
```

Expected: FAIL because the current implementation still has `SettingForm` inside `.line-param-field`, still uses SVG icons for width/length, and does not define the approved line type preview sizing.

## Task 2: Update `lineParam.vue` Template Structure

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`

- [ ] **Step 1: Replace the template block**

Replace the full `<template>` block in `goto-web/src/views/goto/components/form/draw/lineParam.vue` with:

```vue
<template>
  <div class="form-group line-param-box">
    <div class="form-label">
      {{ formLabel }}
    </div>
    <div class="form-content line-param-toolbar">
      <!-- 颜色 -->
      <SettingForm
        class="line-param-setting line-param-setting--color"
        :container="setting.color"
      >
        <el-tooltip content="颜色" placement="top">
          <div class="line-param-field line-param-field--color">
            <ColorPicker
              :parent="setting.color"
              :field-name="'value'"
              @modify="handleModify"
            />
          </div>
        </el-tooltip>
      </SettingForm>

      <!-- 线型 -->
      <SettingForm
        class="line-param-setting line-param-setting--type"
        :container="setting.lineType"
      >
        <el-tooltip content="线型" placement="top">
          <div class="line-param-field line-param-field--type">
            <div class="line-param-type-preview">
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
            </div>
          </div>
        </el-tooltip>
      </SettingForm>

      <!-- 宽度 -->
      <SettingForm
        class="line-param-setting line-param-setting--width"
        :container="setting.lineWidth"
      >
        <el-tooltip content="宽度" placement="top">
          <div class="line-param-field line-param-field--number line-param-field--width">
            <span class="line-param-badge line-param-badge--width">W</span>
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
        </el-tooltip>
      </SettingForm>

      <!-- 长度 -->
      <SettingForm
        class="line-param-setting line-param-setting--length"
        :container="setting.length"
      >
        <el-tooltip content="长度" placement="top">
          <div class="line-param-field line-param-field--number line-param-field--length">
            <span class="line-param-badge line-param-badge--length">L</span>
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
        </el-tooltip>
      </SettingForm>
    </div>
  </div>
</template>
```

- [ ] **Step 2: Confirm `lineEnd` remains absent**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
rg -n "lineEnd|setting\\.lineEnd" src/views/goto/components/form/draw/lineParam.vue
```

Expected: no matches.

## Task 3: Update Scoped CSS To Match Approved Preview

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/lineParam.vue`

- [ ] **Step 1: Replace the style block**

Replace the full `<style scoped lang="scss">` block in `goto-web/src/views/goto/components/form/draw/lineParam.vue` with:

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
  --line-param-control-height: 36px;

  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  row-gap: 8px;
  width: 100%;
  min-width: 0;
}

.line-param-setting {
  flex: 0 0 auto;
  min-width: 0;
}

.line-param-field {
  display: flex;
  align-items: center;
  height: var(--line-param-control-height);
  border: 1px solid var(--draw-border-normal, var(--border-color));
  border-radius: 6px;
  background: var(--draw-bg-primary, var(--card-bg));
  box-sizing: border-box;
  overflow: hidden;
  transition: border-color 0.2s ease, background-color 0.2s ease;

  &:hover {
    border-color: var(--primary-color);
    background: var(--primary-color-light);
  }
}

.line-param-field--color {
  width: var(--line-param-control-height);
  justify-content: center;
  padding: 4px;
}

.line-param-field--type {
  width: 70px;
  justify-content: center;
  padding: 0 5px;
}

.line-param-type-preview {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 20px;
  border-radius: 4px;
  overflow: hidden;
}

.line-param-field--number {
  width: 104px;
  gap: 6px;
  padding: 4px;
}

.line-param-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 24px;
  width: 24px;
  height: 26px;
  border-radius: 5px;
  background: rgba(255, 255, 255, 0.03);
  color: var(--draw-text-secondary, var(--text-secondary));
  font-size: 13px;
  font-weight: 800;
  line-height: 1;
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
    height: 26px;
    line-height: 26px;
    padding: 0 4px;
    border: none;
    border-radius: 5px;
    background: var(--draw-bg-secondary, rgba(255, 255, 255, 0.04));
    text-align: center;
    font-size: 16px;
    font-weight: 800;
    color: var(--draw-text-primary, var(--text-main));
  }

  .el-input-number__decrease,
  .el-input-number__increase {
    display: none;
  }
}

::v-deep .line-param-field--color {
  .el-popover__reference-wrapper {
    display: block;
    width: 100%;
    min-width: 0;
    height: 100%;
  }

  .color-reference {
    width: 100%;
    height: 100%;
    border: 2px solid #fff !important;
    border-radius: 8px;
  }
}

::v-deep .line-param-type-preview {
  .form-group,
  .form-content,
  .image-show-container {
    width: 100%;
    height: 100%;
    min-width: 0;
  }

  .image-show-container {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .select-image-option {
    width: 100%;
    height: 20px;
  }

  .select-image-option img {
    width: 100%;
    height: 20px;
    object-fit: contain;
  }
}
</style>
```

- [ ] **Step 2: Run the focused test and confirm it passes**

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

- [ ] **Step 1: Run the focused unit test**

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

Check these conditions:

- `轴线` or any line parameter with `length.visible === false` does not show an empty length shell.
- `刻度线` or any line parameter with all four fields visible shows color, line type, width, and length in one row when space allows.
- Color is a square control.
- Line type control has little empty space between the white preview and the outer shell.
- Line type preview is taller than the first implementation and uses smaller radius, close to the approved HTML preview.
- Width shows `W`; length shows `L`.
- All four controls have the same visual height.
- Tooltips still show `颜色`, `线型`, `宽度`, and `长度`.

- [ ] **Step 4: Commit the implementation**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
git status --short
git add src/views/goto/components/form/draw/lineParam.vue tests/unit/styles/lineParamCompactToolbar.spec.js
git commit -m "Polish line param compact toolbar"
```

Expected: a commit containing only `lineParam.vue` and `lineParamCompactToolbar.spec.js`.
