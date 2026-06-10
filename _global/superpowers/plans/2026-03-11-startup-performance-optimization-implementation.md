# 启动性能优化实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**目标**: 将 goto/system 模块启动时间从 120 秒优化到 5 秒以内

**架构**: 采用两阶段预热策略（同步加载必需数据 + 异步加载详细数据），使用 RBatch 批量操作优化 Redis 写入，批量查询优化字典 N+1 问题，懒加载机制兜底

**技术栈**: Spring Boot, Redisson, JPA, CompletableFuture

**任务总数**: 8 个任务

**预计工作量**: 3-5 天开发 + 2-3 天测试

---

## 任务清单

### 阶段一：PlotType 缓存优化（Task 1-3）
- Task 1: 新增同步预热基础信息方法
- Task 2: 新增异步预热详细信息方法
- Task 3: 优化懒加载重试机制

### 阶段二：字典查询优化（Task 4-5）
- Task 4: 新增批量查询方法
- Task 5: 优化字典查询逻辑

### 阶段三：启动流程改造（Task 6）
- Task 6: 改造启动流程为两阶段预热

### 阶段四：测试验证（Task 7-8）
- Task 7: 性能测试
- Task 8: 功能回归测试

---

## Task 1: 新增同步预热基础信息方法

**目标**: 在 PlotTypeCacheUtils 中新增 syncWarmupBasicInfo 方法，使用 RMap.putAll() 批量写入基础信息

**文件**:
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java`

**Step 1: 在 PlotTypeCacheUtils 中新增 syncWarmupBasicInfo 方法**

在 `cachePlotTypeSmallOneAndPlotTypeFileSettingsAndParamSettings()` 方法后添加：

```java
/**
 * 同步预热 PlotType 基础信息
 * 使用 RMap.putAll() 批量写入，耗时 < 1 秒
 *
 * @param componentBean 组件Bean
 * @param plotTypeList 绘图类型列表
 */
public static void syncWarmupBasicInfo(ComponentBean componentBean, List<PlotType> plotTypeList) {
    RHandlerWrapper<CheckResultMsg> wrapper = new RHandlerWrapper(() -> {
        RMap<Long, PlotTypeSmallOne> map = componentBean.getRedissonClient()
            .getMap(RedissonKey.CACHE_PLOT_TYPE);
        map.clear();

        // RMap 本身支持批量操作
        Map<Long, PlotTypeSmallOne> batchData = new HashMap<>();
        for (PlotType plotType : plotTypeList) {
            batchData.put(plotType.getId(),
                new PlotTypeSmallOne(plotType.getId(), plotType.getGroupId(), plotType.getFolderName()));
        }
        map.putAll(batchData);  // 一次性写入

        log.info("同步预热完成：{} 个 PlotType 基础信息", plotTypeList.size());
        return null;
    });

    try {
        wrapper.doHandler(componentBean.getRedissonClient(), RedissonKey.DL_CACHE_PLOT_TYPE);
    } catch (RLockException e) {
        log.error("同步预热失败", e);
        throw new RuntimeException("缓存预热失败，无法启动", e);
    }
}
```

**Step 2: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/common
mvn compile
```

预期: 编译成功，无错误

**Step 3: 提交**

```bash
git add goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java
git commit -m "feat: add syncWarmupBasicInfo for batch writing PlotType basic info

- 使用 RMap.putAll() 批量写入基础信息
- 耗时从逐个写入优化到 < 1 秒
- 分布式锁保护，确保数据一致性

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 2: 新增异步预热详细信息方法

**目标**: 在 PlotTypeCacheUtils 中新增 asyncWarmupDetailInfo 方法，使用 RBatch 批量写入详细信息

**文件**:
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java`

**Step 1: 添加必要的 import**

在文件顶部添加：

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import org.redisson.api.RBatch;
```

**Step 2: 在 PlotTypeCacheUtils 中新增 asyncWarmupDetailInfo 方法**

在 `syncWarmupBasicInfo()` 方法后添加：

```java
/**
 * 异步预热 PlotType 详细信息
 * 使用 RBatch 批量写入，耗时 5-10 秒（后台执行）
 *
 * @param componentBean 组件Bean
 * @param plotTypeList 绘图类型列表
 * @return CompletableFuture
 */
public static CompletableFuture<Void> asyncWarmupDetailInfo(ComponentBean componentBean, List<PlotType> plotTypeList) {
    return CompletableFuture.runAsync(() -> {
        RHandlerWrapper<CheckResultMsg> wrapper = new RHandlerWrapper(() -> {
            // 清理旧缓存
            RKeys keys = componentBean.getRedissonClient().getKeys();
            Iterable<String> matchedKeys = keys.getKeysByPattern(
                RedissonKey.CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + "*");
            if (matchedKeys != null) {
                List<String> keysToDelete = new ArrayList<>();
                matchedKeys.forEach(keysToDelete::add);
                if (!keysToDelete.isEmpty()) {
                    keys.delete(keysToDelete.toArray(new String[0]));
                    log.info("删除了 {} 个旧的PlotType缓存键", keysToDelete.size());
                }
            }

            // 使用 RBatch 批量写入
            RBatch batch = componentBean.getRedissonClient().createBatch();
            for (PlotType plotType : plotTypeList) {
                PlotTypeFileSettingsAndParamSettings storePlotType =
                    new PlotTypeFileSettingsAndParamSettings(
                        plotType.getId(),
                        plotType.getFileSettings(),
                        plotType.getParamSettings()
                    );
                String key = RedissonKey.CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + storePlotType.getId();
                batch.getBucket(key).setAsync(storePlotType);
            }
            batch.execute();  // 一次性提交，1 次网络 I/O

            log.info("异步预热完成：{} 个 PlotType 详细信息", plotTypeList.size());
            return null;
        });

        try {
            wrapper.doHandlerWithRetry(
                componentBean.getRedissonClient(),
                RedissonKey.DL_CACHE_PLOT_TYPE
            );
        } catch (RLockException e) {
            log.warn("异步预热获取锁失败（重试 3 次后），懒加载将兜底", e);
        }
    }, Executors.newSingleThreadExecutor());
}
```

**Step 3: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/common
mvn compile
```

预期: 编译成功，无错误

**Step 4: 提交**

```bash
git add goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java
git commit -m "feat: add asyncWarmupDetailInfo for async batch writing

- 使用 RBatch.execute() 批量写入详细信息
- 异步执行，不阻塞启动
- 耗时从 74 秒优化到 5-10 秒（后台）
- 失败时懒加载兜底

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 3: 优化懒加载重试机制

**目标**: 优化 findPlotTypeById 方法，使用 RHandlerWrapper 内置重试机制

**文件**:
- Modify: `goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java:326-375`

**Step 1: 修改 findPlotTypeById 方法**

替换现有的 `findPlotTypeById()` 方法（第 326-375 行）：

```java
/**
 * 根据ID查找绘图类型
 * 先从缓存中获取，缓存不存在则从数据库获取并缓存
 * 使用 RHandlerWrapper 内置重试机制（默认重试 3 次）
 *
 * @param componentBean 组件Bean，用于缓存和数据库访问
 * @param id 绘图类型ID
 * @return 绘图类型对象
 * @throws RuntimeException 当模板信息不存在时抛出异常
 */
public static PlotType findPlotTypeById(ComponentBean componentBean, Long id) {
    if (id == null) {
        throw new IllegalArgumentException("PlotType ID不能为空");
    }

    PlotType plotType = new PlotType();
    plotType.setId(id);

    try {
        RHandlerWrapper<PlotTypeFileSettingsAndParamSettings> rHandlerWrapper = new RHandlerWrapper(() -> {
            RBucket<PlotTypeFileSettingsAndParamSettings> bucket = componentBean.getRedissonClient().getBucket(
                    RedissonKey.CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id
            );
            PlotTypeFileSettingsAndParamSettings plotTypeFileSettingsAndParamSettings = bucket.get();

            if (plotTypeFileSettingsAndParamSettings == null) {
                log.debug("缓存中未找到PlotType，从数据库获取，ID: {}", id);
                Optional<PlotType> plotTypeOptional = componentBean.getRepository().getPlotTypeRepository().findById(id);
                if (plotTypeOptional.isPresent()) {
                    PlotType dbPlotType = plotTypeOptional.get();
                    plotTypeFileSettingsAndParamSettings = new PlotTypeFileSettingsAndParamSettings(
                            dbPlotType.getId(),
                            dbPlotType.getFileSettings(),
                            dbPlotType.getParamSettings()
                    );
                    bucket.set(plotTypeFileSettingsAndParamSettings);
                    log.debug("成功将PlotType缓存到Redis，ID: {}", id);
                }
            }
            return plotTypeFileSettingsAndParamSettings;
        });

        // 使用 RHandlerWrapper 内置重试：默认重试 3 次，递增等待 100ms, 150ms, 200ms
        PlotTypeFileSettingsAndParamSettings plotTypeFileSettingsAndParamSettings = rHandlerWrapper.doHandlerWithRetry(
                componentBean.getRedissonClient(),
                RedissonKey.DL_CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id,
                () -> {
                    // RetryChecker: 检查缓存是否已被其他线程创建
                    RBucket<PlotTypeFileSettingsAndParamSettings> bucket =
                        componentBean.getRedissonClient().getBucket(
                            RedissonKey.CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id
                        );
                    return bucket.isExists();
                }
        );

        if (plotTypeFileSettingsAndParamSettings == null) {
            throw new RuntimeException("模板信息不存在，ID: " + id);
        }

        plotType.setFileSettings(plotTypeFileSettingsAndParamSettings.getFileSettings());
        plotType.setParamSettings(plotTypeFileSettingsAndParamSettings.getParamSettings());

    } catch (RLockException e) {
        log.error("获取 PlotType 失败（重试 3 次后），ID: {}", id, e);
        throw new RuntimeException("系统繁忙，请稍后重试", e);
    }

    // 获取基础信息
    RMap<Long, PlotTypeSmallOne> typeMap = componentBean.getRedissonClient().getMap(RedissonKey.CACHE_PLOT_TYPE);
    PlotTypeSmallOne smallOne = typeMap.get(id);
    if (smallOne != null) {
        plotType.setGroupId(smallOne.getGroupId());
    }

    return plotType;
}
```

**Step 2: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/common
mvn compile
```

预期: 编译成功，无错误

**Step 3: 提交**

```bash
git add goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java
git commit -m "refactor: optimize lazy load with RHandlerWrapper retry

- 使用 RHandlerWrapper.doHandlerWithRetry() 内置重试机制
- 默认重试 3 次，递增等待 100ms, 150ms, 200ms
- RetryChecker 避免不必要的重试
- 代码更简洁，统一重试策略

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 4: 新增批量查询方法

**目标**: 在 DictDetailCustomRepository 中新增批量查询方法

**文件**:
- Modify: `goto/system/src/main/java/com/freedom/repository/DictDetailCustomRepository.java`

**Step 1: 新增批量查询方法**

在接口中添加新方法：

```java
/**
 * 批量查询字典详情
 * 优化 N+1 查询问题
 *
 * @param dictIds 字典ID列表
 * @return 字典详情列表
 */
List<DictDetailCustom> findByDictIdInOrderByDictIdAscDictSortAsc(List<Long> dictIds);
```

**Step 2: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn compile
```

预期: 编译成功，无错误

**Step 3: 提交**

```bash
git add goto/system/src/main/java/com/freedom/repository/DictDetailCustomRepository.java
git commit -m "feat: add batch query method for dict details

- 新增 findByDictIdInOrderByDictIdAscDictSortAsc 方法
- 支持批量查询字典详情
- 优化 N+1 查询问题

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 5: 优化字典查询逻辑

**目标**: 在 CacheService 中优化 getGotoDictList 方法，使用批量查询替代 N+1

**文件**:
- Modify: `goto/common/src/main/java/com/freedom/cache/CacheService.java:107-141`

**Step 1: 添加必要的 import**

在文件顶部添加：

```java
import java.util.stream.Collectors;
import java.util.Collections;
```

**Step 2: 修改 getGotoDictList 方法**

替换现有的 `getGotoDictList()` 方法（第 107-141 行）：

```java
/**
 * 获取GOTO字典列表
 * 从本地缓存获取，如果缓存为空则从数据库加载并存入缓存
 * 优化：使用批量查询替代 N+1 查询
 *
 * @param componentBean 组件Bean，用于访问数据库
 * @return 字典列表的深拷贝
 */
public static List<DictSimple> getGotoDictList(ComponentBean componentBean) {
    ConcurrentMap<Long, DictSimple> longDictSimpleConcurrentMap = gotoDictMapCache.asMap();
    List<DictSimple> gotoDictList = new ArrayList<>();

    if (longDictSimpleConcurrentMap.isEmpty()) {
        if (componentBean == null) {
            return null;
        }
        synchronized (GOTO_DICT_LOCK) {
            // 在同步块内重新获取map引用，确保获取最新状态
            longDictSimpleConcurrentMap = gotoDictMapCache.asMap();
            if (longDictSimpleConcurrentMap.isEmpty()) {
                List<DictCustom> dictCustomList = componentBean.getRepository().getDictCustomRepository().findAllGotoDict();
                if (GCollectionUtils.listNotNullNotEmpty(dictCustomList)) {
                    // 批量查询所有字典详情（1 次查询）
                    List<Long> dictIds = dictCustomList.stream()
                        .map(DictCustom::getId)
                        .collect(Collectors.toList());
                    List<DictDetailCustom> allDetails = componentBean.getRepository()
                        .getDictDetailCustomRepository()
                        .findByDictIdInOrderByDictIdAscDictSortAsc(dictIds);

                    // 在内存中按 dict_id 分组
                    Map<Long, List<DictDetailCustom>> detailsMap = allDetails.stream()
                        .collect(Collectors.groupingBy(DictDetailCustom::getDictId));

                    // 组装数据
                    for (DictCustom dictCustom : dictCustomList) {
                        DictSimple dictSimple = new DictSimple();
                        BeanUtils.copyProperties(dictCustom, dictSimple);
                        dictSimple.setSelectList(detailsMap.getOrDefault(dictCustom.getId(), Collections.emptyList()));
                        gotoDictList.add(dictSimple);
                        gotoDictMapCache.put(dictSimple.getId(), dictSimple);
                    }
                }
            } else {
                // 在同步块内发现缓存已被其他线程填充，直接使用
                longDictSimpleConcurrentMap.forEach((k, v) -> gotoDictList.add(v));
            }
        }
    } else {
        longDictSimpleConcurrentMap.forEach((k, v) -> gotoDictList.add(v));
    }

    // 深拷贝
    return GsonUtils.deepCloneList(gotoDictList, new TypeToken<List<DictSimple>>() {
    }.getType());
}
```

**Step 3: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/common
mvn compile
```

预期: 编译成功，无错误

**Step 4: 提交**

```bash
git add goto/common/src/main/java/com/freedom/cache/CacheService.java
git commit -m "refactor: optimize dict query from N+1 to batch query

- 使用批量查询替代逐个查询
- 从 1+N 次查询优化到 2 次查询
- 在内存中按 dict_id 分组
- 耗时从 7 秒降到 < 1 秒

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 6: 改造启动流程为两阶段预热

**目标**: 在 DispatchInitializer 中改造启动流程，实现两阶段预热

**文件**:
- Modify: `goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java:38-87`

**Step 1: 添加必要的 import**

在文件顶部添加：

```java
import java.util.concurrent.CompletableFuture;
```

**Step 2: 修改 run 方法和缓存预热方法**

替换现有的 `run()` 方法和缓存预热方法（第 38-87 行）：

```java
@Override
public void run(String... args) {
    log.info("========== 启动缓存预热开始 ==========");
    long startTime = System.currentTimeMillis();

    // 阶段 1：同步预热（< 2 秒）
    syncCacheWarming();

    long syncTime = System.currentTimeMillis() - startTime;
    log.info("同步预热完成，耗时: {} ms", syncTime);

    // 阶段 2：异步预热（不阻塞启动）
    CompletableFuture<Void> asyncTask = asyncCacheWarming();

    // 注册异步任务完成回调
    asyncTask.whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("异步预热失败，懒加载将兜底", ex);
        } else {
            long totalTime = System.currentTimeMillis() - startTime;
            log.info("异步预热完成，总耗时: {} ms", totalTime);
        }
    });

    log.info("========== 启动缓存预热结束（同步阶段） ==========");

    /**
     * 重启重新添加任务到延迟队列
     */
    longTimeUserDrawMarkAndNotice.addDelayedTaskIfAbsent();
    /**
     * 通用Websocket-注册前端和后端通信操作名称对应的处理器
     */
    CommonWebSocket.registerReceiveMessageHandler(WebSocketOptionEnum.GROUP_LIST_IMAGE_BASE64.getOption(), new GroupListHandler(componentBean));
    CommonWebSocket.registerReceiveMessageHandler(WebSocketOptionEnum.SHOW_HTML_ANALYSIS_SUCCESS_UPLOAD_SVG.getOption(), new ShowHtmlAnalysisSuccessUploadSvgHandler(componentBean));
    CommonWebSocket.registerReceiveMessageHandler(WebSocketOptionEnum.WECHAT_SCAN_LOGIN.getOption(), new WechatScanLoginHandler(componentBean));
}

/**
 * 同步缓存预热
 * 加载前端首页必需的数据
 */
private void syncCacheWarming() {
    // 1. PlotType 基础信息
    List<PlotType> plotTypeList = componentBean.getRepository()
        .getPlotTypeRepository()
        .findAllByShowOrderBySequence(YesOrNo.YES.getType());
    PlotTypeCacheUtils.syncWarmupBasicInfo(componentBean, plotTypeList);

    // 2. 字典数据（已优化 N+1）
    CacheService.getGotoDictList(componentBean);

    // 3. 表单模块列表
    CacheService.getFormModuleList(componentBean);
}

/**
 * 异步缓存预热
 * 后台加载详细数据
 */
private CompletableFuture<Void> asyncCacheWarming() {
    List<PlotType> plotTypeList = componentBean.getRepository()
        .getPlotTypeRepository()
        .findAllByShowOrderBySequence(YesOrNo.YES.getType());
    return PlotTypeCacheUtils.asyncWarmupDetailInfo(componentBean, plotTypeList);
}
```

**Step 3: 删除旧的缓存预热方法**

删除原有的 `redisCacheWarming()` 和 `localCacheWarming()` 方法（如果存在）

**Step 4: 编译验证**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn compile
```

预期: 编译成功，无错误

**Step 5: 提交**

```bash
git add goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java
git commit -m "refactor: implement two-phase cache warmup

- 阶段 1：同步预热必需数据（< 2 秒）
- 阶段 2：异步预热详细数据（5-10 秒，不阻塞）
- 启动时间从 120 秒优化到 < 5 秒
- 异步失败时懒加载兜底

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## Task 7: 性能测试

**目标**: 验证启动性能优化效果

**文件**:
- Test: 手动测试

**Step 1: 启动应用并记录时间**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto/system
mvn spring-boot:run
```

观察日志输出：
- 查找 "同步预热完成，耗时: XXX ms"
- 查找 "异步预热完成，总耗时: XXX ms"

**Step 2: 验证性能指标**

检查以下指标是否达标：

| 指标 | 目标 | 实际 | 是否达标 |
|------|------|------|---------|
| 同步预热耗时 | < 5 秒 | ___ ms | ✅/❌ |
| 异步预热耗时 | < 10 秒 | ___ ms | ✅/❌ |
| 启动完成时间 | < 5 秒 | ___ ms | ✅/❌ |

**Step 3: 测试懒加载机制**

```bash
# 清空某个 PlotType 的详细缓存
redis-cli DEL "CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID1"

# 访问接口，触发懒加载
curl -X POST http://localhost:8080/api/plot/plotData \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "groupId": 1}' \
  -w "\n响应时间: %{time_total}s\n"
```

预期: 响应时间 < 500ms，数据正常返回

**Step 4: 记录测试结果**

创建测试报告：`docs/plans/2026-03-11-performance-test-result.md`

---

## Task 8: 功能回归测试

**目标**: 验证优化后功能无回归

**文件**:
- Test: 手动测试

**Step 1: 测试前端首页加载**

1. 打开浏览器访问前端首页
2. 验证 PlotType 列表正常显示
3. 验证图片和名称正常加载

预期: 首页加载正常，无错误

**Step 2: 测试详情页加载**

1. 点击任意一个 PlotType
2. 验证详细参数正常加载
3. 验证文件设置正常显示

预期: 详情页加载正常，无错误

**Step 3: 测试字典数据**

1. 访问需要字典数据的页面
2. 验证下拉框选项正常显示
3. 验证字典数据完整

预期: 字典数据正常，无遗漏

**Step 4: 测试并发场景**

使用 JMeter 或 ab 工具进行并发测试：

```bash
# 使用 ab 工具测试
ab -n 100 -c 10 -p data.json -T application/json http://localhost:8080/api/plot/plotData
```

预期: 无缓存击穿，无错误，响应时间稳定

**Step 5: 记录测试结果**

更新测试报告：`docs/plans/2026-03-11-performance-test-result.md`

**Step 6: 最终提交**

```bash
git add docs/plans/2026-03-11-performance-test-result.md
git commit -m "test: add performance and regression test results

- 启动时间：120s → < 5s（96% 提升）
- 同步预热：< 2s
- 异步预热：< 10s
- 懒加载：< 500ms
- 功能回归测试通过

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## 验收标准

### 必须满足（P0）
- ✅ 启动时间 < 5 秒
- ✅ 所有功能正常（无回归）
- ✅ 无缓存击穿（分布式锁保护）
- ✅ 异步预热失败不影响启动

### 应该满足（P1）
- ✅ 异步预热完成时间 < 10 秒
- ✅ 懒加载响应时间 < 500ms
- ✅ 缓存命中率 > 90%
- ✅ 字典查询耗时 < 1 秒

---

## 回滚方案

如果优化后出现问题，可以快速回滚：

```bash
# 1. 回滚代码
git revert HEAD~6..HEAD

# 2. 重新编译
cd /mnt/f/IdeaProjects/goto-software
mvn clean install -DskipTests

# 3. 重启应用
systemctl restart goto-system

# 4. 验证启动
tail -f logs/application.log | grep "缓存预热"
```

---

## 注意事项

1. **分布式锁**: 所有缓存操作都使用分布式锁保护，确保多实例环境下的数据一致性
2. **懒加载兜底**: 异步预热失败不影响功能，懒加载机制自动兜底
3. **重试机制**: 使用 RHandlerWrapper 内置重试，默认重试 3 次
4. **监控告警**: 建议配置监控指标和告警规则，及时发现问题
5. **灰度发布**: 建议先在测试环境验证，再逐步灰度到生产环境

---

## 相关文档

- 设计方案: `docs/plans/2026-03-11-startup-performance-optimization-design.md`
- RabbitMQ 重构计划: `docs/plans/2026-03-09-rabbitmq-to-redis-stream-implementation-v2.md`
