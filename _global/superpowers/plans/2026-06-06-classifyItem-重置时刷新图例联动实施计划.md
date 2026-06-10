# classifyItem 重置时刷新图例联动实施计划

## 目标

当用户点击重置（清空 by 字段）时，立即调用 `classifyLegendRefresh` 刷新图例列表状态。

## 文件修改

**Modify:** `goto-web/src/views/goto/components/form/draw/by.vue`
- 新增导入 `classifyLegendRefresh`
- 在 `clearBy` 方法末尾添加 `classifyLegendRefresh` 调用

## 实施步骤

**Step 1: 修改 by.vue**

在文件顶部添加导入：
```javascript
import { classifyLegendRefresh } from "@/constant/classifyLegendRefresh"
```

在 `clearBy` 方法中添加调用：
```javascript
// 刷新图例列表状态
classifyLegendRefresh({
  topParamSettingsList: this.topParamSettingsList
})
```

**Step 2: 运行前端测试**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constants/classifyLegendRefresh.spec.js --runInBand
```

**Step 3: 提交**

```bash
git add goto-web/src/views/goto/components/form/draw/by.vue
git commit -m "feat: 图例重置时刷新图例列表联动"
```
