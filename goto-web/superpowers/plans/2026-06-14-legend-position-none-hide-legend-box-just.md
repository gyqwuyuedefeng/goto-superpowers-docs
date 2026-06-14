# Legend Position None Hide Legend Box Just Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Hide the runtime "图例设置" alignment parameter "对齐方式" when "图例位置" is set to "不显示".

**Architecture:** The runtime association flow already routes `legendSettings` changes into `legendSettingsParamModify`. Extend that existing data-layer visibility update by toggling `legendBoxJust.visible` beside the other legend controls. Keep runtime rendering unchanged because the draw form already reads visibility from `setting.legendBoxJust`.

**Tech Stack:** Vue 2, Vue CLI 3, Jest unit tests, existing `goto-web/src/constant/modify/commonParamModify` association helpers.

---

## File Structure

- Modify: `goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js`
  - Responsibility: central runtime association logic for `legendSettings` field visibility and derived direction values.
- Create: `goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js`
  - Responsibility: focused Jest coverage for `legendPosition` changing to `none` and back to a visible position.

No component files should be modified. In particular, do not change `goto-web/src/views/goto/components/form/plotSetting/legendSettingsParam.vue`, because the requested scope excludes the plot setting/configuration editor.

## Task 1: Add Focused Unit Coverage

**Files:**
- Create: `goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js`
- Modify: none
- Test: `goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js`

- [ ] **Step 1: Create the failing test file**

Add this complete file:

```js
/* eslint-env jest */

const { legendSettingsParamModify } = require('@/constant/modify/commonParamModify/legendSettingsParamModify')

function createLegendSettings(positionValue) {
  return {
    paramName: 'legendSettings',
    legendPosition: {
      value: {
        value: positionValue
      }
    },
    legendInsideCoord: {
      visible: true
    },
    legendDirection: {
      visible: true,
      value: {
        value: 'horizontal'
      }
    },
    legendBoxJust: {
      visible: true,
      value: {
        value: 'left'
      }
    },
    legendTitle: {
      visible: true
    },
    legendText: {
      visible: true
    },
    legendKeyWidth: {
      visible: true
    },
    legendKeyHeight: {
      visible: true
    },
    legendBarWidth: {
      visible: true
    },
    legendBarHeight: {
      visible: true
    }
  }
}

function createCommonParams(setting) {
  return {
    setting,
    params: {
      optionName: 'legendPosition'
    }
  }
}

describe('legendSettingsParamModify', () => {
  let consoleLogSpy

  beforeEach(() => {
    consoleLogSpy = jest.spyOn(console, 'log').mockImplementation(() => {})
  })

  afterEach(() => {
    consoleLogSpy.mockRestore()
  })

  test('图例位置为不显示时隐藏对齐方式', () => {
    const setting = createLegendSettings('none')

    legendSettingsParamModify(createCommonParams(setting))

    expect(setting.legendInsideCoord.visible).toBe(false)
    expect(setting.legendDirection.visible).toBe(false)
    expect(setting.legendBoxJust.visible).toBe(false)
    expect(setting.legendTitle.visible).toBe(false)
    expect(setting.legendText.visible).toBe(false)
    expect(setting.legendKeyWidth.visible).toBe(false)
    expect(setting.legendKeyHeight.visible).toBe(false)
    expect(setting.legendBarWidth.visible).toBe(false)
    expect(setting.legendBarHeight.visible).toBe(false)
  })

  test('图例位置恢复为可见位置时恢复对齐方式', () => {
    const setting = createLegendSettings('left')
    setting.legendInsideCoord.visible = false
    setting.legendDirection.visible = false
    setting.legendBoxJust.visible = false
    setting.legendTitle.visible = false
    setting.legendText.visible = false
    setting.legendKeyWidth.visible = false
    setting.legendKeyHeight.visible = false
    setting.legendBarWidth.visible = false
    setting.legendBarHeight.visible = false

    legendSettingsParamModify(createCommonParams(setting))

    expect(setting.legendInsideCoord.visible).toBe(false)
    expect(setting.legendDirection.visible).toBe(true)
    expect(setting.legendDirection.value.value).toBe('vertical')
    expect(setting.legendBoxJust.visible).toBe(true)
    expect(setting.legendTitle.visible).toBe(true)
    expect(setting.legendText.visible).toBe(true)
    expect(setting.legendKeyWidth.visible).toBe(true)
    expect(setting.legendKeyHeight.visible).toBe(true)
    expect(setting.legendBarWidth.visible).toBe(true)
    expect(setting.legendBarHeight.visible).toBe(true)
  })
})
```

- [ ] **Step 2: Run the focused test and verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

Expected result before implementation:

```text
FAIL tests/unit/constant/modify/legendSettingsParamModify.spec.js
  legendSettingsParamModify
    ✕ 图例位置为不显示时隐藏对齐方式
```

The meaningful failure should be that `setting.legendBoxJust.visible` remains `true` when the test expects `false`.

## Task 2: Toggle `legendBoxJust.visible` In Runtime Association Logic

**Files:**
- Modify: `goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js`
- Test: `goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js`

- [ ] **Step 1: Update the `none` branch**

In `goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js`, add one line in the `legendPosition.value.value === 'none'` branch:

```js
            commonParams.setting.legendBoxJust.visible = false
```

The hidden branch should become:

```js
        if (commonParams.setting.legendPosition.value.value === 'none') {
            commonParams.setting.legendInsideCoord.visible = false
            commonParams.setting.legendDirection.visible = false
            commonParams.setting.legendBoxJust.visible = false
            commonParams.setting.legendTitle.visible = false
            commonParams.setting.legendText.visible = false
            commonParams.setting.legendKeyWidth.visible = false
            commonParams.setting.legendKeyHeight.visible = false
            commonParams.setting.legendBarWidth.visible = false
            commonParams.setting.legendBarHeight.visible = false
        } else {
```

- [ ] **Step 2: Update the visible-position branch**

In the `else` branch, add one line to restore the alignment setting:

```js
            commonParams.setting.legendBoxJust.visible = true
```

The restored branch should begin:

```js
        } else {
            commonParams.setting.legendInsideCoord.visible = true
            commonParams.setting.legendDirection.visible = true
            commonParams.setting.legendBoxJust.visible = true
            commonParams.setting.legendTitle.visible = true
            commonParams.setting.legendText.visible = true
            commonParams.setting.legendKeyWidth.visible = true
            commonParams.setting.legendKeyHeight.visible = true
            commonParams.setting.legendBarWidth.visible = true
            commonParams.setting.legendBarHeight.visible = true
            if (commonParams.setting.legendPosition.value.value === 'inside') {
```

- [ ] **Step 3: Run the focused test and verify it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

Expected result:

```text
PASS tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

- [ ] **Step 4: Run related association tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/associationParamModify.spec.js tests/unit/constant/modify/publicFunction.spec.js tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

Expected result:

```text
PASS tests/unit/constant/associationParamModify.spec.js
PASS tests/unit/constant/modify/publicFunction.spec.js
PASS tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

- [ ] **Step 5: Run lint on touched source and test files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/constant/modify/commonParamModify/legendSettingsParamModify.js tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

Expected result: command exits with status 0 and prints no lint errors.

- [ ] **Step 6: Review the source diff**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git diff -- goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js
```

Expected source diff:

```diff
+            commonParams.setting.legendBoxJust.visible = false
+            commonParams.setting.legendBoxJust.visible = true
```

The test diff should contain only `goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js`.

- [ ] **Step 7: Commit implementation**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/constant/modify/commonParamModify/legendSettingsParamModify.js goto-web/tests/unit/constant/modify/legendSettingsParamModify.spec.js
git commit -m "fix: hide legend alignment when legend is disabled"
```

Expected result:

```text
[<branch> <commit>] fix: hide legend alignment when legend is disabled
```

Only the source file and the focused test file should be included in this commit.

## Manual Verification

- [ ] Open a plot in the runtime drawing panel with more than one enabled legend.
- [ ] Open "图例设置".
- [ ] Confirm "对齐方式" is visible when "图例位置" is "左侧", "右侧", "顶部", "底部", or "内部".
- [ ] Change "图例位置" to "不显示".
- [ ] Confirm "对齐方式" hides together with the other legend-specific controls.
- [ ] Change "图例位置" back to a visible position.
- [ ] Confirm "对齐方式" appears again when more than one legend is enabled.
- [ ] Open the plot setting/configuration editor and confirm its "图例设置" configuration UI was not changed by this implementation.

## Self-Review Notes

- Spec coverage: the plan covers runtime-only behavior, hide on `none`, restore on visible positions, preserving direction auto-update, and not changing the plot setting editor.
- Placeholder scan: this plan contains concrete files, code snippets, commands, and expected outcomes.
- Type consistency: the plan uses the existing property names `legendPosition`, `legendBoxJust`, `visible`, and `value.value` from the current runtime data structure.
