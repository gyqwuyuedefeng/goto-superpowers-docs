# goto-web Draw Form Label Color Unification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Unify title-like label colors in `goto-web/src/views/goto/components/form/draw` so dark theme uses pure white labels while light theme remains readable, without changing input values, placeholders, or dropdown item colors.

**Architecture:** Keep the change inside the existing draw theme pipeline. Define one new semantic token in draw theme variables, route draw-scoped label selectors to that token through `foundation.scss` mixins, and only add component-local scoped overrides where Vue SFC scoped selectors beat the shared rules. Use the existing `.draw-theme-scope` namespace as the only global style anchor.

**Tech Stack:** Vue 2 SFCs, SCSS, Element UI, draw theme mixins, GitNexus impact analysis, Jest for existing JS unit tests where applicable

---

## File Map

### Primary files

- Modify: `goto-web/src/assets/styles/draw/_variables.scss`
  Responsibility: define `--draw-form-label-color` light/default and dark mappings in the authoritative draw token source.

- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`
  Responsibility: route draw-scoped base label selectors to `--draw-form-label-color` without coloring non-label descendants.

- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`
  Responsibility: unify draw-wide collapse header text and arrow color through the shared token inside the existing draw collapse styling block.

- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`
  Responsibility: replace modern-lab label gradient text with explicit semantic label color and WebKit-safe text fill behavior.

### Possible component-level补漏 files

- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/textParam.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/input.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/classifyItem.vue`

Responsibility: add the smallest scoped override only if shared draw rules lose to local scoped styles.

### Verification files

- Check existing draw style host: `goto-web/src/assets/styles/draw/foundation.scss`
- Check theme source: `goto-web/src/utils/theme.js`
- Run pre-commit impact/scope check via GitNexus before commit

## Task 1: Confirm symbol impact before style edits

**Files:**
- Inspect: `goto-web/src/assets/styles/draw/_variables.scss`
- Inspect: `goto-web/src/assets/styles/draw/_form-base.scss`
- Inspect: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`

- [ ] **Step 1: Read the exact mixin and token names before running GitNexus**

Run:

```bash
sed -n '1,220p' goto-web/src/assets/styles/draw/_variables.scss
sed -n '1,220p' goto-web/src/assets/styles/draw/_form-base.scss
sed -n '340,430p' goto-web/src/assets/styles/draw/_modern-lab-theme.scss
```

Expected: confirm the actual exported mixin/symbol names present in the files before using them in GitNexus.

- [ ] **Step 2: Run GitNexus impact analysis for each edited style unit**

Run the required impact checks before editing any symbol-like style unit or mixin:

```bash
# If the repo MCP warns the index is stale, run:
npx gitnexus analyze
```

Use GitNexus:

```text
gitnexus_query({query: "GTextParam draw label color"})
gitnexus_query({query: "GLegendSettingsParam GTextParam"})
gitnexus_query({query: "drawGroupOptional GTextParam"})
gitnexus_context({name: "<confirmed indexed Vue symbol>"})
gitnexus_impact({target: "<confirmed indexed Vue symbol>", direction: "upstream"})
```

Execution note:

- If GitNexus MCP tools are available in the agent environment, use them directly.
- If the MCP tools are unavailable in the current execution environment, stop and surface that blocker instead of silently skipping the required impact step.
- If GitNexus reports the index is stale, run `npx gitnexus analyze` from the repo root before retrying the impact checks.
- Do not guess the `target` name. First use `gitnexus_query` or `gitnexus_context` to identify the exact indexed symbol name GitNexus expects.
- For this style-oriented change, prefer impact analysis on indexed Vue consumer symbols in the affected execution flow, such as `GTextParam`, `GLegendSettingsParam`, `drawGroupOptional`, and any Vue component actually edited in Task 6.
- If a specific SCSS mixin is not indexed as a symbol, do not dead-end the task there; instead, run impact on the nearest indexed Vue consumer symbols that exercise that style path, then use `gitnexus_detect_changes()` before each commit to confirm the scope stays constrained.

Expected: identify direct callers/consumers and confirm the blast radius stays inside existing draw theme consumers. If any result is HIGH or CRITICAL, stop and surface it before editing.

- [ ] **Step 3: Read exact current implementations**

Run:

```bash
sed -n '1,220p' goto-web/src/assets/styles/draw/_variables.scss
sed -n '1,220p' goto-web/src/assets/styles/draw/_form-base.scss
sed -n '340,430p' goto-web/src/assets/styles/draw/_modern-lab-theme.scss
sed -n '1,120p' goto-web/src/assets/styles/draw/foundation.scss
```

Expected: confirm `_variables.scss` is the only authoritative variable emitter, `foundation.scss` keeps draw rules inside `.draw-theme-scope`, and modern-lab label gradient properties match the spec.

- [ ] **Step 4: Commit nothing yet**

Expected: no file changes yet. This task is only for blast-radius confirmation.

## Task 2: Add the semantic label color token

**Files:**
- Modify: `goto-web/src/assets/styles/draw/_variables.scss`

- [ ] **Step 1: Write a failing visual expectation note**

Record the expected behavior in the work log or commit notes before editing:

```text
Dark theme expected: draw title labels use #fff via --draw-form-label-color.
Light theme expected: draw title labels stay readable via var(--draw-text-primary).
No change expected for input values, placeholders, or dropdown items.
```

Expected: a clear success target exists before implementation.

- [ ] **Step 2: Add the new token in the authoritative token source**

Implement in `goto-web/src/assets/styles/draw/_variables.scss` by editing only the existing authoritative theme-emission blocks and adding only the new token:

```scss
:root,
html[data-theme='light'] {
  /* keep all existing declarations unchanged */
  --draw-form-label-color: var(--draw-text-primary);
}

html[data-theme='dark'] {
  /* keep all existing declarations unchanged */
  --draw-form-label-color: #fff;
}
```

Before editing, verify whether the active shipped path is:

- the legacy emission block inside `_variables.scss`
- or another path that imports `_variables.scss` and emits the same theme blocks into the shipped bundle

Implementation rule:

- Do not assume specific mixin names or flags beyond what is present in the file.
- Add `--draw-form-label-color` inside the existing authoritative `:root, html[data-theme='light']` and `html[data-theme='dark']` emission path that is already shipping today.
- Do not create a second competing definition site for the same variable.

Expected: default/light mapping stays readable; dark mapping is pure white; dark block remains after the default/light block; the new variable is added in the path that actually ships.

- [ ] **Step 3: Verify the token definition**

Run:

```bash
rg -n -- "--draw-form-label-color" goto-web/src/assets/styles/draw
```

Expected: only `_variables.scss` defines the variable value; other files only consume it.

- [ ] **Step 4: Commit the token change**

Use GitNexus before committing:

```text
gitnexus_detect_changes({scope: "all"})
```

```bash
git add goto-web/src/assets/styles/draw/_variables.scss
git commit -m "feat: add draw form label color token"
```

## Task 3: Route shared draw label selectors to the token

**Files:**
- Modify: `goto-web/src/assets/styles/draw/_form-base.scss`

- [ ] **Step 1: Update only title nodes, not wrappers as a whole**

Implement the shared selector changes as minimal edits to the existing selectors only:

```scss
.form-label {
  color: var(--draw-form-label-color);
}

.form-label-wrapper .form-label {
  color: var(--draw-form-label-color);
}
```

Expected: only `.form-label` text is recolored; `SyncIcon` and other wrapper children are not unintentionally turned white.

Placement rule:

- Keep these selectors inside the existing `draw-form-base` mixin so they are emitted through the current `foundation.scss` pattern:

```scss
.draw-theme-scope {
  @include draw-form-base;
}
```

- Do not emit bare global `.form-label` or `.form-label-wrapper` selectors outside the draw namespace.
- Do not introduce a second scoping mechanism such as `.draw-theme-scope &` inside the mixin.
- Do not rewrite unrelated layout declarations such as `.form-group`, spacing, sizing, or flex rules in `_form-base.scss`.

- [ ] **Step 2: Verify no wrapper-wide color remains**

Run:

```bash
rg -n "form-label-wrapper|draw-form-label-color" goto-web/src/assets/styles/draw/_form-base.scss
```

Expected: wrapper layout remains, but color is attached to `.form-label`, not the whole wrapper.

- [ ] **Step 3: Commit the shared base selector change**

Use GitNexus before committing:

```text
gitnexus_detect_changes({scope: "all"})
```

```bash
git add goto-web/src/assets/styles/draw/_form-base.scss
git commit -m "feat: route draw base labels to semantic token"
```

## Task 4: Replace modern-lab gradient labels with explicit color

**Files:**
- Modify: `goto-web/src/assets/styles/draw/_modern-lab-theme.scss`

- [ ] **Step 1: Remove gradient behavior and set explicit semantic color**

Replace the modern-lab `.form-label` rule with WebKit-safe explicit coloring:

```scss
.modern-lab-form-group {
  .form-label {
    color: var(--draw-form-label-color);
    font-size: 13px;
    font-weight: 500;
    margin-bottom: 6px;
    letter-spacing: 0.01em;
    background: none;
    background-image: none;
    -webkit-background-clip: border-box;
    background-clip: border-box;
    -webkit-text-fill-color: currentColor;
  }
}
```

Expected: no residual gradient, no transparent text in WebKit, dark theme labels render pure white.

- [ ] **Step 2: Verify old gradient properties are gone**

Run:

```bash
rg -n "background-clip|text-fill|linear-gradient" goto-web/src/assets/styles/draw/_modern-lab-theme.scss
```

Expected: the old gradient text block is removed or replaced with the explicit rule above.

- [ ] **Step 3: Commit the modern-lab cleanup**

Use GitNexus before committing:

```text
gitnexus_detect_changes({scope: "all"})
```

```bash
git add goto-web/src/assets/styles/draw/_modern-lab-theme.scss
git commit -m "feat: unify modern lab label colors"
```

## Task 5: Unify draw collapse header text and arrow color

**Files:**
- Modify: `goto-web/src/assets/styles/draw/_form-components.scss`

- [ ] **Step 1: Add explicit shared collapse header selectors**

Update the draw collapse section so both title text and arrow consume the semantic token:

```scss
.el-collapse-item__header {
  color: var(--draw-form-label-color);
}

.el-collapse-item__header .el-collapse-item__arrow,
.el-collapse-item__header .el-icon-arrow-right {
  color: var(--draw-form-label-color);
}
```

Expected: `textParam.vue` “其他设置” and similar draw collapse headers use the same label color as neighboring title labels, including the arrow icon.

Placement rule:

- Add these selectors at the existing draw collapse block in `_form-components.scss`, which is already emitted under `.draw-theme-scope` by `foundation.scss`.
- Do not add unanchored global `.el-collapse-item__header` rules.
- Do not narrow the selector with a component-specific parent such as `.list-param-item` unless manual verification proves a shared draw-wide selector still loses.

- [ ] **Step 2: Verify there are no competing hard-coded draw collapse colors**

Run:

```bash
rg -n "el-collapse-item__header|el-collapse-item__arrow|el-icon-arrow-right|color:" goto-web/src/assets/styles/draw goto-web/src/views/goto/components/form/draw
```

Expected: any remaining conflicting gray arrow/title colors are identified for cleanup or scoped补漏.

- [ ] **Step 3: Commit the shared collapse header change**

Use GitNexus before committing:

```text
gitnexus_detect_changes({scope: "all"})
```

```bash
git add goto-web/src/assets/styles/draw/_form-components.scss
git commit -m "feat: unify draw collapse header colors"
```

## Task 6: Add only the minimum scoped overrides if shared rules lose

**Files:**
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/textParam.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectObject.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/input.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectSimple.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue`
- Verify and modify only if needed: `goto-web/src/views/goto/components/form/draw/classifyItem.vue`

- [ ] **Step 1: Inspect candidate scoped components**

Run:

```bash
sed -n '1,240p' goto-web/src/views/goto/components/form/draw/textParam.vue
sed -n '1,320p' goto-web/src/views/goto/components/form/draw/selectObject.vue
sed -n '150,260p' goto-web/src/views/goto/components/form/draw/input.vue
sed -n '1,240p' goto-web/src/views/goto/components/form/draw/selectSimple.vue
sed -n '1,260p' goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue
sed -n '620,700p' goto-web/src/views/goto/components/form/draw/classifyItem.vue
```

Expected: identify only the components where scoped selectors still beat the shared `.draw-theme-scope` rules.

- [ ] **Step 2: Add the smallest possible scoped补漏 if necessary**

Before editing any Vue SFC in this task, run impact analysis on the actual component symbol or exported component name present in that file. If GitNexus does not index the component symbol, stop and surface the blocker instead of guessing compliance.

Suggested discovery flow for each Vue file:

```text
gitnexus_query({query: "textParam.vue GTextParam"})
gitnexus_query({query: "selectObject.vue component"})
gitnexus_query({query: "classifyItem.vue component"})
gitnexus_context({name: "<confirmed component symbol>"})
gitnexus_impact({target: "<confirmed component symbol>", direction: "upstream"})
```

Allowed pattern:

```scss
.component-root :deep(.form-label) {
  color: var(--draw-form-label-color);
}

.component-root :deep(.form-label-wrapper .form-label) {
  color: var(--draw-form-label-color);
}

.component-root :deep(.el-collapse-item__header),
.component-root :deep(.el-collapse-item__header .el-collapse-item__arrow) {
  color: var(--draw-form-label-color);
}
```

Shared draw global SCSS files such as `_form-components.scss`, `_form-base.scss`, and `_modern-lab-theme.scss` must not use `:deep(...)`. Those files should use normal selectors because they are compiled through `foundation.scss` under `.draw-theme-scope`, not through Vue SFC scoped CSS.

Not allowed:

```scss
color: #fff;
color: white;
```

Expected: all补漏 continue to consume the shared semantic token; no new literal colors are introduced.

- [ ] **Step 3: Commit only if any scoped overrides were required**

Use GitNexus before committing:

```text
gitnexus_detect_changes({scope: "all"})
```

```bash
git add goto-web/src/views/goto/components/form/draw/textParam.vue \
  goto-web/src/views/goto/components/form/draw/selectObject.vue \
  goto-web/src/views/goto/components/form/draw/input.vue \
  goto-web/src/views/goto/components/form/draw/selectSimple.vue \
  goto-web/src/views/goto/components/form/draw/selectMultipleObject.vue \
  goto-web/src/views/goto/components/form/draw/classifyItem.vue
git commit -m "fix: align draw scoped label colors"
```

If no component files changed, skip this commit.

## Task 7: Verify behavior in both themes

**Files:**
- Verify host views that mount `.draw-theme-scope`

- [ ] **Step 1: Run available automated checks**

First confirm the repo-standard package manager and scripts from `goto-web/package.json` and the repo lockfile.

Run:

```bash
cat goto-web/package.json
ls goto-web/package-lock.json goto-web/pnpm-lock.yaml goto-web/yarn.lock
```

Expected: verify which package manager the repo actually uses and that `dev`, `lint`, and `test:unit` scripts exist before invoking them.

Then run:

```bash
cd goto-web
<repo package manager> run lint
<repo package manager> run test:unit -- --runInBand
```

Expected: lint passes and the repo-standard unit test entrypoint still passes. If the script contract does not support extra file arguments, do not append them.

- [ ] **Step 2: Run a targeted static regression scan**

Run:

```bash
rg -n -- "#fff|white" goto-web/src/assets/styles/draw goto-web/src/views/goto/components/form/draw
rg -n -- "--draw-form-label-color" goto-web/src/assets/styles/draw goto-web/src/views/goto/components/form/draw
```

Expected: the new semantic variable is used where needed; no scattered literal white values were added to component code.

- [ ] **Step 3: Manually verify in the UI**

Check in the browser:

1. From `goto-web`, run `npm run dev`.
   If `package.json` or the lockfile shows a different package manager, use the repo-standard equivalent command instead.
2. Open the app route that resolves to `goto-web/src/views/goto/plot/type/draw.vue`, which is registered in `goto-web/src/router/routers.js` as `/goto/plot/draw/:groupId/:typeId`.
3. Navigate to a view that renders `drawGroupOptional.vue` or `legendSettingsParam.vue`, because those are known hosts of `textParam.vue`.
4. Confirm the host root includes `.draw-theme-scope`.
5. Use the UI theme switcher from `goto-web/src/views/goto/components/ThemeToggle.vue` or manually set the theme through the browser console:

```js
localStorage.setItem('goto-theme', 'dark')
document.documentElement.setAttribute('data-theme', 'dark')
```

6. Refresh if needed, then inspect `textParam.vue`.
7. Confirm the labels “字体粗细”“字体颜色”“字体大小”“其他设置” are visually unified and pure white.
8. Confirm collapse arrow color matches the header text.
9. Switch back to light theme using the toggle or:

```js
localStorage.setItem('goto-theme', 'light')
document.documentElement.setAttribute('data-theme', 'light')
```

10. Refresh if needed.
11. Confirm labels remain readable and are not white on light backgrounds.
12. Confirm input values, placeholders, and dropdown items did not change color semantics.

Expected: passes all twelve checks.

- [ ] **Step 4: Commit verification-safe cleanup if needed**

If verification required tiny selector cleanup after manual testing:

```bash
git add goto-web/src/assets/styles/draw/_variables.scss \
  goto-web/src/assets/styles/draw/_form-base.scss \
  goto-web/src/assets/styles/draw/_modern-lab-theme.scss \
  goto-web/src/assets/styles/draw/_form-components.scss \
  goto-web/src/views/goto/components/form/draw
git commit -m "fix: polish draw label color unification"
```

## Task 8: Final scope verification and handoff

**Files:**
- Verify changed files only

- [ ] **Step 1: Run GitNexus change detection before final commit/push**

Use GitNexus:

```text
gitnexus_detect_changes({scope: "all"})
```

Expected: affected files and flows stay limited to the planned draw styling surface.

- [ ] **Step 2: Inspect git diff summary**

At the start of implementation, record the starting commit:

```bash
git rev-parse HEAD
```

Then at the end compare from that recorded base SHA to current HEAD.

Run:

```bash
git status --short
git diff --stat <start_sha>..HEAD
```

Expected: only expected draw style files and minimal component overrides are present across the full implementation, not just the last commit.

- [ ] **Step 3: Prepare completion note**

Summarize:

- Which shared files changed
- Whether any component-level scoped补漏 were needed
- Which tests/verification commands ran
- Manual light/dark verification outcome

- [ ] **Step 4: Do not forget post-commit index freshness**

If implementation commits changed code and a later graph-based task depends on GitNexus:

```bash
npx gitnexus analyze
```

If embeddings already exist, preserve them:

```bash
npx gitnexus analyze --embeddings
```
