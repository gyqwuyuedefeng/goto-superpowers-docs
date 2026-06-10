# goto-web Checkbox Style Unification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Unify `goto-web` checkbox visuals across `Element UI`, custom native checkboxes, and `Handsontable` using the approved `Soft Panel` direction, while preserving existing behavior and supporting light/dark themes.

**Architecture:** Build one shared checkbox theme layer in `src/assets/styles/checkbox.scss`, import it globally, then migrate each checkbox family to consume the same tokens instead of hardcoded colors. Keep two densities: default for panels/forms and compact for legend rows, floating action bars, and table cells. Finish by retuning `Handsontable`'s checkbox tokens and the custom select-all checkbox so the table follows the same family without losing density.

**Tech Stack:** Vue 2.6, Element UI 2.15, Handsontable 15, SCSS (`sass-loader@10`), Jest (`@vue/cli-plugin-unit-jest`), existing style guard tests in `goto-web/tests/unit/styles`

---

## Preconditions

- Approved spec: `docs/superpowers/specs/2026-03-25-goto-web-checkbox-style-unification-design.md`
- Implementation repo: `/mnt/f/IdeaProjects/goto-software/goto-web`
- The user explicitly requested isolated execution in the frontend project's local worktree directory. Do not implement in the current `goto-web` working tree.
- `goto-web` already has a valid ignored worktree directory: `goto-web/.worktrees/`
- The current `goto-web` working tree is dirty (`src/views/login.vue` modified). Treat that as a hard stop against in-place coding.
- `goto-web/package.json` uses Windows-style `set NODE_OPTIONS=...` scripts. In Bash, call `npx jest ...` directly and use `NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service ...` for build commands.

### Required worktree setup

Run these commands before touching code:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
git worktree add .worktrees/checkbox-soft-panel-20260325 -b checkbox-soft-panel-20260325
cd .worktrees/checkbox-soft-panel-20260325
npm install
git status --short
```

Expected:

- Worktree path exists: `/mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325`
- `git status --short` is empty inside the new worktree

Run the baseline tests before implementation:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/classifyLegendParamTheme.spec.js --runInBand
npx jest tests/unit/styles/drawDarkTheme.spec.js --runInBand
```

If baseline tests fail, stop and ask whether to proceed or investigate. Do not stack checkbox work on top of an already failing baseline.

## Scope Check

This plan covers one coherent subsystem: checkbox styling in `goto-web`, centered on `src/views/goto` plus the shared `Handsontable` component used by the frontend tools. It does not attempt to redesign radios, switches, or unrelated theme surfaces.

## Implementation Notes

- Start each coding task with `@test-driven-development`: write or extend the guard first, run it, then implement the minimal change.
- End each coding task with the targeted Jest command listed in that task before committing.
- Keep the style contract centralized in `src/assets/styles/checkbox.scss`. If a component only needs layout or density, keep that locally; if it needs color, border, checkmark, or dark-theme logic, move that into the shared layer.
- Use existing semantic theme variables instead of inventing a second theme system:
  - `--surface-elevated`
  - `--surface-muted`
  - `--border-default`
  - `--border-strong`
  - `--accent-solid`
  - `--text-on-accent`
  - `--focus-ring`
- Remove or shrink local `::v-deep` checkbox overrides once the shared layer covers them. Leaving old overrides in place defeats the whole rollout.
- Keep density split explicit:
  - default panel/form size: `18px`
  - compact form/list size: `16px`
  - table size: `15px` to `16px`
- Plain `el-checkbox` consumers with no local styling, such as `Step1Upload.vue`, `Step5Transform.vue`, and the recursive checkbox inside `SyncIcon.vue`, should ideally need no file changes. They still must be manually verified after the global layer lands.
- Every `git add` in the coding session must use exact file paths. Never use `git add .`.

## File Map

### New test to create

- `goto-web/tests/unit/styles/checkboxTheme.spec.js`
  - Guards the shared checkbox style contract, import wiring, compact density, dark-theme token usage, and `Handsontable` checkbox token mapping

### Global shared style files to create or modify

- `goto-web/src/assets/styles/checkbox.scss`
  - New shared checkbox theme layer for `Element UI`, custom native checkboxes, and shared density variants
- `goto-web/src/assets/styles/index.scss`
  - Import the new checkbox layer in the global style stack
- `goto-web/src/assets/styles/element-ui-dark.scss`
  - Remove or reduce scattered checkbox-only overrides so the shared checkbox file owns that contract
- `goto-web/src/assets/styles/handsontable/handsontable-custom.scss`
  - Map `Handsontable` checkbox variables and the custom select-all checkbox to the shared contract

### Custom native checkbox consumers to modify

- `goto-web/src/views/goto/components/form/draw/checkbox.vue`
  - Replace hardcoded size/color/checkmark rules with shared variables
- `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
  - Align its duplicated custom checkbox block with the shared contract
- `goto-web/src/views/goto/components/form/draw/list.vue`
  - Retune the floating mini checkbox to use the same border, fill, and checkmark language
- `goto-web/src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue`
  - Remove leftover deep overrides that fight the shared custom checkbox contract

### Element UI checkbox-heavy consumers to modify

- `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
  - Keep compact sizing and hidden label behavior, but remove local color ownership
- `goto-web/src/views/goto/components/form/plotSetting/classifyParam.vue`
  - Remove hardcoded checkbox card/label palette and let the shared layer drive visuals
- `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
  - Replace local checkbox palette values with theme tokens or shared defaults
- `goto-web/src/views/goto/components/form/plotSetting/facetParams.vue`
  - Same checkbox list/card cleanup as `MarkColumnSelector.vue`
- `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`
  - Keep bordered checkbox layout, but align checked/unselected card treatment with shared tokens

### Existing tests to modify

- `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js`
  - Guard that `compact-checkbox` only controls density/label visibility, not hardcoded checkbox colors
- `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
  - Optionally extend if the shared custom checkbox contract needs a compiled-SCSS assertion

### Verify-only files

- `goto-web/src/views/goto/components/SyncIcon.vue`
- `goto-web/src/views/goto/process/components/Step1Upload.vue`
- `goto-web/src/views/goto/process/components/Step5Transform.vue`
- `goto-web/src/views/components/handsontable.vue`

These files should only change if the shared layer cannot cover them cleanly.

## Task 1: Establish The Shared Checkbox Theme Contract

**Files:**
- Create: `goto-web/tests/unit/styles/checkboxTheme.spec.js`
- Create: `goto-web/src/assets/styles/checkbox.scss`
- Modify: `goto-web/src/assets/styles/index.scss`
- Modify: `goto-web/src/assets/styles/element-ui-dark.scss`

- [ ] **Step 1: Write the failing shared contract test**

Create `goto-web/tests/unit/styles/checkboxTheme.spec.js` with source and compile guards:

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
  const outputPath = path.join(os.tmpdir(), `checkbox-theme-${Date.now()}-${process.pid}.css`)

  execFileSync('npx', ['sass', inputPath, outputPath], {
    cwd: projectRoot,
    stdio: 'pipe'
  })

  const css = fs.readFileSync(outputPath, 'utf8')
  fs.unlinkSync(outputPath)
  return css
}

describe('checkbox theme contract', () => {
  test('imports the shared checkbox layer globally', () => {
    const indexSource = readSource('src/assets/styles/index.scss')
    expect(indexSource).toContain('@import "checkbox";')
  })

  test('defines soft-panel checkbox tokens and density variants', () => {
    const source = readSource('src/assets/styles/checkbox.scss')

    expect(source).toContain('--checkbox-size: 18px;')
    expect(source).toContain('--checkbox-size-compact: 16px;')
    expect(source).toContain('--checkbox-size-table: 15px;')
    expect(source).toContain('--checkbox-radius: 6px;')
    expect(source).toContain('--checkbox-radius-compact: 4px;')
    expect(source).toMatch(/\\.el-checkbox__inner/)
    expect(source).toMatch(/\\.checkbox-container/)
    expect(source).toMatch(/\\.select-all-checkbox/)
    expect(source).toMatch(/html\\[data-theme=['"]dark['"]\\]/)
  })

  test('compiles the shared stylesheet with element and custom checkbox selectors', () => {
    const css = compileScss('src/assets/styles/checkbox.scss')

    expect(css).toContain('.el-checkbox__inner')
    expect(css).toContain('.checkbox-container')
    expect(css).toContain('.select-all-checkbox')
  })
})
```

- [ ] **Step 2: Run the new test to verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- FAIL because `checkbox.scss` does not exist yet and `index.scss` does not import it

- [ ] **Step 3: Implement the shared checkbox stylesheet**

Create `goto-web/src/assets/styles/checkbox.scss` and centralize the visual contract there. Use this structure:

```scss
:root,
html[data-theme='light'] {
  --checkbox-size: 18px;
  --checkbox-size-compact: 16px;
  --checkbox-size-table: 15px;
  --checkbox-radius: 6px;
  --checkbox-radius-compact: 4px;
  --checkbox-border: var(--border-default);
  --checkbox-border-hover: var(--border-strong);
  --checkbox-bg: var(--surface-elevated);
  --checkbox-bg-hover: var(--surface-muted);
  --checkbox-checked-bg: var(--accent-solid);
  --checkbox-checked-border: var(--accent-solid);
  --checkbox-icon: var(--text-on-accent);
  --checkbox-focus-ring: var(--focus-ring);
  --checkbox-disabled-bg: var(--surface-muted);
  --checkbox-disabled-border: var(--border-subtle);
}

html[data-theme='dark'] {
  --checkbox-bg: var(--surface-elevated);
  --checkbox-bg-hover: var(--surface-muted);
  --checkbox-border: var(--border-default);
  --checkbox-border-hover: var(--border-strong);
  --checkbox-disabled-bg: var(--surface-muted);
  --checkbox-disabled-border: var(--border-subtle);
}

.el-checkbox {
  --checkbox-size-current: var(--checkbox-size);
  --checkbox-radius-current: var(--checkbox-radius);

  .el-checkbox__inner {
    width: var(--checkbox-size-current);
    height: var(--checkbox-size-current);
    border-radius: var(--checkbox-radius-current);
    border-color: var(--checkbox-border);
    background: var(--checkbox-bg);
    transition: all 0.2s ease;
  }

  .el-checkbox__input.is-checked .el-checkbox__inner,
  .el-checkbox__input.is-indeterminate .el-checkbox__inner {
    background: var(--checkbox-checked-bg);
    border-color: var(--checkbox-checked-border);
  }

  .el-checkbox__input.is-focus .el-checkbox__inner {
    box-shadow: 0 0 0 3px var(--checkbox-focus-ring);
  }
}

.compact-checkbox,
.checkbox--compact {
  --checkbox-size-current: var(--checkbox-size-compact);
  --checkbox-radius-current: var(--checkbox-radius-compact);
}

.checkbox-container,
.select-all-checkbox {
  /* custom/native checkbox family reuses the same variables */
}
```

Then:

- import `checkbox.scss` in `src/assets/styles/index.scss`
- trim `element-ui-dark.scss` so checkbox-specific dark overrides do not fight the shared file
- leave radio/switch dark rules in place if they are unrelated

- [ ] **Step 4: Run the contract test again**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- PASS with the new global import and shared selector contract in place

- [ ] **Step 5: Commit the shared checkbox foundation**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
git add tests/unit/styles/checkboxTheme.spec.js src/assets/styles/checkbox.scss src/assets/styles/index.scss src/assets/styles/element-ui-dark.scss
git commit -m "feat: add shared checkbox style contract"
```

## Task 2: Migrate The Custom Native Checkbox Family

**Files:**
- Modify: `goto-web/tests/unit/styles/checkboxTheme.spec.js`
- Modify: `goto-web/src/views/goto/components/form/draw/checkbox.vue`
- Modify: `goto-web/src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/list.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue`

- [ ] **Step 1: Extend the guard for custom checkbox consumers**

Add a new test to `goto-web/tests/unit/styles/checkboxTheme.spec.js`:

```js
test('custom checkbox consumers use shared tokens instead of hardcoded legacy colors', () => {
  const drawCheckbox = readSource('src/views/goto/components/form/draw/checkbox.vue')
  const statsCheckbox = readSource('src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue')
  const listSource = readSource('src/views/goto/components/form/draw/list.vue')
  const listMarkSource = readSource('src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue')

  expect(drawCheckbox).toContain('var(--checkbox-border)')
  expect(drawCheckbox).toContain('var(--checkbox-checked-bg)')
  expect(drawCheckbox).not.toContain('#2196f3')
  expect(statsCheckbox).toContain('var(--checkbox-border)')
  expect(statsCheckbox).not.toMatch(/border:\\s*0\\.1em solid #000000/)
  expect(listSource).toContain('var(--checkbox-border)')
  expect(listSource).toContain('var(--checkbox-checked-bg)')
  expect(listMarkSource).not.toMatch(/checkbox-container input:checked ~ \\.checkmark[\\s\\S]*var\\(--primary-color\\)/)
})
```

- [ ] **Step 2: Run the guard to verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- FAIL because these files still contain old hardcoded checkbox rules and duplicate deep overrides

- [ ] **Step 3: Rewire custom checkbox implementations to the shared contract**

Apply the minimum changes:

- In `draw/checkbox.vue`, replace the hardcoded `21px` / `#2196f3` / black-border implementation with shared variables:

```scss
.checkbox-container {
  --checkbox-size-current: var(--checkbox-size, 18px);
  --checkbox-radius-current: var(--checkbox-radius, 6px);
}

.checkmark {
  width: var(--checkbox-size-current);
  height: var(--checkbox-size-current);
  background: var(--checkbox-bg);
  border: 1.5px solid var(--checkbox-border);
  border-radius: var(--checkbox-radius-current);
}

.checkbox-container input:checked ~ .checkmark {
  background: var(--checkbox-checked-bg);
  border-color: var(--checkbox-checked-border);
}
```

- In `checkboxBasicStatisticsMethod.vue`, replace the duplicated custom checkbox block with the same token-driven contract
- In `list.vue`, retune `.fab-checkbox-ui` to use the shared border, background, checked fill, and compact radius
- In `listMarkColumnContentParam.vue`, remove the deep override that forces `var(--primary-color)` and let the shared custom checkbox styling win

- [ ] **Step 4: Re-run the shared checkbox test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- PASS with no remaining legacy hardcoded custom-checkbox colors in these files

- [ ] **Step 5: Commit the custom checkbox migration**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
git add tests/unit/styles/checkboxTheme.spec.js src/views/goto/components/form/draw/checkbox.vue src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue src/views/goto/components/form/draw/list.vue src/views/goto/components/form/plotSetting/listMarkColumnContentParam.vue
git commit -m "feat: unify custom checkbox visuals"
```

## Task 3: Normalize Element UI Checkbox Consumers And Compact Variants

**Files:**
- Modify: `goto-web/tests/unit/styles/checkboxTheme.spec.js`
- Modify: `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyLegendParam.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/classifyParam.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- Modify: `goto-web/src/views/goto/components/form/plotSetting/facetParams.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`

- [ ] **Step 1: Extend the failing guards for compact and list-style Element checkboxes**

Update `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js` with a compact-variant assertion:

```js
test('compact checkbox keeps only density and label-visibility overrides locally', () => {
  const source = readSource('src/views/goto/components/form/draw/classifyLegendParam.vue')

  expect(source).toMatch(/compact-checkbox[\\s\\S]*--checkbox-size-current:\\s*var\\(--checkbox-size-compact\\)/)
  expect(source).toMatch(/compact-checkbox[\\s\\S]*el-checkbox__label[\\s\\S]*display:\\s*none/)
  expect(source).not.toMatch(/compact-checkbox[\\s\\S]*border-color:/)
  expect(source).not.toMatch(/compact-checkbox[\\s\\S]*background:/)
})
```

Add a second guard to `goto-web/tests/unit/styles/checkboxTheme.spec.js`:

```js
test('checkbox-heavy goto forms stop owning checkbox palette locally', () => {
  const classifyParam = readSource('src/views/goto/components/form/plotSetting/classifyParam.vue')
  const markSelector = readSource('src/views/goto/components/form/plotSetting/MarkColumnSelector.vue')
  const facetParams = readSource('src/views/goto/components/form/plotSetting/facetParams.vue')

  expect(classifyParam).toMatch(/file-group[\\s\\S]*background-color:\\s*var\\(--card-bg\\)/)
  expect(classifyParam).not.toMatch(/file-group[\\s\\S]*background-color:\\s*#fff/)
  expect(markSelector).toMatch(/mark-selector-container[\\s\\S]*border:\\s*1px solid var\\(--border-color\\)/)
  expect(markSelector).not.toMatch(/mark-selector-container[\\s\\S]*background-color:\\s*#fff/)
  expect(facetParams).toMatch(/file-mark-selector[\\s\\S]*background-color:\\s*var\\(--card-bg\\)/)
  expect(facetParams).not.toMatch(/file-mark-selector[\\s\\S]*background-color:\\s*#fafafa/)
})
```

- [ ] **Step 2: Run both tests to verify they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/classifyLegendParamTheme.spec.js --runInBand
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- FAIL because `classifyLegendParam.vue` still owns checkbox color/border styles and checkbox list pages still depend on local palette rules

- [ ] **Step 3: Strip local palette ownership from Element UI consumers**

Implement the cleanup:

- In `classifyLegendParam.vue`, keep only compact sizing and hidden-label behavior. Replace local color ownership with shared CSS variables:

```scss
.compact-checkbox {
  --checkbox-size-current: var(--checkbox-size-compact);
  --checkbox-radius-current: var(--checkbox-radius-compact);

  ::v-deep .el-checkbox__label {
    display: none;
  }
}
```

- In `classifyParam.vue`, `MarkColumnSelector.vue`, and `facetParams.vue`, keep the card/list layout styles but remove checkbox-inner color and local checked/unchecked palette ownership unless it is purely layout
- In `classifyChooseHeaderInfo.vue`, keep the bordered checkbox layout and align the checked/unselected panel treatment to shared theme tokens instead of hardcoded checkbox colors

- [ ] **Step 4: Re-run the targeted tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/classifyLegendParamTheme.spec.js --runInBand
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- PASS with compact density still intact and checkbox palette centralized

- [ ] **Step 5: Commit the Element UI consumer cleanup**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
git add tests/unit/styles/checkboxTheme.spec.js tests/unit/styles/classifyLegendParamTheme.spec.js src/views/goto/components/form/draw/classifyLegendParam.vue src/views/goto/components/form/plotSetting/classifyParam.vue src/views/goto/components/form/plotSetting/MarkColumnSelector.vue src/views/goto/components/form/plotSetting/facetParams.vue src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue
git commit -m "feat: align element checkbox consumers with shared theme"
```

## Task 4: Retune Handsontable Checkboxes To The Shared Compact Contract

**Files:**
- Modify: `goto-web/tests/unit/styles/checkboxTheme.spec.js`
- Modify: `goto-web/src/assets/styles/handsontable/handsontable-custom.scss`
- Verify: `goto-web/src/views/components/handsontable.vue`

- [ ] **Step 1: Add a failing Handsontable checkbox guard**

Add this test to `goto-web/tests/unit/styles/checkboxTheme.spec.js`:

```js
test('handsontable checkbox tokens and the select-all checkbox use the shared compact contract', () => {
  const source = readSource('src/assets/styles/handsontable/handsontable-custom.scss')

  expect(source).toMatch(/--ht-checkbox-border-color:\\s*var\\(--checkbox-border\\)/)
  expect(source).toMatch(/--ht-checkbox-background-color:\\s*var\\(--checkbox-bg\\)/)
  expect(source).toMatch(/--ht-checkbox-checked-background-color:\\s*var\\(--checkbox-checked-bg\\)/)
  expect(source).toMatch(/\\.select-all-checkbox[\\s\\S]*width:\\s*var\\(--checkbox-size-table\\)/)
  expect(source).toMatch(/\\.select-all-checkbox[\\s\\S]*border-radius:\\s*var\\(--checkbox-radius-compact\\)/)
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- FAIL because `handsontable-custom.scss` still hardcodes checkbox blues, grays, radii, and shadow values

- [ ] **Step 3: Rewire Handsontable checkbox variables and the custom select-all checkbox**

In `goto-web/src/assets/styles/handsontable/handsontable-custom.scss`:

- map the `--ht-checkbox-*` variables to the shared checkbox contract instead of standalone hardcoded values
- retune `::v-deep .select-all-checkbox` to use the table density variables

Use this target structure:

```scss
[data-theme='light'] .ht-theme-main-dark-auto,
[data-theme='dark'] .ht-theme-main-dark-auto {
  --ht-checkbox-border-color: var(--checkbox-border);
  --ht-checkbox-background-color: var(--checkbox-bg);
  --ht-checkbox-focus-ring-color: var(--checkbox-focus-ring);
  --ht-checkbox-checked-border-color: var(--checkbox-checked-border);
  --ht-checkbox-checked-background-color: var(--checkbox-checked-bg);
  --ht-checkbox-checked-icon-color: var(--checkbox-icon);
}

::v-deep .select-all-checkbox {
  width: var(--checkbox-size-table);
  height: var(--checkbox-size-table);
  border-radius: var(--checkbox-radius-compact);
  background: var(--checkbox-bg);
  border: 1.5px solid var(--checkbox-border);
}
```

`handsontable.vue` should remain verify-only unless the stylesheet cannot target the existing markup cleanly.

- [ ] **Step 4: Re-run the shared checkbox test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
```

Expected:

- PASS with `Handsontable` now following the same compact checkbox family

- [ ] **Step 5: Commit the Handsontable checkbox alignment**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
git add tests/unit/styles/checkboxTheme.spec.js src/assets/styles/handsontable/handsontable-custom.scss
git commit -m "feat: unify handsontable checkbox styling"
```

## Task 5: Verify Plain Consumers, Full Style Tests, And Browser Smoke Checks

**Files:**
- Verify: `goto-web/src/views/goto/components/SyncIcon.vue`
- Verify: `goto-web/src/views/goto/process/components/Step1Upload.vue`
- Verify: `goto-web/src/views/goto/process/components/Step5Transform.vue`
- Verify: `goto-web/src/views/components/handsontable.vue`

- [ ] **Step 1: Run the targeted style suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
npx jest tests/unit/styles/checkboxTheme.spec.js --runInBand
npx jest tests/unit/styles/classifyLegendParamTheme.spec.js --runInBand
npx jest tests/unit/styles/drawDarkTheme.spec.js --runInBand
```

Expected:

- PASS on all three commands

- [ ] **Step 2: Run a production-style frontend build**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service build --mode staging
```

Expected:

- successful build with no SCSS compile errors introduced by the new checkbox layer

- [ ] **Step 3: Do a manual light/dark smoke check**

Open the frontend and verify both light and dark themes for:

- `src/views/goto/components/form/draw/classifyLegendParam.vue`
- `src/views/goto/components/form/plotSetting/classifyParam.vue`
- `src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- `src/views/goto/components/form/plotSetting/facetParams.vue`
- `src/views/goto/statistics/components/checkboxBasicStatisticsMethod.vue`
- `src/views/components/handsontable.vue`
- `src/views/goto/process/components/Step1Upload.vue`
- `src/views/goto/process/components/Step5Transform.vue`
- recursive checkbox in `src/views/goto/components/SyncIcon.vue`

For each page, check:

- default
- hover
- checked
- indeterminate
- disabled
- focus

- [ ] **Step 4: Review diff scope and only commit if manual smoke checks forced follow-up fixes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/checkbox-soft-panel-20260325
git status --short
git diff --stat
```

Expected:

- working tree is clean if no manual follow-up fixes were needed
- if the manual smoke check exposed a final polish item, stage only the touched files and create one more exact-file commit
- no unrelated files from the dirty original working tree appear in this worktree

## Completion Criteria

The implementation is complete only when all of the following are true:

1. `goto-web` uses one shared checkbox style layer for `Element UI`, custom native checkboxes, and `Handsontable`
2. `classifyLegendParam.vue` keeps compact layout but no longer owns checkbox palette
3. custom checkbox implementations no longer hardcode legacy blue/black visuals
4. `Handsontable` header and cell checkboxes match the shared compact family
5. light and dark themes both pass manual smoke checks
6. the targeted Jest style suite and build command both pass inside the isolated `goto-web/.worktrees/...` branch
