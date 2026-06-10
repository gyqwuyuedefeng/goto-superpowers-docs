# Module Type 7 by Mark Column Index Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `moduleType === 7` 分类参数记住 `classifyItem.by` 对应 `byFileMarkColumnList` 中的索引，`by` 为空时跳过重置，多标记列时精确定位不再错误 fallback。

**Architecture:** 变更集中在 `src/constant/paramSettingsReset.js`，新增 `computeByMarkColumnIndex` 和 `resolveByColumnName` helper，修改 `shouldProcessClassifyItem` 和 `resolveBoundMarkHeaderInfo`。`draw.vue` 首次加载调用计算，`resetClassifyItemByDefault` 清空 by 时同步清空索引。

**Tech Stack:** Vue 2 前端项目、CommonJS/Jest 单元测试、`@` alias 映射到 `src`、现有 `scripts/run-unit-tests.js` 测试启动器。

---

## File Structure

- Modify: `src/constant/paramSettingsReset.js`
  - 修改 `shouldProcessClassifyItem`：`by` 为空直接跳过
  - 注释保留 `hasValidConfiguration`
  - 新增 `resolveByColumnName`：columnName 全局匹配（无 fallback）
  - 修改 `resolveBoundMarkHeaderInfo`：用 `byMarkColumnIndex` 定位
  - 新增 `computeByMarkColumnIndex`：首次加载全局计算索引
- Modify: `src/constant/modify/publicFunction.js`
  - `resetClassifyItemByDefault`：清空 `by` 时同步 `byMarkColumnIndex = null`
- Modify: `src/views/goto/plot/type/draw.vue`
  - `obtainPlotData`：首次加载调用 `computeByMarkColumnIndex`
- Modify: `tests/unit/constant/paramSettingsReset.spec.js`
  - 新增多标记列索引定位、by 为空跳过 的测试用例

---

## Task 1: Add Failing Tests For by-empty Skip And Multi Mark Column Index

**Files:**
- Modify: `tests/unit/constant/paramSettingsReset.spec.js`

- [ ] **Step 1: Add by-empty skip test**

Append to `tests/unit/constant/paramSettingsReset.spec.js`:

```js
/**
 * 被测试对象：moduleType=7 shouldProcessClassifyItem。
 * 测试目的：验证 by 为空时直接跳过，不进入重置流程。
 */
test('by 为空时跳过重置', () => {
  const classifyItem = createManualClassifyItem({
    by: null,
    valueList: [{ name: null, value: 0, key: '' }]
  })
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('Group', ['A', 'B'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBeNull()
})

/**
 * 被测试对象：moduleType=7 shouldProcessClassifyItem。
 * 测试目的：验证 by 为空字符串时也跳过。
 */
test('by 为空字符串时跳过重置', () => {
  const classifyItem = createManualClassifyItem({
    by: '',
    valueList: [{ name: null, value: 0, key: '' }]
  })
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('Group', ['A', 'B'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('')
})
```

- [ ] **Step 2: Add multi mark column index test - matched**

Append:

```js
/**
 * 被测试对象：moduleType=7 resolveBoundMarkHeaderInfo 索引用定位。
 * 测试目的：多标记列时 byMarkColumnIndex=1 定位到第二个标记列，不 fallback 到第一个。
 */
test('多标记列索引用定位到第二个标记列', () => {
  const classifyItem = createManualClassifyItem({
    by: 'Type',
    byMarkColumnIndex: 1
  })
  const setting = createClassifySetting(classifyItem, [
    { filedName: 'group', fileSettingIndex: 0 },
    { filedName: 'type', fileSettingIndex: 0 }
  ])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [
        { field: 'group', type: 'SELECT_COLUMN', positionParam: 1 },
        { field: 'type', type: 'SELECT_COLUMN', positionParam: 2 }
      ],
      headerInfoList: [
        createHeader('Group', ['A', 'B'], 0),
        createHeader('Type', ['X', 'Y'], 2)
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Type')
  expect(classifyItem.valueList.map(v => v.name)).toEqual(['X', 'Y'])
})
```

- [ ] **Step 3: Add multi mark column index test - BINDING_MISSING**

Append:

```js
/**
 * 被测试对象：moduleType=7 resolveBoundMarkHeaderInfo 索引定位。
 * 测试目的：对应索引的标记列未标记时 BINDING_MISSING，保持 classifyItem 不变。
 */
test('索引对应标记列未标记时跳过', () => {
  const classifyItem = createManualClassifyItem({
    by: 'Type',
    byMarkColumnIndex: 1,
    valueList: [
      { name: 'Old_X', value: 10 }
    ]
  })
  const setting = createClassifySetting(classifyItem, [
    { filedName: 'group', fileSettingIndex: 0 },
    { filedName: 'type', fileSettingIndex: 0 }
  ])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [
        { field: 'group', type: 'SELECT_COLUMN', positionParam: 1 },
        { field: 'type', type: 'SELECT_COLUMN', positionParam: 2 }
      ],
      headerInfoList: [
        createHeader('Group', ['A', 'B'], 1),
        createHeader('Type', ['X', 'Y'], 0)
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Type')
  expect(classifyItem.valueList).toEqual([
    { name: 'Old_X', value: 10 }
  ])
})
```

- [ ] **Step 4: Add backward compat test - no byMarkColumnIndex**

Append:

```js
/**
 * 被测试对象：moduleType=7 resolveBoundMarkHeaderInfo 兼容旧数据。
 * 测试目的：byMarkColumnIndex 为 undefined 时回退 columnName 全局匹配。
 */
test('无 byMarkColumnIndex 时回退 columnName 匹配', () => {
  const classifyItem = createManualClassifyItem({
    by: 'Group'
  })
  const setting = createClassifySetting(classifyItem, [
    { filedName: 'group', fileSettingIndex: 0 }
  ])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [
        { field: 'group', type: 'SELECT_COLUMN', positionParam: 1 }
      ],
      headerInfoList: [
        createHeader('Group', ['A', 'B'], 1)
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.valueList.map(v => v.name)).toEqual(['A', 'B'])
})
```

- [ ] **Step 5: Add backward compat test - columnName not found**

Append:

```js
/**
 * 被测试对象：moduleType=7 resolveBoundMarkHeaderInfo 兼容旧数据。
 * 测试目的：无 byMarkColumnIndex 且 columnName 不匹配时 BINDING_MISSING，不动数据。
 */
test('无 byMarkColumnIndex 且列名不匹配时跳过', () => {
  const classifyItem = createManualClassifyItem({
    by: 'OldColumn',
    valueList: [
      { name: 'Old_X', value: 10 }
    ]
  })
  const setting = createClassifySetting(classifyItem, [
    { filedName: 'group', fileSettingIndex: 0 }
  ])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [
        { field: 'group', type: 'SELECT_COLUMN', positionParam: 1 }
      ],
      headerInfoList: [
        createHeader('NewColumn', ['A', 'B'], 1)
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('OldColumn')
  expect(classifyItem.valueList).toEqual([
    { name: 'Old_X', value: 10 }
  ])
})
```

- [ ] **Step 6: Run tests to verify they fail**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: The new tests FAIL.

- [ ] **Step 7: Commit failing tests**

```bash
git add tests/unit/constant/paramSettingsReset.spec.js
git commit -m "test: add failing tests for by-empty skip and multi mark column index"
```

---

## Task 2: Modify shouldProcessClassifyItem (by-empty Skip)

**Files:**
- Modify: `src/constant/paramSettingsReset.js`

- [ ] **Step 1: Comment out hasValidConfiguration**

In `src/constant/paramSettingsReset.js`, wrap the entire `hasValidConfiguration` function with a block comment `/* ... */`:

```js
/*
 * hasValidConfiguration - 保留供后续参考
 * 用于判断 classifyItem 是否已有有效配置（非初次使用）
 * 2026-05-30: by 为空直接跳过，此函数暂不再使用
 *
function hasValidConfiguration(classifyItem) {
  console.log("hasValidConfiguration 检查:", {
    valueList: classifyItem.valueList,
    by: classifyItem.by
  })

  if (!CommonUtils.isNotEmptyArray(classifyItem.valueList)) {
    console.log("→ valueList 为空，返回 false")
    return false
  }

  if (classifyItem.valueList.length === 1) {
    const item = classifyItem.valueList[0]
    if (item.key === "" || item.key == null) {
      const defaultValue = obtainDefaultValue(classifyItem.type, classifyItem.valueType)
      console.log("→ 单个空key元素，对比默认值:", {
        currentValue: item.value,
        defaultValue: defaultValue
      })
      if (item.value === defaultValue || item.value == null) {
        console.log("→ 是默认值，返回 false")
        return false
      }
    }
  }

  console.log("→ 有有效配置，返回 true")
  return true
}
*/
```

- [ ] **Step 2: Replace shouldProcessClassifyItem**

```js
function shouldProcessClassifyItem(classifyItem) {
  if (!classifyItem.by) {
    console.log(`跳过处理：by 为空 ${classifyItem.paramName || ''}`)
    return false
  }
  return true
}
```

- [ ] **Step 3: Run by-empty tests only**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand -t "by 为空"
```

Expected: The 2 "by 为空" tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src/constant/paramSettingsReset.js
git commit -m "fix: skip classparam reset when by is empty"
```

---

## Task 3: Add computeByMarkColumnIndex And Modify resolveBoundMarkHeaderInfo

**Files:**
- Modify: `src/constant/paramSettingsReset.js`

- [ ] **Step 1: Add resolveByColumnName internal function**

Add before `resolveBoundMarkHeaderInfo`:

```js
/**
 * 回退逻辑：按 columnName 全局匹配（兼容无 byMarkColumnIndex 的旧数据）
 * 与旧逻辑的区别：不再 fallback 到 candidates[0]，找不到直接 BINDING_MISSING
 */
function resolveByColumnName(commonParams, classifyParamObject, classifyItem) {
  const candidates = resolveBoundMarkHeaderInfoList(commonParams, classifyParamObject)

  if (!CommonUtils.isNotEmptyArray(candidates)) {
    return { state: BINDING_MISSING, headerInfo: null }
  }

  const matchedHeaderInfo = candidates.find(
    headerInfo => headerInfo.columnName === classifyItem.by
  )

  if (!matchedHeaderInfo) {
    return { state: BINDING_MISSING, headerInfo: null }
  }

  return { state: BOUND_MATCHED, headerInfo: matchedHeaderInfo }
}
```

- [ ] **Step 2: Replace resolveBoundMarkHeaderInfo**

```js
function resolveBoundMarkHeaderInfo(commonParams, classifyParamObject, classifyItem) {
  const idx = classifyItem.byMarkColumnIndex
  const markColumnList = classifyParamObject.byFileMarkColumnList

  // 无索引 → 回退 columnName 全局匹配（兼容旧数据）
  if (idx == null || !CommonUtils.isNotEmptyArray(markColumnList)) {
    return resolveByColumnName(commonParams, classifyParamObject, classifyItem)
  }

  const markColumn = markColumnList[idx]
  if (!markColumn) {
    return { state: BINDING_MISSING, headerInfo: null }
  }

  const headerInfo = resolveHeaderInfoForMarkColumn(commonParams, markColumn)
  if (!headerInfo || headerInfo.columnName !== classifyItem.by) {
    return { state: BINDING_MISSING, headerInfo: null }
  }

  return { state: BOUND_MATCHED, headerInfo }
}
```

- [ ] **Step 3: Add computeByMarkColumnIndex function**

Add after `shouldProcessClassifyItem` and before `reSetParamSettings`:

```js
/**
 * 首次加载时计算 classifyItem 对应的 byFileMarkColumnList 索引
 * 遍历所有 moduleType=7 的 setting，对 by 非空的 classifyItem，
 * 解析每个标记列的当前 headerInfo 找 columnName 匹配的索引
 * @param {Array} topParamSettingsList - 顶层参数列表
 * @param {Array} fileSettings - 文件设置列表
 */
function computeByMarkColumnIndex(topParamSettingsList, fileSettings) {
  if (!CommonUtils.isNotEmptyArray(topParamSettingsList)) return
  if (!CommonUtils.isNotEmptyArray(fileSettings)) return

  topParamSettingsList.forEach(group => {
    if (group.moduleType !== -1) return
    if (!CommonUtils.isNotEmptyArray(group.paramSettingsList)) return

    group.paramSettingsList.forEach(setting => {
      if (setting.moduleType !== 7) return
      if (!CommonUtils.isNotEmptyArray(setting.byFileMarkColumnList)) return
      if (!CommonUtils.isNotEmptyArray(setting.classifyItemList)) return

      setting.classifyItemList.forEach(classifyItem => {
        if (!classifyItem.by) {
          classifyItem.byMarkColumnIndex = null
          return
        }

        let matchedIndex = null
        setting.byFileMarkColumnList.forEach((markColumn, idx) => {
          if (matchedIndex !== null) return
          const headerInfo = resolveHeaderInfoForMarkColumn(
            { fileSettings: fileSettings },
            markColumn
          )
          if (headerInfo && headerInfo.columnName === classifyItem.by) {
            matchedIndex = idx
          }
        })

        classifyItem.byMarkColumnIndex = matchedIndex
      })
    })
  })
}
```

- [ ] **Step 4: Add computeByMarkColumnIndex to exports**

In the `export` block at the bottom:

```js
  shouldProcessClassifyItem,
  computeByMarkColumnIndex,
  BOUND_MATCHED,
```

- [ ] **Step 5: Run all paramSettingsReset tests**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: ALL tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/constant/paramSettingsReset.js
git commit -m "feat: add byMarkColumnIndex support for precise mark column matching"
```

---

## Task 4: Clear byMarkColumnIndex In resetClassifyItemByDefault

**Files:**
- Modify: `src/constant/modify/publicFunction.js`

- [ ] **Step 1: Add byMarkColumnIndex clearing**

In `resetClassifyItemByDefault`, find the line:

```js
  setValue(classifyItem, "by", null)
```

Add right after it:

```js
  setValue(classifyItem, "byMarkColumnIndex", null)
```

The end of the function should look like:

```js
  setValue(classifyItem, "by", null)
  setValue(classifyItem, "byMarkColumnIndex", null)
}
```

- [ ] **Step 2: Commit**

```bash
git add src/constant/modify/publicFunction.js
git commit -m "fix: clear byMarkColumnIndex when resetting classifyItem by"
```

---

## Task 5: Call computeByMarkColumnIndex In draw.vue On Initial Load

**Files:**
- Modify: `src/views/goto/plot/type/draw.vue`

- [ ] **Step 1: Import computeByMarkColumnIndex**

Find the existing import from `@/constant/paramSettingsReset` or add a new one. Add `computeByMarkColumnIndex` to the import:

```js
import { computeByMarkColumnIndex } from '@/constant/paramSettingsReset'
```

- [ ] **Step 2: Call in obtainPlotData**

In `obtainPlotData()`, find:

```js
          if (res.code === 200) {
            this.plot = res.data
```

Add right after `this.plot = res.data`:

```js
            // 计算 moduleType=7 分类参数的 byMarkColumnIndex
            computeByMarkColumnIndex(
              this.plot.userDraw.paramSettingsList,
              this.plot.userDraw.fileSettings
            )
```

- [ ] **Step 3: Commit**

```bash
git add src/views/goto/plot/type/draw.vue
git commit -m "feat: compute byMarkColumnIndex on initial plot data load"
```

---

## Task 6: Compatibility Verification

**Files:**
- No new file changes expected.

- [ ] **Step 1: Run all related tests**

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js tests/unit/constant/associationParamModify.spec.js --runInBand
```

Expected: ALL tests PASS.

- [ ] **Step 2: Run lint on changed files**

```bash
npx eslint src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js src/views/goto/plot/type/draw.vue tests/unit/constant/paramSettingsReset.spec.js
```

Expected: PASS or auto-fixable only. If auto-fix needed:

```bash
npx eslint --fix src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js src/views/goto/plot/type/draw.vue tests/unit/constant/paramSettingsReset.spec.js
```

- [ ] **Step 3: Check the diff**

```bash
git diff --stat
```

Expected: Only the 4 files listed above are changed.

- [ ] **Step 4: Commit any lint fixes**

```bash
git add -u
git commit -m "style: lint fixes for byMarkColumnIndex changes"
```

Skip if no changes.

---

## Task 7: Manual Validation Checklist

**Files:**
- No required file changes.

- [ ] **Step 1: by is initially empty**

Open a plot with a moduleType=7 classify param that has never been configured. Upload a file, mark a column, save.

Expected:
- After upload → classifyItem unchanged (by still null/empty)
- After mark + save → classifyItem refreshes with marked column content

- [ ] **Step 2: by has value, same column still exists**

Same plot, already configured with `by='Group'`. Upload a new file that still has column 'Group'.

Expected:
- After upload → `reSetParamSettings` refreshes using the correct byFileMarkColumnList index
- by stays 'Group', valueList updates to new file's unique values

- [ ] **Step 3: Multiple mark columns, select second column**

Configure 2 mark columns (e.g., 'group' and 'type'). Set classifyItem.by to a column from the second mark column. Upload, re-mark, save.

Expected:
- After save → classifyItem refreshes using the second mark column's content, not the first

- [ ] **Step 4: Clear by**

Click clear/reset on the by field. Then upload and re-mark.

Expected:
- by is cleared, byMarkColumnIndex is null
- After re-mark + save → correctly uses the newly marked column

---

## Task 3b: Update Existing BOUND_CHANGED Test

**Files:**
- Modify: `tests/unit/constant/paramSettingsReset.spec.js`

The old `BOUND_CHANGED 时更新 by 并复用旧 valueList` test relied on `candidates[0]` fallback (when `by='Group'` doesn't match `columnName='NewGroup'`, fall back to first candidate). New behavior: columnName doesn't match → BINDING_MISSING → keep old data.

- [ ] **Step 1: Update the BOUND_CHANGED test**

Replace the `BOUND_CHANGED 时更新 by 并复用旧 valueList` test with:

```js
/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证 by 与标记列名不匹配时，保留旧数据不修改。
 * 覆盖范围：旧 fallback 行为已移除，columnName 不匹配则 BINDING_MISSING。
 * 前置条件：by='Group' 但标记列当前选中 'NewGroup'，无 byMarkColumnIndex。
 */
test('BOUND_CHANGED 时保留旧 by 不修改', () => {
  const classifyItem = createManualClassifyItem()
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('NewGroup', ['X', 'Y'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  // by 不匹配任何标记列 → 保持不变
  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.byFileSettingIndex).toBe(0)
  expect(classifyItem.legend.title.value).toBe('Group')
  expect(classifyItem.valueList).toEqual([
    { name: 'A', value: 10 },
    { name: 'B', value: 20 }
  ])
})
```

- [ ] **Step 2: Commit**

```bash
git add tests/unit/constant/paramSettingsReset.spec.js
git commit -m "test: update BOUND_CHANGED test for removed candidates[0] fallback"
```

Run this after Task 3 Step 4 (before the final test run).
