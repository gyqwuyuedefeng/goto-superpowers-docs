# goto-web 画图首页预览图批量懒加载设计

## 1. 背景

当前画图首页位于 `goto-web/src/views/goto/plot/index.vue`。现有链路是：

1. `groupList()` 返回分组和图类型结构。
2. 前端用 IndexedDB 的 `group-list-i-j-image` key 回填图片。
3. 前端发送 `group_list_image_base64` WebSocket 消息。
4. 后端 `GroupListHandler` 按分组索引 `i` 和类型索引 `j` 逐张推送图片。

这个设计存在两个核心问题：

1. 图片和卡片通过数组位置绑定，分组或类型排序变化、隐藏项过滤、增删配置后容易错位。
2. 前端持久化保存图片并全量接收 WebSocket 回图，不是真正按视口懒加载。

本次目标是把画图首页预览图改成稳定业务 ID 驱动的批量 HTTP 懒加载，并停止该页面的本地持久化图片缓存。

## 2. 已确认决策

1. 允许同时修改前端和后端。
2. 采用方案 B：批量 HTTP 懒加载。
3. 不再在该页面使用 WebSocket 让后端按索引推图。
4. 前端不再用 IndexedDB、localStorage 等持久化缓存图片。
5. 允许页面生命周期内使用内存状态去重，避免同一张卡片滚动进出视口时重复请求。
6. 后端接口只接受稳定业务 ID，不接受前端传入的 `fileName` 或 `folderName` 作为读图依据。

## 3. 目标

1. 用 `groupId + typeId` 或等价稳定 ID 绑定图片和卡片，彻底避免 `i/j` 错位。
2. 只在卡片进入视口附近后请求图片，实现真正懒加载。
3. 合并短时间内触发的卡片请求，避免逐张 HTTP 请求造成请求风暴。
4. 前端只保留页面内状态，不持久化保存图片。
5. 后端从现有后端缓存或文件读取预览图，返回每张卡片的独立状态。
6. 单张图失败或缺失不影响同批其他卡片，也不影响页面结构展示和绘图入口。

## 4. 非目标

1. 不重做画图首页视觉风格。
2. 不修改 `plot/draw` 绘图页业务逻辑。
3. 不改造其他页面的 WebSocket 使用方式。
4. 不在本轮引入 CDN 或独立图片服务。
5. 不实现前端跨页面、跨刷新图片缓存。

## 5. 总体架构

首页加载拆成两条链路：

1. 结构链路：`groupList()` 返回分组、类型、排序、显示权限过滤后的轻量结构。
2. 预览图链路：前端根据视口触发批量 HTTP 请求，后端按稳定业务 ID 返回图片结果。

`index.vue` 不再打开 IndexedDB，不再读取本地图片缓存，不再发送 `commonWebSocketMessageSendToServer` 请求预览图，也不再处理 `groupListImageBase64(data)`。

## 6. 后端接口设计

新增接口：

`POST api/plot/group_preview_batch`

请求体：

```json
{
  "items": [
    {
      "groupId": 1,
      "typeId": 11
    }
  ]
}
```

响应体：

```json
{
  "code": 200,
  "data": {
    "items": [
      {
        "groupId": 1,
        "typeId": 11,
        "cardKey": "1:11",
        "status": "ready",
        "image": "data:image/png;base64,..."
      },
      {
        "groupId": 1,
        "typeId": 12,
        "cardKey": "1:12",
        "status": "empty",
        "image": null
      }
    ]
  }
}
```

### 6.1 请求校验

1. `items` 为空时返回空 `items`，不报错。
2. 单批数量上限固定为 20；超过上限时返回参数错误，不做静默截断。
3. `groupId` 和 `typeId` 必须非空。
4. 后端不能信任前端传入的文件路径信息。

### 6.2 查图规则

后端根据当前登录用户计算可见范围，复用 `PlotGroupUtils.currentUserCanViewHiddenPlot()` 和 `PlotGroupUtils.filterPlotGroupSmallListByVisibility(...)` 的语义。

处理流程：

1. 从 `PlotGroupUtils.obtainStorePlotGroupSmallList(componentBean)` 获取后端缓存结构。
2. 按当前用户权限过滤分组和类型。
3. 用 `groupId + typeId` 在过滤后的结构里反查 `PlotGroupSmall` 和 `PlotTypeSmall`。
4. 命中后调用现有 `PlotTypeUtils.obtainPlotSampleResultBase64(componentBean, groupId, fileName, folderName)` 读取预览图。
5. 未命中、图片为空、读取异常分别映射为独立状态。

### 6.3 单项状态

1. `ready`：读取到图片，返回 `image`。
2. `empty`：卡片可见，但后端明确没有预览图。
3. `failed`：单张图读取异常。
4. 请求的 `groupId/typeId` 不在当前用户可见范围内时，返回 `failed`，且不返回隐藏项存在与否的细节。

单项失败不应导致整个接口失败。只有认证、权限、请求体格式、服务不可用等全局问题才走接口级错误。

## 7. 前端状态设计

在 `index.vue` 中维护页面生命周期内的内存状态：

```js
previewMap: {
  "1:11": {
    status: "idle",
    image: null,
    retryCount: 0,
    error: null
  }
}
```

卡片状态：

1. `idle`：尚未进入懒加载范围。
2. `queued`：已进入视口附近，等待批量合并。
3. `loading`：已发送批量请求，等待响应。
4. `ready`：图片可展示。
5. `empty`：后端明确无预览图。
6. `failed`：加载失败。

状态展示：

1. `idle / queued / loading` 显示加载占位，不显示“暂无预览”。
2. `ready` 显示真实图片。
3. `empty` 显示“暂无预览”。
4. `failed` 显示失败态。
5. 点击绘图逻辑保持不变。

## 8. 前端懒加载队列

### 8.1 初始化

`groupList()` 成功后：

1. 遍历分组和类型。
2. 给每张卡片生成 `cardKey = group.id + ':' + plotType.id`。
3. 初始化 `previewMap[cardKey]` 为 `idle`。
4. 渲染完整分组和卡片结构。

### 8.2 视口触发

使用 `IntersectionObserver` 监听卡片容器。卡片进入视口附近时：

1. 如果状态是 `idle`，更新为 `queued`。
2. 将 `{ groupId, typeId, cardKey }` 加入待请求队列。
3. 通过短延迟合并请求，建议 80-120ms。

### 8.3 批量请求

队列发送规则：

1. 每批最多 20 张，与后端批量上限保持一致。
2. 同一 `cardKey` 在 `queued / loading / ready / empty` 状态下不重复请求。
3. `failed` 可在再次进入视口时重试，建议最多 1 次。
4. 接口返回后按 `cardKey` 或 `groupId/typeId` 回填，不使用数组索引。
5. 接口整体失败时，本批 `loading` 卡片转为 `failed`。

### 8.4 生命周期清理

组件销毁时：

1. 断开 `IntersectionObserver`。
2. 清理队列合并 timer。
3. 标记组件已销毁，忽略已过期响应。

## 9. 兼容与迁移

1. `group_list_image_base64` WebSocket handler 可以先保留给其他潜在调用方，但 `index.vue` 不再触发它。
2. `plot_draw_images` IndexedDB store 可以先不全局删除，避免影响未知旧逻辑；本页面停止使用即可。
3. `PlotTypeSmall.imageBase64` 字段如果仍由后端缓存模型保留，本页面不依赖它作为图片来源。
4. 后续如果要彻底删除旧 WebSocket 选项和 IndexedDB store，需要另开清理任务确认没有其他页面依赖。

## 10. 错误处理

1. `groupList()` 失败：页面进入结构级错误态，提供重试入口或沿用现有错误处理。
2. 批量预览接口失败：只影响当前批图片，不影响分组结构和卡片点击。
3. 单张图片失败：该卡片进入 `failed`，其他卡片继续更新。
4. 单张图片为空：该卡片进入 `empty`，不再反复请求。
5. 响应中包含未知 `cardKey`：忽略，避免污染当前页面状态。

## 11. 测试与验证

### 11.1 前端验证

1. `groupList()` 成功后渲染分组和卡片结构，但不会立即请求全部图片。
2. 只有进入视口附近的卡片会进入队列。
3. 多张卡片短时间触发时会合并成批量请求。
4. 响应按 `cardKey/groupId/typeId` 回填，不依赖数组索引。
5. `ready / empty / failed` 状态展示正确。
6. `index.vue` 不再调用 IndexedDB 图片读写方法。
7. `index.vue` 不再发送 `group_list_image_base64` WebSocket 消息。
8. 组件销毁后不再更新已卸载页面状态。

### 11.2 后端验证

1. 批量接口只返回当前用户可见分组和类型的图片。
2. 普通用户请求隐藏分组或隐藏类型时拿不到图片。
3. 管理员按既有规则可访问隐藏项。
4. 单张图读取异常只影响该项，接口仍返回其他项结果。
5. 空图片返回 `empty`。
6. 请求数量超过上限时行为符合固定规则。

### 11.3 建议命令

前端优先运行项目现有校验命令，例如：

```bash
npm run lint
```

后端优先运行批量接口和可见性相关测试，例如：

```bash
mvn -pl system -Dtest=PlotServiceImplPlotGroupVisibilityTest,GroupListHandlerVisibilityTest test
```

实际实施后应按新增测试类名称补充更精确的命令。

## 12. 直接影响文件

预期实施影响：

1. `goto-web/src/views/goto/plot/index.vue`
2. `goto-web/src/api/plot.js`
3. `goto/system/src/main/java/com/freedom/rest/PlotController.java`
4. `goto/system/src/main/java/com/freedom/service/PlotService.java`
5. `goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java`
6. 后端新增请求/响应 DTO，具体包路径按现有项目风格确定。
7. 前后端相关测试文件。

## 13. 最终结论

本次重构采用批量 HTTP 懒加载，而不是继续使用 WebSocket 索引回图，也不是前端持久化缓存图片。

最终链路是：

`结构先返回 -> 卡片进入视口 -> 前端批量请求稳定 ID -> 后端从缓存取图 -> 前端按 cardKey 更新页面内状态`

这能同时解决错位、全量缓存、非懒加载和请求风暴问题，并且保留后续升级到图片 URL 服务化的空间。
