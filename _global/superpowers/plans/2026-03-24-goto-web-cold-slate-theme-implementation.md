# goto-web Cold Slate Theme Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Roll out the approved `Cold Slate` light/dark theme to `goto-web`'s `src/views/goto` core pages by adding semantic theme tokens, keeping the legacy variables compatible, and polishing the most visible tool surfaces.

**Architecture:** Build the rollout in three layers. First, replace the application-level theme variables with the approved semantic `surface / text / accent / border / shadow` contract while preserving existing variable names. Second, bridge `draw` tokens onto that semantic layer so shared form controls and `draw-theme-scope` pages inherit the new theme without ad hoc overrides. Third, retune the highest-traffic `goto` components so overlays, focus states, buttons, and tags use the new tokens instead of hardcoded colors or legacy shadows.

**Tech Stack:** Vue 2.6, Element UI 2.15, Vue CLI 3, SCSS (`sass-loader@10`), Jest (`@vue/cli-plugin-unit-jest`), existing style guard tests in `goto-web/tests/unit/styles`

---

## Preconditions

- Work from the repository root: `/mnt/f/IdeaProjects/goto-software`
- Use the approved spec as the source of truth:
  `docs/superpowers/specs/2026-03-24-goto-web-goto-cold-slate-theme-design.md`
- The worktree is already dirty in unrelated areas, so every `git add` in this plan must name exact files. Never use `git add .`.
- `goto-web/package.json` uses Windows-style `set NODE_OPTIONS=...` scripts. In this Bash environment, run `npx jest ...` directly and use `NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service ...` for any manual serve/build commands.
- GitNexus MCP tools/resources are not available in the current planning session. If they are available in the implementation session, run impact analysis before editing symbol-like mixins/functions and run change detection before each commit, per `AGENTS.md`. If they remain unavailable, use the Jest guard files in this plan plus exact `git diff --stat` review as the fallback safety net.

## Scope Check

This plan covers one coherent subsystem: the `goto-web` theme contract plus its immediate `src/views/goto` consumers. It does not attempt to restyle unrelated login or non-`goto` pages beyond whatever they inherit automatically from the shared compatibility variables.

## Implementation Notes

- Start each coding task with `@test-driven-development`: write the failing guard first, run it, then implement the minimum change.
- End each coding task with the targeted Jest command listed in that task before committing.
- Use `@verification-before-completion` discipline at the end: run the full style test bundle, compile the critical SCSS entrypoints, then do a manual light/dark smoke check in the browser.
- Keep the semantic token source in `theme-variables.scss`. `draw` should map to it, not fork a second theme system.
- Preserve compatibility for existing callers of `--primary-color`, `--card-bg`, `--input-bg`, `--hover-bg`, `--text-main`, `--border-color`, and related variables.
- The spec intentionally separates `accent-primary` from `accent-solid`; do not collapse them back together in dark mode.

## File Map

### New tests to create

- `goto-web/tests/unit/styles/coldSlateTheme.spec.js`
  - Guard the new semantic token contract and the approved accessible color pairs

### Existing tests to modify

- `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
  - Guard `draw` token mapping onto the semantic theme layer
- `goto-web/tests/unit/styles/drawRegressionFixes.spec.js`
  - Guard the `goto` page chrome and control polish against regression

### Global theme files to modify

- `goto-web/src/assets/styles/theme-variables.scss`
  - Add the semantic `surface / text / accent / border / shadow` variables and map the legacy variables to them
- `goto-web/src/assets/styles/dark-mode-global.scss`
  - Replace remaining hardcoded dark values with semantic tokens
- `goto-web/src/assets/styles/element-ui-dark.scss`
  - Align dialogs, dropdowns, inputs, and other global Element UI dark surfaces with the new semantic theme tokens

### Draw bridge files to modify

- `goto-web/src/assets/styles/draw/_variables.scss`
  - Map `--draw-*` tokens to semantic theme variables with safe legacy fallbacks
- `goto-web/src/assets/styles/draw/foundation.scss`
  - Ensure the shared `draw-theme-scope` entrypoint consumes the new token layer
- `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
  - Retune shared card/control polish to the new radius/shadow contract
- `goto-web/src/assets/styles/draw/_element-overrides.scss`
  - Carry the new focus ring, surface, and overlay tokens into body-level poppers and Element UI overrides

### Core `goto` page/component files to modify

- `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`
  - Upgrade zoom controls and tab chrome to the `Cold Slate` overlay/radius treatment
- `goto-web/src/views/goto/toolsBox/type/tool.vue`
  - Upgrade the tool panel shell, icon buttons, input focus, and download button tokens
- `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`
  - Upgrade the card shell, help alerts, radio cards, upload area, and action buttons
- `goto-web/src/views/goto/components/form/draw/input.vue`
  - Add semantic focus ring and improved number-stepper chrome
- `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
  - Remove hardcoded icon black, retune selected tag surfaces, keep overflow safe

### Verify-only files

- `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
  - Confirm the existing full-width upload fix still looks correct after the token shift
- `goto-web/src/assets/styles/index.scss`
  - Verify the global import order still gives `theme-variables.scss` to `element-ui-dark.scss`

## Task 1: Lock The Global Cold Slate Theme Contract

**Files:**
- Create: `goto-web/tests/unit/styles/coldSlateTheme.spec.js`
- Modify: `goto-web/src/assets/styles/theme-variables.scss`
- Modify: `goto-web/src/assets/styles/dark-mode-global.scss`
- Modify: `goto-web/src/assets/styles/element-ui-dark.scss`

- [ ] **Step 1: Write the failing global theme contract test**

Create `goto-web/tests/unit/styles/coldSlateTheme.spec.js` with source assertions and a small WCAG helper:

```js
const fs = require('fs')
const path = require('path')

function readSource(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

function hexToRgb(hex) {
  const value = hex.replace('#', '')
  return [0, 2, 4].map(index => parseInt(value.slice(index, index + 2), 16) / 255)
}

function linearize(channel) {
  return channel <= 0.03928
    ? channel / 12.92
    : Math.pow((channel + 0.055) / 1.055, 2.4)
}

function contrast(hexA, hexB) {
  const lum = hex => {
    const [r, g, b] = hexToRgb(hex).map(linearize)
    return 0.2126 * r + 0.7152 * g + 0.0722 * b
  }

  const a = lum(hexA)
  const b = lum(hexB)
  const high = Math.max(a, b)
  const low = Math.min(a, b)
  return (high + 0.05) / (low + 0.05)
}

describe('Cold Slate theme contract', () => {
  test('defines semantic light and dark tokens plus legacy bridges', () => {
    const source = readSource('src/assets/styles/theme-variables.scss')

    expect(source).toContain('--surface-canvas: #f5f7fb;')
    expect(source).toContain('--surface-panel: #fcfdff;')
    expect(source).toContain('--surface-canvas: #0f1724;')
    expect(source).toContain('--surface-panel: #161f2d;')
    expect(source).toContain('--accent-primary: #4666e5;')
    expect(source).toContain('--accent-primary: #7c8cff;')
    expect(source).toContain('--accent-solid: #586ce0;')
    expect(source).toContain('--text-on-accent: #ffffff;')
    expect(source).toContain('--body-color: var(--surface-canvas);')
    expect(source).toContain('--card-bg: var(--surface-panel);')
    expect(source).toContain('--primary-color: var(--accent-solid);')
  })

  test('keeps the approved text and button combinations AA-safe', () => {
    expect(contrast('#111827', '#f5f7fb')).toBeGreaterThanOrEqual(4.5)
    expect(contrast('#5f6b7a', '#fcfdff')).toBeGreaterThanOrEqual(4.5)
    expect(contrast('#e8edf5', '#0f1724')).toBeGreaterThanOrEqual(4.5)
    expect(contrast('#a7b4c7', '#161f2d')).toBeGreaterThanOrEqual(4.5)
    expect(contrast('#ffffff', '#4666e5')).toBeGreaterThanOrEqual(4.5)
    expect(contrast('#ffffff', '#586ce0')).toBeGreaterThanOrEqual(4.5)
  })
})
```

- [ ] **Step 2: Run the new test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/coldSlateTheme.spec.js --runInBand
```

Expected:

- FAIL because `theme-variables.scss` does not yet define the semantic tokens or the new dark solid accent split

- [ ] **Step 3: Implement the semantic global theme layer**

Update `theme-variables.scss` so the light and dark blocks follow the approved spec. Use this structure:

```scss
:root,
html[data-theme='light'] {
  color-scheme: light;

  --surface-canvas: #f5f7fb;
  --surface-panel: #fcfdff;
  --surface-elevated: #ffffff;
  --surface-muted: #eef2f8;
  --surface-overlay: rgba(252, 253, 255, 0.78);

  --text-primary: #111827;
  --text-secondary: #5f6b7a;
  --text-muted: #7d8a9b;
  --text-inverse: #f8fbff;

  --accent-primary: #4666e5;
  --accent-solid: #4666e5;
  --accent-soft: #e8efff;
  --accent-strong: #3d5add;
  --text-on-accent: #ffffff;

  --border-subtle: #e6ebf2;
  --border-default: #d7dfea;
  --border-strong: #b8c4d6;

  --shadow-xs: 0 1px 2px rgba(15, 23, 36, 0.04);
  --shadow-sm: 0 10px 24px rgba(15, 23, 36, 0.06);
  --shadow-md: 0 18px 48px rgba(15, 23, 36, 0.10);
  --shadow-lg: 0 24px 64px rgba(15, 23, 36, 0.14);

  --focus-ring: rgba(70, 102, 229, 0.18);

  --primary-color: var(--accent-solid);
  --primary-color-light: var(--accent-soft);
  --primary-color-btn-light: var(--accent-strong);
  --body-color: var(--surface-canvas);
  --card-bg: var(--surface-panel);
  --input-bg: var(--surface-elevated);
  --hover-bg: var(--surface-muted);
}

html[data-theme='dark'] {
  color-scheme: dark;

  --surface-canvas: #0f1724;
  --surface-panel: #161f2d;
  --surface-elevated: #1c2636;
  --surface-muted: #243044;
  --surface-overlay: rgba(19, 30, 45, 0.78);

  --text-primary: #e8edf5;
  --text-secondary: #a7b4c7;
  --text-muted: #8290a6;
  --text-inverse: #0f1724;

  --accent-primary: #7c8cff;
  --accent-solid: #586ce0;
  --accent-soft: #1d2b4d;
  --accent-strong: #4f63d8;
  --text-on-accent: #ffffff;
}
```

Then update the remaining global dark files so they consume semantic variables instead of hardcoded GitHub-style colors. For example:

```scss
html[data-theme='dark'] {
  .rightPanel {
    background: var(--surface-panel) !important;
    border-left: 1px solid var(--border-default) !important;
  }

  *::-webkit-scrollbar-thumb {
    background-color: var(--border-default) !important;
    border: 2px solid var(--surface-panel) !important;
  }
}
```

And in `element-ui-dark.scss`, move shell-like controls onto the semantic layer:

```scss
html[data-theme='dark'] {
  .el-dialog,
  .el-card,
  .el-select-dropdown,
  .el-dropdown-menu {
    background: var(--surface-panel) !important;
    border-color: var(--border-default) !important;
    box-shadow: var(--shadow-md) !important;
  }
}
```

- [ ] **Step 4: Re-run the targeted test and grep away stale dark literals**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/coldSlateTheme.spec.js --runInBand
rg -n "#0d1117|#161b22|#30363d|#818cf8" src/assets/styles/theme-variables.scss src/assets/styles/dark-mode-global.scss src/assets/styles/element-ui-dark.scss
```

Expected:

- Jest PASS
- `rg` prints no matches for the old GitHub-style dark palette in those three files

- [ ] **Step 5: Commit the global theme layer**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/tests/unit/styles/coldSlateTheme.spec.js \
  goto-web/src/assets/styles/theme-variables.scss \
  goto-web/src/assets/styles/dark-mode-global.scss \
  goto-web/src/assets/styles/element-ui-dark.scss
git commit -m "feat: add cold slate semantic theme variables"
```

## Task 2: Bridge Draw Tokens To The Semantic Theme Layer

**Files:**
- Modify: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
- Modify: `goto-web/src/assets/styles/draw/_variables.scss`
- Modify: `goto-web/src/assets/styles/draw/foundation.scss`
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
- Modify: `goto-web/src/assets/styles/draw/_element-overrides.scss`

- [ ] **Step 1: Extend the draw token test so it fails on the old mapping**

In `drawDarkTheme.spec.js`, add assertions like:

```js
test('maps draw tokens onto semantic app tokens', () => {
  const source = readSource('src/assets/styles/draw/_variables.scss')

  expect(source).toMatch(/--draw-bg-primary:\s*var\(--surface-panel,\s*var\(--card-bg\)\);/)
  expect(source).toMatch(/--draw-bg-secondary:\s*var\(--surface-elevated,\s*var\(--input-bg\)\);/)
  expect(source).toMatch(/--draw-bg-tertiary:\s*var\(--surface-muted,\s*var\(--hover-bg\)\);/)
  expect(source).toMatch(/--draw-active-color:\s*var\(--accent-primary,\s*var\(--primary-color\)\);/)
  expect(source).toMatch(/--draw-active-solid:\s*var\(--accent-solid,\s*var\(--primary-color\)\);/)
  expect(source).toMatch(/--draw-focus-ring:\s*var\(--focus-ring\);/)
  expect(source).toMatch(/--draw-shadow-card:\s*var\(--shadow-sm\);/)
  expect(source).toMatch(/--draw-shadow-card-hover:\s*var\(--shadow-md\);/)
  expect(source).toMatch(/--draw-overlay-bg:\s*var\(--surface-overlay\);/)
})
```

Add one shared-skin assertion to make sure the bridge is actually consumed:

```js
const skinSource = readSource('src/assets/styles/draw/_modern-lab-theme.scss')
expect(skinSource).toMatch(/var\(--draw-focus-ring\)/)
```

- [ ] **Step 2: Run the focused draw theme tests and confirm they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawDarkTheme.spec.js --runInBand
```

Expected:

- FAIL because `_variables.scss` does not yet expose the new semantic draw bridge tokens
- FAIL because `_modern-lab-theme.scss` is not yet consuming a `draw` focus ring token

- [ ] **Step 3: Implement the draw-to-semantic bridge**

Update `_variables.scss` so `draw` consumes semantic tokens first and legacy variables second:

```scss
@mixin draw-theme-aware-tokens {
  --draw-bg-primary: var(--surface-panel, var(--card-bg));
  --draw-bg-secondary: var(--surface-elevated, var(--input-bg));
  --draw-bg-tertiary: var(--surface-muted, var(--hover-bg));

  --draw-border-light: var(--border-subtle, var(--border-color-light, var(--border-color)));
  --draw-border-normal: var(--border-default, var(--border-color));
  --draw-border-dark: var(--border-strong, var(--border-color-strong, var(--border-color)));

  --draw-text-primary: var(--text-primary, var(--text-main));
  --draw-text-secondary: var(--text-secondary);
  --draw-text-disabled: var(--text-muted, var(--text-tertiary, var(--text-secondary)));

  --draw-active-color: var(--accent-primary, var(--primary-color));
  --draw-active-solid: var(--accent-solid, var(--primary-color));
  --draw-focus-ring: var(--focus-ring);
  --draw-overlay-bg: var(--surface-overlay);

  --draw-shadow-card: var(--shadow-sm);
  --draw-shadow-card-hover: var(--shadow-md);
  --draw-shadow-zoom: var(--shadow-sm);
}
```

Then make the shared draw skin actually consume the bridge:

```scss
.modern-lab-form-group {
  .form-content-item:focus-within,
  :deep(.el-input__inner:focus) {
    box-shadow: 0 0 0 3px var(--draw-focus-ring);
  }
}
```

Also make body-level poppers use the new overlay/panel tokens in `_element-overrides.scss` where appropriate:

```scss
.draw-theme-popper.el-select-dropdown.el-popper {
  background: var(--draw-bg-primary) !important;
  border-color: var(--draw-border-normal) !important;
  box-shadow: var(--shadow-md) !important;
}
```

- [ ] **Step 4: Re-run the focused draw theme tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawDarkTheme.spec.js --runInBand
```

Expected:

- PASS for the new draw bridge assertions
- PASS for the existing `html[data-theme]` and scope assertions

- [ ] **Step 5: Commit the draw bridge changes**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/tests/unit/styles/drawDarkTheme.spec.js \
  goto-web/src/assets/styles/draw/_variables.scss \
  goto-web/src/assets/styles/draw/foundation.scss \
  goto-web/src/assets/styles/draw/_modern-lab-theme.scss \
  goto-web/src/assets/styles/draw/_element-overrides.scss
git commit -m "feat: bridge draw tokens to cold slate theme"
```

## Task 3: Polish The Core `goto` Chrome And Shared Controls

**Files:**
- Modify: `goto-web/tests/unit/styles/drawRegressionFixes.spec.js`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/input.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Verify: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`

- [ ] **Step 1: Extend the regression guards to reflect the approved polish**

In `drawRegressionFixes.spec.js`, add targeted assertions like:

```js
test('PlotResultCanvas upgrades zoom chrome to the overlay treatment', () => {
  const source = readSource('src/views/goto/plot/type/components/PlotResultCanvas.vue')

  expect(source).toMatch(/\.zoom-controls\s*{[\s\S]*background:\s*var\(--draw-overlay-bg,\s*var\(--draw-bg-primary\)\);/)
  expect(source).toMatch(/\.zoom-controls\s*{[\s\S]*backdrop-filter:\s*blur\(16px\) saturate\(120%\);/)
  expect(source).toMatch(/\.zoom-controls\s*{[\s\S]*border-radius:\s*14px;/)
})

test('tool page uses semantic accent text and panel shadows', () => {
  const source = readSource('src/views/goto/toolsBox/type/tool.vue')

  expect(source).toMatch(/#property-board\s*{[\s\S]*box-shadow:\s*var\(--shadow-md\);/)
  expect(source).toMatch(/\.download \.button__text\s*{[\s\S]*color:\s*var\(--text-on-accent\);/)
  expect(source).toMatch(/#idraw-scale ::v-deep \.el-input__inner:focus\s*{[\s\S]*box-shadow:\s*0 0 0 3px var\(--focus-ring\);/)
})

test('shared draw inputs and multi-selects consume semantic focus and tag tokens', () => {
  const inputSource = readSource('src/views/goto/components/form/draw/input.vue')
  const selectSource = readSource('src/views/goto/components/form/draw/selectMultipleObject.vue')

  expect(inputSource).toMatch(/:deep\(\.el-input__inner\)\s*{[\s\S]*&:focus\s*{[\s\S]*box-shadow:\s*0 0 0 3px var\(--draw-focus-ring\);/)
  expect(selectSource).toMatch(/::v-deep \.el-tag\s*{[\s\S]*background:\s*var\(--accent-soft\);/)
  expect(selectSource).toMatch(/::v-deep \.el-select \.el-input \.el-input__suffix .el-input__suffix-inner i\s*{[\s\S]*color:\s*var\(--draw-text-secondary\);/)
})
```

- [ ] **Step 2: Run the regression suite and confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawRegressionFixes.spec.js --runInBand
```

Expected:

- FAIL because the component files still use the older radius/shadow/focus/tag treatments

- [ ] **Step 3: Implement the component polish**

Apply the approved polish in the component styles:

`PlotResultCanvas.vue`

```scss
.zoom-controls {
  background: var(--draw-overlay-bg, var(--draw-bg-primary));
  backdrop-filter: blur(16px) saturate(120%);
  border-radius: 14px;
  box-shadow: var(--shadow-sm);
}

#show.el-tabs.el-tabs--border-card > .el-tabs__header {
  background: var(--draw-overlay-bg) !important;
}
```

`tool.vue`

```scss
#property-board {
  background-color: var(--surface-panel);
  box-shadow: var(--shadow-md);
}

#idraw-scale ::v-deep .el-input__inner:focus,
.modern-input-wrap ::v-deep .el-input:focus-within {
  box-shadow: 0 0 0 3px var(--focus-ring);
}

.download .button__text,
.download .svg {
  color: var(--text-on-accent);
  fill: var(--text-on-accent);
}
```

`excelFormatConvert.vue`

```scss
.convert-card {
  border-radius: 14px;
  box-shadow: var(--shadow-md);
}

.help-section ::v-deep .el-alert--info.is-light {
  background: var(--surface-panel);
}
```

`input.vue`

```scss
:deep(.el-input__inner) {
  &:focus {
    border-color: var(--draw-active-solid);
    box-shadow: 0 0 0 3px var(--draw-focus-ring);
  }
}
```

`selectMultipleObject.vue`

```scss
::v-deep .el-tag {
  background: var(--accent-soft);
  color: var(--accent-strong);
  border: 1px solid transparent;
  border-radius: 999px;
}

::v-deep .el-select .el-input .el-input__suffix .el-input__suffix-inner i {
  color: var(--draw-text-secondary);
}
```

While applying those changes, keep the existing width/overflow fixes in `PlotSidePanel.vue` and `selectMultipleObject.vue` intact.

- [ ] **Step 4: Re-run the regression suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest tests/unit/styles/drawRegressionFixes.spec.js --runInBand
```

Expected:

- PASS for the new overlay/radius/focus/tag assertions
- PASS for the pre-existing upload-width and dark regression assertions

- [ ] **Step 5: Commit the component polish**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add \
  goto-web/tests/unit/styles/drawRegressionFixes.spec.js \
  goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue \
  goto-web/src/views/goto/toolsBox/type/tool.vue \
  goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue \
  goto-web/src/views/goto/components/form/draw/input.vue \
  goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue
git commit -m "feat: polish goto cold slate surfaces"
```

## Task 4: Run Full Verification And Manual Smoke Checks

**Files:**
- Verify: `goto-web/tests/unit/styles/coldSlateTheme.spec.js`
- Verify: `goto-web/tests/unit/styles/drawDarkTheme.spec.js`
- Verify: `goto-web/tests/unit/styles/drawRegressionFixes.spec.js`
- Verify: `goto-web/tests/unit/styles/classifyLegendParamTheme.spec.js`
- Verify: `goto-web/tests/unit/styles/coordinatePointTheme.spec.js`
- Verify: `goto-web/src/assets/styles/theme-variables.scss`
- Verify: `goto-web/src/assets/styles/draw/foundation.scss`
- Verify: the browser rendering of `/goto/plot` and `/goto/toolsBox`

- [ ] **Step 1: Run the full style-oriented Jest bundle**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx jest \
  tests/unit/styles/coldSlateTheme.spec.js \
  tests/unit/styles/drawDarkTheme.spec.js \
  tests/unit/styles/drawRegressionFixes.spec.js \
  tests/unit/styles/classifyLegendParamTheme.spec.js \
  tests/unit/styles/coordinatePointTheme.spec.js \
  --runInBand
```

Expected:

- PASS on all five style suites

- [ ] **Step 2: Compile the critical SCSS entrypoints directly**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx sass src/assets/styles/theme-variables.scss /tmp/theme-variables.css
npx sass src/assets/styles/draw/foundation.scss /tmp/draw-foundation.css
```

Expected:

- Both commands exit `0`
- No Sass syntax errors from the new token layer or draw bridge

- [ ] **Step 3: Grep for hardcoded color leftovers in the polished component files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
rg -n "#000000|#ffffff|rgba\\(255,\\s*255,\\s*255|rgba\\(0,\\s*0,\\s*0" \
  src/views/goto/plot/type/components/PlotResultCanvas.vue \
  src/views/goto/toolsBox/type/tool.vue \
  src/views/goto/toolsBox/type/excelFormatConvert.vue \
  src/views/goto/components/form/draw/input.vue \
  src/views/goto/components/form/draw/selectMultipleObject.vue
```

Expected:

- No stray black/white hardcoded values remain in those component style blocks except intentional non-theme assets that you explicitly reviewed

- [ ] **Step 4: Do a manual light/dark browser smoke check**

Run the dev server in Bash:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service serve
```

Manual checks:

1. Open the `goto` plot page and toggle light/dark. Verify the result tabs, zoom controls, right panel, and input focus rings match the new `Cold Slate` hierarchy.
2. Open the `goto` tools page and inspect the tool shell, icon buttons, scale input, and download button in both themes.
3. Open the Excel format convert page from the tools flow and verify the help alerts, radio cards, upload panel, and primary buttons use the new surfaces and shadows.
4. Re-check `PlotSidePanel` after the token shift to ensure the existing full-width uploader still looks correct.

- [ ] **Step 5: Review the final diff and commit any last verification-only fixes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git diff --stat
git status --short
```

If Step 4 required small final fixes, commit them with an exact file list:

```bash
git add \
  goto-web/src/assets/styles/theme-variables.scss \
  goto-web/src/assets/styles/draw/_variables.scss \
  goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue
git commit -m "chore: finalize cold slate theme verification"
```

Replace that `git add` list with the exact files you actually touched during the final smoke-fix pass. Do not add unrelated files.

If there are no code changes after verification, do not create an empty commit.

## Plan Review

Because the current session was not explicitly authorized for subagent delegation, this plan should be self-reviewed against `writing-plans/plan-document-reviewer-prompt.md` before execution. Use the following checklist:

- No placeholder steps or `TODO` text remain
- The plan stays aligned with `docs/superpowers/specs/2026-03-24-goto-web-goto-cold-slate-theme-design.md`
- Every task names exact files, exact commands, and an observable success/failure condition
- The plan does not require whole-site restyling beyond the `goto` core pages

## Execution Handoff

After the plan is accepted, choose one of these execution modes:

1. **Subagent-Driven (recommended)**: Use `superpowers:subagent-driven-development` and execute one task per fresh worker with review between tasks.
2. **Inline Execution**: Use `superpowers:executing-plans` and execute the tasks in this session with checkpoints.
