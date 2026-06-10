# PlotGroupSmall show缓存补齐 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `PlotGroupSmall` 统一补齐 `show` 字段，并确保所有构造、缓存写入、缓存回读与异常回退链路都能透传该字段。

**Architecture:** 在 `common` 模块内扩展 `PlotGroupSmall`、`PlotGroupProjection` 和 `PlotGroupRepository` 的最小数据面，让 `show` 从数据库投影进入缓存模型；同时补齐 `PlotGroupCacheUtils` 与 `PlotGroupUtils` 的缓存写入、缓存回组装和旧缓存读取失败时的自动重建逻辑。通过 `common` 模块单测锁定字段映射和缓存回退行为，避免部署后旧 Redis 缓存导致的读取异常。

**Tech Stack:** Java 17, Spring Boot, Spring Data JPA, Redisson, JUnit 5, Mockito

---

### Task 1: 锁定失败测试

**Files:**
- Create: `goto/common/src/test/java/com/freedom/util/PlotGroupCacheUtilsTest.java`
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java`
- Modify: `goto/common/src/main/java/com/freedom/util/PlotGroupUtils.java`

- [ ] **Step 1: 编写失败测试**

为以下行为补测试：
- `obtainPlotGroupListFromDatabase` 会把 projection 中的 `show` 映射到 `PlotGroup`
- `obtainPlotGroupSmallListFromDatabase` 会把 `PlotGroup.show` 透传到 `PlotGroupSmall.show`
- `obtainStorePlotGroupSmallList` 在缓存读取异常时会回退到重建逻辑

- [ ] **Step 2: 运行测试并确认失败**

Run: `mvn -pl common -Dtest=PlotGroupCacheUtilsTest test`
Expected: 测试因 `show` 未映射或缓存异常未回退而失败

### Task 2: 实现字段透传

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/model/small/PlotGroupSmall.java`
- Modify: `goto/common/src/main/java/com/freedom/projection/PlotGroupProjection.java`
- Modify: `goto/common/src/main/java/com/freedom/repository/PlotGroupRepository.java`
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotGroupCacheUtils.java`
- Modify: `goto/common/src/main/java/com/freedom/util/PlotGroupUtils.java`

- [ ] **Step 1: 扩展数据模型与投影**

给 `PlotGroupSmall` 增加 `show` 字段；给 `PlotGroupProjection` 增加 `getShow()`；把 repository 查询 SQL 补齐 `show` 列。

- [ ] **Step 2: 写最小实现**

补齐以下链路：
- map/list 缓存写入时写入 `show`
- projection 转 `PlotGroup` 时设置 `show`
- `PlotGroupSmall` 构造时设置 `show`
- 从缓存回组装 `PlotGroup` 时恢复 `show`
- list 缓存读取异常时回退到缓存重建

- [ ] **Step 3: 运行测试确认通过**

Run: `mvn -pl common -Dtest=PlotGroupCacheUtilsTest test`
Expected: PASS

### Task 3: 回归验证

**Files:**
- Verify: `goto/common/src/test/java/com/freedom/util/PlotGroupCacheUtilsTest.java`

- [ ] **Step 1: 运行相关回归测试**

Run: `mvn -pl common -Dtest=PlotGroupCacheUtilsTest,PlotTypeUtilsTest test`
Expected: PASS

- [ ] **Step 2: 检查改动影响范围**

Run: GitNexus `detect_changes(scope: "all")`
Expected: 仅影响 `PlotGroupSmall`、缓存工具、projection/repository 及新增测试
