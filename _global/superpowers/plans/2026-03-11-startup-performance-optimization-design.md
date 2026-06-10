# goto/system 模块启动性能优化设计方案

> **设计日期**: 2026-03-11
> **设计目标**: 将启动时间从 120 秒优化到 5 秒以内
> **优化方案**: 两阶段预热 + 批量操作 + 异步化

---

## 一、问题分析

### 1.1 当前问题

启动缓存预热耗时约 **2 分钟**，主要瓶颈：

1. **PlotType 缓存（最严重）**
   - 位置：`goto/common/src/main/java/com/freedom/util/cache/PlotTypeCacheUtils.java:117-128`
   - 问题：62 个 PlotType 对象逐个写入 Redis，耗时 **74 秒**
   - 原因：每次 `bucket.set()` 都是一次网络 I/O

2. **字典查询 N+1 问题**
   - 位置：`goto/common/src/main/java/com/freedom/cache/CacheService.java:121`
   - 问题：重复查询 `sys_dict_detail` 表，每个 dict_id 单独查询
   - 原因：循环中逐个查询，假设 50 个字典 = 1 + 50 = 51 次查询
   - 耗时：约 **7 秒**

3. **缓存预热阻塞启动**
   - 位置：`goto/system/src/main/java/com/freedom/runner/DispatchInitializer.java:67-68`
   - 问题：同步执行，阻塞主线程
   - 影响：应用启动后 2 分钟才能对外提供服务

### 1.2 优化目标

- **启动时间**: 从 120 秒优化到 **< 5 秒**
- **PlotType 缓存**: 从 74 秒优化到 **5-10 秒**（异步）
- **字典查询**: 从 7 秒优化到 **< 1 秒**
- **用户体验**: 启动后立即可用，异步预热对用户无感知

---

## 二、架构设计

### 2.1 整体架构

```
启动流程（DispatchInitializer.run()）
│
├─ 阶段 1：同步预热（< 2 秒）✅ 阻塞启动
│  ├─ 1.1 PlotType 基础信息预热
│  │   └─ RMap 批量写入：CACHE_PLOT_TYPE
│  │       (62 个 PlotTypeSmallOne: id, groupId, folderName)
│  │
│  └─ 1.2 字典数据预热（优化 N+1）
│      ├─ 批量查询：SELECT * FROM sys_dict_detail WHERE dict_id IN (...)
│      ├─ 内存分组：按 dict_id 分组
│      └─ 本地缓存：Caffeine Cache
│
├─ 启动完成 🚀 对外提供服务
│
└─ 阶段 2：异步预热（后台 5-10 秒）⚡ 不阻塞启动
   └─ 2.1 PlotType 详细信息预热
       ├─ 使用 CompletableFuture.runAsync()
       ├─ RBatch 批量写入：CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id
       │   (62 个 PlotTypeFileSettingsAndParamSettings: fileSettings, paramSettings)
       └─ 完成后记录日志
```

### 2.2 关键设计点

1. **阶段划分依据**
   - **阶段 1（同步）**: 前端首页必需的数据（列表展示）
   - **阶段 2（异步）**: 点击时才需要的数据（详细参数）

2. **性能提升来源**
   - PlotType 基础信息：RMap 批量操作（< 1 秒）
   - 字典查询：从 N+1 优化到 1 次查询（从 7 秒降到 < 1 秒）
   - PlotType 详细信息：异步 + RBatch（不阻塞启动）

3. **兜底机制**
   - 懒加载：`findPlotTypeById()` 已有分布式锁保护
   - 降级策略：异步失败不影响启动，懒加载兜底

---

## 三、核心组件设计

### 3.1 PlotType 缓存优化

#### 3.1.1 阶段 1：同步预热基础信息

```java
/**
 * 同步预热 PlotType 基础信息
 * 耗时: < 1 秒
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
        map.putAll(batchData);  // ✅ 一次性写入

        log.info("同步预热完成：{} 个 PlotType 基础信息", plotTypeList.size());
        return null;
    });

    wrapper.doHandler(componentBean.getRedissonClient(), RedissonKey.DL_CACHE_PLOT_TYPE);
}
```

#### 3.1.2 阶段 2：异步预热详细信息

```java
/**
 * 异步预热 PlotType 详细信息
 * 耗时: 5-10 秒（后台执行）
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
            batch.execute();  // ✅ 一次性提交，1 次网络 I/O

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

### 3.2 字典查询优化

#### 3.2.1 新增批量查询方法

```java
// DictDetailCustomRepository.java
public interface DictDetailCustomRepository extends JpaRepository<DictDetailCustom, Long> {
    // 原有方法
    List<DictDetailCustom> findByDictIdOrderByDictSortAsc(Long dictId);

    // ✅ 新增：批量查询方法
    List<DictDetailCustom> findByDictIdInOrderByDictIdAscDictSortAsc(List<Long> dictIds);
}
```

#### 3.2.2 优化查询逻辑

```java
// CacheService.java
public static List<DictSimple> getGotoDictList(ComponentBean componentBean) {
    ConcurrentMap<Long, DictSimple> longDictSimpleConcurrentMap = gotoDictMapCache.asMap();
    List<DictSimple> gotoDictList = new ArrayList<>();

    if (longDictSimpleConcurrentMap.isEmpty()) {
        synchronized (GOTO_DICT_LOCK) {
            longDictSimpleConcurrentMap = gotoDictMapCache.asMap();
            if (longDictSimpleConcurrentMap.isEmpty()) {
                // 查询所有字典
                List<DictCustom> dictCustomList = componentBean.getRepository()
                    .getDictCustomRepository().findAllGotoDict();

                if (GCollectionUtils.listNotNullNotEmpty(dictCustomList)) {
                    // ✅ 批量查询所有字典详情（1 次查询）
                    List<Long> dictIds = dictCustomList.stream()
                        .map(DictCustom::getId)
                        .collect(Collectors.toList());
                    List<DictDetailCustom> allDetails = componentBean.getRepository()
                        .getDictDetailCustomRepository()
                        .findByDictIdInOrderByDictIdAscDictSortAsc(dictIds);

                    // ✅ 在内存中按 dict_id 分组
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
                longDictSimpleConcurrentMap.forEach((k, v) -> gotoDictList.add(v));
            }
        }
    } else {
        longDictSimpleConcurrentMap.forEach((k, v) -> gotoDictList.add(v));
    }

    return GsonUtils.deepCloneList(gotoDictList, new TypeToken<List<DictSimple>>() {}.getType());
}
```

### 3.3 启动流程改造

```java
// DispatchInitializer.java
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

    // 其他启动任务...
    longTimeUserDrawMarkAndNotice.addDelayedTaskIfAbsent();
    CommonWebSocket.registerReceiveMessageHandler(...);
}

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

private CompletableFuture<Void> asyncCacheWarming() {
    List<PlotType> plotTypeList = componentBean.getRepository()
        .getPlotTypeRepository()
        .findAllByShowOrderBySequence(YesOrNo.YES.getType());
    return PlotTypeCacheUtils.asyncWarmupDetailInfo(componentBean, plotTypeList);
}
```

---

## 四、数据流设计

### 4.1 启动流程时序图

```
应用启动
    │
    ├─ Spring Boot 初始化
    │
    ├─ CommandLineRunner.run() 触发
    │   │
    │   ├─ [同步阶段] syncCacheWarming()
    │   │   │
    │   │   ├─ 1. 查询数据库
    │   │   │   ├─ findAllByShowOrderBySequence() → 62 个 PlotType
    │   │   │   ├─ findAllGotoDict() → 50 个 Dict
    │   │   │   └─ findByDictIdIn() → 批量查询 DictDetail (1 次查询)
    │   │   │
    │   │   ├─ 2. 写入 Redis (< 2 秒)
    │   │   │   ├─ RMap.putAll() → CACHE_PLOT_TYPE (基础信息)
    │   │   │   └─ 分布式锁保护: DL_CACHE_PLOT_TYPE
    │   │   │
    │   │   └─ 3. 写入本地缓存
    │   │       ├─ Caffeine Cache → gotoDictMapCache
    │   │       └─ Caffeine Cache → formModuleListCache
    │   │
    │   ├─ 启动完成 ✅ (耗时 < 5 秒)
    │   │
    │   └─ [异步阶段] asyncCacheWarming()
    │       │
    │       └─ CompletableFuture.runAsync()
    │           │
    │           ├─ 1. 清理旧缓存
    │           │   └─ keys.delete(CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID*)
    │           │
    │           ├─ 2. RBatch 批量写入 (5-10 秒)
    │           │   ├─ batch.getBucket(key).setAsync() × 62
    │           │   ├─ batch.execute() → 1 次网络 I/O
    │           │   └─ 分布式锁保护: DL_CACHE_PLOT_TYPE
    │           │
    │           └─ 3. 完成回调
    │               ├─ 成功: log.info("异步预热完成")
    │               └─ 失败: log.error("异步预热失败，懒加载将兜底")
    │
    └─ 其他启动任务
```

### 4.2 用户访问流程

#### 场景 1：异步预热已完成（正常情况）

```
用户访问 /plot/type/draw/:groupId/:typeId
    │
    ├─ 前端: obtainPlotData()
    │   └─ API: POST /api/plot/plotData
    │
    ├─ 后端: PlotServiceImpl.plotData()
    │   │
    │   ├─ PlotTypeCacheUtils.findPlotTypeById(id)
    │   │   │
    │   │   ├─ 1. 获取详细信息 (分布式锁保护)
    │   │   │   ├─ RBucket.get(CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id)
    │   │   │   └─ ✅ 缓存命中 (异步预热已完成)
    │   │   │
    │   │   └─ 2. 获取基础信息
    │   │       └─ RMap.get(CACHE_PLOT_TYPE, id)
    │   │
    │   └─ 返回数据 (< 100ms)
    │
    └─ 前端渲染
```

#### 场景 2：异步预热未完成（懒加载兜底）

```
用户访问 /plot/type/draw/:groupId/:typeId
    │
    ├─ 前端: obtainPlotData()
    │   └─ API: POST /api/plot/plotData
    │
    ├─ 后端: PlotServiceImpl.plotData()
    │   │
    │   ├─ PlotTypeCacheUtils.findPlotTypeById(id)
    │   │   │
    │   │   ├─ 1. 尝试获取详细信息 (分布式锁保护)
    │   │   │   ├─ RBucket.get(CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id)
    │   │   │   └─ ❌ 缓存未命中 (异步预热未完成)
    │   │   │
    │   │   ├─ 2. 懒加载触发 (分布式锁保护 + 重试机制)
    │   │   │   ├─ 查询数据库: findById(id)
    │   │   │   ├─ 写入缓存: bucket.set(plotTypeFileSettingsAndParamSettings)
    │   │   │   └─ 返回数据
    │   │   │
    │   │   └─ 3. 获取基础信息
    │   │       └─ RMap.get(CACHE_PLOT_TYPE, id)
    │   │
    │   └─ 返回数据 (< 500ms，首次稍慢)
    │
    └─ 前端渲染
```

### 4.3 缓存键设计

```
基础信息: CACHE_PLOT_TYPE (RMap)
├─ Key: Long (plotTypeId)
└─ Value: PlotTypeSmallOne {id, groupId, folderName}

详细信息: CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id (RBucket)
├─ Key: String (前缀 + plotTypeId)
└─ Value: PlotTypeFileSettingsAndParamSettings {id, fileSettings, paramSettings}

分布式锁: DL_CACHE_PLOT_TYPE (全局锁)
分布式锁: DL_CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id (单个 PlotType 锁)
```

---

## 五、错误处理和降级策略

### 5.1 异常场景分析

| 场景 | 影响范围 | 降级策略 | 恢复方案 |
|------|---------|---------|---------|
| **同步预热失败** | 启动失败 | ❌ 无降级（必须成功） | 重启应用 |
| **异步预热失败** | 首次访问变慢 | ✅ 懒加载兜底 | 后台重试 |
| **Redis 连接失败** | 缓存不可用 | ✅ 直接查询数据库 | 恢复 Redis 连接 |
| **分布式锁超时** | 单次请求失败 | ✅ 重试机制（RHandlerWrapper） | 自动恢复 |
| **数据库查询超时** | 启动变慢 | ⚠️ 增加超时时间 | 优化查询 |
| **并发缓存击穿** | 数据库压力大 | ✅ 分布式锁保护 | 自动恢复 |

### 5.2 错误处理实现

#### 5.2.1 异步预热失败处理

```java
// DispatchInitializer.java
private CompletableFuture<Void> asyncCacheWarming() {
    List<PlotType> plotTypeList = componentBean.getRepository()
        .getPlotTypeRepository()
        .findAllByShowOrderBySequence(YesOrNo.YES.getType());

    return PlotTypeCacheUtils.asyncWarmupDetailInfo(componentBean, plotTypeList)
        .exceptionally(ex -> {
            // 异步预热失败，记录错误但不影响启动
            log.error("异步预热失败，懒加载将兜底", ex);
            return null;
        });
}
```

#### 5.2.2 懒加载重试机制（使用 RHandlerWrapper 内置重试）

```java
// PlotTypeCacheUtils.java
public static PlotType findPlotTypeById(ComponentBean componentBean, Long id) {
    if (id == null) {
        throw new IllegalArgumentException("PlotType ID不能为空");
    }

    PlotType plotType = new PlotType();
    plotType.setId(id);

    RHandlerWrapper<PlotTypeFileSettingsAndParamSettings> wrapper = new RHandlerWrapper(() -> {
        RBucket<PlotTypeFileSettingsAndParamSettings> bucket =
            componentBean.getRedissonClient().getBucket(
                RedissonKey.CACHE_PLOT_TYPE_PARAM_SETTINGS_FILE_SETTINGS_BY_ID + id
            );

        PlotTypeFileSettingsAndParamSettings data = bucket.get();

        if (data == null) {
            log.debug("缓存未命中，触发懒加载，ID: {}", id);

            Optional<PlotType> plotTypeOptional = componentBean.getRepository()
                .getPlotTypeRepository().findById(id);

            if (plotTypeOptional.isPresent()) {
                PlotType dbPlotType = plotTypeOptional.get();
                data = new PlotTypeFileSettingsAndParamSettings(
                    dbPlotType.getId(),
                    dbPlotType.getFileSettings(),
                    dbPlotType.getParamSettings()
                );
                bucket.set(data);
                log.debug("懒加载完成，ID: {}", id);
            } else {
                throw new RuntimeException("PlotType 不存在，ID: " + id);
            }
        }

        return data;
    });

    try {
        // ✅ 使用 RHandlerWrapper 内置重试：默认重试 3 次，递增等待 100ms, 150ms, 200ms
        PlotTypeFileSettingsAndParamSettings data = wrapper.doHandlerWithRetry(
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

        if (data != null) {
            plotType.setFileSettings(data.getFileSettings());
            plotType.setParamSettings(data.getParamSettings());
        }

    } catch (RLockException e) {
        log.error("获取 PlotType 失败（重试 3 次后），ID: {}", id, e);
        throw new RuntimeException("系统繁忙，请稍后重试", e);
    }

    // 获取基础信息
    RMap<Long, PlotTypeSmallOne> typeMap = componentBean.getRedissonClient()
        .getMap(RedissonKey.CACHE_PLOT_TYPE);
    PlotTypeSmallOne smallOne = typeMap.get(id);
    if (smallOne != null) {
        plotType.setGroupId(smallOne.getGroupId());
    }

    return plotType;
}
```

### 5.3 监控指标

| 指标 | 说明 | 阈值 |
|------|------|------|
| `cache.warmup.sync` | 同步预热耗时 | < 5 秒 |
| `cache.warmup.async` | 异步预热耗时 | < 10 秒 |
| `cache.lazy.load` | 懒加载触发次数 | < 10% |
| `cache.lock.failure` | 分布式锁失败次数 | < 5% |
| `cache.hit` | 缓存命中率 | > 90% |

---

## 六、测试验证计划

### 6.1 性能测试

| 测试项 | 优化前 | 优化后 | 提升 |
|--------|--------|--------|------|
| 启动时间 | 120 秒 | < 5 秒 | **96%** |
| PlotType 缓存 | 74 秒（同步） | 5-10 秒（异步） | **87%** |
| 字典查询 | 7 秒（N+1） | < 1 秒（批量） | **86%** |
| 首次访问 | < 100ms（缓存命中） | < 500ms（懒加载） | 可接受 |

### 6.2 功能测试

| 用例 ID | 测试场景 | 预期结果 |
|---------|---------|---------|
| TC-01 | 正常启动 | 启动时间 < 5 秒 |
| TC-02 | 异步预热完成 | 所有缓存写入成功 |
| TC-03 | 异步预热失败 | 懒加载兜底，不影响功能 |
| TC-04 | 首页加载 | 列表正常显示 |
| TC-05 | 点击详情 | 详细数据正常加载 |
| TC-06 | 缓存未命中 | 懒加载触发，数据正常返回 |
| TC-07 | 并发访问 | 无缓存击穿，数据一致 |

### 6.3 验收标准

#### 必须满足（P0）
- ✅ 启动时间 < 5 秒
- ✅ 所有功能正常（无回归）
- ✅ 无缓存击穿（分布式锁保护）
- ✅ 异步预热失败不影响启动

#### 应该满足（P1）
- ✅ 异步预热完成时间 < 10 秒
- ✅ 懒加载响应时间 < 500ms
- ✅ 缓存命中率 > 90%
- ✅ 字典查询耗时 < 1 秒

---

## 七、实施计划

### 7.1 改动文件清单

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `PlotTypeCacheUtils.java` | 新增 + 修改 | 新增两阶段预热方法 |
| `DictDetailCustomRepository.java` | 新增 | 新增批量查询方法 |
| `CacheService.java` | 修改 | 优化字典查询逻辑 |
| `DispatchInitializer.java` | 修改 | 改造启动流程 |

### 7.2 实施步骤

1. **开发阶段**（3-5 天）
   - 实现两阶段预热逻辑
   - 优化字典查询
   - 改造启动流程
   - 单元测试

2. **测试阶段**（2-3 天）
   - 性能测试
   - 功能测试
   - 并发测试
   - 回归测试

3. **灰度发布**（1-2 周）
   - 测试环境验证
   - 生产环境单实例灰度
   - 监控观察
   - 全量发布

### 7.3 回滚方案

```bash
# 快速回滚
git revert <commit-hash>
systemctl restart goto-system
```

---

## 八、风险评估

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| 异步预热失败 | 首次访问变慢 | 低 | 懒加载兜底 |
| 分布式锁竞争 | 启动变慢 | 低 | 重试机制 |
| 缓存击穿 | 数据库压力 | 极低 | 分布式锁保护 |
| 功能回归 | 业务异常 | 低 | 充分测试 + 灰度发布 |

---

## 九、总结

### 9.1 核心优化点

1. **两阶段预热**：同步加载必需数据，异步加载详细数据
2. **批量操作**：RMap.putAll() + RBatch.execute() 减少网络 I/O
3. **批量查询**：优化字典查询从 N+1 到 2 次查询
4. **懒加载兜底**：异步失败不影响功能，分布式锁保护

### 9.2 预期收益

- **启动时间**：从 120 秒降到 < 5 秒（**96% 提升**）
- **用户体验**：启动后立即可用，无感知优化
- **系统稳定性**：分布式锁保护，无缓存击穿风险
- **可维护性**：利用现有机制，代码改动最小

---

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:writing-plans to create implementation plan.
