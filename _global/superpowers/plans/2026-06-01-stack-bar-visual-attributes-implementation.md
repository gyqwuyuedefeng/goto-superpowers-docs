# Stack Bar Visual Attributes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement stack bar parameter linkage for circular styles, axis tabs, radial facet strip positions, and label setting visibility.

**Architecture:** Keep stack-bar-specific rules in `goto-web/src/constant/modify/stackBarModify.js`. Reuse existing parameter lookup utilities, add focused helpers for setting `disabledIndexList`, tab visibility, fallback selection values, and label visibility. Only update `facetParams.vue` to pass the computed disabled indexes into the existing select component.

**Tech Stack:** Vue 2, Element UI, Jest, existing `goto-web` modify constants.

---

## File Structure

- Modify: `goto-web/src/constant/modify/stackBarModify.js`
  - Owns all stack bar linkage rules.
  - Adds helpers for:
    - style value lookup and fallback
    - X-axis numeric-column style disabling
    - axis tab and circular parameter visibility
    - radial facet strip position disabling
    - label setting visibility
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
  - Passes `disabledIndexList` to `GSelectObject` for `radialStripPosition`.
- Create: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`
  - Covers the stack bar linkage rules with minimal mocked `commonParams`.

## Task 1: Add Stack Bar Unit Test Harness

**Files:**
- Create: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`
- Read: `goto-web/tests/unit/constant/modify/vennModify.spec.js`
- Read: `goto-web/tests/unit/constant/modify/scatterModify.spec.js`

- [ ] **Step 1: Create a failing Jest spec file**

Create `goto-web/tests/unit/constant/modify/stackBarModify.spec.js` with this complete starter content:

```js
/* eslint-env jest */
jest.mock('@/constant/associationParamModify', () => ({
  localFindFromTopParamSettingsList: jest.fn((name, commonParams) => {
    const groups = commonParams.topParamSettingsList || []
    for (const group of groups) {
      const settings = group.paramSettingsList || []
      for (const setting of settings) {
        if (setting.paramName === name) {
          return setting
        }
      }
    }
    return null
  }),
  localFindGroupFromTopParamSettingsListByName: jest.fn((groupName, commonParams) => {
    const groups = commonParams.topParamSettingsList || []
    for (const group of groups) {
      if (group.name === groupName) {
        return group
      }
    }
    return null
  }),
  findColumnAllContentByFileNameFromFileSettings: jest.fn(),
  findAllColumnNameByFileNameFromFileSettings: jest.fn(),
  replaceNewSetting: jest.fn(),
  setSettingsVisible: jest.fn((commonParams, visible, ...names) => {
    const groups = commonParams.topParamSettingsList || []
    for (const name of names) {
      for (const group of groups) {
        const settings = group.paramSettingsList || []
        for (const setting of settings) {
          if (setting.paramName === name) {
            setting.visible = visible
          }
        }
      }
    }
  })
}))

jest.mock('@/constant/classifyItem', () => ({
  fillChooseColumnUniqueList: jest.fn(),
  obtainChooseColumnUniqueList: jest.fn()
}))

jest.mock('@/constant/modify/publicFunction', () => ({
  facetParamsModify: jest.fn(),
  plotNumModify: jest.fn(),
  obtainFilteredHeaderInfoList: jest.fn(),
  xDisplayReSetOptionList: jest.fn(),
  findFilteredHeaderInfoListByGroupIdAndKey: jest.fn(),
  updateSettingsByPlotGroupIdAndPlotTypeId: jest.fn()
}))

jest.mock('@/utils/CommonUtils', () => ({
  isNotEmptyArray: jest.fn((value) => Array.isArray(value) && value.length > 0),
  isNotEmptyObject: jest.fn((value) => value !== null && typeof value === 'object' && Object.keys(value).length > 0),
  deepClone: jest.fn((value) => JSON.parse(JSON.stringify(value)))
}))

jest.mock('@/utils/BitwiseUtils', () => ({
  getBit: jest.fn()
}))

jest.mock('@/api/plotType', () => ({}))

const PublicFunctions = require('@/constant/modify/publicFunction')
const {
  doStackBarModify,
  stackBarSaveFileMarkSuccess
} = require('@/constant/modify/stackBarModify')

function createSelectSetting(paramName, value, selectValues) {
  return {
    moduleType: 3,
    paramName,
    value,
    visible: true,
    selectList: selectValues.map((item, index) => ({
      id: index + 1,
      label: item.label,
      value: item.value
    }))
  }
}

function createValueSelect(value, selectValues) {
  return {
    value: {
      value,
      selectList: selectValues.map((item, index) => ({
        id: index + 1,
        label: item.label,
        value: item.value
      }))
    },
    visible: true
  }
}

function createSetting(paramName, value, visible = true) {
  return {
    paramName,
    value,
    visible
  }
}

function buildCommonParams({
  styleValue = 'default',
  facetTypeValue = 'none',
  radialStripValue = 'inside',
  classLabelPos = 'none',
  valueLabelPos = 'none',
  xAxisNumberColumn = false,
  fileSettingIndex = 0
} = {}) {
  const styleSetting = createSelectSetting('type', styleValue, [
    { label: '默认', value: 'default' },
    { label: '环形（X轴弯曲）', value: 'circular_x' },
    { label: '环形（Y轴弯曲）', value: 'circular_y' }
  ])
  const facetParams = {
    paramName: 'facetParams',
    type: createValueSelect(facetTypeValue, [
      { label: '无分面', value: 'none' },
      { label: '环绕分面', value: 'wrap' },
      { label: '网格分面', value: 'grid' },
      { label: '环形分面', value: 'radial' }
    ]),
    radialStripPosition: createValueSelect(radialStripValue, [
      { label: '内侧', value: 'inside' },
      { label: '外侧', value: 'outside' },
      { label: '起始', value: 'start' },
      { label: '终止', value: 'end' }
    ])
  }
  const visualGroup = {
    moduleType: -1,
    name: '视觉属性',
    visible: true,
    paramSettingsList: [
      styleSetting,
      createSetting('start', 0, true),
      createSetting('end', 270, true),
      createSetting('innerRadius', 0.5, true)
    ]
  }
  const labelGroup = {
    moduleType: -1,
    name: '标签设置',
    visible: true,
    paramSettingsList: [
      createSetting('classLabelPos', classLabelPos, true),
      createSetting('valueLabelPos', valueLabelPos, true),
      createSetting('classLabelOrientation', 'vertical', true),
      createSetting('valueLabelOrientation', 'horizontal', true),
      createSetting('valueLabelDigits', 2, true),
      createSetting('autofitAxis', 'FALSE', true),
      createSetting('classLabels', {}, true),
      createSetting('valueLabels', {}, true)
    ]
  }
  return {
    setting: styleSetting,
    params: null,
    fileSettingIndex,
    groupId: 22,
    plotTypeId: 135,
    userDrawId: 'test-user-draw-id',
    vueContext: {
      $set(target, key, value) {
        target[key] = value
      }
    },
    fileSettings: [
      {
        fileMarkContainerList: [
          { field: 'x_column', positionParam: 0, type: 'SELECT_COLUMN' }
        ],
        headerInfoList: [
          {
            columnName: 'X',
            numberColumn: xAxisNumberColumn,
            status: 1
          }
        ]
      }
    ],
    topParamSettingsList: [
      visualGroup,
      labelGroup,
      {
        moduleType: -1,
        name: 'X轴设置',
        visible: true,
        paramSettingsList: []
      },
      {
        moduleType: -1,
        name: 'Y轴设置',
        visible: true,
        paramSettingsList: []
      },
      {
        moduleType: -1,
        name: '角度轴设置',
        visible: true,
        paramSettingsList: []
      },
      {
        moduleType: -1,
        name: '径向轴设置',
        visible: true,
        paramSettingsList: []
      },
      {
        moduleType: -1,
        name: '分面参数',
        visible: true,
        paramSettingsList: [facetParams]
      }
    ]
  }
}

function findSetting(commonParams, paramName) {
  const groups = commonParams.topParamSettingsList || []
  for (const group of groups) {
    const settings = group.paramSettingsList || []
    for (const setting of settings) {
      if (setting.paramName === paramName) {
        return setting
      }
    }
  }
  return null
}

function findGroup(commonParams, groupName) {
  return commonParams.topParamSettingsList.find(group => group.name === groupName)
}

describe('stackBarModify visual attributes', () => {
  beforeEach(() => {
    PublicFunctions.findFilteredHeaderInfoListByGroupIdAndKey.mockReset()
    PublicFunctions.findFilteredHeaderInfoListByGroupIdAndKey.mockImplementation((commonParams, key) => {
      if (key !== 'x_column') {
        return []
      }
      const headerInfo = commonParams.fileSettings[0].headerInfoList[0]
      return [{ ...headerInfo, fileSettingIndex: 0 }]
    })
    PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId.mockClear()
    PublicFunctions.facetParamsModify.mockClear()
  })

  test('numeric x-axis disables circular styles and falls back to default', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_x',
      xAxisNumberColumn: true
    })

    stackBarSaveFileMarkSuccess(commonParams)

    const styleSetting = findSetting(commonParams, 'type')
    expect(styleSetting.disabledIndexList).toEqual([1, 2])
    expect(styleSetting.value).toBe('default')
    expect(findSetting(commonParams, 'start').visible).toBe(false)
    expect(findSetting(commonParams, 'end').visible).toBe(false)
    expect(findSetting(commonParams, 'innerRadius').visible).toBe(false)
    expect(findGroup(commonParams, 'X轴设置').visible).toBe(true)
    expect(findGroup(commonParams, 'Y轴设置').visible).toBe(true)
    expect(findGroup(commonParams, '角度轴设置').visible).toBe(false)
    expect(findGroup(commonParams, '径向轴设置').visible).toBe(false)
    expect(PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId).toHaveBeenCalledWith(
      [styleSetting],
      commonParams
    )
  })

  test('non-numeric x-axis enables circular styles', () => {
    const commonParams = buildCommonParams({
      styleValue: 'default',
      xAxisNumberColumn: false
    })

    stackBarSaveFileMarkSuccess(commonParams)

    const styleSetting = findSetting(commonParams, 'type')
    expect(styleSetting.disabledIndexList).toEqual([])
  })

  test('circular style hides Cartesian axis tabs and shows circular parameters', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_y'
    })
    commonParams.setting = findSetting(commonParams, 'type')

    doStackBarModify(commonParams)

    expect(findSetting(commonParams, 'start').visible).toBe(true)
    expect(findSetting(commonParams, 'end').visible).toBe(true)
    expect(findSetting(commonParams, 'innerRadius').visible).toBe(true)
    expect(findGroup(commonParams, 'X轴设置').visible).toBe(false)
    expect(findGroup(commonParams, 'Y轴设置').visible).toBe(false)
    expect(findGroup(commonParams, '角度轴设置').visible).toBe(true)
    expect(findGroup(commonParams, '径向轴设置').visible).toBe(true)
  })
})
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: FAIL. The first failure should show that `disabledIndexList` is `undefined` or `type` changes do not update visibility.

- [ ] **Step 3: Commit the failing tests**

```bash
git add goto-web/tests/unit/constant/modify/stackBarModify.spec.js
git commit -m "test: add stack bar visual attribute linkage coverage"
```

## Task 2: Implement Style and Axis Visibility Linkage

**Files:**
- Modify: `goto-web/src/constant/modify/stackBarModify.js:1-123`
- Test: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`

- [ ] **Step 1: Update imports**

In `goto-web/src/constant/modify/stackBarModify.js`, extend the existing association import so it includes `localFindGroupFromTopParamSettingsListByName`:

```js
import {
  localFindFromTopParamSettingsList,
  findColumnAllContentByFileNameFromFileSettings,
  findAllColumnNameByFileNameFromFileSettings,
  replaceNewSetting,
  setSettingsVisible,
  localFindGroupFromTopParamSettingsListByName
} from "@/constant/associationParamModify"
```

- [ ] **Step 2: Add constants after the existing comment block**

Insert this code after the `commonParams` comment and before `doStackBarModify`:

```js
const STACK_BAR_STYLE_DEFAULT = "default"
const STACK_BAR_STYLE_CIRCULAR_X = "circular_x"
const STACK_BAR_STYLE_CIRCULAR_Y = "circular_y"
const STACK_BAR_CIRCULAR_STYLES = [
  STACK_BAR_STYLE_CIRCULAR_X,
  STACK_BAR_STYLE_CIRCULAR_Y
]
```

- [ ] **Step 3: Replace `doStackBarModify` with style dispatch**

Replace the whole current `doStackBarModify` function with:

```js
function doStackBarModify(commonParams) {
  const paramName = commonParams.setting.paramName
  if (paramName === "plotNum") {
    plotNumModify(commonParams)
  } else if (paramName === "facetParams") {
    PublicFunctions.facetParamsModify(commonParams)
    if (
      CommonUtils.isNotEmptyObject(commonParams.params) &&
      commonParams.params.optionName === "type"
    ) {
      radialStripPositionModify(commonParams)
    }
  } else if (paramName === "type") {
    styleModify(commonParams)
  } else if (paramName === "classLabelPos" || paramName === "valueLabelPos") {
    labelSettingsModify(commonParams)
  }
}
```

- [ ] **Step 4: Add style helper functions after `plotNumModify`**

Insert this complete helper block after `plotNumModify`:

```js
function styleModify(commonParams) {
  styleSettingsVisibleModify(commonParams)
  radialStripPositionModify(commonParams)
  labelSettingsModify(commonParams)
}

function getStyleSetting(commonParams) {
  return localFindFromTopParamSettingsList("type", commonParams)
}

function getStyleValue(commonParams) {
  const styleSetting = getStyleSetting(commonParams)
  return styleSetting ? styleSetting.value : null
}

function getSelectOptionIndexListByValues(setting, values) {
  const disabledIndexList = []
  if (!setting || !CommonUtils.isNotEmptyArray(setting.selectList)) {
    return disabledIndexList
  }

  for (let index = 0; index < setting.selectList.length; index++) {
    const option = setting.selectList[index]
    if (option && values.indexOf(option.value) !== -1) {
      disabledIndexList.push(index)
    }
  }
  return disabledIndexList
}

function setDisabledIndexList(commonParams, setting, disabledIndexList) {
  if (!setting) {
    return
  }
  if (commonParams.vueContext && commonParams.vueContext.$set) {
    commonParams.vueContext.$set(setting, "disabledIndexList", disabledIndexList)
  } else {
    setting.disabledIndexList = disabledIndexList
  }
}

function findFirstEnabledOptionValue(setting, disabledIndexList) {
  if (!setting || !CommonUtils.isNotEmptyArray(setting.selectList)) {
    return null
  }

  for (let index = 0; index < setting.selectList.length; index++) {
    const option = setting.selectList[index]
    if (
      option &&
      disabledIndexList.indexOf(index) === -1 &&
      option.value !== undefined &&
      option.value !== null
    ) {
      return option.value
    }
  }
  return null
}

function fallbackIfDisabled(commonParams, setting, disabledIndexList, fallbackValue) {
  if (!setting) {
    return false
  }
  const currentIndex = getSelectOptionIndexListByValues(setting, [setting.value])[0]
  const currentDisabled =
    currentIndex !== undefined && disabledIndexList.indexOf(currentIndex) !== -1
  if (!currentDisabled) {
    return false
  }

  const nextValue =
    fallbackValue !== undefined && fallbackValue !== null
      ? fallbackValue
      : findFirstEnabledOptionValue(setting, disabledIndexList)
  if (nextValue === null) {
    return false
  }

  setting.value = nextValue
  PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId(
    [setting],
    commonParams
  )
  return true
}

function isXAxisNumberColumn(commonParams) {
  const xAxisHeaderInfoList =
    PublicFunctions.findFilteredHeaderInfoListByGroupIdAndKey(
      commonParams,
      "x_column"
    )
  if (!CommonUtils.isNotEmptyArray(xAxisHeaderInfoList)) {
    return false
  }

  for (let index = 0; index < xAxisHeaderInfoList.length; index++) {
    const headerInfo = xAxisHeaderInfoList[index]
    if (headerInfo && headerInfo.numberColumn === true) {
      return true
    }
  }
  return false
}

function styleDisabledModify(commonParams) {
  const styleSetting = getStyleSetting(commonParams)
  if (!styleSetting) {
    return
  }

  const disabledIndexList = isXAxisNumberColumn(commonParams)
    ? getSelectOptionIndexListByValues(styleSetting, STACK_BAR_CIRCULAR_STYLES)
    : []

  setDisabledIndexList(commonParams, styleSetting, disabledIndexList)

  if (disabledIndexList.length > 0) {
    fallbackIfDisabled(
      commonParams,
      styleSetting,
      disabledIndexList,
      STACK_BAR_STYLE_DEFAULT
    )
  }
}

function setGroupVisible(commonParams, visible, ...groupNames) {
  for (let index = 0; index < groupNames.length; index++) {
    const group = localFindGroupFromTopParamSettingsListByName(
      groupNames[index],
      commonParams
    )
    if (group) {
      group.visible = visible
    }
  }
}

function styleSettingsVisibleModify(commonParams) {
  const styleValue = getStyleValue(commonParams)
  const circular = STACK_BAR_CIRCULAR_STYLES.indexOf(styleValue) !== -1

  setSettingsVisible(commonParams, circular, "start", "end", "innerRadius")
  setGroupVisible(commonParams, !circular, "X轴设置", "Y轴设置")
  setGroupVisible(commonParams, circular, "角度轴设置", "径向轴设置")
}
```

- [ ] **Step 5: Update save mark success to apply style rules**

Replace `stackBarSaveFileMarkSuccess` with:

```js
function stackBarSaveFileMarkSuccess(commonParams) {
  plotNumModify(commonParams)
  if (commonParams.fileSettingIndex === 1) {
    xDisplayReSetOptionList(commonParams)
  }
  styleDisabledModify(commonParams)
  styleSettingsVisibleModify(commonParams)
  radialStripPositionModify(commonParams)
  labelSettingsModify(commonParams)
}
```

This intentionally runs the X-axis check for all file mark saves because the stack bar dataset is file index `0`, while metadata behavior still remains guarded by file index `1`.

- [ ] **Step 6: Run the focused test**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: The three tests from Task 1 pass, or failures only mention `radialStripPositionModify` / `labelSettingsModify` being undefined before later tasks.

- [ ] **Step 7: If undefined helper failures appear, add temporary empty helpers**

If the test fails because `radialStripPositionModify` or `labelSettingsModify` is not defined, add these temporary functions after `styleSettingsVisibleModify`:

```js
function radialStripPositionModify(commonParams) {}

function labelSettingsModify(commonParams) {}
```

Run the same focused test again. Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add goto-web/src/constant/modify/stackBarModify.js goto-web/tests/unit/constant/modify/stackBarModify.spec.js
git commit -m "feat: link stack bar style to x-axis type"
```

## Task 3: Implement Radial Facet Strip Position Linkage

**Files:**
- Modify: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`
- Modify: `goto-web/src/constant/modify/stackBarModify.js`
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue:158-170`

- [ ] **Step 1: Add failing radial facet tests**

Append these tests before the closing brace of the existing `describe('stackBarModify visual attributes', () => { })` block:

```js
  test('circular x style disables radial strip start and end positions', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_x',
      facetTypeValue: 'radial',
      radialStripValue: 'start'
    })
    commonParams.setting = findSetting(commonParams, 'type')

    doStackBarModify(commonParams)

    const facetParams = findSetting(commonParams, 'facetParams')
    expect(facetParams.radialStripPosition.disabledIndexList).toEqual([2, 3])
    expect(facetParams.radialStripPosition.value.value).toBe('inside')
    expect(PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId).toHaveBeenCalledWith(
      [facetParams],
      commonParams
    )
  })

  test('circular y style disables radial strip inside and outside positions', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_y',
      facetTypeValue: 'radial',
      radialStripValue: 'inside'
    })
    commonParams.setting = findSetting(commonParams, 'type')

    doStackBarModify(commonParams)

    const facetParams = findSetting(commonParams, 'facetParams')
    expect(facetParams.radialStripPosition.disabledIndexList).toEqual([0, 1])
    expect(facetParams.radialStripPosition.value.value).toBe('start')
  })

  test('non-radial facet clears radial strip disabled positions', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_x',
      facetTypeValue: 'wrap',
      radialStripValue: 'start'
    })
    commonParams.setting = findSetting(commonParams, 'type')

    doStackBarModify(commonParams)

    const facetParams = findSetting(commonParams, 'facetParams')
    expect(facetParams.radialStripPosition.disabledIndexList).toEqual([])
    expect(facetParams.radialStripPosition.value.value).toBe('start')
  })

  test('facet type changes still call public facet modify before stack bar radial constraints', () => {
    const commonParams = buildCommonParams({
      styleValue: 'circular_x',
      facetTypeValue: 'radial'
    })
    const facetParams = findSetting(commonParams, 'facetParams')
    commonParams.setting = facetParams
    commonParams.params = {
      optionName: 'type'
    }

    doStackBarModify(commonParams)

    expect(PublicFunctions.facetParamsModify).toHaveBeenCalledWith(commonParams)
    expect(facetParams.radialStripPosition.disabledIndexList).toEqual([2, 3])
  })
```

- [ ] **Step 2: Run focused test to verify radial cases fail**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: FAIL on the new radial facet assertions.

- [ ] **Step 3: Replace temporary radial helper with implementation**

In `stackBarModify.js`, replace `function radialStripPositionModify(commonParams) {}` with:

```js
function getFacetTypeValue(facetParamsSetting) {
  return facetParamsSetting &&
    facetParamsSetting.type &&
    facetParamsSetting.type.value
    ? facetParamsSetting.type.value.value
    : null
}

function getRadialStripPositionValue(facetParamsSetting) {
  return facetParamsSetting &&
    facetParamsSetting.radialStripPosition &&
    facetParamsSetting.radialStripPosition.value
    ? facetParamsSetting.radialStripPosition.value.value
    : null
}

function setRadialStripPositionValue(facetParamsSetting, value) {
  if (
    facetParamsSetting &&
    facetParamsSetting.radialStripPosition &&
    facetParamsSetting.radialStripPosition.value
  ) {
    facetParamsSetting.radialStripPosition.value.value = value
  }
}

function getNestedSelectList(setting) {
  if (
    setting &&
    setting.value &&
    CommonUtils.isNotEmptyArray(setting.value.selectList)
  ) {
    return setting.value.selectList
  }
  if (setting && CommonUtils.isNotEmptyArray(setting.selectList)) {
    return setting.selectList
  }
  return []
}

function getNestedSelectDisabledIndexes(setting, disabledValues) {
  const selectList = getNestedSelectList(setting)
  const disabledIndexList = []
  for (let index = 0; index < selectList.length; index++) {
    const option = selectList[index]
    if (option && disabledValues.indexOf(option.value) !== -1) {
      disabledIndexList.push(index)
    }
  }
  return disabledIndexList
}

function findFirstEnabledNestedSelectValue(setting, disabledIndexList) {
  const selectList = getNestedSelectList(setting)
  for (let index = 0; index < selectList.length; index++) {
    const option = selectList[index]
    if (
      option &&
      disabledIndexList.indexOf(index) === -1 &&
      option.value !== undefined &&
      option.value !== null
    ) {
      return option.value
    }
  }
  return null
}

function setRadialStripDisabledIndexList(commonParams, radialStripPositionSetting, disabledIndexList) {
  if (!radialStripPositionSetting) {
    return
  }
  if (commonParams.vueContext && commonParams.vueContext.$set) {
    commonParams.vueContext.$set(
      radialStripPositionSetting,
      "disabledIndexList",
      disabledIndexList
    )
  } else {
    radialStripPositionSetting.disabledIndexList = disabledIndexList
  }
}

function radialStripPositionModify(commonParams) {
  const facetParamsSetting = localFindFromTopParamSettingsList(
    "facetParams",
    commonParams
  )
  if (!facetParamsSetting || !facetParamsSetting.radialStripPosition) {
    return
  }

  let disabledValues = []
  if (getFacetTypeValue(facetParamsSetting) === "radial") {
    const styleValue = getStyleValue(commonParams)
    if (styleValue === STACK_BAR_STYLE_CIRCULAR_X) {
      disabledValues = ["start", "end"]
    } else if (styleValue === STACK_BAR_STYLE_CIRCULAR_Y) {
      disabledValues = ["inside", "outside"]
    }
  }

  const disabledIndexList = getNestedSelectDisabledIndexes(
    facetParamsSetting.radialStripPosition,
    disabledValues
  )
  setRadialStripDisabledIndexList(
    commonParams,
    facetParamsSetting.radialStripPosition,
    disabledIndexList
  )

  const currentValue = getRadialStripPositionValue(facetParamsSetting)
  const currentIndex = getNestedSelectDisabledIndexes(
    facetParamsSetting.radialStripPosition,
    [currentValue]
  )[0]
  if (
    currentIndex !== undefined &&
    disabledIndexList.indexOf(currentIndex) !== -1
  ) {
    const fallbackValue = findFirstEnabledNestedSelectValue(
      facetParamsSetting.radialStripPosition,
      disabledIndexList
    )
    if (fallbackValue !== null) {
      setRadialStripPositionValue(facetParamsSetting, fallbackValue)
      PublicFunctions.updateSettingsByPlotGroupIdAndPlotTypeId(
        [facetParamsSetting],
        commonParams
      )
    }
  }
}
```

- [ ] **Step 4: Update `facetParams.vue` to pass disabled indexes**

In `goto-web/src/views/goto/components/form/draw/facetParams.vue`, update the radial strip `GSelectObject` block from:

```vue
          :list="setting.radialStripPosition.value.selectList"
          :modify-params="buildModifyParams('radialStripPosition')"
```

to:

```vue
          :list="setting.radialStripPosition.value.selectList"
          :disabled-index-list="setting.radialStripPosition.disabledIndexList || []"
          :modify-params="buildModifyParams('radialStripPosition')"
```

- [ ] **Step 5: Run focused test**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: PASS for style and radial facet tests.

- [ ] **Step 6: Commit**

```bash
git add goto-web/src/constant/modify/stackBarModify.js goto-web/src/views/goto/components/form/draw/facetParams.vue goto-web/tests/unit/constant/modify/stackBarModify.spec.js
git commit -m "feat: constrain stack bar radial facet positions"
```

## Task 4: Implement Label Setting Visibility Linkage

**Files:**
- Modify: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`
- Modify: `goto-web/src/constant/modify/stackBarModify.js`

- [ ] **Step 1: Add failing label visibility tests**

Append these tests inside the same `describe` block:

```js
  test('hides class label orientation and font when class label position is none', () => {
    const commonParams = buildCommonParams({
      classLabelPos: 'none',
      valueLabelPos: 'inner-top'
    })
    commonParams.setting = findSetting(commonParams, 'classLabelPos')

    doStackBarModify(commonParams)

    expect(findSetting(commonParams, 'classLabelOrientation').visible).toBe(false)
    expect(findSetting(commonParams, 'classLabels').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabelOrientation').visible).toBe(true)
    expect(findSetting(commonParams, 'valueLabelDigits').visible).toBe(true)
    expect(findSetting(commonParams, 'valueLabels').visible).toBe(true)
    expect(findSetting(commonParams, 'autofitAxis').visible).toBe(true)
  })

  test('hides value label fields when value label position is none', () => {
    const commonParams = buildCommonParams({
      classLabelPos: 'outer-top',
      valueLabelPos: 'none'
    })
    commonParams.setting = findSetting(commonParams, 'valueLabelPos')

    doStackBarModify(commonParams)

    expect(findSetting(commonParams, 'classLabelOrientation').visible).toBe(true)
    expect(findSetting(commonParams, 'classLabels').visible).toBe(true)
    expect(findSetting(commonParams, 'valueLabelOrientation').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabelDigits').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabels').visible).toBe(false)
    expect(findSetting(commonParams, 'autofitAxis').visible).toBe(true)
  })

  test('when both label positions are none only keeps position controls visible', () => {
    const commonParams = buildCommonParams({
      classLabelPos: 'none',
      valueLabelPos: 'none'
    })
    commonParams.setting = findSetting(commonParams, 'classLabelPos')

    doStackBarModify(commonParams)

    expect(findSetting(commonParams, 'classLabelPos').visible).toBe(true)
    expect(findSetting(commonParams, 'valueLabelPos').visible).toBe(true)
    expect(findSetting(commonParams, 'classLabelOrientation').visible).toBe(false)
    expect(findSetting(commonParams, 'classLabels').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabelOrientation').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabelDigits').visible).toBe(false)
    expect(findSetting(commonParams, 'valueLabels').visible).toBe(false)
    expect(findSetting(commonParams, 'autofitAxis').visible).toBe(false)
  })
```

- [ ] **Step 2: Run focused test to verify label cases fail**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: FAIL on the new label visibility assertions.

- [ ] **Step 3: Replace temporary label helper with implementation**

In `stackBarModify.js`, replace `function labelSettingsModify(commonParams) {}` with:

```js
function labelSettingsModify(commonParams) {
  const classLabelPosSetting = localFindFromTopParamSettingsList(
    "classLabelPos",
    commonParams
  )
  const valueLabelPosSetting = localFindFromTopParamSettingsList(
    "valueLabelPos",
    commonParams
  )

  const classLabelVisible =
    classLabelPosSetting && classLabelPosSetting.value !== "none"
  const valueLabelVisible =
    valueLabelPosSetting && valueLabelPosSetting.value !== "none"

  setSettingsVisible(
    commonParams,
    classLabelVisible,
    "classLabelOrientation",
    "classLabels"
  )
  setSettingsVisible(
    commonParams,
    valueLabelVisible,
    "valueLabelOrientation",
    "valueLabelDigits",
    "valueLabels"
  )
  setSettingsVisible(
    commonParams,
    classLabelVisible || valueLabelVisible,
    "autofitAxis"
  )

  if (classLabelPosSetting) {
    classLabelPosSetting.visible = true
  }
  if (valueLabelPosSetting) {
    valueLabelPosSetting.visible = true
  }
}
```

- [ ] **Step 4: Run focused test**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto-web/src/constant/modify/stackBarModify.js goto-web/tests/unit/constant/modify/stackBarModify.spec.js
git commit -m "feat: link stack bar label setting visibility"
```

## Task 5: Verification and Cleanup

**Files:**
- Review: `goto-web/src/constant/modify/stackBarModify.js`
- Review: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
- Review: `goto-web/tests/unit/constant/modify/stackBarModify.spec.js`

- [ ] **Step 1: Remove unused temporary helpers or imports**

Run:

```bash
cd goto-web
npm run lint -- src/constant/modify/stackBarModify.js src/views/goto/components/form/draw/facetParams.vue tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: PASS. If lint reports unused imports already present in `stackBarModify.js`, do not perform broad cleanup. Remove only imports made newly unused by this implementation.

- [ ] **Step 2: Run the focused unit tests**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/stackBarModify.spec.js
```

Expected: PASS.

- [ ] **Step 3: Run related modify tests**

Run:

```bash
cd goto-web
npm run test:unit -- tests/unit/constant/modify/vennModify.spec.js tests/unit/constant/modify/scatterModify.spec.js tests/unit/constant/modify/publicFunction.spec.js
```

Expected: PASS. These protect the existing `disabledIndexList`, group visibility, and common modify behavior used as references.

- [ ] **Step 4: Run full lint if focused lint passes**

Run:

```bash
cd goto-web
npm run lint
```

Expected: PASS, or only pre-existing warnings/errors outside the files touched in this plan. If unrelated lint failures appear, record them in the final implementation summary and keep the task-specific lint command as the acceptance signal.

- [ ] **Step 5: Run production build if dependencies are available**

Run:

```bash
cd goto-web
npm run build:prod
```

Expected: PASS. If the Windows-style `set NODE_OPTIONS=...` script fails under Bash, retry with:

```bash
cd goto-web
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service build
```

Expected: PASS.

- [ ] **Step 6: Manual browser verification**

Start the dev server:

```bash
cd goto-web
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service serve
```

Expected: local URL printed by Vue CLI, usually `http://localhost:9527/` or `http://localhost:8080/`.

Verify these UI states in a stack bar draw page:

1. Mark X axis as a numeric column. The style select disables `环形（X轴弯曲）` and `环形（Y轴弯曲)`.
2. If style was circular before marking numeric X, it falls back to `默认`.
3. Style `默认` hides `起始角度`, `终止角度`, `内圈半径`, `角度轴设置`, and `径向轴设置`; it shows `X轴设置` and `Y轴设置`.
4. Style `环形（X轴弯曲）` or `环形（Y轴弯曲）` shows circular parameters and circular axis tabs; it hides `X轴设置` and `Y轴设置`.
5. With facet type `环形分面`, style `环形（X轴弯曲）` disables radial strip `起始` and `终止`.
6. With facet type `环形分面`, style `环形（Y轴弯曲）` disables radial strip `内侧` and `外侧`.
7. Setting both label positions to `不显示` leaves only the two position controls visible in `标签设置`.

- [ ] **Step 7: Commit verification cleanup**

If Task 5 changed files, commit them:

```bash
git add goto-web/src/constant/modify/stackBarModify.js goto-web/src/views/goto/components/form/draw/facetParams.vue goto-web/tests/unit/constant/modify/stackBarModify.spec.js
git commit -m "test: verify stack bar visual attribute linkage"
```

If Task 5 did not change files, do not create an empty commit.

## Self-Review Notes

- Spec coverage:
  - X-axis numeric style disabling and fallback: Task 1 and Task 2.
  - Default/circular tab and parameter visibility: Task 1 and Task 2.
  - Radial facet strip option disabling and fallback: Task 3.
  - `facetParams.vue` UI disabled-index passthrough: Task 3.
  - Label setting visibility: Task 4.
  - Verification: Task 5.
- Placeholder scan: no placeholder steps remain; every code-changing step includes exact code or exact replacement text.
- Type consistency:
  - `disabledIndexList` is stored on normal select settings and on `facetParams.radialStripPosition`, matching existing `GSelectObject` prop behavior.
  - `facetParams.type.value.value` and `facetParams.radialStripPosition.value.value` match the provided real parameter shape.
  - `commonParams.vueContext.$set` is used only where adding/replacing reactive properties matters.
