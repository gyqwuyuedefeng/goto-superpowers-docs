# Plot Home Layered Loading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the approved `结构先出、首屏优先、非首屏延后、协议可收口、缓存可校验` loading flow for the plot homepage across `goto-web` and `goto`.

**Architecture:** Build the feature in two repos with isolated worktrees. First, add a deterministic preview-state model and stable cache keys on the frontend so the page can render skeleton, ready, empty, failed, and timeout states without depending on ad hoc `imageBase64` checks. Second, extend the backend plot preview payload and WebSocket handler so requests carry batch priority and responses carry per-card status plus an explicit `batch_done` message that cleanly closes each round. Finally, wire the page to request first-screen previews before deferred batches and verify the full flow with targeted frontend and backend checks.

**Tech Stack:** Vue 2.6, Element UI 2.15, Jest, IndexedDB, Spring Boot 3.3, Maven, JUnit 5

---

## Preconditions

- Use the approved spec as the source of truth:
  `docs/superpowers/specs/2026-03-25-goto-web-plot-home-layered-loading-design.md`
- Work only inside the isolated repo worktrees created for this feature:
  - Frontend: `/mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325`
  - Backend: `/mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325`
- Keep the root repo (`/mnt/f/IdeaProjects/goto-software`) only for docs and coordination. Do not edit `goto-web` or `goto` from their primary worktrees.
- `goto-web` needs `npm install --legacy-peer-deps` in a fresh worktree because plain `npm install` fails on the `@idraw/studio` / `react` peer dependency conflict.
- `goto/system/pom.xml` currently hardcodes `<skipTests>true</skipTests>`. Every targeted backend test command in this plan must pass `-DskipTests=false`, and the output must show real test execution instead of `Tests are skipped`.
- GitNexus MCP tools are not available in the current session. During implementation, use targeted tests plus `git diff --stat` / exact-file `git status` reviews as the fallback safety net before each commit.
- The main `goto-web` workspace is already dirty in unrelated files (`src/views/login.vue`), so every `git add` must name exact files. Never use `git add .`.

## Scope Check

This plan covers one coherent feature: the plot-home preview loading flow. It spans two repos because the current behavior is split between the Vue page and the Java WebSocket handler, but it still produces one user-facing outcome: a stable plot homepage that shows structure immediately, prioritizes first-screen previews, and closes every preview batch deterministically.

## Implementation Notes

- Start each coding task with `@test-driven-development`: write the failing test first, run it, then implement the minimum code to pass it.
- End each coding task with the targeted command listed in that task before committing.
- Keep repo commits separate:
  - `goto-web` commits happen only inside the frontend worktree
  - `goto` commits happen only inside the backend worktree
- Do not rename the existing WebSocket option string in this feature. Keep `group_list_image_base64` as the transport option and evolve the request/response payload shape behind it.
- Do not show `暂无预览` until the backend explicitly says a card is `empty`.
- Prefer a small pure helper on the frontend for preview-state transitions instead of packing every transformation into `index.vue`.
- Prefer small DTOs on the backend for preview batches instead of raw nested maps; this keeps the handler testable.

## File Map

### Frontend tests to create

- `goto-web/tests/unit/plot/plotPreviewState.spec.js`
  - Guards preview-card state creation, stable cache keys, and batch-summary transitions
- `goto-web/tests/unit/plot/plotPreviewTransport.spec.js`
  - Guards the frontend transport contract: callback cleanup, request payload shape, and explicit `batch_done` handling
- `goto-web/tests/unit/plot/plotHomeLoadingContract.spec.js`
  - Guards the homepage source contract so loading / empty / failed / timeout states stay visually distinct

### Frontend files to create

- `goto-web/src/views/goto/plot/previewState.js`
  - Pure helper for preview keys, state constants, card-state transitions, and first-screen / deferred batch bookkeeping

### Frontend files to modify

- `goto-web/src/views/goto/plot/index.vue`
  - Render page-level and card-level states, schedule first-screen vs deferred batches, and consume the new preview protocol
- `goto-web/src/views/components/websocket/index.vue`
  - Add callback cleanup so preview batch handlers can unregister after `batch_done`
- `goto-web/src/utils/IndexedDBUtils.js`
  - Add Promise-based upsert/delete helpers needed by deterministic cache writes

### Backend tests to create

- `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsPreviewVersionTest.java`
  - Guards preview-file cache key / version generation
- `goto/system/src/test/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssemblerTest.java`
  - Guards high-priority filtering, card-status payloads, and `batch_done` emission

### Backend files to create

- `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchRequest.java`
  - Incoming WebSocket payload for `group_list_image_base64`
- `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewCardMessage.java`
  - Outgoing per-card payload for `ready / empty / failed`
- `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchDoneMessage.java`
  - Outgoing explicit end-of-batch payload
- `goto/system/src/main/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssembler.java`
  - Pure assembler that turns cached plot cards plus a batch request into per-card messages and a trailing `batch_done`

### Backend files to modify

- `goto/common/src/main/java/com/freedom/model/small/PlotTypeSmall.java`
  - Carry stable preview metadata alongside the existing plot identifiers
- `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`
  - Build preview file path, stable preview key, and preview version from the sample SVG
- `goto/common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java`
  - Populate the new preview metadata when caching `PlotTypeSmall`
- `goto/system/src/main/java/com/freedom/model/ws/sendMessageHandler/GroupListHandler.java`
  - Parse batch requests, emit per-card statuses, and always send `batch_done`

### Verify-only files

- `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
  - Confirm `groupList()` continues to return light structure data only
- `goto/common/src/main/java/com/freedom/constant/WebSocketOptionEnum.java`
  - Confirm the option list still exposes `group_list_image_base64` without needing a rename

## Task 1: Lock The Frontend Preview-State Helper

**Files:**
- Create: `goto-web/tests/unit/plot/plotPreviewState.spec.js`
- Create: `goto-web/src/views/goto/plot/previewState.js`

- [ ] **Step 1: Write the failing preview-state unit test**

Create `goto-web/tests/unit/plot/plotPreviewState.spec.js`:

```js
import {
  PREVIEW_STATUS,
  buildPreviewKey,
  createPreviewCard,
  applyPreviewResult,
  finalizeBatchSummary
} from '@/views/goto/plot/previewState'

describe('plot preview state helper', () => {
  test('builds a stable cache key from business ids and version', () => {
    expect(buildPreviewKey({ groupId: 10, typeId: 22, previewVersion: '1712345' }))
      .toBe('plot-preview:10:22:1712345')
  })

  test('creates queued cards and only flips to empty when backend says empty', () => {
    const card = createPreviewCard({ groupId: 1, typeId: 2, previewVersion: 'v1' })

    expect(card.status).toBe(PREVIEW_STATUS.IDLE)

    const loading = applyPreviewResult(card, { status: PREVIEW_STATUS.LOADING })
    expect(loading.status).toBe(PREVIEW_STATUS.LOADING)

    const empty = applyPreviewResult(loading, { status: PREVIEW_STATUS.EMPTY })
    expect(empty.status).toBe(PREVIEW_STATUS.EMPTY)
  })

  test('marks batch completion without requiring a successful image payload', () => {
    expect(finalizeBatchSummary({
      requestId: 'req-1',
      rangeType: 'first_screen',
      summary: { total: 4, ready: 0, empty: 3, failed: 1 }
    })).toEqual({
      requestId: 'req-1',
      rangeType: 'first_screen',
      total: 4,
      ready: 0,
      empty: 3,
      failed: 1,
      done: true
    })
  })
})
```

- [ ] **Step 2: Run the test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewState.spec.js --runInBand
```

Expected:

- FAIL because `previewState.js` does not exist yet

- [ ] **Step 3: Write the minimal preview-state helper**

Create `goto-web/src/views/goto/plot/previewState.js` with:

```js
export const PREVIEW_STATUS = {
  IDLE: 'idle',
  CACHED: 'cached',
  QUEUED: 'queued',
  LOADING: 'loading',
  READY: 'ready',
  EMPTY: 'empty',
  FAILED: 'failed',
  TIMEOUT: 'timeout'
}

export function buildPreviewKey({ groupId, typeId, previewVersion }) {
  return `plot-preview:${groupId}:${typeId}:${previewVersion || 'na'}`
}

export function createPreviewCard({ groupId, typeId, previewVersion }) {
  return {
    groupId,
    typeId,
    previewVersion,
    cacheKey: buildPreviewKey({ groupId, typeId, previewVersion }),
    status: PREVIEW_STATUS.IDLE,
    imageBase64: ''
  }
}

export function applyPreviewResult(card, result) {
  return {
    ...card,
    ...result
  }
}

export function finalizeBatchSummary(message) {
  return {
    requestId: message.requestId,
    rangeType: message.rangeType,
    total: message.summary.total,
    ready: message.summary.ready,
    empty: message.summary.empty,
    failed: message.summary.failed,
    done: true
  }
}
```

- [ ] **Step 4: Run the test to confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewState.spec.js --runInBand
```

Expected:

- PASS with 3 tests passing

- [ ] **Step 5: Commit the helper**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
git add tests/unit/plot/plotPreviewState.spec.js src/views/goto/plot/previewState.js
git commit -m "test: add plot preview state helper"
```

## Task 2: Add Stable Preview Metadata On The Backend Structure Payload

**Files:**
- Create: `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsPreviewVersionTest.java`
- Modify: `goto/common/src/main/java/com/freedom/model/small/PlotTypeSmall.java`
- Modify: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java`
- Verify: `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`

- [ ] **Step 1: Write the failing preview-version backend test**

Create `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsPreviewVersionTest.java`:

```java
package com.freedom.util;

import org.junit.jupiter.api.Test;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

class PlotTypeUtilsPreviewVersionTest {

    @Test
    void buildsStablePreviewKeyFromBusinessIdsAndVersion() {
        assertEquals(
                "plot-preview:3:9:1712345",
                PlotTypeUtils.buildPreviewCacheKey(3L, 9L, "1712345")
        );
    }

    @Test
    void derivesPreviewVersionFromFileTimestamp() throws Exception {
        Path svgPath = Files.createTempFile("plot-preview-version", ".svg");
        File svgFile = svgPath.toFile();
        Files.writeString(svgPath, "<svg></svg>");

        String version = PlotTypeUtils.buildPreviewVersion(svgFile);

        assertNotNull(version);
    }
}
```

- [ ] **Step 2: Run the test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl common -DskipTests=false -Dtest=PlotTypeUtilsPreviewVersionTest test
```

Expected:

- FAIL because `buildPreviewCacheKey` / `buildPreviewVersion` do not exist yet

- [ ] **Step 3: Implement preview metadata on cached plot cards**

Update the backend metadata model.

In `goto/common/src/main/java/com/freedom/model/small/PlotTypeSmall.java`, add:

```java
private String previewCacheKey;
private String previewVersion;
```

In `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`, add:

```java
public static String buildPreviewCacheKey(Long groupId, Long typeId, String previewVersion) {
    return String.format("plot-preview:%s:%s:%s", groupId, typeId, previewVersion == null ? "na" : previewVersion);
}

public static String buildPreviewVersion(File previewFile) {
    if (previewFile == null || !previewFile.exists()) {
        return null;
    }
    return String.valueOf(previewFile.lastModified());
}
```

In `goto/common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java`, when creating `PlotTypeSmall`, populate `previewVersion` from the sample SVG file and `previewCacheKey` from `groupId + plotTypeId + previewVersion`.

Do not reintroduce eager `imageBase64` into the structure payload.

- [ ] **Step 4: Run the test to confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl common -DskipTests=false -Dtest=PlotTypeUtilsPreviewVersionTest test
```

Expected:

- PASS with 2 tests passing

- [ ] **Step 5: Commit the metadata contract**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
git add \
  common/src/test/java/com/freedom/util/PlotTypeUtilsPreviewVersionTest.java \
  common/src/main/java/com/freedom/model/small/PlotTypeSmall.java \
  common/src/main/java/com/freedom/util/PlotTypeUtils.java \
  common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java
git commit -m "feat: add plot preview metadata contract"
```

## Task 3: Implement The Backend Preview Batch Protocol

**Files:**
- Create: `goto/system/src/test/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssemblerTest.java`
- Create: `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchRequest.java`
- Create: `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewCardMessage.java`
- Create: `goto/system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchDoneMessage.java`
- Create: `goto/system/src/main/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssembler.java`
- Modify: `goto/system/src/main/java/com/freedom/model/ws/sendMessageHandler/GroupListHandler.java`
- Verify: `goto/common/src/main/java/com/freedom/constant/WebSocketOptionEnum.java`

- [ ] **Step 1: Write the failing assembler test**

Create `goto/system/src/test/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssemblerTest.java`:

```java
package com.freedom.model.ws.sendMessageHandler;

import com.freedom.model.dto.plot.PlotPreviewBatchDoneMessage;
import com.freedom.model.dto.plot.PlotPreviewBatchRequest;
import com.freedom.model.dto.plot.PlotPreviewCardMessage;
import com.freedom.model.small.PlotGroupSmall;
import com.freedom.model.small.PlotTypeSmall;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

class PlotPreviewBatchAssemblerTest {

    @Test
    void emitsCardStatusesAndBatchDoneForRequestedCards() {
        PlotTypeSmall card = new PlotTypeSmall(2L, "example", "folder", null);
        card.setPreviewCacheKey("plot-preview:1:2:v1");
        card.setPreviewVersion("v1");

        PlotGroupSmall group = new PlotGroupSmall();
        group.setId(1L);
        group.setPlotTypeSmallList(List.of(card));

        PlotPreviewBatchRequest request = new PlotPreviewBatchRequest();
        request.setRequestId("req-1");
        request.setRangeType("first_screen");
        request.setCardKeys(List.of("plot-preview:1:2:v1"));

        PlotPreviewBatchAssembler.Result result =
                new PlotPreviewBatchAssembler().assemble(
                        request,
                        List.of(group),
                        preview -> null
                );

        PlotPreviewCardMessage cardMessage = result.getCardMessages().get(0);
        PlotPreviewBatchDoneMessage done = result.getDoneMessage();

        assertEquals("empty", cardMessage.getStatus());
        assertEquals("req-1", done.getRequestId());
        assertEquals("first_screen", done.getRangeType());
        assertEquals(1, done.getSummary().get("empty"));
    }
}
```

- [ ] **Step 2: Run the test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl system -DskipTests=false -Dtest=PlotPreviewBatchAssemblerTest test
```

Expected:

- FAIL because the DTOs and assembler do not exist yet

- [ ] **Step 3: Implement DTOs and the explicit batch protocol**

Create DTOs with the following minimum fields:

`PlotPreviewBatchRequest.java`

```java
private String requestId;
private String priority;
private String rangeType;
private List<String> cardKeys;
```

`PlotPreviewCardMessage.java`

```java
private String requestId;
private String cardKey;
private String status;
private String image;
private String version;
```

`PlotPreviewBatchDoneMessage.java`

```java
private String requestId;
private String rangeType;
private Map<String, Integer> summary;
private String type = "batch_done";
```

Create `PlotPreviewBatchAssembler.java` with a pure method that:

1. Filters cards by requested `cardKeys`
2. Resolves each card into `ready / empty / failed`
3. Tracks `ready / empty / failed` counts
4. Builds one trailing `PlotPreviewBatchDoneMessage`

Then update `GroupListHandler.java` so it:

1. Parses `SendMessage.params` into `PlotPreviewBatchRequest`
2. Delegates message construction to `PlotPreviewBatchAssembler`
3. Sends every `PlotPreviewCardMessage`
4. Always sends the trailing `batch_done` message from the assembler

Keep the transport option string `group_list_image_base64` unchanged.

- [ ] **Step 4: Run the test to confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl system -DskipTests=false -Dtest=PlotPreviewBatchAssemblerTest test
```

Expected:

- PASS with the assembler proving both per-card status and `batch_done` emission

- [ ] **Step 5: Commit the protocol change**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
git add \
  system/src/test/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssemblerTest.java \
  system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchRequest.java \
  system/src/main/java/com/freedom/model/dto/plot/PlotPreviewCardMessage.java \
  system/src/main/java/com/freedom/model/dto/plot/PlotPreviewBatchDoneMessage.java \
  system/src/main/java/com/freedom/model/ws/sendMessageHandler/PlotPreviewBatchAssembler.java \
  system/src/main/java/com/freedom/model/ws/sendMessageHandler/GroupListHandler.java
git commit -m "feat: batch plot preview websocket responses"
```

## Task 4: Upgrade Frontend Transport And Cache Writes

**Files:**
- Create: `goto-web/tests/unit/plot/plotPreviewTransport.spec.js`
- Modify: `goto-web/src/views/components/websocket/index.vue`
- Modify: `goto-web/src/utils/IndexedDBUtils.js`

- [ ] **Step 1: Write the failing frontend transport test**

Create `goto-web/tests/unit/plot/plotPreviewTransport.spec.js`:

```js
const fs = require('fs')
const path = require('path')

function read(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

describe('plot preview transport contract', () => {
  test('websocket component exposes callback cleanup', () => {
    const source = read('src/views/components/websocket/index.vue')
    expect(source).toContain('removeMessageCallback(uuid)')
  })

  test('indexed db helper exposes promise-based upsert for preview writes', () => {
    const source = read('src/utils/IndexedDBUtils.js')
    expect(source).toContain('putData(db, storeName, data)')
  })
})
```

- [ ] **Step 2: Run the test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewTransport.spec.js --runInBand
```

Expected:

- FAIL because callback cleanup and `putData` do not exist yet

- [ ] **Step 3: Implement cleanup and deterministic cache writes**

In `goto-web/src/views/components/websocket/index.vue`, add:

```js
removeMessageCallback(uuid) {
  Reflect.deleteProperty(this.optionMap, uuid)
}
```

Call it when the plot homepage receives `batch_done`, so stale batch callbacks do not accumulate in `optionMap`.

In `goto-web/src/utils/IndexedDBUtils.js`, add Promise-based helpers:

```js
putData(db, storeName, data) {
  return new Promise((resolve, reject) => {
    const request = db.transaction([storeName], 'readwrite').objectStore(storeName).put(data)
    request.onsuccess = () => resolve(data)
    request.onerror = event => reject(event)
  })
}
```

Add a matching Promise-based `removeData` wrapper so the page can sequence cache writes and deletes deterministically.

- [ ] **Step 4: Run the test to confirm it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewTransport.spec.js --runInBand
```

Expected:

- PASS with 2 tests passing

- [ ] **Step 5: Commit the transport changes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
git add \
  tests/unit/plot/plotPreviewTransport.spec.js \
  src/views/components/websocket/index.vue \
  src/utils/IndexedDBUtils.js
git commit -m "feat: support deterministic plot preview transport"
```

## Task 5: Wire The Plot Homepage To First-Screen And Deferred Preview Batches

**Files:**
- Create: `goto-web/tests/unit/plot/plotHomeLoadingContract.spec.js`
- Modify: `goto-web/src/views/goto/plot/index.vue`
- Verify: `goto-web/src/views/goto/plot/previewState.js`

- [ ] **Step 1: Write the failing homepage loading-contract test**

Create `goto-web/tests/unit/plot/plotHomeLoadingContract.spec.js`:

```js
const fs = require('fs')
const path = require('path')

function read(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

describe('plot homepage loading contract', () => {
  test('homepage distinguishes loading, empty, and failed preview states', () => {
    const source = read('src/views/goto/plot/index.vue')

    expect(source).toContain("plot-card--loading")
    expect(source).toContain("plot-card--empty")
    expect(source).toContain("plot-card--failed")
  })

  test('homepage schedules explicit first-screen batches before deferred batches', () => {
    const source = read('src/views/goto/plot/index.vue')

    expect(source).toContain("rangeType: 'first_screen'")
    expect(source).toContain("rangeType: 'deferred'")
  })
})
```

- [ ] **Step 2: Run the test to confirm it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotHomeLoadingContract.spec.js --runInBand
```

Expected:

- FAIL because `index.vue` still only checks `imageBase64` vs `暂无预览`

- [ ] **Step 3: Implement the homepage state machine and batch scheduling**

Refactor `goto-web/src/views/goto/plot/index.vue` so it:

1. Builds normalized preview cards from `groupList()` using `previewState.js`
2. Opens in `booting`, then moves to `struct_ready` after structure data arrives
3. Reads IndexedDB using `previewCacheKey`, marks hits as `cached`, and leaves misses as `idle`
4. Sends one `high` / `first_screen` WebSocket batch for the visible cards
5. Sends `deferred` batches only after first-screen completion or near-scroll events
6. Applies `ready / empty / failed / timeout` per-card states
7. Calls `this.$refs.ws.removeMessageCallback(requestId)` when `batch_done` arrives

Keep the UI explicit:

```vue
<div v-if="plotType.previewStatus === 'loading'" class="image-slot loading-state">...</div>
<div v-else-if="plotType.previewStatus === 'failed'" class="image-slot failed-state">...</div>
<div v-else-if="plotType.previewStatus === 'empty'" class="image-slot empty-state">...</div>
```

Add the matching CSS modifiers:

```scss
.plot-card--loading {}
.plot-card--empty {}
.plot-card--failed {}
```

- [ ] **Step 4: Run the targeted frontend checks**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewState.spec.js tests/unit/plot/plotPreviewTransport.spec.js tests/unit/plot/plotHomeLoadingContract.spec.js --runInBand
```

Expected:

- PASS with all plot-preview frontend tests passing

- [ ] **Step 5: Commit the page integration**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
git add \
  tests/unit/plot/plotHomeLoadingContract.spec.js \
  src/views/goto/plot/index.vue \
  src/views/goto/plot/previewState.js
git commit -m "feat: prioritize first-screen plot previews"
```

## Task 6: Verify The End-To-End Flow Before Handoff

**Files:**
- Verify only; no planned file creation

- [ ] **Step 1: Re-run all targeted frontend plot-preview checks**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/plot/plotPreviewState.spec.js tests/unit/plot/plotPreviewTransport.spec.js tests/unit/plot/plotHomeLoadingContract.spec.js --runInBand
```

Expected:

- PASS with all plot-preview frontend tests green

- [ ] **Step 2: Re-run all targeted backend preview checks**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl common -DskipTests=false -Dtest=PlotTypeUtilsPreviewVersionTest test
mvn -pl system -DskipTests=false -Dtest=PlotPreviewBatchAssemblerTest test
```

Expected:

- PASS with real test execution output, not `Tests are skipped`

- [ ] **Step 3: Smoke-check the manual first-screen flow**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
NODE_OPTIONS=--openssl-legacy-provider npx vue-cli-service serve
```

Manual checklist:

1. Open `/goto/plot`
2. Confirm structure renders before previews
3. Confirm first-screen cards fill before lower cards
4. Confirm missing preview cards show `暂无预览` only after backend empty response
5. Confirm failed cards show a failed state instead of hanging skeletons forever

- [ ] **Step 4: Review exact diffs in both repos**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
git diff --stat master...HEAD

cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
git diff --stat master...HEAD
```

Expected:

- Only plot-preview related files appear in each repo

- [ ] **Step 5: Create the final feature commits if any task was left uncommitted**

If verification uncovered small fixes, commit them in the repo where they belong using exact file lists. Keep frontend and backend commits separate.

## Review Notes

- The frontend baseline check already passed in the dedicated worktree:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web/.worktrees/plot-home-layered-loading-20260325
npx jest tests/unit/styles/drawArchitecture.spec.js --runInBand
```

- The backend baseline build already succeeded, but the default command skipped tests because of the current `surefire` config:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/.worktrees/plot-home-layered-loading-20260325
mvn -pl system -Dtest=PlotTypeControllerTest test
```

Use that as a reminder to include `-DskipTests=false` on every real backend test run in this feature.
