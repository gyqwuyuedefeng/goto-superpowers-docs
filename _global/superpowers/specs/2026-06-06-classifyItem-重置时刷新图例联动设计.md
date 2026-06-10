# 图例设置联动改进：重置 by 时刷新图例列表

## 背景

当前 `classifyLegendRefresh` 在以下场景被调用：
- `deleteFileModify`（删除文件后）
- `uploadFileModify`（上传文件后）
- `replaceFileModify`（替换文件后）
- `refreshClassifyItemList`（刷新分类项列表）

但 `by.vue` 的 `clearBy` 方法（重置 by 字段）只调用了 `resetClassifyItemByDefault`，没有触发 `classifyLegendRefresh`。

## 目标

当用户点击重置（清空 by 字段）时，立即刷新图例列表：
1. 删除对应的图例项
2. 根据图例数量更新"图例设置"分组的显隐状态

## 修改点

### 文件：`goto-web/src/views/goto/components/form/draw/by.vue`

**导入**
```javascript
import { classifyLegendRefresh } from "@/constant/classifyLegendRefresh"
```

**修改 clearBy 方法**
```javascript
clearBy() {
  console.log("====================clearBy")
  PublicFunctions.resetClassifyItemByDefault(
    {
      topParamSettingsList: this.topParamSettingsList,
      vueContext: this
    },
    this.classifyItem
  )
  console.log("after clearBy, classifyItem.by =", this.classifyItem.by)
  
  // 刷新图例列表状态
  classifyLegendRefresh({
    topParamSettingsList: this.topParamSettingsList
  })
  
  this.$forceUpdate()
  // 触发 modify 回调，通知父组件重置操作
  this.$emit("modify", this.classifyItem)
}
```

## 非目标

- 不修改其他调用 `resetClassifyItemByDefault` 的场景
- 不修改 `classifyLegendRefresh` 本身的逻辑

## 测试设计

1. 清空 by 后验证图例列表中对应图例项被删除
2. 清空最后一个 by 后验证"图例设置"分组隐藏
3. 清空前有多个 by，清空一个后验证其他图例项不受影响

## 验收标准

- 点击重置清空 by 后，图例列表立即更新
- 图例为空时"图例设置"分组自动隐藏
- 不影响其他功能
