# TempPathRoot Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a filesystem-only cleanup task that removes expired files and directories under `tempPathRoot` using a default retention period of 12 hours.

**Architecture:** Keep the implementation aligned with the existing `quartz/task` style by splitting orchestration from deletion mechanics. A dedicated executor handles path normalization, expiration checks, safe recursive deletion, and counters; a task class only resolves configuration, invokes the executor, and writes summary logs. Retention is configurable through existing `RProperties` binding so environment YAML files remain the source of truth.

**Tech Stack:** Spring Boot, Lombok, JUnit 5, Mockito, AssertJ, Java NIO

---

## File Structure

**Create**

- `goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupExecutor.java`
- `goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupTask.java`
- `goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupExecutorTest.java`
- `goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupTaskTest.java`

**Modify**

- `goto/common/src/main/java/com/freedom/properties/RProperties.java`
- `goto/common/src/main/resources/application-common-home.yml`
- `goto/common/src/main/resources/application-common-public.yml`
- `goto/common/src/main/resources/application-common-wsl.yml`
- `goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java`

**Operational follow-up**

- Register a Quartz job using the existing platform scheduler with:
  - `beanName = tempPathRootCleanupTask`
  - `methodName = run`
  - recommended cron = `0 0 * * * ?`

No framework code changes are planned for Quartz itself unless implementation reveals the project currently lacks a registration path for this task.

### Task 1: Add Retention Configuration

**Files:**

- Modify: `goto/common/src/main/java/com/freedom/properties/RProperties.java`
- Modify: `goto/common/src/main/resources/application-common-home.yml`
- Modify: `goto/common/src/main/resources/application-common-public.yml`
- Modify: `goto/common/src/main/resources/application-common-wsl.yml`

- [ ] **Step 1: Write the failing configuration binding test**

Add a focused test method to an existing property-binding-friendly test class or create a small new test if needed. The test should assert that `RProperties` exposes a temp cleanup retention field with a default value path coming from YAML.

```java
assertThat(rProperties.getTempCleanupExpireHours()).isEqualTo(12);
```

- [ ] **Step 2: Run the targeted test to verify it fails**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupTaskTest test`

Expected: FAIL because `getTempCleanupExpireHours()` does not exist yet, or because the property is not wired.

- [ ] **Step 3: Add the new property to `RProperties` and YAML**

Implement a new field on `RProperties`:

```java
private Integer tempCleanupExpireHours;
```

Add this key to all three environment files under `properties.config.r`:

```yaml
tempCleanupExpireHours: 12
```

Keep the property close to `tempPathRoot` so the temp-storage configuration remains grouped.

- [ ] **Step 4: Re-run the targeted test to verify it passes**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupTaskTest test`

Expected: PASS for the configuration assertion or compile success for the new field usage.

- [ ] **Step 5: Commit**

```bash
git add goto/common/src/main/java/com/freedom/properties/RProperties.java \
  goto/common/src/main/resources/application-common-home.yml \
  goto/common/src/main/resources/application-common-public.yml \
  goto/common/src/main/resources/application-common-wsl.yml
git commit -m "feat: add temp path cleanup retention config"
```

### Task 2: Implement Safe Cleanup Executor With TDD

**Files:**

- Create: `goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupExecutor.java`
- Create: `goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupExecutorTest.java`

- [ ] **Step 1: Write the failing executor tests**

Create `TempPathRootCleanupExecutorTest` using `@TempDir` and cover these behaviors:

```java
@Test
void 根目录不存在时_返回空统计且不抛异常() {}

@Test
void 过期文件和目录会被删除_未过期内容会保留() {}

@Test
void 删除单个路径失败时_继续处理剩余内容() {}

@Test
void 符号链接过期时_只删除链接本身() {}
```

Design the tests around a simple result object, for example:

```java
assertThat(result.getScannedCount()).isEqualTo(3);
assertThat(result.getDeletedSuccessCount()).isEqualTo(2);
assertThat(result.getDeletedFailureCount()).isEqualTo(1);
```

For deletion failure injection, subclass the executor in the test and override a protected delete hook instead of relying on platform-specific file locking.

- [ ] **Step 2: Run the executor tests to verify they fail**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupExecutorTest test`

Expected: FAIL because the executor class and result model do not exist yet.

- [ ] **Step 3: Write the minimal executor implementation**

Implement `TempPathRootCleanupExecutor` with:

- a public method like:

```java
public CleanupResult execute(Path tempRoot, Duration retention)
```

- path normalization and root guard
- top-level scan of `tempRoot`
- expiration check based on `lastModified`
- recursive deletion for directories
- symlink-safe behavior using `LinkOption.NOFOLLOW_LINKS`
- per-path try/catch isolation
- a small immutable or Lombok-backed `CleanupResult` value type nested inside the executor if that keeps scope small

Expose protected hooks where tests need deterministic control:

```java
protected FileTime readLastModifiedTime(Path path) throws IOException
protected void deletePath(Path path) throws IOException
```

Implementation should delete children before parents:

```java
try (Stream<Path> stream = Files.walk(path, FileVisitOption.FOLLOW_LINKS)) {
    // do not actually follow symlinks during delete; sort deepest-first
}
```

If `FOLLOW_LINKS` complicates symlink safety, use plain `Files.walk(path)` and guard with `Files.isSymbolicLink(path)` before recursing.

- [ ] **Step 4: Re-run the executor tests to verify they pass**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupExecutorTest test`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupExecutor.java \
  goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupExecutorTest.java
git commit -m "feat: add temp path root cleanup executor"
```

### Task 3: Add Task Orchestration And Summary Logging

**Files:**

- Create: `goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupTask.java`
- Create: `goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupTaskTest.java`
- Modify: `goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java`

- [ ] **Step 1: Write the failing task tests**

Create `TempPathRootCleanupTaskTest` following the style of `LongTimeUnusedUserDrawRemoveTaskTest`. Cover:

```java
@Test
void 根目录为空时_任务仍会调用执行器并记录统计() {}

@Test
void 任务运行时_会从RProperties读取tempPathRoot和保留小时数() {}

@Test
void 执行器抛异常时_任务捕获后不中断调用方() {}
```

Mock graph should include:

- `ComponentBean`
- `com.freedom.bean.Properties`
- `ConfigProperties`
- `RProperties`
- `TempPathRootCleanupExecutor`

Expected interaction:

```java
verify(cleanupExecutor).execute(Path.of(tempPathRoot), Duration.ofHours(12));
```

- [ ] **Step 2: Run the task tests to verify they fail**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupTaskTest test`

Expected: FAIL because the task class does not exist yet.

- [ ] **Step 3: Implement the task class**

Implement `TempPathRootCleanupTask` as a Spring `@Service` with constructor injection:

```java
@Service
@RequiredArgsConstructor
public class TempPathRootCleanupTask {
    private final ComponentBean componentBean;
    private final TempPathRootCleanupExecutor cleanupExecutor;

    public void run() { ... }
}
```

Behavior:

- read `tempPathRoot` and `tempCleanupExpireHours` from `componentBean.getProperties().getConfigProperties().getR()`
- normalize null or invalid retention values back to `12`
- log start, root path, retention hours, and summary counts
- catch unexpected exceptions, log them, and return without throwing

Add the new test class to `QuartzSuite` if that suite is still used as the module-level aggregator.

- [ ] **Step 4: Re-run the task tests to verify they pass**

Run: `mvn -pl goto/system -am -Dtest=TempPathRootCleanupTaskTest test`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add goto/system/src/main/java/com/freedom/quartz/task/TempPathRootCleanupTask.java \
  goto/system/src/test/java/com/freedom/quartz/task/TempPathRootCleanupTaskTest.java \
  goto/system/src/test/java/com/freedom/quartz/QuartzSuite.java
git commit -m "feat: add temp path root cleanup task"
```

### Task 4: Run Focused Verification And Record Quartz Registration

**Files:**

- Modify: `docs/superpowers/plans/2026-03-28-temp-path-root-cleanup-implementation.md`

- [ ] **Step 1: Run the focused test set**

Run:

```bash
mvn -f /mnt/f/IdeaProjects/goto-software/goto/pom.xml -pl system -am \
  -Dtest=TempPathRootCleanupExecutorTest,TempPathRootCleanupTaskTest,LongTimeUnusedUserDrawRemoveTaskTest,UserDrawRemoveExecutorTest \
  -Dsurefire.failIfNoSpecifiedTests=false \
  test
```

Expected: PASS

- [ ] **Step 2: If the focused suite passes, run the quartz-related suite**

Run:

```bash
mvn -f /mnt/f/IdeaProjects/goto-software/goto/pom.xml -pl system -am \
  -Dtest=QuartzSuite \
  -Dsurefire.failIfNoSpecifiedTests=false \
  test
```

Expected: PASS

- [ ] **Step 3: Record deployment-time scheduler wiring**

Document in the implementation notes or release notes that production registration should create a Quartz job with:

```text
beanName=tempPathRootCleanupTask
methodName=run
cronExpression=0 0 * * * ?
```

If a code-based trigger is later required, treat that as a separate change instead of extending this implementation.

Recorded deployment handoff:

- Quartz registration should use `beanName=tempPathRootCleanupTask`
- Quartz registration should use `methodName=run`
- Quartz registration should use `cronExpression=0 0 * * * ?`

- [ ] **Step 4: Commit**

```bash
git add docs/superpowers/plans/2026-03-28-temp-path-root-cleanup-implementation.md
git commit -m "docs: finalize temp path root cleanup implementation plan"
```

## Notes For The Implementer

- Do not use `git reset --hard` or revert unrelated working tree changes.
- Keep the cleanup limited to `tempPathRoot`; do not expand to chart roots or other temp-like locations.
- Prefer `java.nio.file.Files` for path-safe operations over `FileUtils.forceDelete`, except where a small helper makes tests simpler.
- If Windows symlink tests are flaky in the local environment, gate that assertion with assumptions and still preserve symlink-safe production logic in code.
- No GitNexus impact analysis is required for this plan document itself, but implementation must run impact analysis before editing symbols per repo policy.
