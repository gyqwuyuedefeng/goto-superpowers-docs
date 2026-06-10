# goto-web `goto` Dialog Responsive Size Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `goto-web/src/views/goto` 下基于统一 `Dialog` 组件的弹窗改造成 `size="sm|md|lg"` 分级响应式体系，统一居中、视口收敛和 body 滚动行为。

**Architecture:** 先把弹窗尺寸计算与布局降级规则抽到纯函数中，用单元测试固定 `sm/md/lg`、小屏断点和安全边距模式；再把 [dialog.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/components/dialog.vue) 改为消费该规则并监听视口变化；最后逐个迁移 `src/views/goto` 下现有 `Dialog` 调用点，删除 `width/height` 传递并改为显式 `size`。

**Tech Stack:** Vue 2, Element UI 2, Jest, Vue Test Utils, ripgrep, GitNexus

---

## File Structure

### Shared Component and Logic

- Modify: `goto-web/src/views/components/dialog.vue`
  - 统一 `Dialog` 对外接口为 `size`
  - 接入响应式尺寸计算
  - 管理 resize 生命周期与安全边距模式
- Create: `goto-web/src/views/components/dialogSize.js`
  - 输出纯函数：按 `size` 和 viewport 计算最终宽度、最大高度和布局模式
- Test: `goto-web/tests/unit/views/components/dialogSize.spec.js`
  - 固定 `sm/md/lg`、`<768px` 断点、`<700px` 高度兜底和安全边距切换规则

### Migrated `Dialog` Call Sites

- Modify: `goto-web/src/views/goto/index.vue`
  - `提示/退出系统` 弹窗改为 `size="sm"`
- Modify: `goto-web/src/views/goto/components/chooseColumnMultipleContent.vue`
  - 表格型列内容选择弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/components/classifyItemChooseColumn.vue`
  - 表格型单列选择弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
  - `选择分析结果` 弹窗改为 `size="lg"`
  - `下载设置` 弹窗改为 `size="md"`
- Modify: `goto-web/src/views/goto/plot/type/fileMark/formatConvertDialog.vue`
  - 删除 `width/height` 透传，改为 `size="lg"`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotTemplateDialog.vue`
  - 模板管理弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotHistoryDialog.vue`
  - 历史记录弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue`
  - 列名选择网格弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesShow.vue`
  - 对比组设置弹窗改为 `size="lg"`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`
  - 分类列名选择网格弹窗改为 `size="lg"`

### Verification and Supporting References

- Reference: `goto-web/package.json`
  - 使用 `npm run test:unit` / `vue-cli-service test:unit` / `npm run lint`
- Reference: `docs/superpowers/specs/2026-03-28-goto-web-goto-dialog-responsive-size-design.md`
  - 实现必须遵守已批准的断点、比例和安全边距规则
- Reference: repo-root `AGENTS.md`
  - 编辑符号前先跑 GitNexus impact
  - 每次提交前先跑 GitNexus detect changes

## Task 0: Prepare GitNexus and Test Harness

**Files:**
- Reference: `/mnt/f/IdeaProjects/goto-software/AGENTS.md`
- Reference: `goto-web/package.json`

- [ ] **Step 1: Check GitNexus index freshness before any symbol edit**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus status
```

Expected:

- index is fresh, or the tool explicitly says analysis is required

- [ ] **Step 2: If the index is stale, refresh it before continuing**

Run when needed:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus analyze
```

Expected:

- GitNexus index refreshed successfully

- [ ] **Step 3: Confirm the test command shape once before writing new tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --help
```

Expected:

- verify whether `--testPathPattern` is supported in this workspace
- if not, swap subsequent commands to `npx jest --runInBand <path>`

## Task 1: Freeze the Responsive Sizing Contract in Tests

**Files:**
- Create: `goto-web/tests/unit/views/components/dialogSize.spec.js`
- Reference: `docs/superpowers/specs/2026-03-28-goto-web-goto-dialog-responsive-size-design.md`

- [ ] **Step 1: Read the approved spec and restate the sizing contract in the test file header**

Add a short comment block capturing the contract:

```js
/**
 * Contract:
 * - sm: 42vw, max-width 720px, max-height 58vh on desktop
 * - md: 60vw, max-width 1040px, max-height 72vh on desktop
 * - lg: 78vw, max-width 1440px, max-height 82vh on desktop
 * - viewport width < 768px => width 92vw, max-height 88vh
 * - viewport height < 700px => clamp max-height with calc(100vh - 32px)
 * - if centered placement would cross 16px top/bottom safety edge => safe-edge mode
 */
```

- [ ] **Step 2: Write the first failing tests for desktop `sm/md/lg`**

```js
const { resolveDialogLayout } = require('@/views/components/dialogSize')

describe('resolveDialogLayout', () => {
  test('returns desktop sm contract', () => {
    expect(resolveDialogLayout({
      size: 'sm',
      viewportWidth: 1440,
      viewportHeight: 900,
      centeredHeight: 480
    })).toMatchObject({
      width: '42vw',
      maxWidth: '720px',
      maxHeight: '58vh',
      mode: 'center'
    })
  })
})
```

- [ ] **Step 3: Run the focused test to verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogSize.spec.js
```

Expected:

- FAIL with module-not-found or missing `resolveDialogLayout`

- [ ] **Step 4: Add failing tests for small-screen collapse and low-height fallback**

```js
test('collapses all sizes to mobile-safe width below 768px', () => {
  expect(resolveDialogLayout({
    size: 'lg',
    viewportWidth: 767,
    viewportHeight: 900,
    centeredHeight: 500
  })).toMatchObject({
    width: '92vw',
    maxHeight: '88vh'
  })
})

test('applies low-height cap below 700px', () => {
  expect(resolveDialogLayout({
    size: 'md',
    viewportWidth: 1280,
    viewportHeight: 680,
    centeredHeight: 520
  }).maxHeight).toBe('min(72vh, calc(100vh - 32px))')
})
```

- [ ] **Step 5: Add failing tests for safe-edge mode**

```js
test('switches to safe-edge mode when centered box would overflow viewport edge', () => {
  expect(resolveDialogLayout({
    size: 'lg',
    viewportWidth: 1280,
    viewportHeight: 720,
    centeredHeight: 710
  })).toMatchObject({
    mode: 'safe-edge',
    topInset: '16px',
    bottomInset: '16px',
    maxHeight: 'calc(100vh - 32px)'
  })
})
```

- [ ] **Step 6: Do not commit red tests; keep the branch dirty until the resolver is green**

Reason:

- this repo may require green commits
- TDD still applies; the red state is local only

## Task 2: Implement the Shared Dialog Size Resolver

**Files:**
- Create: `goto-web/src/views/components/dialogSize.js`
- Test: `goto-web/tests/unit/views/components/dialogSize.spec.js`

- [ ] **Step 1: Create the minimal resolver to satisfy the tests**

```js
export function resolveDialogLayout({ size = 'md', viewportWidth, viewportHeight, centeredHeight }) {
  const desktopMap = {
    sm: { width: '42vw', maxWidth: '720px', maxHeight: '58vh' },
    md: { width: '60vw', maxWidth: '1040px', maxHeight: '72vh' },
    lg: { width: '78vw', maxWidth: '1440px', maxHeight: '82vh' }
  }

  const isSmallViewport = viewportWidth < 768
  const isLowHeightViewport = viewportHeight < 700
  const base = isSmallViewport
    ? { width: '92vw', maxWidth: '92vw', maxHeight: '88vh' }
    : desktopMap[size] || desktopMap.md

  const next = {
    ...base,
    mode: 'center',
    topInset: null,
    bottomInset: null
  }

  if (!isSmallViewport && isLowHeightViewport) {
    next.maxHeight = `min(${base.maxHeight}, calc(100vh - 32px))`
  }

  const centeredTop = Math.max((viewportHeight - centeredHeight) / 2, 0)
  const centeredBottom = centeredTop + centeredHeight
  if (centeredTop < 16 || centeredBottom > viewportHeight - 16) {
    next.mode = 'safe-edge'
    next.topInset = '16px'
    next.bottomInset = '16px'
    next.maxHeight = 'calc(100vh - 32px)'
  }

  return next
}
```

- [ ] **Step 2: Run the focused test suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogSize.spec.js
```

Expected:

- PASS for the new sizing resolver contract

- [ ] **Step 3: If edge cases fail, fix the resolver before touching Vue files**

Examples:

- unknown `size` falls back to `md`
- safe-edge mode should override previous `vh` cap
- small-screen mode should still expose deterministic `maxWidth`

- [ ] **Step 4: Commit the shared resolver**

Before commit, verify staged scope with GitNexus:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialogSize.js goto-web/tests/unit/views/components/dialogSize.spec.js
npx gitnexus detect-changes --scope staged
```

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "feat: add goto dialog responsive size resolver"
```

## Task 3: Integrate the Resolver into `Dialog`

**Files:**
- Modify: `goto-web/src/views/components/dialog.vue`
- Modify: `goto-web/src/views/components/dialogSize.js`
- Test: `goto-web/tests/unit/views/components/dialogSize.spec.js`

- [ ] **Step 1: Run GitNexus impact analysis before editing the `Dialog` symbol**

First identify the actual symbol names exposed by the `Dialog` component:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus query --query "goto-web Dialog component open close confirm cancel"
```

Then run upstream impact for each symbol that will be modified inside `goto-web/src/views/components/dialog.vue`, typically:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus impact --target "open" --direction upstream
npx gitnexus impact --target "confirm" --direction upstream
npx gitnexus impact --target "cancel" --direction upstream
```

If the GitNexus CLI shape differs in this repo, use the configured MCP-equivalent. Record:

- direct callers
- affected process/risk level
- any HIGH/CRITICAL warnings

If any impacted symbol returns HIGH or CRITICAL risk:

- stop editing immediately
- capture the blast radius in the implementation notes
- update the execution context before touching the symbol

- [ ] **Step 2: Add a failing integration-style test for `Dialog` source contract**

Create a source-level assertion in `goto-web/tests/unit/views/components/dialogSize.spec.js` or a new `goto-web/tests/unit/views/components/dialog.spec.js`:

```js
const fs = require('fs')
const path = require('path')

const source = fs.readFileSync(
  path.resolve(__dirname, '../../../../src/views/components/dialog.vue'),
  'utf8'
)

expect(source).toMatch(/size:\s*{\s*type:\s*String/)
expect(source).toMatch(/import\s*{\s*resolveDialogLayout\s*}\s*from\s*['"]\.\/dialogSize['"]/)
expect(source).not.toMatch(/width:\s*{\s*type:\s*Number/)
expect(source).not.toMatch(/height:\s*{\s*type:\s*Number/)
```

- [ ] **Step 3: Implement the minimal `Dialog` API migration**

Target shape:

```js
props: {
  size: {
    type: String,
    default: function() {
      return 'md'
    }
  },
  fullscreen: { ... }
}
```

And in the template:

```vue
<el-dialog
  :width="dialogWidth"
  :custom-class="dialogCustomClass"
>
```

Add Vue state/computed/methods to:

- read `window.innerWidth` / `window.innerHeight`
- call `resolveDialogLayout`
- apply `dialogWidth`, `dialogMaxHeight`, safe-edge class, and center class
- add/remove `resize` listeners in lifecycle hooks
- skip responsive logic when `fullscreen === true`

- [ ] **Step 4: Replace direct DOM height writes with style/class application**

Expected direction:

```js
applyDialogLayout() {
  const dialogEl = this.$refs.dialog && this.$refs.dialog.$refs.dialog
  if (!dialogEl || this.fullscreen) {
    return
  }
  const centeredHeight = dialogEl.getBoundingClientRect().height
  const layout = resolveDialogLayout(...)
  dialogEl.style.width = layout.width
  dialogEl.style.maxWidth = layout.maxWidth
  dialogEl.style.maxHeight = layout.maxHeight
  dialogEl.style.marginTop = layout.mode === 'safe-edge' ? '16px' : ''
}
```

Adjust the CSS so that:

- default mode uses flex centering
- safe-edge mode aligns within viewport with `16px` top/bottom margin
- `.el-dialog__body` continues to own scrolling
- resize handling is debounced or throttled enough to avoid layout thrash

- [ ] **Step 5: Run focused tests and lint on the shared component**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialog
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogSize.spec.js
npm run lint -- src/views/components/dialog.vue src/views/components/dialogSize.js
```

Expected:

- tests PASS
- lint PASS

- [ ] **Step 6: Commit the component migration**

Before commit, verify scope with GitNexus:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialog.vue goto-web/src/views/components/dialogSize.js goto-web/tests/unit/views/components/dialogSize.spec.js goto-web/tests/unit/views/components/dialog.spec.js
npx gitnexus detect-changes --scope staged
```

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "feat: add responsive size modes to goto dialog"
```

## Task 4: Migrate the Small and Medium Dialog Call Sites

**Files:**
- Modify: `goto-web/src/views/goto/index.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Test: `goto-web/tests/unit/views/goto/toolsBox/type/tool.spec.js`

- [ ] **Step 1: Run GitNexus impact for the components you will edit in this task**

Run upstream impact for the Vue component symbols corresponding to:

- `goto-web/src/views/goto/index.vue`
- `goto-web/src/views/goto/toolsBox/type/tool.vue`

Record direct callers and any HIGH/CRITICAL warnings before editing.

If any impacted symbol returns HIGH or CRITICAL risk:

- stop the task
- record the blast radius
- resolve the risk before changing the component

- [ ] **Step 2: Add failing source-level assertions for migrated call sites**

Extend `goto-web/tests/unit/views/goto/toolsBox/type/tool.spec.js` and add a new source-level test for `goto-web/src/views/goto/index.vue`:

```js
const fs = require('fs')
const path = require('path')

const source = fs.readFileSync(
  path.resolve(__dirname, '../../../../../../src/views/goto/toolsBox/type/tool.vue'),
  'utf8'
)

expect(source).toMatch(/<Dialog(?=[\s\S]*size="md")/)
expect(source).not.toMatch(/:width="800"/)
```

For `index.vue`:

```js
const indexSource = fs.readFileSync(
  path.resolve(__dirname, '../../../../../../src/views/goto/index.vue'),
  'utf8'
)

expect(indexSource).toMatch(/<Dialog(?=[\s\S]*size="sm")/)
expect(indexSource).not.toMatch(/:width="500"/)
```

- [ ] **Step 3: Migrate the call sites**

Required mapping:

- `goto-web/src/views/goto/index.vue` => `size="sm"`
- `goto-web/src/views/goto/toolsBox/type/tool.vue` `下载设置` => `size="md"`

Keep all existing business props and events.

- [ ] **Step 4: Run the focused test suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/goto/toolsBox/type/tool.spec.js
```

Expected:

- PASS

- [ ] **Step 5: Commit the small/medium migration**

Before commit, verify scope with GitNexus:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/goto/index.vue goto-web/src/views/goto/toolsBox/type/tool.vue goto-web/tests/unit/views/goto/toolsBox/type/tool.spec.js
npx gitnexus detect-changes --scope staged
```

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "refactor: migrate goto dialog sm and md call sites"
```

## Task 5: Migrate the Large Dialog Call Sites

**Files:**
- Modify: `goto-web/src/views/goto/components/chooseColumnMultipleContent.vue`
- Modify: `goto-web/src/views/goto/components/classifyItemChooseColumn.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Modify: `goto-web/src/views/goto/plot/type/fileMark/formatConvertDialog.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotTemplateDialog.vue`
- Modify: `goto-web/src/views/goto/plot/type/components/PlotHistoryDialog.vue`
- Modify: `goto-web/src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/comparesShow.vue`
- Modify: `goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`

- [ ] **Step 1: Run GitNexus impact for each large-dialog component symbol before editing**

Run upstream impact for the component symbols corresponding to:

- `chooseColumnMultipleContent.vue`
- `classifyItemChooseColumn.vue`
- `tool.vue`
- `formatConvertDialog.vue`
- `PlotTemplateDialog.vue`
- `PlotHistoryDialog.vue`
- `chooseHeaderInfo.vue`
- `comparesShow.vue`
- `classifyChooseHeaderInfo.vue`

If any symbol reports HIGH/CRITICAL risk, stop and record the blast radius before implementing the file changes.

- [ ] **Step 2: Add or extend source-level migration tests**

Create a new source-contract test file, for example:

`goto-web/tests/unit/views/goto/dialogUsage.spec.js`

```js
const fs = require('fs')
const path = require('path')

const cases = [
  ['src/views/goto/components/chooseColumnMultipleContent.vue', 'size="lg"'],
  ['src/views/goto/components/classifyItemChooseColumn.vue', 'size="lg"'],
  ['src/views/goto/plot/type/fileMark/formatConvertDialog.vue', 'size="lg"'],
  ['src/views/goto/plot/type/components/PlotTemplateDialog.vue', 'size="lg"'],
  ['src/views/goto/plot/type/components/PlotHistoryDialog.vue', 'size="lg"'],
  ['src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue', 'size="lg"'],
  ['src/views/goto/components/form/draw/comparesShow.vue', 'size="lg"'],
  ['src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue', 'size="lg"']
]
```

Assert each file:

- contains `<Dialog` with `size="lg"`
- no longer contains `:width=` or `:height=`

- [ ] **Step 3: Migrate the table/grid/history dialogs to `size="lg"`**

Required mapping:

- `chooseColumnMultipleContent.vue` => `lg`
- `classifyItemChooseColumn.vue` => `lg`
- `toolsBox/type/tool.vue` `选择分析结果` => `lg`
- `formatConvertDialog.vue` => `lg`
- `PlotTemplateDialog.vue` => `lg`
- `PlotHistoryDialog.vue` => `lg`
- `chooseHeaderInfo.vue` => `lg`
- `comparesShow.vue` => `lg`
- `classifyChooseHeaderInfo.vue` => `lg`

Delete old props like:

```vue
:width="800"
:height="600"
:width="width"
:height="height"
```

- [ ] **Step 4: Remove dead local state only if it becomes unused**

Examples to check after migration:

- `formatConvertDialog.vue` local `width` / `height`
- `chooseHeaderInfo.vue` props `dialogWidth` / `dialogHeight` or derived local `width` / `height`
- `classifyChooseHeaderInfo.vue` props `dialogWidth` / `dialogHeight` or derived local `width` / `height`

Keep the change minimal:

- remove only unused size plumbing
- do not refactor unrelated business logic

- [ ] **Step 5: Run the focused migration tests and lint**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/goto/dialogUsage.spec.js
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/goto/toolsBox/type/tool.spec.js
npm run lint -- src/views/goto/index.vue src/views/goto/components/chooseColumnMultipleContent.vue src/views/goto/components/classifyItemChooseColumn.vue src/views/goto/toolsBox/type/tool.vue src/views/goto/plot/type/fileMark/formatConvertDialog.vue src/views/goto/plot/type/components/PlotTemplateDialog.vue src/views/goto/plot/type/components/PlotHistoryDialog.vue src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue src/views/goto/components/form/draw/comparesShow.vue src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue
```

Expected:

- migration tests PASS
- existing `tool.spec.js` stays PASS
- lint PASS

- [ ] **Step 6: Commit the large-dialog migration**

Before commit, verify scope with GitNexus:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/goto/index.vue goto-web/src/views/goto/components/chooseColumnMultipleContent.vue goto-web/src/views/goto/components/classifyItemChooseColumn.vue goto-web/src/views/goto/toolsBox/type/tool.vue goto-web/src/views/goto/plot/type/fileMark/formatConvertDialog.vue goto-web/src/views/goto/plot/type/components/PlotTemplateDialog.vue goto-web/src/views/goto/plot/type/components/PlotHistoryDialog.vue goto-web/src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue goto-web/src/views/goto/components/form/draw/comparesShow.vue goto-web/src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue goto-web/tests/unit/views/goto/dialogUsage.spec.js goto-web/tests/unit/views/goto/toolsBox/type/tool.spec.js
npx gitnexus detect-changes --scope staged
```

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "refactor: migrate goto dialog lg call sites"
```

## Task 6: End-to-End Verification and Change Scope Review

**Files:**
- Modify: `goto-web/src/views/components/dialog.vue`
- Modify: migrated `goto-web/src/views/goto/**/*.vue` files
- Test: `goto-web/tests/unit/views/components/dialogSize.spec.js`
- Test: `goto-web/tests/unit/views/components/dialog.spec.js`
- Test: `goto-web/tests/unit/views/goto/dialogUsage.spec.js`
- Reference: `docs/superpowers/specs/2026-03-28-goto-web-goto-dialog-responsive-size-design.md`

- [ ] **Step 1: Run the complete focused verification pack**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialog
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogSize.spec.js
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/goto/dialogUsage.spec.js
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/goto/toolsBox/type/tool.spec.js
npm run lint -- src/views/components/dialog.vue src/views/components/dialogSize.js src/views/goto
```

Expected:

- all targeted tests PASS
- lint PASS

- [ ] **Step 2: Perform manual viewport verification**

Open the affected pages and verify:

- `sm`: `goto/index.vue` logout confirm stays centered and compact
- `md`: `toolsBox/type/tool.vue` `下载设置` remains readable and not oversized
- `lg`: history/template/choose-header/compare dialogs keep footer visible and body scrolls
- viewport width `<768px` collapses to `92vw`
- low height / resized desktop window triggers safe-edge mode with `16px` insets

- [ ] **Step 3: Run GitNexus change detection before final commit/hand-off**

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus detect-changes --scope all
```

If the repository exposes GitNexus only through MCP tools, run the equivalent `detect_changes` call and confirm:

- changed symbols stay within the planned component and call sites
- no unexpected execution flows are touched

- [ ] **Step 4: Capture final scope summary**

Document in the handoff:

- which files moved to `sm`
- which files moved to `md`
- which files moved to `lg`
- whether any direct `el-dialog` remained untouched by design
- whether any content-layout regressions were found during manual checks

- [ ] **Step 5: Commit the final verification if any files changed**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialog.vue goto-web/src/views/components/dialogSize.js goto-web/src/views/goto goto-web/tests/unit/views/components/dialogSize.spec.js goto-web/tests/unit/views/components/dialog.spec.js goto-web/tests/unit/views/goto/dialogUsage.spec.js goto-web/tests/unit/views/goto/toolsBox/type/tool.spec.js
npx gitnexus detect-changes --scope staged
git commit -m "test: verify goto dialog responsive size migration"
```

## Notes for the Implementer

- Do not expand scope to direct `el-dialog` users such as `goto/user/center/updatePass.vue` and `goto/user/center/updateEmail.vue`
- Do not preserve `width/height` as long-term public props on `Dialog`
- Keep migration semantic: choose `size` by usage, not by copying old pixels
- Prefer source-contract tests and pure-function tests over brittle DOM-heavy modal tests
- If `dialog.vue` needs a helper CSS class for safe-edge mode, keep it local to the component and do not invent a project-wide dialog design system in this task
