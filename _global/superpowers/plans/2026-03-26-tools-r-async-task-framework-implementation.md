# Tools R Async Task Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert image conversion and merge-image export onto the same asynchronous R-task workflow already used by Excel reshape, while leaving a reusable framework for future tool-page generators.

**Architecture:** Keep each tool's request/task/stage/download specifier independent, but standardize the lifecycle: service validates and prepares files, handler declares `source + param`, `CommonTaskThreadPool` executes, stage validates output and writes Redis metadata, frontend waits for WebSocket success and then downloads by `businessType + taskId`. Reuse the existing image-convert and excel-convert interaction pattern for the merge-image tool instead of synchronous generation at download time.

**Tech Stack:** Spring Boot, RabbitMQ task dispatch, existing `CommonTaskThreadPool` R execution pipeline, Redis, Vue 2 + Element UI, WebSocket notice mixin, JUnit 5 + Mockito.

---

## File Map

**Existing files to modify**

- `goto/common/src/main/java/com/freedom/constant/TaskType.java`
- `goto/common/src/main/java/com/freedom/model/commonTask/ImageConvertTask.java`
- `goto/system/src/main/java/com/freedom/rest/ImageConvertController.java`
- `goto/system/src/main/java/com/freedom/rest/ToolsController.java`
- `goto/system/src/main/java/com/freedom/service/ImageConvertService.java`
- `goto/system/src/main/java/com/freedom/service/ToolsService.java`
- `goto/system/src/main/java/com/freedom/service/impl/ImageConvertServiceImpl.java`
- `goto/system/src/main/java/com/freedom/service/impl/ToolsServiceImpl.java`
- `goto/system/src/main/java/com/freedom/download/specifier/ImageConvertDownloadSpecifier.java`
- `goto/system/src/main/java/com/freedom/model/file/download/impl/MergeImageDownload.java`
- `goto/task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java`
- `goto/task/src/main/java/com/freedom/model/stage/CommonTaskStage.java`
- `goto/task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java`
- `goto/system/src/test/java/com/freedom/service/impl/ToolsServiceImplTest.java`
- `goto-web/src/api/tools.js`
- `goto-web/src/views/goto/toolsBox/type/imageConvert.vue`
- `goto-web/src/views/goto/toolsBox/type/tool.vue`

**Expected new backend files**

- `goto/common/src/main/java/com/freedom/model/commonTask/MergeImageTask.java`
- `goto/common/src/main/java/com/freedom/model/dto/MergeImageTaskRequest.java`
- `goto/task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java`
- `goto/task/src/main/java/com/freedom/model/stage/ImageConvertTaskStage.java`
- `goto/task/src/main/java/com/freedom/model/stage/MergeImageTaskStage.java`
- `goto/task/src/main/java/com/freedom/model/stage/AbstractToolResultTaskStage.java`
- `goto/system/src/main/java/com/freedom/download/specifier/MergeImageDownloadSpecifier.java`
- `goto/task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java`

**Expected new or expanded frontend support**

- Add merge-image task submit API in `goto-web/src/api/tools.js`
- Add merge-image async state and WebSocket handling in `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Optionally add a tiny shared local helper in the same Vue file if no existing mixin fits the current code style

**Relevant references to read before coding**

- `goto/task/src/main/java/com/freedom/handler/FileReshapeMessageHandler.java`
- `goto/task/src/main/java/com/freedom/model/stage/FileReshapeTaskStage.java`
- `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`
- `goto-web/src/utils/DownloadUtils.js`

## Task 1: Lock Down Current Boundaries And Impact

**Files:**
- Read: `goto/task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java`
- Read: `goto/system/src/main/java/com/freedom/service/impl/ImageConvertServiceImpl.java`
- Read: `goto/system/src/main/java/com/freedom/model/file/download/impl/MergeImageDownload.java`
- Read: `goto-web/src/views/goto/toolsBox/type/tool.vue`

- [ ] **Step 1: Run GitNexus impact analysis for every symbol that will be edited**

Run the relevant impact checks before touching code, for example:

```text
gitnexus_impact({target: "ImageConvertMessageHandler", direction: "upstream"})
gitnexus_impact({target: "ImageConvertServiceImpl", direction: "upstream"})
gitnexus_impact({target: "ToolsServiceImpl", direction: "upstream"})
gitnexus_impact({target: "MergeImageDownload", direction: "upstream"})
```

Expected: identify direct callers, affected processes, and whether any edit is HIGH/CRITICAL risk.

- [ ] **Step 2: Record the blast radius in the working notes**

Capture the direct callers and any critical paths:

- Image convert submit path
- Merge-image download path
- Download specifier path
- Task stage registration path

Expected: one short note per symbol so later tasks know what must be regression-tested.

- [ ] **Step 3: Inspect the current test baseline**

Run:

```bash
./mvnw -pl goto/task -Dtest=FileReshapeMessageHandlerTest,ImageConvertMessageHandlerTest test
./mvnw -pl goto/system -Dtest=ToolsServiceImplTest test
```

Expected: capture current pass/fail state before introducing new behavior.

## Task 2: Create A Reusable Tool-Result Stage Pattern

**Files:**
- Create: `goto/task/src/main/java/com/freedom/model/stage/AbstractToolResultTaskStage.java`
- Create: `goto/task/src/main/java/com/freedom/model/stage/ImageConvertTaskStage.java`
- Create: `goto/task/src/main/java/com/freedom/model/stage/MergeImageTaskStage.java`
- Modify: `goto/task/src/main/java/com/freedom/model/stage/CommonTaskStage.java`
- Test: `goto/task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java`

- [ ] **Step 1: Write the failing stage tests or assertions first**

Add tests that express the expected result contract:

- success path stores `taskId`
- success path stores `fileName`
- success path stores `outputPath`
- success path rejects missing or empty output file

Expected: test fails because the shared stage or tool-specific stages do not exist yet.

- [ ] **Step 2: Implement the abstract stage helper**

Create a small base stage that:

- reads `CommonTask<?>`
- validates the output file path returned by the task data
- builds a common Redis payload with `taskId`, `businessType`, `fileName`, `outputPath`, `createTime`
- exposes template hooks for tool-specific extra fields

Expected: no tool-specific business logic in the base class beyond validation and Redis persistence.

- [ ] **Step 3: Implement `ImageConvertTaskStage` and `MergeImageTaskStage`**

Each stage should:

- extend the abstract helper
- define its Redis key prefix
- define extra metadata fields

Expected: image convert and merge image can each store their own download metadata while still sharing the common contract.

- [ ] **Step 4: Wire stage registration**

Update the relevant handler `@PostConstruct` initialization so `CommonTaskThreadPool.STAGE_MAPPING` knows about the new stages.

Expected: tasks use the right stage automatically after thread-pool execution.

- [ ] **Step 5: Run focused tests**

Run:

```bash
./mvnw -pl goto/task -Dtest=ImageConvertMessageHandlerTest test
```

Expected: PASS, or a narrower actionable failure tied to the new stage contract.

- [ ] **Step 6: Commit**

```bash
git add goto/task/src/main/java/com/freedom/model/stage/AbstractToolResultTaskStage.java \
  goto/task/src/main/java/com/freedom/model/stage/ImageConvertTaskStage.java \
  goto/task/src/main/java/com/freedom/model/stage/MergeImageTaskStage.java \
  goto/task/src/main/java/com/freedom/model/stage/CommonTaskStage.java \
  goto/task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java
git commit -m "refactor: add reusable tool task result stages"
```

## Task 3: Move Image Convert To The Declarative R Task Pipeline

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/model/commonTask/ImageConvertTask.java`
- Modify: `goto/system/src/main/java/com/freedom/service/ImageConvertService.java`
- Modify: `goto/system/src/main/java/com/freedom/service/impl/ImageConvertServiceImpl.java`
- Modify: `goto/task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java`
- Modify: `goto/system/src/main/java/com/freedom/download/specifier/ImageConvertDownloadSpecifier.java`
- Test: `goto/task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java`

- [ ] **Step 1: Write or extend the failing tests**

Add tests for:

- `ImageConvertMessageHandler` registers thread-pool key and stage mapping
- handler sets `source` to `fileConvert.R`
- handler builds `fileConvert(inputFile=..., outputFile=..., dpi=300)` or equivalent parameter string
- service stores input/output paths on the task instead of shell script fields

Expected: FAIL because the current code still depends on `scriptPath` and imperative shell execution.

- [ ] **Step 2: Simplify `ImageConvertTask` to model R execution inputs**

Keep only fields needed by the new lifecycle:

- input file path
- output file path
- file name for download
- file formats
- optional dpi
- any Redis-visible metadata

Expected: `scriptPath` and shell-only fields are removed or deprecated in one pass.

- [ ] **Step 3: Move file preparation into `ImageConvertServiceImpl`**

Implement:

- base64 normalization and input-file write before enqueue
- task-directory creation by `taskId`
- Java-path and R-path mapping for both input and output
- population of the revised `ImageConvertTask`

Expected: the handler no longer needs to decode base64 or touch local file IO.

- [ ] **Step 4: Rewrite `ImageConvertMessageHandler` in FileReshape style**

Make the handler:

- register `KEY_CALLABLE`
- register `STAGE_MAPPING`
- set the source list for `fileConvert.R`
- build the `fileConvert(...)` invocation with fixed `dpi = 300`
- submit the task through `SystemRebootInitUtils.commonTaskThreadPool.add(...)`

Expected: no more shell facade calls and no more direct Redis writes in the handler.

- [ ] **Step 5: Adjust the download specifier**

Update `ImageConvertDownloadSpecifier` to read the new Redis payload shape and reconstruct the physical output path correctly.

Expected: image convert downloads still work via `businessType = imageConvert`.

- [ ] **Step 6: Run focused tests**

Run:

```bash
./mvnw -pl goto/task -Dtest=ImageConvertMessageHandlerTest test
./mvnw -pl goto/system -Dtest=ImageConvertServiceImplTest test
```

Expected: PASS for handler/service tests or clear failures in path-building assumptions.

- [ ] **Step 7: Commit**

```bash
git add goto/common/src/main/java/com/freedom/model/commonTask/ImageConvertTask.java \
  goto/system/src/main/java/com/freedom/service/ImageConvertService.java \
  goto/system/src/main/java/com/freedom/service/impl/ImageConvertServiceImpl.java \
  goto/task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java \
  goto/system/src/main/java/com/freedom/download/specifier/ImageConvertDownloadSpecifier.java \
  goto/task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java
git commit -m "refactor: move image convert to common r task pipeline"
```

## Task 4: Introduce An Asynchronous Merge-Image Task

**Files:**
- Create: `goto/common/src/main/java/com/freedom/model/commonTask/MergeImageTask.java`
- Create: `goto/common/src/main/java/com/freedom/model/dto/MergeImageTaskRequest.java`
- Create: `goto/task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java`
- Create: `goto/system/src/main/java/com/freedom/download/specifier/MergeImageDownloadSpecifier.java`
- Modify: `goto/common/src/main/java/com/freedom/constant/TaskType.java`
- Modify: `goto/system/src/main/java/com/freedom/rest/ToolsController.java`
- Modify: `goto/system/src/main/java/com/freedom/service/ToolsService.java`
- Modify: `goto/system/src/main/java/com/freedom/service/impl/ToolsServiceImpl.java`
- Modify: `goto/system/src/main/java/com/freedom/model/file/download/impl/MergeImageDownload.java`
- Test: `goto/task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java`
- Test: `goto/system/src/test/java/com/freedom/service/impl/ToolsServiceImplTest.java`

- [ ] **Step 1: Write the failing tests first**

Cover:

- task request validation rejects fewer than two images
- service returns `taskId/status=processing`
- handler builds the merge-image R parameter payload
- result metadata lands in Redis for download lookup

Expected: FAIL because no async merge-image task exists yet.

- [ ] **Step 2: Add the new task type and request/task models**

Create:

- `MergeImageTaskRequest` for controller/service payloads
- `MergeImageTask` for queue execution payloads

Include only fields needed for:

- `typeList`
- normalized image descriptors
- output archive name / output path
- task metadata for download

Expected: request shape is explicit and separate from the old synchronous download payload.

- [ ] **Step 3: Extend the tools controller/service API**

Add a new submit endpoint such as:

```text
POST /api/tools/merge_image/submit
```

Service behavior:

- validate input
- assign `taskId`
- prepare temp input/output directories
- normalize uploaded-base64 and analysis-result file references
- enqueue the new `MERGE_IMAGE` common task
- return `status = processing`

Expected: the submit API mirrors image-convert and excel-convert task submission.

- [ ] **Step 4: Move merge-image generation out of synchronous download**

Extract the business logic now buried inside `MergeImageDownload`:

- coordinate conversion
- global bounds calculation
- `lb/rt` normalization
- R `imageList` script assembly

Place it in the async task flow so the handler can prepare `source` and `param`, or a focused helper can prepare the normalized task payload before handler submission.

Expected: `MergeImageDownload` stops performing live generation.

- [ ] **Step 5: Implement `MergeImageMessageHandler`**

Match the FileReshape pattern:

- register thread-pool key
- register stage mapping
- set needed R source files
- build the merge-image function invocation
- submit to `CommonTaskThreadPool`

Expected: generation is now asynchronous and R-driven.

- [ ] **Step 6: Add a merge-image download specifier**

Implement `businessType = mergeImage` so downloads resolve by `taskId` from Redis metadata written by `MergeImageTaskStage`.

Expected: no fallback to `GLOBAL_DOWNLOAD_CONSTANT.DOWNLOAD_TYPE.MERGE_IMAGE`.

- [ ] **Step 7: Reduce `MergeImageDownload` to compatibility behavior or remove generation from it**

Choose one:

- keep it as a thin compatibility layer that reads generated output only, or
- stop routing new tool-page traffic through it entirely

Expected: no code path should generate merge-image output during the file-download request itself.

- [ ] **Step 8: Run focused tests**

Run:

```bash
./mvnw -pl goto/task -Dtest=MergeImageMessageHandlerTest test
./mvnw -pl goto/system -Dtest=ToolsServiceImplTest test
```

Expected: PASS for new task submission and handler assembly behavior.

- [ ] **Step 9: Commit**

```bash
git add goto/common/src/main/java/com/freedom/constant/TaskType.java \
  goto/common/src/main/java/com/freedom/model/commonTask/MergeImageTask.java \
  goto/common/src/main/java/com/freedom/model/dto/MergeImageTaskRequest.java \
  goto/task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java \
  goto/system/src/main/java/com/freedom/rest/ToolsController.java \
  goto/system/src/main/java/com/freedom/service/ToolsService.java \
  goto/system/src/main/java/com/freedom/service/impl/ToolsServiceImpl.java \
  goto/system/src/main/java/com/freedom/model/file/download/impl/MergeImageDownload.java \
  goto/system/src/main/java/com/freedom/download/specifier/MergeImageDownloadSpecifier.java \
  goto/task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java \
  goto/system/src/test/java/com/freedom/service/impl/ToolsServiceImplTest.java
git commit -m "feat: add async merge image r task pipeline"
```

## Task 5: Align Tool-Page Frontends With The Shared Async Workflow

**Files:**
- Modify: `goto-web/src/api/tools.js`
- Modify: `goto-web/src/views/goto/toolsBox/type/imageConvert.vue`
- Modify: `goto-web/src/views/goto/toolsBox/type/tool.vue`
- Reference: `goto-web/src/views/goto/toolsBox/type/excelFormatConvert.vue`
- Reference: `goto-web/src/utils/DownloadUtils.js`

- [ ] **Step 1: Add the failing UI acceptance checklist**

Write down the expected flows before editing:

- image convert still submits and downloads by `taskId`
- merge image no longer downloads immediately on confirm
- merge image shows in-progress state
- merge image only enables final download after WebSocket success

Expected: a manual checklist that will be run after frontend edits.

- [ ] **Step 2: Add the merge-image submit API wrapper**

In `goto-web/src/api/tools.js`, add a new method for the async merge-image submit endpoint.

Expected: the Vue page no longer uses `downloadDirectByParams(...)` for merge-image generation.

- [ ] **Step 3: Keep `imageConvert.vue` aligned to the revised backend contract**

Verify and adjust:

- request payload shape
- success-path `taskId` handling
- download request parameters
- any message text that still implies synchronous conversion

Expected: no UX regression for image convert.

- [ ] **Step 4: Rework `tool.vue` download flow into task submission**

Add:

- `commonWebSocketOnMessageNotice` mixin if not already present
- merge-task progress state
- merge-task result state and `taskId`
- submit action on dialog confirm
- download action using `DownloadUtils.downloadByBusinessType('mergeImage', ...)`

Expected: "confirm download settings" starts generation, and a separate action downloads the completed output.

- [ ] **Step 5: Remove old direct-download coupling**

Delete or isolate the old call:

```javascript
this.GLOBAL_DOWNLOAD_CONSTANT.downloadDirectByParams(...)
```

Expected: new tool-page flow uses the business download API rather than the legacy direct binary endpoint for merge-image.

- [ ] **Step 6: Run the manual frontend checklist**

Verify in a browser or by component smoke testing:

- image convert submit and success state
- merge image submit and status updates
- merge image final download

Expected: both pages follow the same user mental model.

- [ ] **Step 7: Commit**

```bash
git add goto-web/src/api/tools.js \
  goto-web/src/views/goto/toolsBox/type/imageConvert.vue \
  goto-web/src/views/goto/toolsBox/type/tool.vue
git commit -m "feat: unify tool page async task interactions"
```

## Task 6: Regression Verification And Change-Scope Audit

**Files:**
- Read: `docs/superpowers/specs/2026-03-26-tools-r-async-task-framework-design.md`
- Read: all modified files from previous tasks

- [ ] **Step 1: Run backend test suites that cover the touched areas**

Run:

```bash
./mvnw -pl goto/task -Dtest=FileReshapeMessageHandlerTest,ImageConvertMessageHandlerTest,MergeImageMessageHandlerTest test
./mvnw -pl goto/system -Dtest=ToolsServiceImplTest,ImageConvertServiceImplTest test
```

Expected: PASS for all touched task/service tests.

- [ ] **Step 2: Run any frontend lint or targeted validation available in this repo**

If the repo has a frontend lint/test command, run it. If not, record that only manual validation was available.

Example:

```bash
npm --prefix goto-web run lint
```

Expected: PASS, or a documented reason if the command does not exist.

- [ ] **Step 3: Run GitNexus changed-scope detection**

Before any final integration or commit stack cleanup, run:

```text
gitnexus_detect_changes({scope: "all"})
```

Expected: touched symbols and execution flows match image convert, merge image, shared tool task stages, and tool-page UI paths only.

- [ ] **Step 4: Manually verify the end-to-end flows**

Check:

- image convert: submit -> queued/running -> success -> download
- merge image: submit -> queued/running -> success -> download
- excel reshape: still submits and downloads successfully

Expected: all three tool pages now fit the same async-task pattern from the user perspective.

- [ ] **Step 5: Final commit or branch handoff**

If all tasks were committed incrementally, create only the final coordination commit if needed. Otherwise:

```bash
git status
git log --oneline --decorate -n 10
```

Expected: clean understanding of the final patch set before review or merge.

## Notes For The Implementer

- Follow the AGENTS rule: run `gitnexus_impact` before editing any function/class/method and warn the user if risk is HIGH or CRITICAL.
- Prefer reusing `FileReshapeMessageHandler` and `FileReshapeTaskStage` patterns over inventing a new execution style.
- Keep task-data models focused. Do not create a generic `ToolTask` object.
- Keep output directories namespaced by `taskId`.
- Do not leave both synchronous and asynchronous merge-image generation paths active for the same tool-page flow.
- If `ImageConvertServiceImplTest` does not exist, create it rather than skipping service-level tests.
- If frontend reusable logic is small, keep it local to `tool.vue`; only create a shared mixin/composable if two pages truly need the same code.
