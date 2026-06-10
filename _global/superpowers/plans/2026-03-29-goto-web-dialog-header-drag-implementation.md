# goto-web Dialog Header Drag Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为统一 `Dialog` 组件增加“仅标题栏空白区域可拖拽”的能力，并确保关闭、重开、`resize`、内容变化后都能回到现有自动布局。

**Architecture:** 继续以 [dialog.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/components/dialog.vue) 的自动布局为基础层，把拖拽实现为叠加的 `transform` 偏移层；所有拖拽状态、命中判断、边界裁剪和回退规则都集中在组件内部，不改业务调用点。测试分成 source-contract 约束和围绕组件内部拖拽逻辑的约束，避免写脆弱的 DOM 交互大集成测试。

**Tech Stack:** Vue 2, Element UI 2, Jest, Vue CLI unit tests, ESLint, GitNexus

---

## File Structure

### Shared Dialog Runtime

- Modify: `goto-web/src/views/components/dialog.vue`
  - 增加拖拽状态、事件绑定、命中判断、边界裁剪、偏移应用、回退清理

### Tests

- Modify: `goto-web/tests/unit/views/components/dialog.spec.js`
  - 锁定拖拽 source contract：标题栏空白区触发、全局事件绑定、拖拽偏移应用、回退清理
- Create: `goto-web/tests/unit/views/components/dialogDrag.spec.js`
  - 为命中判断、拖拽阈值、边界裁剪、回退条件写纯逻辑测试

### References

- Reference: `docs/superpowers/specs/2026-03-29-goto-web-dialog-header-drag-design.md`
- Reference: repo-root `AGENTS.md`

## Task 0: Prepare GitNexus and Verification Shape

**Files:**
- Reference: `/mnt/f/IdeaProjects/goto-software/AGENTS.md`
- Reference: `goto-web/package.json`

- [ ] **Step 1: Check GitNexus index freshness**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus status
```

Expected:

- index is up-to-date, or you learn that `analyze` is required first

- [ ] **Step 2: Refresh index only if stale**

Run when needed:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus analyze
```

Expected:

- index refresh succeeds

- [ ] **Step 3: Confirm focused test command shape**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --help
```

Expected:

- confirm `--testPathPattern` is usable in this workspace

## Task 1: Freeze Drag Contract in Tests

**Files:**
- Modify: `goto-web/tests/unit/views/components/dialog.spec.js`
- Create: `goto-web/tests/unit/views/components/dialogDrag.spec.js`
- Reference: `docs/superpowers/specs/2026-03-29-goto-web-dialog-header-drag-design.md`

- [ ] **Step 1: Add failing source-contract assertions for dialog drag plumbing**

Extend `dialog.spec.js` with checks like:

```js
expect(source).toMatch(/dragging:\s*false/)
expect(source).toMatch(/dragOffsetX:\s*0/)
expect(source).toMatch(/dragOffsetY:\s*0/)
expect(source).toMatch(/document\.addEventListener\(\s*["']mousemove["']/)
expect(source).toMatch(/document\.addEventListener\(\s*["']mouseup["']/)
expect(source).toMatch(/document\.removeEventListener\(\s*["']mousemove["']/)
expect(source).toMatch(/document\.removeEventListener\(\s*["']mouseup["']/)
expect(source).toMatch(/transform:\s*translate/)
```

- [ ] **Step 2: Add failing pure-logic tests for drag rules**

Create `dialogDrag.spec.js` with explicit contract tests for:

- 空白区命中判定
- 命中 `.el-dialog__title` / `.el-dialog__headerbtn` / `button` / `input` / `a` / `[data-no-drag]` 时不启动
- `4px` 拖拽阈值
- 左 `24px` / 顶 `48px` / 底 `24px` 可见边界
- 关闭/重开/resize/内容变化/safe-edge 时重置偏移

Example skeleton:

```js
const fs = require('fs')
const path = require('path')
const dialogPath = path.resolve(__dirname, '../../../../src/views/components/dialog.vue')
const source = fs.readFileSync(dialogPath, 'utf8')

test('dialog drag logic declares threshold and boundary constants', () => {
  expect(source).toMatch(/4/)
  expect(source).toMatch(/24/)
  expect(source).toMatch(/48/)
  expect(source).toMatch(/viewportWidth/)
})
```

- [ ] **Step 3: Run tests to verify RED state**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialog.spec.js
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogDrag.spec.js
```

Expected:

- FAIL for missing drag helpers / missing source-contract implementation

- [ ] **Step 4: Keep red tests uncommitted**

Do not commit until the implementation is green.

## Task 2: Implement Drag Rule Methods Inside Dialog

**Files:**
- Modify: `goto-web/src/views/components/dialog.vue`
- Test: `goto-web/tests/unit/views/components/dialogDrag.spec.js`

- [ ] **Step 1: Run GitNexus impact for the symbol(s) you will edit**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus impact "dialog.vue" --repo goto-web --direction upstream
```

If the CLI can identify actual exported symbols, prefer those exact targets instead of the file name.

If any result is HIGH or CRITICAL:

- stop implementation
- record the blast radius
- resolve the risk before editing

- [ ] **Step 2: Add minimal drag rule methods inside `dialog.vue` to satisfy the tests**

Expected direction:

```js
const DIALOG_DRAG_THRESHOLD = 4
const DIALOG_DRAG_VISIBLE_LEFT = 24
const DIALOG_DRAG_VISIBLE_TOP = 48
const DIALOG_DRAG_VISIBLE_BOTTOM = 24

shouldStartDialogDrag({ deltaX, deltaY }) {
  return Math.hypot(deltaX, deltaY) > DIALOG_DRAG_THRESHOLD
}

isDialogHeaderDragAllowed(target) {
  // reject title, close button, interactive elements, [data-no-drag]
}

clampDialogDragOffset({ nextOffsetX, nextOffsetY, dialogRect, viewportWidth, viewportHeight }) {
  // keep 24px left visible, right edge within viewport,
  // 48px top header visible, 24px bottom visible
}
```

- [ ] **Step 3: Run the focused helper test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogDrag.spec.js
```

Expected:

- PASS

- [ ] **Step 4: Verify staged scope before commit**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialog.vue goto-web/tests/unit/views/components/dialogDrag.spec.js
npx gitnexus help detect-changes
```

If `detect-changes` exists, run it on staged scope. If not, record that the installed CLI lacks the command.

- [ ] **Step 5: Commit the helper layer**

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "feat: add dialog drag rule methods"
```

## Task 3: Complete Drag Runtime Integration and Cleanup

**Files:**
- Modify: `goto-web/src/views/components/dialog.vue`
- Modify: `goto-web/tests/unit/views/components/dialog.spec.js`
- Test: `goto-web/tests/unit/views/components/dialogDrag.spec.js`

- [ ] **Step 1: Run GitNexus impact before editing dialog runtime**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus impact "dialog.vue" --repo goto-web --direction upstream
```

If the CLI supports symbol-level targets for this Vue file, use them.

If any result is HIGH or CRITICAL:

- stop implementation
- record the blast radius
- resolve the risk before editing

- [ ] **Step 2: Add drag state, bound listener refs, and reset plumbing**

Target state shape:

```js
data() {
  return {
    dragging: false,
    dragReady: false,
    dragOffsetX: 0,
    dragOffsetY: 0,
    dragStartX: 0,
    dragStartY: 0,
    dragOriginX: 0,
    dragOriginY: 0,
    boundHandleDialogDragMove: null,
    boundHandleDialogDragEnd: null
  }
}
```

Also add a single reset method:

```js
resetDialogDragState() {
  this.dragging = false
  this.dragReady = false
  this.dragOffsetX = 0
  this.dragOffsetY = 0
  this.dragStartX = 0
  this.dragStartY = 0
  this.dragOriginX = 0
  this.dragOriginY = 0
}
```

- [ ] **Step 3: Add header drag start logic**

Expected direction:

```js
handleDialogHeaderMouseDown(event) {
  if (this.fullscreen || !this.isDialogVisible) {
    return
  }
  if (!this.isDialogHeaderDragAllowed(event.target)) {
    return
  }
  this.dragStartX = event.clientX
  this.dragStartY = event.clientY
  this.dragOriginX = this.dragOffsetX
  this.dragOriginY = this.dragOffsetY
  this.dragReady = true
  this.boundHandleDialogDragMove = this.boundHandleDialogDragMove || this.handleDialogDragMove.bind(this)
  this.boundHandleDialogDragEnd = this.boundHandleDialogDragEnd || this.handleDialogDragEnd.bind(this)
  document.addEventListener('mousemove', this.boundHandleDialogDragMove)
  document.addEventListener('mouseup', this.boundHandleDialogDragEnd)
}
```

Bind this only to the header region after the dialog DOM is available.

- [ ] **Step 4: Add move/end logic and apply transform**

Expected direction:

```js
handleDialogDragMove(event) {
  const deltaX = event.clientX - this.dragStartX
  const deltaY = event.clientY - this.dragStartY
  if (!this.dragging && !this.shouldStartDialogDrag({ deltaX, deltaY })) {
    return
  }
  this.dragging = true
  const dialogRect = this.getDialogElements().dialogEl.getBoundingClientRect()
  const nextOffset = this.clampDialogDragOffset(...)
  this.dragOffsetX = nextOffset.x
  this.dragOffsetY = nextOffset.y
  this.applyDialogDragTransform()
}

handleDialogDragEnd() {
  this.dragging = false
  this.dragReady = false
  document.removeEventListener('mousemove', this.boundHandleDialogDragMove)
  document.removeEventListener('mouseup', this.boundHandleDialogDragEnd)
}
```

`applyDialogDragTransform()` should set:

```js
if (this.dragOffsetX === 0 && this.dragOffsetY === 0) {
  dialogEl.style.transform = ''
  return
}
dialogEl.style.transform = `translate(${this.dragOffsetX}px, ${this.dragOffsetY}px)`
```

- [ ] **Step 5: Reset drag state on all required fallback paths**

Must reset on:

- close
- open
- resize
- content height change
- transition into `safe-edge`

Implementation guidance:

- call `resetDialogDragState()` before reapplying automatic layout
- clear `dialogEl.style.transform` when drag is reset
- clamp must include right-edge handling so the dialog right side cannot exceed `viewportWidth`
- detect `safe-edge` transition by comparing previous mode and next mode inside `applyDialogLayout()`
- if `previousMode !== 'safe-edge' && nextMode === 'safe-edge'`, clear drag state before applying the final layout
- add one dedicated cleanup method for document listeners and call it from both `close()` and `beforeDestroy()`

- [ ] **Step 6: Run focused tests and ESLint**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialog.spec.js
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialogDrag.spec.js
npx eslint src/views/components/dialog.vue tests/unit/views/components/dialog.spec.js tests/unit/views/components/dialogDrag.spec.js
```

Expected:

- all tests PASS
- focused eslint PASS

- [ ] **Step 7: Verify staged scope before commit**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialog.vue goto-web/tests/unit/views/components/dialog.spec.js goto-web/tests/unit/views/components/dialogDrag.spec.js
npx gitnexus help detect-changes
```

If `detect-changes` exists, run it; otherwise record that it is unavailable in this CLI.

- [ ] **Step 8: Commit the dialog drag integration**

```bash
cd /mnt/f/IdeaProjects/goto-software
git commit -m "feat: add dialog header drag behavior"
```

## Task 4: Final Verification and Manual Runtime Check

**Files:**
- Modify: `goto-web/src/views/components/dialog.vue`
- Test: `goto-web/tests/unit/views/components/dialog.spec.js`
- Test: `goto-web/tests/unit/views/components/dialogDrag.spec.js`

- [ ] **Step 1: Run the complete focused verification pack**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx vue-cli-service test:unit --runInBand --testPathPattern=tests/unit/views/components/dialog
npx eslint src/views/components/dialog.vue tests/unit/views/components/dialog.spec.js tests/unit/views/components/dialogDrag.spec.js
```

Expected:

- tests PASS
- focused eslint PASS

- [ ] **Step 2: Perform manual browser verification**

Verify in a real browser:

- 默认打开时仍然自动居中
- 按住标题栏空白区域可拖拽
- 标题文字不可拖
- 关闭按钮不可拖
- 轻微抖动小于 `4px` 不进入拖拽
- 拖拽后释放，当前位置在当前打开周期内保持
- 关闭再打开恢复默认居中
- `resize` 后恢复自动布局
- 内容异步变化后恢复自动布局
- 超长内容进入 `safe-edge` 时不会保留旧拖拽偏移

- [ ] **Step 3: Capture GitNexus status and scope evidence**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
npx gitnexus status
npx gitnexus help detect-changes
git status --short
```

If `detect-changes` still does not exist, record that limitation in the handoff.

- [ ] **Step 4: Commit final verification only if any verification files changed**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/components/dialog.vue goto-web/tests/unit/views/components/dialog.spec.js goto-web/tests/unit/views/components/dialogDrag.spec.js
git commit -m "test: verify dialog header drag behavior"
```

## Notes for the Implementer

- Do not expand scope to touch `goto` business call sites
- Keep drag behavior entirely inside the shared `Dialog`
- Do not convert dialog positioning to `left/top`
- Automatic layout remains authoritative; drag offset is temporary overlay state
- If a runtime browser check shows the header selector is too broad, tighten the hit-test logic instead of widening drag scope
