# goto-web 画图首页预览图批量懒加载 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the plot home page's index-based WebSocket preview image loading with stable-ID batch HTTP lazy loading and remove persistent frontend image caching from that page.

**Architecture:** Add a backend batch preview endpoint that accepts `groupId/typeId`, applies the same visibility filtering as `groupList()`, and returns per-card `ready/empty/failed` results. Add a small frontend API wrapper and a focused preview loader helper so `index.vue` can render structure first, observe cards near the viewport, batch visible cards into HTTP requests, and update page-local preview state by `cardKey`.

**Tech Stack:** Java 17, Spring Boot MVC, JUnit 5, Mockito static mocks, Vue 2, Element UI, Jest, IntersectionObserver.

---

## File Structure

Backend files:

- Create `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchRequest.java`
  - Request DTO for `POST /api/plot/group_preview_batch`.
- Create `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchResponse.java`
  - Response DTO with one item per requested preview.
- Modify `goto/system/src/main/java/com/freedom/service/PlotService.java`
  - Add `groupPreviewBatch(PlotGroupPreviewBatchRequest request)`.
- Modify `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
  - Implement validation, visibility filtering, stable lookup, and per-item status mapping.
- Modify `goto/system/src/main/java/com/freedom/rest/PlotController.java`
  - Add `POST group_preview_batch` endpoint.
- Create `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplGroupPreviewBatchTest.java`
  - Service-level tests for visibility, empty image, failed item, and batch limit.
- Create `goto/system/src/test/java/com/freedom/rest/PlotControllerGroupPreviewBatchTest.java`
  - Controller delegation test.

Frontend files:

- Modify `goto-web/src/api/plot.js`
  - Add `groupPreviewBatch(data)`.
- Create `goto-web/src/views/goto/plot/previewBatchLoader.js`
  - Pure helpers for card keys, group normalization, preview state, response application, and batch queueing.
- Modify `goto-web/src/views/goto/plot/index.vue`
  - Remove IndexedDB/WebSocket image logic and wire `IntersectionObserver` + batch loader.
- Create `goto-web/tests/unit/api/plot.spec.js`
  - API wrapper test.
- Create `goto-web/tests/unit/views/goto/plot/previewBatchLoader.spec.js`
  - Pure helper tests.
- Create `goto-web/tests/unit/views/goto/plot/indexSource.spec.js`
  - Source contract test that prevents reintroducing IndexedDB and `group_list_image_base64` on this page.

---

### Task 1: Backend Service Failing Tests

**Files:**
- Create: `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplGroupPreviewBatchTest.java`

- [ ] **Step 1: Write the failing service test**

Create `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplGroupPreviewBatchTest.java` with this content:

```java
package com.freedom.service.impl;

import com.freedom.bean.ComponentBean;
import com.freedom.config.UserDrawTaskProperties;
import com.freedom.model.CommonResponse;
import com.freedom.model.dto.PlotGroupPreviewBatchRequest;
import com.freedom.model.dto.PlotGroupPreviewBatchResponse;
import com.freedom.model.small.PlotGroupSmall;
import com.freedom.model.small.PlotTypeSmall;
import com.freedom.service.async.AsyncUserPlotTypeStatService;
import com.freedom.service.dto.JwtUserDto;
import com.freedom.service.dto.UserLoginDto;
import com.freedom.util.PlotGroupUtils;
import com.freedom.util.PlotTypeUtils;
import com.freedom.util.SecurityUtils;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.MockedStatic;
import org.mockito.junit.jupiter.MockitoExtension;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.mockito.Mockito.CALLS_REAL_METHODS;
import static org.mockito.Mockito.mockStatic;

@ExtendWith(MockitoExtension.class)
class PlotServiceImplGroupPreviewBatchTest {

    @Mock
    private ComponentBean componentBean;

    @Mock
    private UserDrawTaskProperties userDrawTaskProperties;

    @Mock
    private AsyncUserPlotTypeStatService asyncUserPlotTypeStatService;

    private PlotServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new PlotServiceImpl(componentBean, userDrawTaskProperties, asyncUserPlotTypeStatService);
    }

    @Test
    void groupPreviewBatch_shouldReturnReadyEmptyAndFailedByStableIdsForNormalUser() throws Exception {
        List<PlotGroupSmall> cachedGroupList = List.of(
                buildPlotGroupSmall(
                        1L,
                        "可见分组",
                        1,
                        1,
                        List.of(
                                buildPlotTypeSmall(11L, "visible.png", "visible-type", 1),
                                buildPlotTypeSmall(12L, "empty.png", "empty-type", 1),
                                buildPlotTypeSmall(13L, "boom.png", "boom-type", 1),
                                buildPlotTypeSmall(14L, "hidden.png", "hidden-type", 0)
                        )
                ),
                buildPlotGroupSmall(
                        2L,
                        "隐藏分组",
                        2,
                        0,
                        List.of(buildPlotTypeSmall(21L, "hidden-group.png", "hidden-group-type", 1))
                )
        );

        PlotGroupPreviewBatchRequest request = buildRequest(
                item(1L, 11L),
                item(1L, 12L),
                item(1L, 13L),
                item(1L, 14L),
                item(2L, 21L),
                item(99L, 99L)
        );

        try (MockedStatic<SecurityUtils> securityUtilsMock = mockStatic(SecurityUtils.class);
             MockedStatic<PlotGroupUtils> plotGroupUtilsMock = mockStatic(PlotGroupUtils.class, CALLS_REAL_METHODS);
             MockedStatic<PlotTypeUtils> plotTypeUtilsMock = mockStatic(PlotTypeUtils.class)) {
            securityUtilsMock.when(SecurityUtils::getCurrentUser).thenReturn(buildCurrentUser(false));
            plotGroupUtilsMock.when(() -> PlotGroupUtils.obtainStorePlotGroupSmallList(componentBean)).thenReturn(cachedGroupList);
            plotTypeUtilsMock.when(() -> PlotTypeUtils.obtainPlotSampleResultBase64(componentBean, 1L, "visible.png", "visible-type"))
                    .thenReturn("data:image/png;base64,visible");
            plotTypeUtilsMock.when(() -> PlotTypeUtils.obtainPlotSampleResultBase64(componentBean, 1L, "empty.png", "empty-type"))
                    .thenReturn("");
            plotTypeUtilsMock.when(() -> PlotTypeUtils.obtainPlotSampleResultBase64(componentBean, 1L, "boom.png", "boom-type"))
                    .thenThrow(new RuntimeException("read failed"));

            CommonResponse response = service.groupPreviewBatch(request);

            assertEquals(200, response.getCode());
            PlotGroupPreviewBatchResponse data = (PlotGroupPreviewBatchResponse) response.getData();
            assertNotNull(data);
            assertEquals(6, data.getItems().size());

            assertPreview(data.getItems().get(0), 1L, 11L, "1:11", "ready", "data:image/png;base64,visible");
            assertPreview(data.getItems().get(1), 1L, 12L, "1:12", "empty", null);
            assertPreview(data.getItems().get(2), 1L, 13L, "1:13", "failed", null);
            assertPreview(data.getItems().get(3), 1L, 14L, "1:14", "failed", null);
            assertPreview(data.getItems().get(4), 2L, 21L, "2:21", "failed", null);
            assertPreview(data.getItems().get(5), 99L, 99L, "99:99", "failed", null);
        }
    }

    @Test
    void groupPreviewBatch_shouldKeepHiddenItemsForAdmin() throws Exception {
        List<PlotGroupSmall> cachedGroupList = List.of(
                buildPlotGroupSmall(
                        2L,
                        "隐藏分组",
                        2,
                        0,
                        List.of(buildPlotTypeSmall(21L, "hidden-group.png", "hidden-group-type", 0))
                )
        );

        PlotGroupPreviewBatchRequest request = buildRequest(item(2L, 21L));

        try (MockedStatic<SecurityUtils> securityUtilsMock = mockStatic(SecurityUtils.class);
             MockedStatic<PlotGroupUtils> plotGroupUtilsMock = mockStatic(PlotGroupUtils.class, CALLS_REAL_METHODS);
             MockedStatic<PlotTypeUtils> plotTypeUtilsMock = mockStatic(PlotTypeUtils.class)) {
            securityUtilsMock.when(SecurityUtils::getCurrentUser).thenReturn(buildCurrentUser(true));
            plotGroupUtilsMock.when(() -> PlotGroupUtils.obtainStorePlotGroupSmallList(componentBean)).thenReturn(cachedGroupList);
            plotTypeUtilsMock.when(() -> PlotTypeUtils.obtainPlotSampleResultBase64(componentBean, 2L, "hidden-group.png", "hidden-group-type"))
                    .thenReturn("data:image/png;base64,admin");

            CommonResponse response = service.groupPreviewBatch(request);

            assertEquals(200, response.getCode());
            PlotGroupPreviewBatchResponse data = (PlotGroupPreviewBatchResponse) response.getData();
            assertPreview(data.getItems().get(0), 2L, 21L, "2:21", "ready", "data:image/png;base64,admin");
        }
    }

    @Test
    void groupPreviewBatch_shouldReturnEmptyItemsForEmptyRequest() {
        CommonResponse response = service.groupPreviewBatch(new PlotGroupPreviewBatchRequest());

        assertEquals(200, response.getCode());
        PlotGroupPreviewBatchResponse data = (PlotGroupPreviewBatchResponse) response.getData();
        assertNotNull(data);
        assertEquals(0, data.getItems().size());
    }

    @Test
    void groupPreviewBatch_shouldRejectMoreThanTwentyItems() {
        List<PlotGroupPreviewBatchRequest.Item> items = new ArrayList<>();
        for (long i = 1; i <= 21; i++) {
            items.add(item(1L, i));
        }
        PlotGroupPreviewBatchRequest request = new PlotGroupPreviewBatchRequest();
        request.setItems(items);

        CommonResponse response = service.groupPreviewBatch(request);

        assertEquals("单批最多只能请求20张预览图", response.getMsg());
    }

    private static PlotGroupPreviewBatchRequest buildRequest(PlotGroupPreviewBatchRequest.Item... items) {
        PlotGroupPreviewBatchRequest request = new PlotGroupPreviewBatchRequest();
        request.setItems(List.of(items));
        return request;
    }

    private static PlotGroupPreviewBatchRequest.Item item(Long groupId, Long typeId) {
        PlotGroupPreviewBatchRequest.Item item = new PlotGroupPreviewBatchRequest.Item();
        item.setGroupId(groupId);
        item.setTypeId(typeId);
        return item;
    }

    private static JwtUserDto buildCurrentUser(boolean isAdmin) {
        UserLoginDto userLoginDto = new UserLoginDto();
        userLoginDto.setId(1L);
        userLoginDto.setIsAdmin(isAdmin);
        return new JwtUserDto(userLoginDto, List.of(), List.of());
    }

    private static PlotGroupSmall buildPlotGroupSmall(Long id, String name, Integer sequence, Integer show, List<PlotTypeSmall> plotTypeSmallList) {
        PlotGroupSmall plotGroupSmall = new PlotGroupSmall(id, name, name, sequence, show);
        plotGroupSmall.setPlotTypeSmallList(plotTypeSmallList);
        return plotGroupSmall;
    }

    private static PlotTypeSmall buildPlotTypeSmall(Long id, String fileName, String folderName, Integer show) throws Exception {
        PlotTypeSmall plotTypeSmall = new PlotTypeSmall(id, fileName, folderName, null);
        Field field = PlotTypeSmall.class.getDeclaredField("show");
        field.setAccessible(true);
        field.set(plotTypeSmall, show);
        return plotTypeSmall;
    }

    private static void assertPreview(
            PlotGroupPreviewBatchResponse.Item item,
            Long groupId,
            Long typeId,
            String cardKey,
            String status,
            String image
    ) {
        assertEquals(groupId, item.getGroupId());
        assertEquals(typeId, item.getTypeId());
        assertEquals(cardKey, item.getCardKey());
        assertEquals(status, item.getStatus());
        assertEquals(image, item.getImage());
    }
}
```

- [ ] **Step 2: Run the service test and verify it fails**

Run:

```bash
mvn -pl system -Dtest=PlotServiceImplGroupPreviewBatchTest test
```

Expected: FAIL at compilation because `PlotGroupPreviewBatchRequest`, `PlotGroupPreviewBatchResponse`, and `PlotServiceImpl#groupPreviewBatch` do not exist.

- [ ] **Step 3: Commit the failing test**

Run:

```bash
git add goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplGroupPreviewBatchTest.java
git commit -m "test: cover plot preview batch service"
```

---

### Task 2: Backend DTOs And Service Implementation

**Files:**
- Create: `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchRequest.java`
- Create: `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchResponse.java`
- Modify: `goto/system/src/main/java/com/freedom/service/PlotService.java`
- Modify: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
- Test: `goto/system/src/test/java/com/freedom/service/impl/PlotServiceImplGroupPreviewBatchTest.java`

- [ ] **Step 1: Add request DTO**

Create `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchRequest.java`:

```java
package com.freedom.model.dto;

import lombok.Data;

import java.io.Serializable;
import java.util.List;

@Data
public class PlotGroupPreviewBatchRequest implements Serializable {
    private static final long serialVersionUID = -3251352201503646244L;

    private List<Item> items;

    @Data
    public static class Item implements Serializable {
        private static final long serialVersionUID = 2810787216559789918L;

        private Long groupId;
        private Long typeId;
    }
}
```

- [ ] **Step 2: Add response DTO**

Create `goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchResponse.java`:

```java
package com.freedom.model.dto;

import lombok.Data;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

@Data
public class PlotGroupPreviewBatchResponse implements Serializable {
    private static final long serialVersionUID = 7145607305457840423L;

    private List<Item> items = new ArrayList<>();

    @Data
    public static class Item implements Serializable {
        private static final long serialVersionUID = -2232274727302519665L;

        private Long groupId;
        private Long typeId;
        private String cardKey;
        private String status;
        private String image;
    }
}
```

- [ ] **Step 3: Add the service contract**

Modify `goto/system/src/main/java/com/freedom/service/PlotService.java`:

```java
import com.freedom.model.dto.PlotGroupPreviewBatchRequest;
```

Add this method after `groupList()`:

```java
    /**
     * Plot分组预览图批量懒加载
     */
    CommonResponse groupPreviewBatch(PlotGroupPreviewBatchRequest request);
```

- [ ] **Step 4: Add service imports and constants**

Modify `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`.

Add imports:

```java
import com.freedom.model.dto.PlotGroupPreviewBatchRequest;
import com.freedom.model.dto.PlotGroupPreviewBatchResponse;
import com.freedom.model.small.PlotGroupSmall;
import com.freedom.model.small.PlotTypeSmall;
```

Add this constant near the top of the class:

```java
    private static final int GROUP_PREVIEW_BATCH_LIMIT = 20;
    private static final String PREVIEW_STATUS_READY = "ready";
    private static final String PREVIEW_STATUS_EMPTY = "empty";
    private static final String PREVIEW_STATUS_FAILED = "failed";
```

- [ ] **Step 5: Implement `groupPreviewBatch`**

Add this method after `groupList()` in `PlotServiceImpl`:

```java
    @Override
    public CommonResponse groupPreviewBatch(PlotGroupPreviewBatchRequest request) {
        PlotGroupPreviewBatchResponse response = new PlotGroupPreviewBatchResponse();
        if (request == null || ObjectUtils.isEmpty(request.getItems())) {
            return ResponseUtils.successData(response);
        }
        if (request.getItems().size() > GROUP_PREVIEW_BATCH_LIMIT) {
            return ResponseUtils.failureMsg("单批最多只能请求20张预览图");
        }

        boolean includeHidden = PlotGroupUtils.currentUserCanViewHiddenPlot();
        List<PlotGroupSmall> visibleGroupList = PlotGroupUtils.filterPlotGroupSmallListByVisibility(
                PlotGroupUtils.obtainStorePlotGroupSmallList(componentBean),
                includeHidden
        );

        for (PlotGroupPreviewBatchRequest.Item requestItem : request.getItems()) {
            response.getItems().add(buildPreviewResponseItem(visibleGroupList, requestItem));
        }
        return ResponseUtils.successData(response);
    }
```

- [ ] **Step 6: Implement preview item helpers**

Add these private methods in `PlotServiceImpl`:

```java
    private PlotGroupPreviewBatchResponse.Item buildPreviewResponseItem(
            List<PlotGroupSmall> visibleGroupList,
            PlotGroupPreviewBatchRequest.Item requestItem
    ) {
        PlotGroupPreviewBatchResponse.Item responseItem = new PlotGroupPreviewBatchResponse.Item();
        Long groupId = requestItem == null ? null : requestItem.getGroupId();
        Long typeId = requestItem == null ? null : requestItem.getTypeId();
        responseItem.setGroupId(groupId);
        responseItem.setTypeId(typeId);
        responseItem.setCardKey(buildPreviewCardKey(groupId, typeId));

        if (groupId == null || typeId == null) {
            responseItem.setStatus(PREVIEW_STATUS_FAILED);
            return responseItem;
        }

        PlotGroupSmall group = findPreviewGroup(visibleGroupList, groupId);
        PlotTypeSmall type = findPreviewType(group, typeId);
        if (group == null || type == null) {
            responseItem.setStatus(PREVIEW_STATUS_FAILED);
            return responseItem;
        }

        try {
            String imageBase64 = PlotTypeUtils.obtainPlotSampleResultBase64(
                    componentBean,
                    group.getId(),
                    type.getFileName(),
                    type.getFolderName()
            );
            if (ObjectUtils.isEmpty(imageBase64)) {
                responseItem.setStatus(PREVIEW_STATUS_EMPTY);
                return responseItem;
            }
            responseItem.setStatus(PREVIEW_STATUS_READY);
            responseItem.setImage(imageBase64);
            return responseItem;
        } catch (Exception e) {
            responseItem.setStatus(PREVIEW_STATUS_FAILED);
            return responseItem;
        }
    }

    private String buildPreviewCardKey(Long groupId, Long typeId) {
        return String.valueOf(groupId) + ":" + String.valueOf(typeId);
    }

    private PlotGroupSmall findPreviewGroup(List<PlotGroupSmall> groupList, Long groupId) {
        if (ObjectUtils.isEmpty(groupList) || groupId == null) {
            return null;
        }
        for (PlotGroupSmall group : groupList) {
            if (group != null && groupId.equals(group.getId())) {
                return group;
            }
        }
        return null;
    }

    private PlotTypeSmall findPreviewType(PlotGroupSmall group, Long typeId) {
        if (group == null || ObjectUtils.isEmpty(group.getPlotTypeSmallList()) || typeId == null) {
            return null;
        }
        for (PlotTypeSmall type : group.getPlotTypeSmallList()) {
            if (type != null && typeId.equals(type.getId())) {
                return type;
            }
        }
        return null;
    }
```

- [ ] **Step 7: Run the service test and verify it passes**

Run:

```bash
mvn -pl system -Dtest=PlotServiceImplGroupPreviewBatchTest test
```

Expected: PASS.

- [ ] **Step 8: Commit backend service implementation**

Run:

```bash
git add \
  goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchRequest.java \
  goto/system/src/main/java/com/freedom/model/dto/PlotGroupPreviewBatchResponse.java \
  goto/system/src/main/java/com/freedom/service/PlotService.java \
  goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java
git commit -m "feat: add plot preview batch service"
```

---

### Task 3: Backend Controller Endpoint

**Files:**
- Modify: `goto/system/src/main/java/com/freedom/rest/PlotController.java`
- Create: `goto/system/src/test/java/com/freedom/rest/PlotControllerGroupPreviewBatchTest.java`

- [ ] **Step 1: Write the failing controller delegation test**

Create `goto/system/src/test/java/com/freedom/rest/PlotControllerGroupPreviewBatchTest.java`:

```java
package com.freedom.rest;

import com.freedom.model.CommonResponse;
import com.freedom.model.dto.PlotGroupPreviewBatchRequest;
import com.freedom.service.PlotService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.http.ResponseEntity;

import static org.junit.jupiter.api.Assertions.assertSame;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class PlotControllerGroupPreviewBatchTest {

    @Mock
    private PlotService plotService;

    @Test
    void groupPreviewBatch_shouldDelegateToPlotService() {
        PlotController controller = new PlotController(plotService);
        PlotGroupPreviewBatchRequest request = new PlotGroupPreviewBatchRequest();
        CommonResponse expected = new CommonResponse();
        expected.setCode(200);
        when(plotService.groupPreviewBatch(request)).thenReturn(expected);

        ResponseEntity<Object> response = controller.groupPreviewBatch(request);

        assertSame(expected, response.getBody());
        verify(plotService).groupPreviewBatch(request);
    }
}
```

- [ ] **Step 2: Run the controller test and verify it fails**

Run:

```bash
mvn -pl system -Dtest=PlotControllerGroupPreviewBatchTest test
```

Expected: FAIL at compilation because `PlotController#groupPreviewBatch` does not exist.

- [ ] **Step 3: Add controller imports**

Modify `goto/system/src/main/java/com/freedom/rest/PlotController.java`.

Add import:

```java
import com.freedom.model.dto.PlotGroupPreviewBatchRequest;
```

- [ ] **Step 4: Add controller endpoint**

Add this method after `groupList()`:

```java
    @PostMapping(value = "group_preview_batch")
    @Log("Plot分组预览图批量数据")
    @Operation(description = "Plot分组预览图批量数据")
    @PreAuthorize("@el.check('plot:group_list')")
    public ResponseEntity<Object> groupPreviewBatch(@RequestBody PlotGroupPreviewBatchRequest request) {
        return new ResponseEntity<>(plotService.groupPreviewBatch(request), HttpStatus.OK);
    }
```

Use `plot:group_list` intentionally so preview access follows the existing plot home page permission.

- [ ] **Step 5: Run backend endpoint tests**

Run:

```bash
mvn -pl system -Dtest=PlotControllerGroupPreviewBatchTest,PlotServiceImplGroupPreviewBatchTest,PlotServiceImplPlotGroupVisibilityTest test
```

Expected: PASS.

- [ ] **Step 6: Commit backend endpoint**

Run:

```bash
git add \
  goto/system/src/main/java/com/freedom/rest/PlotController.java \
  goto/system/src/test/java/com/freedom/rest/PlotControllerGroupPreviewBatchTest.java
git commit -m "feat: expose plot preview batch endpoint"
```

---

### Task 4: Frontend API Wrapper

**Files:**
- Modify: `goto-web/src/api/plot.js`
- Create: `goto-web/tests/unit/api/plot.spec.js`

- [ ] **Step 1: Write the failing API wrapper test**

Create `goto-web/tests/unit/api/plot.spec.js`:

```js
/* eslint-env jest */
const mockRequest = jest.fn()

jest.mock("@/utils/request", () => mockRequest)

const { groupPreviewBatch } = require("@/api/plot")

describe("plot api", () => {
  beforeEach(() => {
    mockRequest.mockReset()
  })

  test("posts group preview batch request to backend", () => {
    const data = {
      items: [
        { groupId: 1, typeId: 11 },
        { groupId: 1, typeId: 12 }
      ]
    }

    groupPreviewBatch(data)

    expect(mockRequest).toHaveBeenCalledWith({
      url: "api/plot/group_preview_batch",
      method: "post",
      data
    })
  })
})
```

- [ ] **Step 2: Run the API test and verify it fails**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/api/plot.spec.js
```

Expected: FAIL because `groupPreviewBatch` is not exported.

- [ ] **Step 3: Add API wrapper**

Modify `goto-web/src/api/plot.js`.

Add this function after `groupList()`:

```js
export function groupPreviewBatch(data) {
  return request({
    url: "api/plot/group_preview_batch",
    method: "post",
    data
  })
}
```

Add it to the default export:

```js
export default {
  groupList,
  groupPreviewBatch,
  groupNameList,
  plotData,
  customDrawPlot,
  customCancelDrawPlot
}
```

- [ ] **Step 4: Run the API test and verify it passes**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/api/plot.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit frontend API wrapper**

Run:

```bash
git add goto-web/src/api/plot.js goto-web/tests/unit/api/plot.spec.js
git commit -m "feat: add plot preview batch api"
```

---

### Task 5: Frontend Preview Loader Helper

**Files:**
- Create: `goto-web/src/views/goto/plot/previewBatchLoader.js`
- Create: `goto-web/tests/unit/views/goto/plot/previewBatchLoader.spec.js`

- [ ] **Step 1: Write failing helper tests**

Create `goto-web/tests/unit/views/goto/plot/previewBatchLoader.spec.js`:

```js
/* eslint-env jest */

const {
  createCardKey,
  normalizeGroupsForPreview,
  createInitialPreviewMap,
  applyPreviewBatchResult,
  createPreviewBatchQueue
} = require("@/views/goto/plot/previewBatchLoader")

describe("previewBatchLoader", () => {
  test("creates stable card keys from groupId and typeId", () => {
    expect(createCardKey(1, 11)).toBe("1:11")
  })

  test("normalizes groups with cardKey without mutating source", () => {
    const source = [
      {
        id: 1,
        name: "Alpha",
        plotTypeSmallList: [{ id: 11 }, { id: 12 }]
      }
    ]

    const result = normalizeGroupsForPreview(source)

    expect(result[0].plotTypeSmallList.map(item => item.cardKey)).toEqual(["1:11", "1:12"])
    expect(source[0].plotTypeSmallList[0].cardKey).toBeUndefined()
  })

  test("creates idle preview state for each card", () => {
    const groups = normalizeGroupsForPreview([
      { id: 1, plotTypeSmallList: [{ id: 11 }, { id: 12 }] }
    ])

    expect(createInitialPreviewMap(groups)).toEqual({
      "1:11": { status: "idle", image: null, retryCount: 0, error: null },
      "1:12": { status: "idle", image: null, retryCount: 0, error: null }
    })
  })

  test("applies ready empty and failed results by cardKey", () => {
    const current = {
      "1:11": { status: "loading", image: null, retryCount: 0, error: null },
      "1:12": { status: "loading", image: null, retryCount: 0, error: null },
      "1:13": { status: "loading", image: null, retryCount: 0, error: null }
    }

    const result = applyPreviewBatchResult(current, [
      { cardKey: "1:11", status: "ready", image: "image-11" },
      { cardKey: "1:12", status: "empty", image: null },
      { cardKey: "1:13", status: "failed", image: null },
      { cardKey: "9:99", status: "ready", image: "ignored" }
    ])

    expect(result["1:11"]).toEqual({ status: "ready", image: "image-11", retryCount: 0, error: null })
    expect(result["1:12"]).toEqual({ status: "empty", image: null, retryCount: 0, error: null })
    expect(result["1:13"]).toEqual({ status: "failed", image: null, retryCount: 0, error: "failed" })
    expect(result["9:99"]).toBeUndefined()
  })

  test("batches queued cards and caps one request at twenty items", async() => {
    let timer
    const loadBatch = jest.fn(() => Promise.resolve({ data: { items: [] } }))
    const queue = createPreviewBatchQueue({
      loadBatch,
      delay: 10,
      batchSize: 20,
      setTimeoutFn: fn => {
        timer = fn
        return 1
      },
      clearTimeoutFn: jest.fn()
    })

    for (let index = 1; index <= 21; index++) {
      queue.enqueue({ groupId: 1, typeId: index, cardKey: `1:${index}` })
    }

    timer()
    await Promise.resolve()

    expect(loadBatch).toHaveBeenCalledTimes(1)
    expect(loadBatch.mock.calls[0][0].items).toHaveLength(20)
    expect(queue.size()).toBe(1)
  })
})
```

- [ ] **Step 2: Run helper tests and verify they fail**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/views/goto/plot/previewBatchLoader.spec.js
```

Expected: FAIL because `previewBatchLoader.js` does not exist.

- [ ] **Step 3: Implement helper**

Create `goto-web/src/views/goto/plot/previewBatchLoader.js`:

```js
export const PREVIEW_STATUS = {
  IDLE: "idle",
  QUEUED: "queued",
  LOADING: "loading",
  READY: "ready",
  EMPTY: "empty",
  FAILED: "failed"
}

export function createCardKey(groupId, typeId) {
  return `${groupId}:${typeId}`
}

export function normalizeGroupsForPreview(groupList) {
  if (!Array.isArray(groupList)) {
    return []
  }
  return groupList.map(group => ({
    ...group,
    plotTypeSmallList: Array.isArray(group.plotTypeSmallList)
      ? group.plotTypeSmallList.map(plotType => ({
        ...plotType,
        cardKey: createCardKey(group.id, plotType.id)
      }))
      : []
  }))
}

export function createInitialPreviewMap(groupList) {
  return groupList.reduce((previewMap, group) => {
    group.plotTypeSmallList.forEach(plotType => {
      previewMap[plotType.cardKey] = {
        status: PREVIEW_STATUS.IDLE,
        image: null,
        retryCount: 0,
        error: null
      }
    })
    return previewMap
  }, {})
}

export function applyPreviewBatchResult(currentMap, items) {
  const nextMap = { ...currentMap }
  if (!Array.isArray(items)) {
    return nextMap
  }
  items.forEach(item => {
    if (!item || !item.cardKey || !nextMap[item.cardKey]) {
      return
    }
    if (item.status === PREVIEW_STATUS.READY) {
      nextMap[item.cardKey] = {
        ...nextMap[item.cardKey],
        status: PREVIEW_STATUS.READY,
        image: item.image || null,
        error: null
      }
      return
    }
    if (item.status === PREVIEW_STATUS.EMPTY) {
      nextMap[item.cardKey] = {
        ...nextMap[item.cardKey],
        status: PREVIEW_STATUS.EMPTY,
        image: null,
        error: null
      }
      return
    }
    nextMap[item.cardKey] = {
      ...nextMap[item.cardKey],
      status: PREVIEW_STATUS.FAILED,
      image: null,
      error: "failed"
    }
  })
  return nextMap
}

export function createPreviewBatchQueue({
  loadBatch,
  delay = 100,
  batchSize = 20,
  setTimeoutFn = window.setTimeout.bind(window),
  clearTimeoutFn = window.clearTimeout.bind(window)
}) {
  const pendingMap = new Map()
  let timer = null

  function flush() {
    timer = null
    const items = Array.from(pendingMap.values()).slice(0, batchSize)
    items.forEach(item => pendingMap.delete(item.cardKey))
    if (items.length === 0) {
      return Promise.resolve()
    }
    return loadBatch({
      items: items.map(item => ({
        groupId: item.groupId,
        typeId: item.typeId
      }))
    })
  }

  function schedule() {
    if (timer !== null) {
      return
    }
    timer = setTimeoutFn(flush, delay)
  }

  return {
    enqueue(item) {
      if (!item || !item.cardKey || pendingMap.has(item.cardKey)) {
        return
      }
      pendingMap.set(item.cardKey, item)
      schedule()
    },
    flush,
    clear() {
      if (timer !== null) {
        clearTimeoutFn(timer)
        timer = null
      }
      pendingMap.clear()
    },
    size() {
      return pendingMap.size
    }
  }
}
```

- [ ] **Step 4: Run helper tests and verify they pass**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/views/goto/plot/previewBatchLoader.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit helper**

Run:

```bash
git add \
  goto-web/src/views/goto/plot/previewBatchLoader.js \
  goto-web/tests/unit/views/goto/plot/previewBatchLoader.spec.js
git commit -m "feat: add plot preview batch loader"
```

---

### Task 6: Wire Plot Page To Batch Lazy Loading

**Files:**
- Modify: `goto-web/src/views/goto/plot/index.vue`
- Create: `goto-web/tests/unit/views/goto/plot/indexSource.spec.js`

- [ ] **Step 1: Write source contract test**

Create `goto-web/tests/unit/views/goto/plot/indexSource.spec.js`:

```js
/* eslint-env jest */
const fs = require("fs")
const path = require("path")

const sourcePath = path.resolve(
  __dirname,
  "../../../../../src/views/goto/plot/index.vue"
)

function readSource() {
  return fs.readFileSync(sourcePath, "utf8")
}

describe("plot index preview loading contract", () => {
  test("does not use IndexedDB or websocket for plot preview images", () => {
    const source = readSource()

    expect(source).not.toContain("IndexedDBUtils")
    expect(source).not.toContain("plot_draw_images")
    expect(source).not.toContain("commonWebSocketMessageSendToServer")
    expect(source).not.toContain("group_list_image_base64")
    expect(source).not.toContain("groupListImageBase64")
    expect(source).not.toContain("getKey(i, j)")
  })

  test("uses stable cardKey batch lazy loading primitives", () => {
    const source = readSource()

    expect(source).toContain("previewMap")
    expect(source).toContain("IntersectionObserver")
    expect(source).toContain("groupPreviewBatch")
    expect(source).toContain("createCardKey")
    expect(source).toContain("queuePreviewLoad")
  })
})
```

- [ ] **Step 2: Run source contract test and verify it fails**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/views/goto/plot/indexSource.spec.js
```

Expected: FAIL because `index.vue` still contains IndexedDB and WebSocket preview code.

- [ ] **Step 3: Replace image template with preview state**

Modify the card wrapper inside `goto-web/src/views/goto/plot/index.vue`.

Replace the current `v-if="CommonUtils.isNotEmptyString(plotType.imageBase64)"` block with:

```vue
            <template v-if="getPreviewState(plotType.cardKey).status === PREVIEW_STATUS.READY">
              <el-image :src="getPreviewState(plotType.cardKey).image" fit="contain">
                <div slot="placeholder" class="image-loading">
                  <i class="el-icon-loading" />
                </div>
                <div slot="error" class="image-slot">
                  <i class="el-icon-picture-outline" />
                </div>
              </el-image>
            </template>
            <div v-else-if="getPreviewState(plotType.cardKey).status === PREVIEW_STATUS.EMPTY" class="image-slot empty-state">
              <i class="el-icon-data-analysis" />
              <span>暂无预览</span>
            </div>
            <div v-else-if="getPreviewState(plotType.cardKey).status === PREVIEW_STATUS.FAILED" class="image-slot failed-state">
              <i class="el-icon-picture-outline" />
              <span>加载失败</span>
            </div>
            <div v-else class="image-loading">
              <i class="el-icon-loading" />
            </div>
```

- [ ] **Step 4: Add card ref and observer metadata**

Modify the `plot-card` element in the template to include stable metadata:

```vue
        <div v-for="plotType in group.plotTypeSmallList" :key="plotType.id" class="plot-card"
          :data-card-key="plotType.cardKey"
          :data-group-id="group.id"
          :data-type-id="plotType.id"
          ref="plotCards"
          @click="drawDesigns(group, plotType)">
```

- [ ] **Step 5: Replace imports**

Modify the script imports in `index.vue`.

Remove:

```js
import { v4 as uuidv4 } from "uuid"
```

Add:

```js
import {
  PREVIEW_STATUS,
  applyPreviewBatchResult,
  createCardKey,
  createInitialPreviewMap,
  createPreviewBatchQueue,
  normalizeGroupsForPreview
} from "./previewBatchLoader"
```

- [ ] **Step 6: Replace data state**

In `data()`, remove these fields:

```js
      databaseName: "goto_software",
      version: 1,
      storeName: "plot_draw_images",
      db: null,
```

Add these fields:

```js
      previewMap: {},
      previewQueue: null,
      previewObserver: null,
      destroyed: false,
      PREVIEW_STATUS,
```

- [ ] **Step 7: Remove IndexedDB setup from `created()`**

Delete this block from `created()`:

```js
    // 打开数据库
    this.IndexedDBUtils.openDB(
      this.databaseName,
      this.version,
      this.storeName
    ).then(db => {
      this.db = db
    })
```

- [ ] **Step 8: Replace mounted structure loading**

Replace the `mounted()` success branch with:

```js
      .then(res => {
        if (res.code === 200) {
          this.groupList = normalizeGroupsForPreview(res.data)
          this.previewMap = createInitialPreviewMap(this.groupList)
          this.previewQueue = createPreviewBatchQueue({
            loadBatch: this.loadPreviewBatch
          })
          this.$nextTick(() => {
            this.observePreviewCards()
            this.$emit("changeMenu")
          })
        }
      })
```

- [ ] **Step 9: Add lifecycle cleanup**

Add `beforeDestroy()` to the component:

```js
  beforeDestroy() {
    this.destroyed = true
    if (this.previewObserver) {
      this.previewObserver.disconnect()
      this.previewObserver = null
    }
    if (this.previewQueue) {
      this.previewQueue.clear()
      this.previewQueue = null
    }
  },
```

- [ ] **Step 10: Replace old image methods with preview methods**

Remove the `getKey`, `setImageBase64FromIndexedDB`, and `groupListImageBase64` methods from `methods`.

Add these methods:

```js
    getPreviewState(cardKey) {
      return this.previewMap[cardKey] || {
        status: PREVIEW_STATUS.IDLE,
        image: null,
        retryCount: 0,
        error: null
      }
    },
    observePreviewCards() {
      if (this.previewObserver) {
        this.previewObserver.disconnect()
      }
      if (typeof IntersectionObserver === "undefined") {
        this.queueAllIdlePreviews()
        return
      }
      this.previewObserver = new IntersectionObserver(
        entries => {
          entries.forEach(entry => {
            if (!entry.isIntersecting) {
              return
            }
            this.queuePreviewLoad(entry.target)
          })
        },
        {
          root: null,
          rootMargin: "240px 0px",
          threshold: 0.01
        }
      )

      const cards = Array.isArray(this.$refs.plotCards)
        ? this.$refs.plotCards
        : [this.$refs.plotCards].filter(Boolean)
      cards.forEach(card => this.previewObserver.observe(card))
    },
    queueAllIdlePreviews() {
      if (!Array.isArray(this.groupList)) {
        return
      }
      this.groupList.forEach(group => {
        group.plotTypeSmallList.forEach(plotType => {
          this.queuePreviewItem(group.id, plotType.id, plotType.cardKey)
        })
      })
    },
    queuePreviewLoad(cardEl) {
      const groupId = Number(cardEl.dataset.groupId)
      const typeId = Number(cardEl.dataset.typeId)
      const cardKey = cardEl.dataset.cardKey || createCardKey(groupId, typeId)
      this.queuePreviewItem(groupId, typeId, cardKey)
      if (this.previewObserver) {
        this.previewObserver.unobserve(cardEl)
      }
    },
    queuePreviewItem(groupId, typeId, cardKey) {
      const current = this.getPreviewState(cardKey)
      if ([PREVIEW_STATUS.QUEUED, PREVIEW_STATUS.LOADING, PREVIEW_STATUS.READY, PREVIEW_STATUS.EMPTY].includes(current.status)) {
        return
      }
      if (current.status === PREVIEW_STATUS.FAILED && current.retryCount >= 1) {
        return
      }
      this.$set(this.previewMap, cardKey, {
        ...current,
        status: PREVIEW_STATUS.QUEUED,
        retryCount: current.status === PREVIEW_STATUS.FAILED ? current.retryCount + 1 : current.retryCount,
        error: null
      })
      this.previewQueue.enqueue({ groupId, typeId, cardKey })
    },
    loadPreviewBatch(payload) {
      const loadingKeys = payload.items.map(item => createCardKey(item.groupId, item.typeId))
      loadingKeys.forEach(cardKey => {
        const current = this.getPreviewState(cardKey)
        this.$set(this.previewMap, cardKey, {
          ...current,
          status: PREVIEW_STATUS.LOADING,
          error: null
        })
      })
      return crudPlot.groupPreviewBatch(payload)
        .then(res => {
          if (this.destroyed) {
            return res
          }
          if (res.code === 200 && res.data) {
            this.previewMap = applyPreviewBatchResult(this.previewMap, res.data.items)
          } else {
            this.markPreviewBatchFailed(loadingKeys)
          }
          return res
        })
        .catch(error => {
          if (!this.destroyed) {
            this.markPreviewBatchFailed(loadingKeys)
          }
          return Promise.reject(error)
        })
    },
    markPreviewBatchFailed(cardKeys) {
      cardKeys.forEach(cardKey => {
        const current = this.getPreviewState(cardKey)
        this.$set(this.previewMap, cardKey, {
          ...current,
          status: PREVIEW_STATUS.FAILED,
          image: null,
          error: "failed"
        })
      })
    },
```

- [ ] **Step 11: Add failed-state style**

Add this style near `.image-slot` styles:

```scss
.failed-state {
  color: #ef4444;
}
```

- [ ] **Step 12: Run plot page source contract test**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/views/goto/plot/indexSource.spec.js
```

Expected: PASS.

- [ ] **Step 13: Run frontend focused tests**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/api/plot.spec.js tests/unit/views/goto/plot/previewBatchLoader.spec.js tests/unit/views/goto/plot/indexSource.spec.js
```

Expected: PASS.

- [ ] **Step 14: Commit page wiring**

Run:

```bash
git add \
  goto-web/src/views/goto/plot/index.vue \
  goto-web/tests/unit/views/goto/plot/indexSource.spec.js
git commit -m "feat: lazy load plot previews by batch"
```

---

### Task 7: Full Verification

**Files:**
- Verify all files changed in Tasks 1-6.

- [ ] **Step 1: Run backend focused regression**

Run:

```bash
mvn -pl system -Dtest=PlotServiceImplGroupPreviewBatchTest,PlotControllerGroupPreviewBatchTest,PlotServiceImplPlotGroupVisibilityTest,GroupListHandlerVisibilityTest test
```

Expected: PASS.

- [ ] **Step 2: Run frontend focused regression**

Run:

```bash
cd goto-web && npm run test:unit -- tests/unit/api/plot.spec.js tests/unit/views/goto/plot/previewBatchLoader.spec.js tests/unit/views/goto/plot/indexSource.spec.js
```

Expected: PASS.

- [ ] **Step 3: Run frontend lint**

Run:

```bash
cd goto-web && npm run lint -- src/api/plot.js src/views/goto/plot/index.vue src/views/goto/plot/previewBatchLoader.js
```

Expected: PASS.

- [ ] **Step 4: Inspect final diff for forbidden old preview paths**

Run:

```bash
rg -n "IndexedDBUtils|plot_draw_images|commonWebSocketMessageSendToServer|group_list_image_base64|groupListImageBase64|getKey\\(i, j\\)" goto-web/src/views/goto/plot/index.vue
```

Expected: no matches.

- [ ] **Step 5: Inspect final diff for stable ID path**

Run:

```bash
rg -n "group_preview_batch|groupPreviewBatch|cardKey|IntersectionObserver|createCardKey" goto-web/src/api/plot.js goto-web/src/views/goto/plot/index.vue goto-web/src/views/goto/plot/previewBatchLoader.js goto/system/src/main/java/com/freedom
```

Expected: matches show the new API wrapper, page observer, helper, controller, service, and DTOs.
