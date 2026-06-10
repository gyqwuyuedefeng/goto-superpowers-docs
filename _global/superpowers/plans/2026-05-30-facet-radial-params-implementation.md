# Facet Radial Params Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the new `radial` facet type with independent radial-specific parameters while keeping existing `none`, `grid`, and `wrap` behavior intact.

**Architecture:** The backend remains the source of the facet parameter schema and dictionary ids through `FacetParams`. The draw form continues to render explicit `setting.*` fields, while `publicFunction.js` remains the single owner of facet type visibility and facet variable color-list linkage.

**Tech Stack:** Java 17, Spring Boot 3.3, Lombok, JUnit 5, Vue 2, Jest, Element UI form components.

---

## File Structure

- Modify: `goto/common/src/main/java/com/freedom/model/form/FacetParams.java`
  - Change facet type dictionary id from `29L` to `73L`.
  - Add `scaleA`, `scaleR`, `spaceA`, `spaceR`, and `radialStripPosition`.
- Create: `goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java`
  - Verify new defaults, dict ids, and R parameter generation.
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
  - Render radial-specific controls.
- Modify: `goto-web/src/constant/modify/publicFunction.js`
  - Add `radial` visibility and `facets` linkage.
- Modify: `goto-web/src/constant/modify/pieModify.js`
  - Ensure pie chart custom facet logic does not re-show cartesian facet controls for `radial`.
- Modify: `goto-web/tests/unit/constant/modify/publicFunction.spec.js`
  - Add tests for radial visibility and color-list linkage.

---

### Task 1: Backend Failing Tests

**Files:**
- Create: `goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java`

- [ ] **Step 1: Add backend tests for radial defaults and R output**

Create `goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java`:

```java
package com.freedom.model.form;

import com.freedom.cache.CacheService;
import com.freedom.domain.DictDetailCustom;
import com.freedom.model.DictSimple;
import com.freedom.model.form.base.InternalParameterItem;
import com.freedom.util.AutoParamCreateUtils;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class FacetParamsRadialDefaultsTest {

    @AfterEach
    void clearDictCache() {
        CacheService.gotoDictMapCache.invalidateAll();
    }

    @Test
    void constructor_shouldUseRadialFacetTypeDictionaryAndDefaults() {
        CacheService.gotoDictMapCache.put(73L, dict(73L, "facetType", detail("none", "无分面"), detail("radial", "环形分面")));
        CacheService.gotoDictMapCache.put(66L, dict(66L, "radialStripPosition", detail("inside", "内部"), detail("outside", "外部")));

        FacetParams params = new FacetParams();

        assertEquals(73L, params.getType().getValue().getDictId());
        assertEquals("none", params.getType().getValue().getValue());
        assertEquals(2, params.getType().getValue().getSelectList().size());
        assertEquals("T", params.getScaleA().getValue());
        assertEquals("F", params.getScaleR().getValue());
        assertEquals("T", params.getSpaceA().getValue());
        assertEquals("T", params.getSpaceR().getValue());
        assertNotNull(params.getRadialStripPosition());
        assertEquals(66L, params.getRadialStripPosition().getValue().getDictId());
        assertEquals("inside", params.getRadialStripPosition().getValue().getValue());
    }

    @Test
    void autoParamCreate_shouldEmitRadialSpecificRParameters() {
        FacetParams params = new FacetParams();
        params.setScaleA(new InternalParameterItem<>("T", false, true));
        params.setScaleR(new InternalParameterItem<>("F", false, true));
        params.setSpaceA(new InternalParameterItem<>("T", false, true));
        params.setSpaceR(new InternalParameterItem<>("T", false, true));

        SelectParam radialStrip = new SelectParam();
        radialStrip.setValue("inside");
        params.setRadialStripPosition(new InternalParameterItem<>(radialStrip, false, true));

        String result = AutoParamCreateUtils.autoParamCreate(params, FacetParams.class);

        assertTrue(result.contains("scaleA=T"));
        assertTrue(result.contains("scaleR=F"));
        assertTrue(result.contains("spaceA=T"));
        assertTrue(result.contains("spaceR=T"));
        assertTrue(result.contains("radial.strip.position=\"inside\""));
    }

    private static DictSimple dict(Long id, String name, DictDetailCustom... details) {
        DictSimple dict = new DictSimple();
        dict.setId(id);
        dict.setName(name);
        dict.setSelectList(List.of(details));
        return dict;
    }

    private static DictDetailCustom detail(String value, String label) {
        DictDetailCustom detail = new DictDetailCustom();
        detail.setValue(value);
        detail.setLabel(label);
        return detail;
    }
}
```

- [ ] **Step 2: Run the backend tests and verify they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=FacetParamsRadialDefaultsTest test
```

Expected: compilation fails because `FacetParams` does not yet have `getScaleA`, `getScaleR`, `getSpaceA`, `getSpaceR`, `getRadialStripPosition`, and `setRadialStripPosition`.

- [ ] **Step 3: Commit the failing tests**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java
git commit -m "test: 覆盖环形分面后端默认参数"
```

---

### Task 2: Backend Implementation

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/model/form/FacetParams.java`
- Test: `goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java`

- [ ] **Step 1: Add radial fields and dictionary initialization**

In `FacetParams.java`, add the new fields after `spaceY` and before `nrow`:

```java
    // scaleA
    @Syncable(nested = true)
    @AutoParamField(name = "scaleA", quotes = false)
    private InternalParameterItem<String> scaleA;
    // scaleR
    @Syncable(nested = true)
    @AutoParamField(name = "scaleR", quotes = false)
    private InternalParameterItem<String> scaleR;
    // spaceA
    @Syncable(nested = true)
    @AutoParamField(name = "spaceA", quotes = false)
    private InternalParameterItem<String> spaceA;
    // spaceR
    @Syncable(nested = true)
    @AutoParamField(name = "spaceR", quotes = false)
    private InternalParameterItem<String> spaceR;
```

Add the independent radial strip field after `stripPosition`:

```java
    // radialStripPosition
    @Syncable(nested = true)
    @AutoParamField(name = "radial.strip.position", quotes = true)
    private InternalParameterItem<SelectParam> radialStripPosition;
```

Change the facet type dictionary id:

```java
        DictSimple dictSimple = CacheService.getGotoDictById(73L);
```

Initialize the new boolean fields after `spaceY`:

```java
        this.scaleA = new InternalParameterItem<>("T", false, false);
        this.scaleR = new InternalParameterItem<>("F", false, false);
        this.spaceA = new InternalParameterItem<>("T", false, false);
        this.spaceR = new InternalParameterItem<>("T", false, false);
```

Initialize the radial strip position after existing `stripPosition` initialization:

```java
        this.radialStripPosition = new InternalParameterItem<>(new SelectParam(), false, false);
        dictSimple = CacheService.getGotoDictById(66L);
        if (dictSimple != null) {
            this.radialStripPosition.getValue().setDictId(dictSimple.getId());
            this.radialStripPosition.getValue().setValue("inside");
            this.radialStripPosition.getValue().setDictName(dictSimple.getName());
            this.radialStripPosition.getValue().setSelectList(dictSimple.getSelectList());
        }
```

- [ ] **Step 2: Run the focused backend tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=FacetParamsRadialDefaultsTest test
```

Expected: PASS.

- [ ] **Step 3: Run the common module test suite**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common test
```

Expected: PASS. If local private dependencies are missing, record the exact missing artifact in the final handoff and still keep the focused test result.

- [ ] **Step 4: Commit backend implementation**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto/common/src/main/java/com/freedom/model/form/FacetParams.java
git commit -m "feat: 新增环形分面后端参数"
```

---

### Task 3: Frontend Failing Tests

**Files:**
- Modify: `goto-web/tests/unit/constant/modify/publicFunction.spec.js`

- [ ] **Step 1: Import `facetParamsModify` in the existing public function test**

Change the require block near the top of `publicFunction.spec.js` to include `facetParamsModify`:

```js
const {
  findFilteredHeaderInfoListByFileMarkColumns,
  saveFileMarkSuccess,
  resetClassifyItemByDefault,
  facetParamsModify
} = require('@/constant/modify/publicFunction')
```

- [ ] **Step 2: Add facetParams radial linkage tests**

Append this test block to `publicFunction.spec.js`:

```js
describe('facetParamsModify radial type', () => {
  function param(visible = false, value = null) {
    return { visible, value }
  }

  function listParam(visible = false, list = []) {
    return { visible, value: { list } }
  }

  function createFacetParams(typeValue) {
    return {
      paramName: 'facetParams',
      type: { value: { value: typeValue } },
      nested: param(false, 'F'),
      scaleX: param(true, 'T'),
      scaleY: param(true, 'T'),
      spaceX: param(true, 'T'),
      spaceY: param(true, 'T'),
      scaleA: param(false, 'T'),
      scaleR: param(false, 'F'),
      spaceA: param(false, 'T'),
      spaceR: param(false, 'T'),
      nrow: param(true, 2),
      stripPosition: { visible: true, value: { value: 'top' } },
      radialStripPosition: { visible: false, value: { value: 'inside' } },
      rows: listParam(true, [{ checked: true, value: 'oldRow', params: { fileSettingIndex: 0 } }]),
      cols: listParam(true, [{ checked: true, value: 'oldCol', params: { fileSettingIndex: 0 } }]),
      facets: listParam(false, [{ checked: true, value: 'Group', params: { fileSettingIndex: 0 } }]),
      fillColorList: { visible: true, value: [{ name: 'old' }] },
      borderColorList: { visible: true, value: [{ name: 'old' }] }
    }
  }

  function createCommonParams(facetParams, optionName) {
    const settings = [
      facetParams,
      { paramName: 'nestLine', visible: true },
      { paramName: 'stripTextX', visible: true },
      { paramName: 'stripTextY', visible: true },
      { paramName: 'stripText', visible: false },
      { paramName: 'stripMargin', visible: false },
      { paramName: 'panelSpace', visible: false }
    ]

    return {
      params: { optionName },
      topParamSettingsList: [
        {
          moduleType: -1,
          paramSettingsList: settings
        }
      ],
      fileSettings: [
        {
          uploadFileInfo: {
            handsontable: {
              header: ['GT_Selected', 'Group'],
              body: [
                [1, 'A'],
                ['1', 'B'],
                [0, 'C'],
                [1, 'A']
              ]
            }
          }
        }
      ],
      vueContext: {
        $set(target, key, value) {
          target[key] = value
        }
      }
    }
  }

  test('type radial shows radial controls and hides cartesian controls', () => {
    const facetParams = createFacetParams('radial')
    const commonParams = createCommonParams(facetParams, 'type')

    facetParamsModify(commonParams)

    expect(facetParams.nested.visible).toBe(true)
    expect(facetParams.facets.visible).toBe(true)
    expect(facetParams.scaleA.visible).toBe(true)
    expect(facetParams.scaleR.visible).toBe(true)
    expect(facetParams.spaceA.visible).toBe(true)
    expect(facetParams.spaceR.visible).toBe(true)
    expect(facetParams.radialStripPosition.visible).toBe(true)
    expect(facetParams.scaleX.visible).toBe(false)
    expect(facetParams.scaleY.visible).toBe(false)
    expect(facetParams.spaceX.visible).toBe(false)
    expect(facetParams.spaceY.visible).toBe(false)
    expect(facetParams.nrow.visible).toBe(false)
    expect(facetParams.stripPosition.visible).toBe(false)
    expect(facetParams.rows.visible).toBe(false)
    expect(facetParams.cols.visible).toBe(false)
    expect(commonParams.topParamSettingsList[0].paramSettingsList.find(item => item.paramName === 'stripText').visible).toBe(true)
    expect(commonParams.topParamSettingsList[0].paramSettingsList.find(item => item.paramName === 'stripTextX').visible).toBe(false)
    expect(facetParams.fillColorList.value).toEqual([])
    expect(facetParams.borderColorList.value).toEqual([])
  })

  test('radial facets generate fill and border color lists like wrap', () => {
    const facetParams = createFacetParams('radial')
    facetParams.facets.value.list = [
      { checked: true, value: 'Group', params: { fileSettingIndex: 0 } }
    ]
    const commonParams = createCommonParams(facetParams, 'facets')

    facetParamsModify(commonParams)

    expect(facetParams.fillColorList.value).toEqual([
      {
        name: 'Group',
        listParam: {
          type: 2,
          valueType: 1,
          checkedVisible: false,
          allowDelete: false,
          allowAdd: false,
          allowEditChecked: false,
          allowEditName: false,
          allowEditValue: true,
          list: [
            { _show: true, checked: true, name: 'A', value: '#ffffff' },
            { _show: true, checked: true, name: 'B', value: '#ffffff' }
          ]
        }
      }
    ])
    expect(facetParams.borderColorList.value).toEqual(facetParams.fillColorList.value)
  })

  test('wrap hides radial-only controls', () => {
    const facetParams = createFacetParams('wrap')
    const commonParams = createCommonParams(facetParams, 'type')

    facetParamsModify(commonParams)

    expect(facetParams.stripPosition.visible).toBe(true)
    expect(facetParams.radialStripPosition.visible).toBe(false)
    expect(facetParams.scaleA.visible).toBe(false)
    expect(facetParams.scaleR.visible).toBe(false)
    expect(facetParams.spaceA.visible).toBe(false)
    expect(facetParams.spaceR.visible).toBe(false)
  })
})
```

- [ ] **Step 3: Run the focused frontend tests and verify they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: FAIL because `radial` is not handled by `facetTypeModify` and `rowsOrColsOrFacetsModify`.

- [ ] **Step 4: Commit the failing frontend tests**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/tests/unit/constant/modify/publicFunction.spec.js
git commit -m "test: 覆盖环形分面前端联动"
```

---

### Task 4: Frontend Implementation

**Files:**
- Modify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
- Modify: `goto-web/src/constant/modify/publicFunction.js`
- Modify: `goto-web/src/constant/modify/pieModify.js`
- Test: `goto-web/tests/unit/constant/modify/publicFunction.spec.js`

- [ ] **Step 1: Add radial controls to the draw facet component**

In `facetParams.vue`, add these controls after the existing `spaceY` block and before `nrow`:

```vue
      <!-- 角度轴刻度自适应 -->
      <SettingForm v-if="setting.scaleA" :container="setting.scaleA">
        <GRadio
          :parent="setting.scaleA"
          :field-name="'value'"
          :option-list="yesNoOptions"
          :form-label="'角度轴刻度自适应'"
          :modify-params="buildModifyParams('scaleA')"
          @modify="handleModify"
        />
      </SettingForm>

      <!-- 径向轴刻度自适应 -->
      <SettingForm v-if="setting.scaleR" :container="setting.scaleR">
        <GRadio
          :parent="setting.scaleR"
          :field-name="'value'"
          :option-list="yesNoOptions"
          :form-label="'径向轴刻度自适应'"
          :modify-params="buildModifyParams('scaleR')"
          @modify="handleModify"
        />
      </SettingForm>

      <!-- 角度轴面板自适应 -->
      <SettingForm v-if="setting.spaceA" :container="setting.spaceA">
        <GRadio
          :parent="setting.spaceA"
          :field-name="'value'"
          :option-list="yesNoOptions"
          :form-label="'角度轴面板自适应'"
          :modify-params="buildModifyParams('spaceA')"
          @modify="handleModify"
        />
      </SettingForm>

      <!-- 径向轴面板自适应 -->
      <SettingForm v-if="setting.spaceR" :container="setting.spaceR">
        <GRadio
          :parent="setting.spaceR"
          :field-name="'value'"
          :option-list="yesNoOptions"
          :form-label="'径向轴面板自适应'"
          :modify-params="buildModifyParams('spaceR')"
          @modify="handleModify"
        />
      </SettingForm>
```

Add the independent radial strip selector after the existing `stripPosition` block:

```vue
      <!-- 分面标签位置 -->
      <SettingForm v-if="setting.radialStripPosition" :container="setting.radialStripPosition">
        <GSelectObject
          :form-label="'分面标签位置'"
          :parent="setting.radialStripPosition.value"
          :field-name="'value'"
          :value-key="'value'"
          :option-name="'label'"
          :option-key="'value'"
          :list="setting.radialStripPosition.value.selectList"
          :modify-params="buildModifyParams('radialStripPosition')"
          @modify="handleModify"
        />
      </SettingForm>
```

Update the component comment so the `setting` prop list includes `scaleA`, `scaleR`, `spaceA`, `spaceR`, and `radialStripPosition`.

- [ ] **Step 2: Add safe visibility helpers in `publicFunction.js`**

Add these helpers near `facetTypeModify`:

```js
function setFacetSettingVisible(facetParamsSetting, visible, ...fieldNames) {
  for (const fieldName of fieldNames) {
    if (facetParamsSetting[fieldName]) {
      facetParamsSetting[fieldName].visible = visible
    }
  }
}

function getFacetTypeValue(facetParamsSetting) {
  return facetParamsSetting &&
    facetParamsSetting.type &&
    facetParamsSetting.type.value
    ? facetParamsSetting.type.value.value
    : null
}
```

- [ ] **Step 3: Update `rowsOrColsOrFacetsModify` to treat radial like wrap**

Replace the existing `wrap` branch condition:

```js
  } else if (facetParamsSetting.type.value.value === "wrap") {
```

with:

```js
  } else if (["wrap", "radial"].includes(getFacetTypeValue(facetParamsSetting))) {
```

- [ ] **Step 4: Update `facetTypeModify` visibility branches**

At the start of `facetTypeModify`, after finding `panelSpaceSetting`, add:

```js
  const facetType = getFacetTypeValue(facetParamsSetting)
```

Change every repeated `facetParamsSetting.type.value.value` condition in `facetTypeModify` to use `facetType`.

In the theme visibility branch, add `radial` alongside `wrap`:

```js
  } else if (facetType === "wrap" || facetType === "radial") {
    if (nestLineSetting) {
      nestLineSetting.visible = isBooleanTrue(facetParamsSetting.nested.value)
    }
    stripTextXSetting.visible = false
    stripTextYSetting.visible = false
    stripTextSetting.visible = true
    stripMarginSetting.visible = true
    if (panelSpaceSetting) {
      panelSpaceSetting.visible = true
    }
  }
```

Before the per-type parameter branch, hide all facet sub-parameters once:

```js
  setFacetSettingVisible(
    facetParamsSetting,
    false,
    "nested",
    "scaleX",
    "scaleY",
    "spaceX",
    "spaceY",
    "scaleA",
    "scaleR",
    "spaceA",
    "spaceR",
    "nrow",
    "stripPosition",
    "radialStripPosition",
    "rows",
    "cols",
    "facets",
    "fillColorList",
    "borderColorList"
  )
```

Then replace the current per-type branch with:

```js
  if (facetType === "grid") {
    setFacetSettingVisible(
      facetParamsSetting,
      true,
      "nested",
      "scaleX",
      "scaleY",
      "spaceX",
      "spaceY",
      "rows",
      "cols",
      "fillColorList",
      "borderColorList"
    )
  } else if (facetType === "wrap") {
    setFacetSettingVisible(
      facetParamsSetting,
      true,
      "nested",
      "scaleX",
      "scaleY",
      "nrow",
      "stripPosition",
      "facets",
      "fillColorList",
      "borderColorList"
    )
  } else if (facetType === "radial") {
    setFacetSettingVisible(
      facetParamsSetting,
      true,
      "nested",
      "scaleA",
      "scaleR",
      "spaceA",
      "spaceR",
      "radialStripPosition",
      "facets",
      "fillColorList",
      "borderColorList"
    )
  }
```

Leave the existing checked-state clearing and color-list clearing logic in place.

- [ ] **Step 5: Guard `nestedModify` for optional `nestLine`**

Keep the current behavior, but ensure it only writes `nestLineSetting.visible` when both settings exist:

```js
  if (facetParamsSetting && nestLineSetting) {
    if (isBooleanTrue(facetParamsSetting.nested.value)) {
      nestLineSetting.visible = true
    } else if (isBooleanFalse(facetParamsSetting.nested.value)) {
      nestLineSetting.visible = false
    }
  }
```

This code already exists; confirm it remains unchanged after refactoring.

- [ ] **Step 6: Update `pieModify.js` radial custom visibility**

In `goto-web/src/constant/modify/pieModify.js`, update the local `facetParamsModify` function to include `radial`:

```js
function facetParamsModify(commonParams) {
  console.log("===================facetParamsModify")
  const facetParamsSetting = localFindFromTopParamSettingsList(
    "facetParams",
    commonParams
  )
  const facetType =
    facetParamsSetting &&
    facetParamsSetting.type &&
    facetParamsSetting.type.value
      ? facetParamsSetting.type.value.value
      : null

  if (facetType === "grid") {
    facetParamsSetting.scaleX.visible = false
    facetParamsSetting.scaleY.visible = false
    facetParamsSetting.spaceX.visible = false
    facetParamsSetting.spaceY.visible = false
  } else if (facetType === "wrap") {
    facetParamsSetting.scaleX.visible = false
    facetParamsSetting.scaleY.visible = false
  } else if (facetType === "radial") {
    if (facetParamsSetting.scaleX) facetParamsSetting.scaleX.visible = false
    if (facetParamsSetting.scaleY) facetParamsSetting.scaleY.visible = false
    if (facetParamsSetting.spaceX) facetParamsSetting.spaceX.visible = false
    if (facetParamsSetting.spaceY) facetParamsSetting.spaceY.visible = false
  }
}
```

- [ ] **Step 7: Run focused frontend tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 8: Run frontend lint on touched files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/views/goto/components/form/draw/facetParams.vue src/constant/modify/publicFunction.js src/constant/modify/pieModify.js tests/unit/constant/modify/publicFunction.spec.js
```

Expected: PASS.

- [ ] **Step 9: Commit frontend implementation**

```bash
cd /mnt/f/IdeaProjects/goto-software
git add goto-web/src/views/goto/components/form/draw/facetParams.vue goto-web/src/constant/modify/publicFunction.js goto-web/src/constant/modify/pieModify.js
git commit -m "feat: 新增环形分面前端联动"
```

---

### Task 5: Full Verification and Cleanup

**Files:**
- Verify: `goto/common/src/main/java/com/freedom/model/form/FacetParams.java`
- Verify: `goto-web/src/views/goto/components/form/draw/facetParams.vue`
- Verify: `goto-web/src/constant/modify/publicFunction.js`
- Verify: `goto-web/src/constant/modify/pieModify.js`
- Verify: `goto-web/tests/unit/constant/modify/publicFunction.spec.js`
- Verify: `goto/common/src/test/java/com/freedom/model/form/FacetParamsRadialDefaultsTest.java`

- [ ] **Step 1: Search for missed old dictionary ids and radial fields**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
rg -n "getGotoDictById\\(29L\\)|radial\\.strip\\.position|radialStripPosition|scaleA|scaleR|spaceA|spaceR" goto/common/src/main/java goto/common/src/test/java goto-web/src goto-web/tests -S
```

Expected:

- No remaining `getGotoDictById(29L)` in `FacetParams.java`.
- `radial.strip.position` appears in `FacetParams.java` and `FacetParamsRadialDefaultsTest.java`.
- `radialStripPosition`, `scaleA`, `scaleR`, `spaceA`, and `spaceR` appear in backend model, frontend component, frontend linkage, and tests.

- [ ] **Step 2: Run focused backend verification**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=FacetParamsRadialDefaultsTest test
```

Expected: PASS.

- [ ] **Step 3: Run focused frontend verification**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/publicFunction.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 4: Inspect git diff**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git log -5 --oneline
git status --short
```

Expected:

- Recent commits include one backend test commit, one backend implementation commit, one frontend test commit, and one frontend implementation commit.
- `git status --short` may still show pre-existing unrelated dirty files from before this task; the only relevant tracked changes should be committed.

- [ ] **Step 5: Final handoff**

Report:

- Backend test command and result.
- Frontend test command and result.
- Lint command and result.
- Any verification command blocked by missing local dependencies, including the exact command and error summary.
