# Module Type 7 上传重置 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `moduleType === 7` 分类参数在文件上传和标记保存后按绑定标记列刷新内容，并在绑定缺失时保留旧 `by` 与旧 `valueList`。

**Architecture:** 变更集中在 `src/constant/paramSettingsReset.js`。新增小型解析 helper 负责从 `setting.byFileMarkColumnList` 找实际标记列；`processClassifyItems` 将完整 `classifyParamObject` 继续传给手动和连续分类处理；手动分类刷新时先重建新 key，再按 key 和顺序复用旧 value。

**Tech Stack:** Vue 2 前端项目、CommonJS/Jest 单元测试、`@` alias 映射到 `src`、现有 `scripts/run-unit-tests.js` 测试启动器。

---

## File Structure

- Modify: `src/constant/paramSettingsReset.js`
  - 负责 `reSetParamSettings`、`processClassifyItems`、`moduleType === 7` 分类参数的上传后重置逻辑。
  - 新增并导出 mark-column 解析和 value 复用 helper，方便单元测试覆盖。
- Create: `tests/unit/constant/paramSettingsReset.spec.js`
  - 聚焦测试 `processClassifyItems` 和新增 helper 行为。
  - 测试数据直接构造 `fileSettings`、`byFileMarkColumnList`、`classifyItemList`，不依赖真实 Vue 组件。
- No change: `src/constant/associationParamModify.js`
  - 上传、删除、保存标记的调用链保持不变，继续通过 `reSetParamSettings(commonParams)` 进入新逻辑。

## Task 1: Add Failing Tests For Manual Classify Reset

**Files:**
- Create: `tests/unit/constant/paramSettingsReset.spec.js`

- [ ] **Step 1: Create the test file**

Create `tests/unit/constant/paramSettingsReset.spec.js` with this content:

```js
/* eslint-env jest */

const {
  processClassifyItems
} = require('@/constant/paramSettingsReset')

function createVueContext() {
  return {
    $set: jest.fn((target, key, value) => {
      target[key] = value
    })
  }
}

function createCommonParams(fileSettings, topParamSettingsList = []) {
  return {
    groupId: 1,
    fileSettings,
    topParamSettingsList,
    vueContext: createVueContext()
  }
}

function createFileSetting({
  fileMarkContainerList = [
    {
      field: 'group',
      type: 'SELECT_COLUMN',
      positionParam: 1
    }
  ],
  headerInfoList = []
} = {}) {
  return {
    fileMarkContainerList,
    headerInfoList
  }
}

function createHeader(columnName, chooseColumnUniqueList, status = 2) {
  return {
    columnName,
    label: columnName,
    field: columnName,
    key: columnName,
    status,
    chooseColumnUniqueList
  }
}

function createManualClassifyItem(overrides = {}) {
  return {
    type: 1,
    valueType: 2,
    by: 'Group',
    byFileSettingIndex: 0,
    legend: {
      title: {
        value: 'Group'
      }
    },
    valueList: [
      { name: 'A', value: 10 },
      { name: 'B', value: 20 }
    ],
    value: null,
    ...overrides
  }
}

function createClassifySetting(classifyItem, byFileMarkColumnList = [
  {
    filedName: 'group',
    fileSettingIndex: 0
  }
]) {
  return {
    moduleType: 7,
    parentModuleType: 7,
    byFileMarkColumnList,
    classifyItemList: [classifyItem]
  }
}

/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证旧标记列仍存在时，按绑定标记列刷新唯一值，并按 key 保留旧 value。
 * 覆盖范围：BOUND_MATCHED 状态、顺序变化、新唯一值增加、非颜色值补齐。
 * 前置条件：byFileMarkColumnList 指向 field=group，实际选中的表头仍是 Group。
 */
test('BOUND_MATCHED 时按绑定标记列刷新并按 key 保留旧值', () => {
  const classifyItem = createManualClassifyItem()
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('Group', ['B', 'A', 'C'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.byFileSettingIndex).toBe(0)
  expect(classifyItem.legend.title.value).toBe('Group')
  expect(classifyItem.valueList).toEqual([
    { name: 'B', value: 20 },
    { name: 'A', value: 10 },
    { name: 'C', value: 20 }
  ])
})

/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证用户重新标记到新列后，by/title/fileSettingIndex 会切换到新列，同时复用旧值。
 * 覆盖范围：BOUND_CHANGED 状态。
 * 前置条件：旧 by 为 Group，绑定标记列实际选中 NewGroup。
 */
test('BOUND_CHANGED 时更新 by 并复用旧 valueList', () => {
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

  expect(classifyItem.by).toBe('NewGroup')
  expect(classifyItem.byFileSettingIndex).toBe(0)
  expect(classifyItem.legend.title.value).toBe('NewGroup')
  expect(classifyItem.valueList).toEqual([
    { name: 'X', value: 10 },
    { name: 'Y', value: 20 }
  ])
})

/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证旧标记列丢失并清理绑定后，不清空 by，也不重置 valueList。
 * 覆盖范围：BINDING_MISSING 状态。
 * 前置条件：byFileMarkColumnList 为空，classifyItem 仍保存旧模板值。
 */
test('BINDING_MISSING 时保留旧 by 和旧 valueList', () => {
  const classifyItem = createManualClassifyItem()
  const setting = createClassifySetting(classifyItem, [])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [],
      headerInfoList: [
        createHeader('Other', ['O'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.byFileSettingIndex).toBe(0)
  expect(classifyItem.legend.title.value).toBe('Group')
  expect(classifyItem.valueList).toEqual([
    { name: 'A', value: 10 },
    { name: 'B', value: 20 }
  ])
})

/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证新标记列唯一值为空时保留旧 valueList，避免上传解析暂态清空模板。
 * 覆盖范围：空 chooseColumnUniqueList 分支。
 * 前置条件：绑定标记列可解析，但 chooseColumnUniqueList 为空。
 */
test('绑定列唯一值为空时保留旧 valueList', () => {
  const classifyItem = createManualClassifyItem()
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('Group', [])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.valueList).toEqual([
    { name: 'A', value: 10 },
    { name: 'B', value: 20 }
  ])
})

/**
 * 被测试对象：moduleType=7 上传后重置逻辑。
 * 测试目的：验证唯一值减少时只保留新 key 对应项。
 * 覆盖范围：新 valueList 比旧 valueList 短。
 * 前置条件：绑定标记列仍是 Group，新文件只剩 B。
 */
test('唯一值减少时列表收缩到新 key', () => {
  const classifyItem = createManualClassifyItem()
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('Group', ['B'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.valueList).toEqual([
    { name: 'B', value: 20 }
  ])
})
```

- [ ] **Step 2: Run the new test to verify it fails**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: FAIL. At least the `BINDING_MISSING` test should fail because current `handleManualClassifyItem` calls the default reset path when it cannot find a valid old `by` column.

- [ ] **Step 3: Keep the failing test uncommitted**

Do not commit yet. The next tasks will make this test file pass, then commit the green implementation and tests together.

## Task 2: Resolve Bound Mark Columns

**Files:**
- Modify: `src/constant/paramSettingsReset.js`
- Test: `tests/unit/constant/paramSettingsReset.spec.js`

- [ ] **Step 1: Add the BitwiseUtils import**

In `src/constant/paramSettingsReset.js`, add this import after the `CommonUtils` import:

```js
import BitwiseUtils from "@/utils/BitwiseUtils"
```

- [ ] **Step 2: Add mark-column state constants**

Add these constants after the imports:

```js
const BOUND_MATCHED = "BOUND_MATCHED"
const BOUND_CHANGED = "BOUND_CHANGED"
const BINDING_MISSING = "BINDING_MISSING"
```

- [ ] **Step 3: Add a safe Vue setter helper**

Add this helper before `processClassifyItems`:

```js
function setReactiveValue(commonParams, target, key, value) {
  if (commonParams && commonParams.vueContext && commonParams.vueContext.$set) {
    commonParams.vueContext.$set(target, key, value)
  } else {
    target[key] = value
  }
}
```

- [ ] **Step 4: Add bound mark-column resolution helpers**

Add these helpers before `processClassifyItems`:

```js
function resolveBoundMarkHeaderInfo(commonParams, classifyParamObject, classifyItem) {
  const candidates = resolveBoundMarkHeaderInfoList(commonParams, classifyParamObject)

  if (!CommonUtils.isNotEmptyArray(candidates)) {
    return {
      state: BINDING_MISSING,
      headerInfo: null
    }
  }

  const matchedHeaderInfo = candidates.find(
    headerInfo => headerInfo.columnName === classifyItem.by
  )
  const headerInfo = matchedHeaderInfo || candidates[0]

  return {
    state: headerInfo.columnName === classifyItem.by ? BOUND_MATCHED : BOUND_CHANGED,
    headerInfo
  }
}

function resolveBoundMarkHeaderInfoList(commonParams, classifyParamObject) {
  if (
    !commonParams ||
    !CommonUtils.isNotEmptyArray(commonParams.fileSettings) ||
    !classifyParamObject ||
    !CommonUtils.isNotEmptyArray(classifyParamObject.byFileMarkColumnList)
  ) {
    return []
  }

  const result = []

  classifyParamObject.byFileMarkColumnList.forEach(markColumn => {
    const headerInfo = resolveHeaderInfoForMarkColumn(commonParams, markColumn)
    if (headerInfo) {
      result.push(headerInfo)
    }
  })

  return result
}

function resolveHeaderInfoForMarkColumn(commonParams, markColumn) {
  if (!markColumn || markColumn.fileSettingIndex == null) {
    return null
  }

  const fileSetting = commonParams.fileSettings[markColumn.fileSettingIndex]
  if (!fileSetting || !CommonUtils.isNotEmptyArray(fileSetting.headerInfoList)) {
    return null
  }

  PublicFunctions.obtainHeaderInfoListColumnNameUniqueList(
    fileSetting.headerInfoList,
    commonParams.fileSettings
  )

  const markContainer = findMarkContainer(fileSetting, markColumn.filedName)
  if (!markContainer) {
    return null
  }

  const headerInfo = fileSetting.headerInfoList.find(
    item => BitwiseUtils.getBit(item.status, markContainer.positionParam) === 1
  )

  if (!headerInfo) {
    return null
  }

  return {
    ...headerInfo,
    fileSettingIndex: markColumn.fileSettingIndex
  }
}

function findMarkContainer(fileSetting, filedName) {
  if (!fileSetting || !CommonUtils.isNotEmptyArray(fileSetting.fileMarkContainerList)) {
    return null
  }

  return fileSetting.fileMarkContainerList.find(
    container => container.field === filedName && container.type === "SELECT_COLUMN"
  )
}
```

- [ ] **Step 5: Export the new constants and helpers**

Add these names to the export block at the bottom of `src/constant/paramSettingsReset.js`:

```js
  BOUND_MATCHED,
  BOUND_CHANGED,
  BINDING_MISSING,
  setReactiveValue,
  resolveBoundMarkHeaderInfo,
  resolveBoundMarkHeaderInfoList,
  resolveHeaderInfoForMarkColumn,
  findMarkContainer,
```

- [ ] **Step 6: Run the test to verify current behavior still fails later**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: FAIL. The resolver exists, but `processClassifyItems` has not switched to it yet.

- [ ] **Step 7: Keep resolver helpers uncommitted**

Do not commit yet. The resolver helpers are only an intermediate state; commit after Task 3 makes the new tests pass.

## Task 3: Preserve Manual Value Lists By Bound Mark State

**Files:**
- Modify: `src/constant/paramSettingsReset.js`
- Test: `tests/unit/constant/paramSettingsReset.spec.js`

- [ ] **Step 1: Pass the full classify parameter object to manual handling**

Replace the `CLASSIFY_MANUAL` branch in `processClassifyItems`:

```js
    } else if (classifyItem.type === CLASSIFY_MANUAL) {
      handleManualClassifyItem(commonParams, classifyParamObject, classifyItem)
    }
```

- [ ] **Step 2: Replace `handleManualClassifyItem`**

Replace the existing `handleManualClassifyItem(commonParams, classifyItem)` function with:

```js
function handleManualClassifyItem(commonParams, classifyParamObject, classifyItem) {
  if (classifyParamObject.parentModuleType !== 7) {
    handleManualClassifyItemLegacy(commonParams, classifyItem)
    return
  }

  const { state, headerInfo } = resolveBoundMarkHeaderInfo(
    commonParams,
    classifyParamObject,
    classifyItem
  )

  if (state === BINDING_MISSING) {
    return
  }

  updateClassifyItemByHeaderInfo(commonParams, classifyItem, headerInfo)
  updateClassifyItemValueList(commonParams, classifyItem, headerInfo)
}

function handleManualClassifyItemLegacy(commonParams, classifyItem) {
  if (classifyItem.by) {
    if (!reSetManualHaveBy(commonParams, classifyItem)) {
      byNotEmptyButColumnNotFound(commonParams, classifyItem)
    }
  } else {
    byEmpty(commonParams, classifyItem)
  }
}
```

- [ ] **Step 3: Add `updateClassifyItemByHeaderInfo`**

Add this helper before `updateClassifyItemValueList`:

```js
function updateClassifyItemByHeaderInfo(commonParams, classifyItem, headerInfo) {
  setReactiveValue(commonParams, classifyItem, "by", headerInfo.columnName)
  setReactiveValue(commonParams, classifyItem, "byFileSettingIndex", headerInfo.fileSettingIndex)

  if (classifyItem.legend && classifyItem.legend.title) {
    setReactiveValue(commonParams, classifyItem.legend.title, "value", headerInfo.columnName)
  }
}
```

- [ ] **Step 4: Replace `updateClassifyItemValueList`**

Replace the existing `updateClassifyItemValueList` function with:

```js
function updateClassifyItemValueList(commonParams, classifyItem, headerInfo) {
  if (!CommonUtils.isNotEmptyArray(headerInfo.chooseColumnUniqueList)) {
    return
  }

  const classifyItemBackup = CommonUtils.deepClone(classifyItem)

  setReactiveValue(commonParams, classifyItem, "valueList", [])
  fillChooseColumnUniqueList(
    commonParams.vueContext,
    classifyItem,
    headerInfo.columnName,
    headerInfo.chooseColumnUniqueList
  )

  if (CommonUtils.isNotEmptyArray(classifyItem.valueList)) {
    restoreClassifyItemValues(commonParams, classifyItem, classifyItemBackup)
  }
}
```

- [ ] **Step 5: Replace `restoreClassifyItemValues`**

Replace the existing `restoreClassifyItemValues` function with:

```js
function restoreClassifyItemValues(commonParams, classifyItem, classifyItemBackup) {
  if (classifyItem.valueType === COLOR) {
    PublicFunctions.reserveColorForClassifyItem(
      commonParams,
      classifyItem,
      classifyItemBackup
    )
    return
  }

  if ([SIZE, THICKNESS, SHAPE, LINE_TYPE].includes(classifyItem.valueType)) {
    syncValueListByKeyAndFallback(classifyItem, classifyItemBackup)
  }
}
```

- [ ] **Step 6: Add key-first value sync helpers**

Add these helpers before `byNotEmptyButColumnNotFound`:

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

- [ ] **Step 7: Export changed and new helpers**

Add these names to the export block:

```js
  handleManualClassifyItemLegacy,
  updateClassifyItemByHeaderInfo,
  syncValueListByKeyAndFallback,
  findAndRemoveMatchedOldItem,
  getValueListItemKey,
  findLastValidValue,
```

- [ ] **Step 8: Run the manual reset tests**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: PASS for the tests created in Task 1.

- [ ] **Step 9: Commit manual reset implementation**

```bash
git add src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
git commit -m "feat: preserve module type 7 manual reset values"
```

## Task 4: Add Color And Continuous Coverage

**Files:**
- Modify: `tests/unit/constant/paramSettingsReset.spec.js`
- Modify: `src/constant/paramSettingsReset.js`

- [ ] **Step 1: Add color and continuous tests**

Append these tests to `tests/unit/constant/paramSettingsReset.spec.js`:

```js
/**
 * 被测试对象：moduleType=7 手动颜色分类上传后重置。
 * 测试目的：验证颜色值沿用现有 reserveColorForClassifyItem 策略，旧颜色优先保留，不够时补模板色。
 * 覆盖范围：COLOR 分支、新唯一值增加。
 * 前置条件：模板色包含一个未被旧颜色占用的新颜色。
 */
test('颜色分类唯一值增加时旧颜色优先保留且不够使用模板色', () => {
  const classifyItem = createManualClassifyItem({
    valueType: 1,
    valueList: [
      { name: 'A', value: '#111111' },
      { name: 'B', value: '#222222' }
    ]
  })
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams(
    [
      createFileSetting({
        headerInfoList: [
          createHeader('Group', ['A', 'B', 'C'])
        ]
      })
    ],
    [
      {
        moduleType: -1,
        paramSettingsList: [
          {
            paramName: 'ManageParamTemplateColor',
            list: [
              { value: '#111111' },
              { value: '#333333' }
            ]
          }
        ]
      }
    ]
  )

  processClassifyItems(commonParams, setting)

  expect(classifyItem.valueList).toEqual([
    { name: 'A', value: '#111111' },
    { name: 'B', value: '#222222' },
    { name: 'C', value: '#333333' }
  ])
})

/**
 * 被测试对象：moduleType=7 连续分类上传后重置。
 * 测试目的：验证连续分类只同步 by/title/fileSettingIndex，不重建离散 valueList。
 * 覆盖范围：CLASSIFY_CONTINUES 的 BOUND_CHANGED 状态。
 * 前置条件：旧 by 为 Group，绑定标记列实际选中 NewGroup。
 */
test('连续分类绑定列变化时只同步 by 和标题', () => {
  const classifyItem = createManualClassifyItem({
    type: 2,
    valueType: 2,
    by: 'Group',
    valueList: null,
    value: [0, 1]
  })
  const setting = createClassifySetting(classifyItem)
  const commonParams = createCommonParams([
    createFileSetting({
      headerInfoList: [
        createHeader('NewGroup', ['1', '2'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('NewGroup')
  expect(classifyItem.byFileSettingIndex).toBe(0)
  expect(classifyItem.legend.title.value).toBe('NewGroup')
  expect(classifyItem.valueList).toBeNull()
  expect(classifyItem.value).toEqual([0, 1])
})

/**
 * 被测试对象：moduleType=7 连续分类上传后重置。
 * 测试目的：验证绑定缺失时连续分类也不会清空旧 by。
 * 覆盖范围：CLASSIFY_CONTINUES 的 BINDING_MISSING 状态。
 * 前置条件：byFileMarkColumnList 为空。
 */
test('连续分类绑定缺失时保留旧 by', () => {
  const classifyItem = createManualClassifyItem({
    type: 2,
    valueType: 2,
    by: 'Group',
    valueList: null,
    value: [0, 1]
  })
  const setting = createClassifySetting(classifyItem, [])
  const commonParams = createCommonParams([
    createFileSetting({
      fileMarkContainerList: [],
      headerInfoList: [
        createHeader('Other', ['1'])
      ]
    })
  ])

  processClassifyItems(commonParams, setting)

  expect(classifyItem.by).toBe('Group')
  expect(classifyItem.valueList).toBeNull()
  expect(classifyItem.value).toEqual([0, 1])
})
```

- [ ] **Step 2: Run tests to verify continuous behavior fails before implementation**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: FAIL on the continuous BOUND_CHANGED case until `handleContinuousClassifyItem` uses the bound mark resolver.

- [ ] **Step 3: Replace `handleContinuousClassifyItem` moduleType 7 logic**

In `src/constant/paramSettingsReset.js`, replace `handleContinuousClassifyItem` with:

```js
function handleContinuousClassifyItem(commonParams, classifyParamObject, classifyItem) {
  console.log("处理连续类型分类项:", classifyItem)

  if (classifyParamObject.parentModuleType === 7) {
    const { state, headerInfo } = resolveBoundMarkHeaderInfo(
      commonParams,
      classifyParamObject,
      classifyItem
    )

    if (state === BINDING_MISSING) {
      return
    }

    updateClassifyItemByHeaderInfo(commonParams, classifyItem, headerInfo)
  } else if (classifyParamObject.parentModuleType === 22) {
    console.log("多映射模式，保持by不变")
  }
}
```

- [ ] **Step 4: Run all paramSettingsReset tests**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 5: Commit color and continuous coverage**

```bash
git add src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
git commit -m "test: cover module type 7 color and continuous reset"
```

## Task 5: Compatibility Verification

**Files:**
- Modify only if tests reveal a regression: `src/constant/paramSettingsReset.js`
- Test: existing unit tests

- [ ] **Step 1: Run existing related tests**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/constant/paramSettingsReset.spec.js tests/unit/constant/associationParamModify.spec.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 2: Run lint on the changed source file**

Run:

```bash
npx eslint src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
```

Expected: PASS. If ESLint reports only auto-fixable formatting, run:

```bash
npx eslint --fix src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
```

Then rerun:

```bash
npx eslint src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
```

Expected: PASS.

- [ ] **Step 3: Check the diff**

Run:

```bash
git diff -- src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
```

Expected: The diff only contains mark-column-bound reset logic and focused tests. It should not modify `associationParamModify.js`, UI components, backend API files, or unrelated generated assets.

- [ ] **Step 4: Commit verification fixes if any were needed**

If Step 2 auto-fixed files or Step 1 required a compatibility adjustment, commit them:

```bash
git add src/constant/paramSettingsReset.js tests/unit/constant/paramSettingsReset.spec.js
git commit -m "fix: align module type 7 reset compatibility"
```

If no files changed after Step 1 and Step 2, do not create an empty commit.

## Task 6: Manual Validation Checklist

**Files:**
- No required file changes.

- [ ] **Step 1: Same marked column name with changed data**

In the running app, upload a file whose bound mark column keeps the same column name but has changed unique values.

Expected:

- `classifyItem.by` remains the same column name.
- New unique values appear in the `moduleType === 7` component.
- Existing values are preserved by old category name when names still exist.
- Extra non-color categories repeat the last old value.
- Extra color categories use existing template/random color behavior.

- [ ] **Step 2: Missing old mark column after upload**

Upload a file that does not contain the old marked column, causing `byFileMarkColumnList` to be empty or unresolved.

Expected:

- `classifyItem.by` remains the old value.
- `classifyItem.valueList` remains unchanged.
- The component waits for the user to mark and save a new column.

- [ ] **Step 3: Re-mark to a different column and save**

Mark a new column for the same `moduleType === 7` component and save marks.

Expected:

- `classifyItem.by` changes to the new column name.
- `legend.title.value` changes to the new column name.
- `byFileSettingIndex` matches the file index of the new mark.
- `valueList` keys come from the new column content.
- Values are reused from the old template as defined in the spec.
