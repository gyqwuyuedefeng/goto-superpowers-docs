# Classify Item Discrete By Value Extension Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a user selects a new by column in discrete mapping mode, preserve existing mapping values and fill extra categories using the last existing non-color value or the existing color extension logic.

**Architecture:** Extract the non-color `valueList` fallback sync logic into `src/constant/classifyValueListSync.js` so both upload/reset and manual by selection can use it without circular imports. Then update `PublicFunctions.doClassifyItemChooseColumnFinish` to back up the old `classifyItem`, rebuild by values, and restore discrete values by type.

**Tech Stack:** Vue 2 front-end codebase, CommonJS/Jest unit tests, project `@` alias, existing `scripts/run-unit-tests.js` test runner.

---

## File Structure

- Create: `src/constant/classifyValueListSync.js`
  - Owns non-color `valueList` sync helpers:
    - `syncValueListByKeyAndFallback(newClassifyItem, oldClassifyItem)`
    - `findAndRemoveMatchedOldItem(oldValueList, newItem)`
    - `getValueListItemKey(item)`
    - `findLastValidValue(valueList)`
- Create: `tests/unit/constant/classifyValueListSync.spec.js`
  - Focused tests for the extracted helper. This locks down the existing upload/reset behavior before moving it.
- Modify: `src/constant/paramSettingsReset.js`
  - Import helper functions from `@/constant/classifyValueListSync`.
  - Remove local duplicate helper implementations.
  - Keep existing exports for compatibility with current tests/imports.
- Modify: `tests/unit/constant/modify/publicFunction.spec.js`
  - Add tests for `doClassifyItemChooseColumnFinish` discrete by selection behavior.
- Modify: `src/constant/modify/publicFunction.js`
  - Import `syncValueListByKeyAndFallback`.
  - Back up old `classifyItem` before `fillChooseColumnUniqueList`.
  - For discrete color, call existing `reserveColorForClassifyItem`.
  - For discrete non-color, call `syncValueListByKeyAndFallback`.

## Task 1: Extract Non-Color ValueList Sync Helper

**Files:**
- Create: `tests/unit/constant/classifyValueListSync.spec.js`
- Create: `src/constant/classifyValueListSync.js`
- Modify: `src/constant/paramSettingsReset.js`

- [ ] **Step 1: Add failing tests for the extracted sync helper**

Create `tests/unit/constant/classifyValueListSync.spec.js`:

```js
/* eslint-env jest */

const {
  syncValueListByKeyAndFallback,
  findAndRemoveMatchedOldItem,
  getValueListItemKey,
  findLastValidValue
} = require('@/constant/classifyValueListSync')

describe('classifyValueListSync', () => {
  /**
   * 被测试对象：syncValueListByKeyAndFallback。
   * 测试目的：验证新旧离散项名称不一致且新列表更长时，先按旧列表顺序继承，超出部分使用旧列表最后一个有效值补齐。
   */
  test('新离散内容超过旧内容时使用最后一个旧值补齐', () => {
    const newClassifyItem = {
      valueList: [
        { name: 'X', value: 0 },
        { name: 'Y', value: 0 },
        { name: 'Z', value: 0 },
        { name: 'W', value: 0 },
        { name: 'V', value: 0 }
      ]
    }
    const oldClassifyItem = {
      valueList: [
        { name: 'A', value: 1 },
        { name: 'B', value: 2 },
        { name: 'C', value: 3 }
      ]
    }

    syncValueListByKeyAndFallback(newClassifyItem, oldClassifyItem)

    expect(newClassifyItem.valueList.map(item => item.value)).toEqual([1, 2, 3, 3, 3])
  })

  /**
   * 被测试对象：syncValueListByKeyAndFallback。
   * 测试目的：验证新旧离散项存在同名项时，优先按 name/key 匹配继承旧值，而不是只按位置继承。
   */
  test('同名离散内容优先按 name 继承旧值', () => {
    const newClassifyItem = {
      valueList: [
        { name: 'B', value: 0 },
        { name: 'A', value: 0 },
        { name: 'X', value: 0 }
      ]
    }
    const oldClassifyItem = {
      valueList: [
        { name: 'A', value: 10 },
        { name: 'B', value: 20 }
      ]
    }

    syncValueListByKeyAndFallback(newClassifyItem, oldClassifyItem)

    expect(newClassifyItem.valueList.map(item => item.value)).toEqual([20, 10, 20])
  })

  /**
   * 被测试对象：getValueListItemKey。
   * 测试目的：验证 key 存在时优先使用 key，兼容后续 valueList 项存在 key 的结构。
   */
  test('valueList 项 key 存在时优先使用 key', () => {
    expect(getValueListItemKey({ key: 'k1', name: 'A' })).toBe('k1')
    expect(getValueListItemKey({ name: 'A' })).toBe('A')
    expect(getValueListItemKey(null)).toBeNull()
  })

  /**
   * 被测试对象：findAndRemoveMatchedOldItem。
   * 测试目的：验证匹配到旧项后会从候选旧列表中移除，避免同一个旧值被重复当成未匹配顺序值消费。
   */
  test('匹配旧项后从候选列表移除', () => {
    const oldValueList = [
      { name: 'A', value: 10 },
      { name: 'B', value: 20 }
    ]

    const matched = findAndRemoveMatchedOldItem(oldValueList, { name: 'A' })

    expect(matched).toEqual({ name: 'A', value: 10 })
    expect(oldValueList).toEqual([{ name: 'B', value: 20 }])
  })

  /**
   * 被测试对象：findLastValidValue。
   * 测试目的：验证最后一个有效 value 按 value !== undefined 判断，允许 null 作为业务值被保留。
   */
  test('查找最后一个 value 不为 undefined 的旧值', () => {
    expect(findLastValidValue([
      { name: 'A', value: 1 },
      { name: 'B' },
      { name: 'C', value: null }
    ])).toBeNull()
  })
})
```

- [ ] **Step 2: Run the new helper tests and verify they fail**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/classifyValueListSync.spec.js --runInBand
```

Expected: FAIL because `@/constant/classifyValueListSync` does not exist yet.

- [ ] **Step 3: Create the helper implementation**

Create `src/constant/classifyValueListSync.js`:

```js
import CommonUtils from "@/utils/CommonUtils"

function syncValueListByKeyAndFallback(newClassifyItem, oldClassifyItem) {
  if (!CommonUtils.isNotEmptyArray(newClassifyItem.valueList)) {
    return
  }

  const oldValueList = CommonUtils.isNotEmptyArray(oldClassifyItem.valueList)
    ? oldClassifyItem.valueList
    : []
  const unmatchedOldValues = [...oldValueList]
  const unmatchedNewItems = []

  newClassifyItem.valueList.forEach(newItem => {
    const matchedOldItem = findAndRemoveMatchedOldItem(unmatchedOldValues, newItem)
    if (matchedOldItem) {
      newItem.value = matchedOldItem.value
      return
    }

    unmatchedNewItems.push(newItem)
  })

  unmatchedNewItems.forEach(newItem => {
    if (unmatchedOldValues.length > 0) {
      const oldItem = unmatchedOldValues.shift()
      newItem.value = oldItem.value
      return
    }

    const lastOldValue = findLastValidValue(oldValueList)
    if (lastOldValue !== undefined) {
      newItem.value = lastOldValue
    }
  })
}

function findAndRemoveMatchedOldItem(oldValueList, newItem) {
  const newKey = getValueListItemKey(newItem)
  const index = oldValueList.findIndex(
    oldItem => getValueListItemKey(oldItem) === newKey
  )

  if (index === -1) {
    return null
  }

  return oldValueList.splice(index, 1)[0]
}

function getValueListItemKey(item) {
  if (!item) {
    return null
  }
  if (item.key !== undefined && item.key !== null) {
    return item.key
  }
  return item.name
}

function findLastValidValue(valueList) {
  for (let index = valueList.length - 1; index >= 0; index--) {
    if (valueList[index] && valueList[index].value !== undefined) {
      return valueList[index].value
    }
  }
  return undefined
}

export {
  syncValueListByKeyAndFallback,
  findAndRemoveMatchedOldItem,
  getValueListItemKey,
  findLastValidValue
}
```

- [ ] **Step 4: Run helper tests and verify they pass**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/classifyValueListSync.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 5: Update `paramSettingsReset.js` to import the helper**

In `src/constant/paramSettingsReset.js`, add this import after the existing `classifyItem` imports:

```js
import {
  syncValueListByKeyAndFallback,
  findAndRemoveMatchedOldItem,
  getValueListItemKey,
  findLastValidValue
} from "@/constant/classifyValueListSync"
```

Then delete these local function declarations from `src/constant/paramSettingsReset.js`:

```js
function syncValueListByKeyAndFallback(newClassifyItem, oldClassifyItem) {
  if (!CommonUtils.isNotEmptyArray(newClassifyItem.valueList)) {
    return
  }

  const oldValueList = CommonUtils.isNotEmptyArray(oldClassifyItem.valueList)
    ? oldClassifyItem.valueList
    : []
  const unmatchedOldValues = [...oldValueList]

  newClassifyItem.valueList.forEach(newItem => {
    const matchedOldItem = findAndRemoveMatchedOldItem(unmatchedOldValues, newItem)
    if (matchedOldItem) {
      newItem.value = matchedOldItem.value
      return
    }

    if (unmatchedOldValues.length > 0) {
      const oldItem = unmatchedOldValues.shift()
      newItem.value = oldItem.value
      return
    }

    const lastOldValue = findLastValidValue(oldValueList)
    if (lastOldValue !== undefined) {
      newItem.value = lastOldValue
    }
  })
}

function findAndRemoveMatchedOldItem(oldValueList, newItem) {
  const newKey = getValueListItemKey(newItem)
  const index = oldValueList.findIndex(
    oldItem => getValueListItemKey(oldItem) === newKey
  )

  if (index === -1) {
    return null
  }

  return oldValueList.splice(index, 1)[0]
}

function getValueListItemKey(item) {
  if (!item) {
    return null
  }
  if (item.key !== undefined && item.key !== null) {
    return item.key
  }
  return item.name
}

function findLastValidValue(valueList) {
  for (let index = valueList.length - 1; index >= 0; index--) {
    if (valueList[index] && valueList[index].value !== undefined) {
      return valueList[index].value
    }
  }
  return undefined
}
```

Keep the existing export block entries:

```js
  syncValueListByKeyAndFallback,
  findAndRemoveMatchedOldItem,
  getValueListItemKey,
  findLastValidValue,
```

- [ ] **Step 6: Run existing reset tests to verify the extraction remains compatible**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js tests/unit/constant/classifyValueListSync.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 7: Commit the helper extraction**

Run:

```bash
git add src/constant/classifyValueListSync.js src/constant/paramSettingsReset.js tests/unit/constant/classifyValueListSync.spec.js
git commit -m "refactor: extract classify value list sync helper"
```

## Task 2: Add Failing Tests For Manual By Selection Value Restoration

**Files:**
- Modify: `tests/unit/constant/modify/publicFunction.spec.js`

- [ ] **Step 1: Import `doClassifyItemChooseColumnFinish` in the public function spec**

In `tests/unit/constant/modify/publicFunction.spec.js`, change the existing destructuring import from:

```js
const {
  findFilteredHeaderInfoListByFileMarkColumns,
  saveFileMarkSuccess,
  resetClassifyItemByDefault,
  facetParamsModify
} = require('@/constant/modify/publicFunction')
```

to:

```js
const {
  findFilteredHeaderInfoListByFileMarkColumns,
  saveFileMarkSuccess,
  resetClassifyItemByDefault,
  facetParamsModify,
  doClassifyItemChooseColumnFinish
} = require('@/constant/modify/publicFunction')
```

- [ ] **Step 2: Add test helpers**

Append these helpers near the top-level helper area in `tests/unit/constant/modify/publicFunction.spec.js`, before the new describe block:

```js
function createVueContext() {
  return {
    $set: jest.fn((target, key, value) => {
      target[key] = value
    })
  }
}

function createManualClassifyItem(overrides = {}) {
  return {
    type: 1,
    valueType: 2,
    by: 'OldGroup',
    byFileSettingIndex: 0,
    legend: {
      title: {
        value: 'OldGroup'
      }
    },
    valueList: [
      { name: 'A', value: 1 },
      { name: 'B', value: 2 },
      { name: 'C', value: 3 }
    ],
    value: null,
    ...overrides
  }
}

function createHeaderInfoList(chooseColumnUniqueList) {
  return [
    {
      columnName: 'NewGroup',
      fileSettingIndex: 0,
      status: 1,
      chooseColumnUniqueList
    }
  ]
}
```

- [ ] **Step 3: Add failing tests for by selection restoration**

Append this describe block to `tests/unit/constant/modify/publicFunction.spec.js`:

```js
/**
 * 被测试对象：doClassifyItemChooseColumnFinish。
 * 测试目的：验证分类项选择新的 by 后，离散映射 valueList 重建时能按旧配置恢复值。
 * 覆盖范围：非颜色最后旧值补齐、同名项优先匹配、颜色保留并补齐、连续映射不走离散恢复。
 */
describe('doClassifyItemChooseColumnFinish discrete value restoration', () => {
  /**
   * 测试场景：离散大小映射选择新 by，新 by 的离散内容数量超过旧 by。
   * 前置条件：旧 valueList 有 3 项，值为 1/2/3；新 by 有 5 个唯一值且名称完全不同。
   * 期望结果：新 valueList 前 3 项按旧顺序继承，后 2 项使用最后一个旧值 3 补齐。
   * 断言重点：非颜色缺失项不回到默认值。
   */
  test('离散非颜色新内容超过旧内容时使用最后旧值补齐', () => {
    const vueContext = createVueContext()
    const classifyItem = createManualClassifyItem({
      valueType: 2
    })

    const headerInfo = doClassifyItemChooseColumnFinish(
      vueContext,
      {},
      classifyItem,
      [],
      createHeaderInfoList(['X', 'Y', 'Z', 'W', 'V']),
      []
    )

    expect(headerInfo.columnName).toBe('NewGroup')
    expect(classifyItem.by).toBe('NewGroup')
    expect(classifyItem.byFileSettingIndex).toBe(0)
    expect(classifyItem.legend.title.value).toBe('NewGroup')
    expect(classifyItem.valueList.map(item => item.name)).toEqual(['X', 'Y', 'Z', 'W', 'V'])
    expect(classifyItem.valueList.map(item => item.value)).toEqual([1, 2, 3, 3, 3])
  })

  /**
   * 测试场景：离散形状映射选择新 by，新旧内容中有同名项。
   * 前置条件：旧 valueList 中 A=23、B=24，新列表顺序为 B/A/X。
   * 期望结果：B 和 A 优先按名称继承原值，X 使用旧列表最后一个有效值。
   * 断言重点：同名项匹配优先级高于纯索引继承。
   */
  test('离散非颜色同名内容优先继承同名旧值', () => {
    const vueContext = createVueContext()
    const classifyItem = createManualClassifyItem({
      valueType: 4,
      valueList: [
        { name: 'A', value: 23 },
        { name: 'B', value: 24 }
      ]
    })

    doClassifyItemChooseColumnFinish(
      vueContext,
      {},
      classifyItem,
      [],
      createHeaderInfoList(['B', 'A', 'X']),
      []
    )

    expect(classifyItem.valueList.map(item => item.value)).toEqual([24, 23, 24])
  })

  /**
   * 测试场景：离散颜色映射选择新 by，新 by 的离散内容数量超过旧 by。
   * 前置条件：旧颜色有 2 个，模板颜色里有未占用颜色。
   * 期望结果：前 2 个颜色保留旧颜色，新增项按现有颜色补齐逻辑使用模板色。
   * 断言重点：颜色类型不使用最后旧色直接重复补齐。
   */
  test('离散颜色新内容超过旧内容时保留旧颜色并按模板色补齐', () => {
    const vueContext = createVueContext()
    const classifyItem = createManualClassifyItem({
      valueType: 1,
      valueList: [
        { name: 'A', value: '#111111' },
        { name: 'B', value: '#222222' }
      ]
    })
    const topParamSettingsList = [
      {
        moduleType: -1,
        paramSettingsList: [
          {
            paramName: 'ManageParamTemplateColor',
            list: [
              { value: '#111111' },
              { value: '#222222' },
              { value: '#333333' },
              { value: '#444444' }
            ]
          }
        ]
      }
    ]

    doClassifyItemChooseColumnFinish(
      vueContext,
      {},
      classifyItem,
      topParamSettingsList,
      createHeaderInfoList(['X', 'Y', 'Z', 'W']),
      []
    )

    expect(classifyItem.valueList.map(item => item.value)).toEqual([
      '#111111',
      '#222222',
      '#333333',
      '#444444'
    ])
  })

  /**
   * 测试场景：连续映射选择 by。
   * 前置条件：分类项为连续颜色映射，旧 valueList 是 low/high 风格。
   * 期望结果：保持连续映射原有处理，不执行离散映射扩展恢复。
   * 断言重点：本需求只影响 type=1 离散映射。
   */
  test('连续映射选择 by 时不执行离散值扩展恢复', () => {
    const vueContext = createVueContext()
    const classifyItem = createManualClassifyItem({
      type: 2,
      valueType: 1,
      valueList: [
        { name: 'low', value: '#111111' },
        { name: 'high', value: '#222222' }
      ]
    })

    doClassifyItemChooseColumnFinish(
      vueContext,
      {},
      classifyItem,
      [],
      createHeaderInfoList(['X', 'Y', 'Z', 'W']),
      []
    )

    expect(classifyItem.type).toBe(2)
    expect(classifyItem.valueList.map(item => item.value)).toEqual(['#f2dbdb', '#ff6767'])
  })
})
```

- [ ] **Step 4: Run the new public function tests and verify they fail**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand -t "doClassifyItemChooseColumnFinish discrete value restoration"
```

Expected: FAIL. The non-color tests should show default values instead of old values; the color test should not preserve the old colors.

- [ ] **Step 5: Commit the failing tests**

Run:

```bash
git add tests/unit/constant/modify/publicFunction.spec.js
git commit -m "test: add failing tests for classify by value restoration"
```

## Task 3: Restore Discrete Values In Manual By Selection

**Files:**
- Modify: `src/constant/modify/publicFunction.js`
- Test: `tests/unit/constant/modify/publicFunction.spec.js`

- [ ] **Step 1: Import the non-color sync helper**

In `src/constant/modify/publicFunction.js`, add this import with the other constant/helper imports near the top of the file:

```js
import { syncValueListByKeyAndFallback } from "@/constant/classifyValueListSync"
```

- [ ] **Step 2: Add a type guard helper**

Add this helper near `syncValueListForClassifyItem` or before `doClassifyItemChooseColumnFinish`:

```js
function shouldRestoreDiscreteClassifyValues(classifyItem) {
  return (
    classifyItem &&
    classifyItem.type === CLASSIFY_MANUAL &&
    CommonUtils.isNotEmptyArray(classifyItem.valueList)
  )
}
```

- [ ] **Step 3: Add the restoration helper**

Add this helper near `shouldRestoreDiscreteClassifyValues`:

```js
function restoreDiscreteClassifyValuesAfterBySelection(
  commonParams,
  classifyItem,
  classifyItemBackup
) {
  if (!shouldRestoreDiscreteClassifyValues(classifyItem)) {
    return
  }

  if (classifyItem.valueType === COLOR) {
    reserveColorForClassifyItem(commonParams, classifyItem, classifyItemBackup)
    return
  }

  if (
    classifyItem.valueType === SIZE ||
    classifyItem.valueType === THICKNESS ||
    classifyItem.valueType === SHAPE ||
    classifyItem.valueType === LINE_TYPE
  ) {
    syncValueListByKeyAndFallback(classifyItem, classifyItemBackup)
  }
}
```

- [ ] **Step 4: Back up the old classify item before rebuilding by values**

In `doClassifyItemChooseColumnFinish`, find:

```js
    const columnName = headerInfo.columnName

    if (GT_SELECTED === columnName) {
      return
    }

    fillChooseColumnUniqueList(
      THIS,
      classifyItem,
      columnName,
      headerInfo.chooseColumnUniqueList
    )
```

Replace it with:

```js
    const columnName = headerInfo.columnName

    if (GT_SELECTED === columnName) {
      return
    }

    const classifyItemBackup = CommonUtils.deepClone(classifyItem)

    fillChooseColumnUniqueList(
      THIS,
      classifyItem,
      columnName,
      headerInfo.chooseColumnUniqueList
    )

    restoreDiscreteClassifyValuesAfterBySelection(
      {
        vueContext: THIS,
        topParamSettingsList
      },
      classifyItem,
      classifyItemBackup
    )
```

- [ ] **Step 5: Remove the old manual color-only refill block**

In `doClassifyItemChooseColumnFinish`, delete this block because `restoreDiscreteClassifyValuesAfterBySelection` now handles color and non-color:

```js
    if (classifyItem.type === CLASSIFY_MANUAL) {
      /**
       * 是颜色类型
       */
      if (classifyItem.valueType === COLOR) {
        // 需要填充的颜色数量
        let needAllColorNum = 0
        if (CommonUtils.isNotEmptyArray(classifyItem.valueList)) {
          needAllColorNum = classifyItem.valueList.length
        }

        if (needAllColorNum > 0) {
          /**
           * 获取对应数量的颜色，优先从模板获取，不够的随机创建
           */
          const colorList = obtainNeedSizeColorList(
            topParamSettingsList,
            needAllColorNum
          )

          /**
           * 开始填充
           */
          for (let index = 0; index < classifyItem.valueList.length; index++) {
            const element = classifyItem.valueList[index]
            element.value = colorList[index]
          }
        }
      }
    }
```

- [ ] **Step 6: Export the new helpers for focused tests and future reuse**

At the bottom export block in `src/constant/modify/publicFunction.js`, add:

```js
  shouldRestoreDiscreteClassifyValues,
  restoreDiscreteClassifyValuesAfterBySelection,
```

Place them near `doClassifyItemChooseColumnFinish`.

- [ ] **Step 7: Run the public function focused tests and verify they pass**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand -t "doClassifyItemChooseColumnFinish discrete value restoration"
```

Expected: PASS.

- [ ] **Step 8: Run the full public function spec**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 9: Commit the implementation**

Run:

```bash
git add src/constant/modify/publicFunction.js tests/unit/constant/modify/publicFunction.spec.js
git commit -m "fix: restore discrete classify values after by selection"
```

## Task 4: Compatibility Verification

**Files:**
- No expected source changes.

- [ ] **Step 1: Run all related unit tests**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/classifyValueListSync.spec.js tests/unit/constant/paramSettingsReset.spec.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 2: Run lint on changed files**

Run:

```bash
npx eslint src/constant/classifyValueListSync.js src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js tests/unit/constant/classifyValueListSync.spec.js tests/unit/constant/modify/publicFunction.spec.js
```

Expected: PASS. If lint reports fixable formatting issues, run:

```bash
npx eslint --fix src/constant/classifyValueListSync.js src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js tests/unit/constant/classifyValueListSync.spec.js tests/unit/constant/modify/publicFunction.spec.js
```

Then rerun the lint command.

- [ ] **Step 3: Inspect the final diff**

Run:

```bash
git diff --stat
git diff -- src/constant/classifyValueListSync.js src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js tests/unit/constant/classifyValueListSync.spec.js tests/unit/constant/modify/publicFunction.spec.js
```

Expected: Only the five planned files are changed. The diff should show:

- New helper file.
- `paramSettingsReset.js` importing helper functions and no longer defining duplicate local helpers.
- `publicFunction.js` backing up and restoring discrete values inside `doClassifyItemChooseColumnFinish`.
- Tests for helper behavior and by selection behavior.

- [ ] **Step 4: Commit lint fixes if any were applied**

Run only if Step 2 changed files:

```bash
git add src/constant/classifyValueListSync.js src/constant/paramSettingsReset.js src/constant/modify/publicFunction.js tests/unit/constant/classifyValueListSync.spec.js tests/unit/constant/modify/publicFunction.spec.js
git commit -m "style: lint classify by value restoration changes"
```

## Manual Validation Checklist

- [ ] Open a plot using `src/views/goto/components/form/draw/classifyItem.vue` with a discrete size/thickness/shape/line-type classify item.
- [ ] Select by column A with 3 unique values and configure values `1/2/3`.
- [ ] Select by column B with 5 unique values.
- [ ] Verify the UI shows 5 items with values `1/2/3/3/3`.
- [ ] Repeat with discrete color: configure 2 old colors, select a by with 4 unique values, and verify old colors remain first while the 2 new entries use template/generated colors.
- [ ] Switch/select a continuous mapping by and verify continuous color low/high behavior remains unchanged.
