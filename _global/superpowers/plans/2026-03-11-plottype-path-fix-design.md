# PlotTypeUtils 路径修复设计

## 问题描述

在 `PlotTypeUtils.obtainParam` 方法中，拼接文件路径时使用了 `obtainUserDrawPath`，该方法返回 Windows 路径格式。但这些路径最终会传递给 R 执行，而 R 运行在 Linux 环境（Docker 容器）中，导致路径格式不匹配。

### 具体问题位置

1. **第 312 行区域**：生成结果文件路径（`FILE_NAME_FIELD_NAME` 参数）
2. **第 332 行区域**：生成上传文件路径（`UploadFilePathParam` 参数）

### 环境差异

**开发环境（application-common-home.yml）：**
- `chartUser`: `F:\IdeaProjects\goto-software\goto-deploy-backend\public\ChartRoot\user` (Windows)
- `chartUserForR`: `/root/software/public/ChartRoot/user` (Linux Docker)
- Java 运行在 Windows，R 运行在 Docker 容器（Linux）

**生产环境（application-common-public.yml）：**
- `chartUser`: `/root/software/public/ChartRoot/user` (Linux)
- `chartUserForR`: `/root/software/public/ChartRoot/user` (Linux)
- Java 和 R 都运行在 Linux 环境，路径相同

## 设计方案

### 核心思路

采用**双路径变量方案**：
- **Windows 路径**：用于 Java 本地文件操作（如 `FileUtil.exist()`）
- **Linux 路径**：用于传递给 R 的参数字符串

这种设计在开发环境和生产环境都能正常工作：
- 开发环境：两个路径不同，分别用于不同目的
- 生产环境：两个路径相同，不会有副作用

### 修改点 1：第 312 行区域（结果文件路径）

**当前代码：**
```java
if (PlotTypeUtils.FILE_NAME_FIELD_NAME.equals(param.paramName)) {
    String pre = "";
    if (plotType.isExample()) {
        pre = obtainPlotSamplePath(componentBean, plotGroup.getFolderName(), plotType.getFolderName());
    } else {
        pre = PlotTypeUtils.obtainUserDrawPath(componentBean, plotType.getUserDraw());
    }
    builder.append(pre + "/" + PlotTypeUtils.RESULT_FOLDER_NAME + "/" + param.getValue());
}
```

**修改后：**
```java
if (PlotTypeUtils.FILE_NAME_FIELD_NAME.equals(param.paramName)) {
    String preForR = "";
    if (plotType.isExample()) {
        preForR = obtainPlotSamplePath(componentBean, plotGroup.getFolderName(), plotType.getFolderName());
    } else {
        preForR = PlotTypeUtils.obtainUserDrawPathForR(componentBean, plotType.getUserDraw());
    }
    builder.append(preForR + "/" + PlotTypeUtils.RESULT_FOLDER_NAME + "/" + param.getValue());
}
```

**说明：** 这里只需要传给 R 的路径，不需要文件检查，所以只用 `ForR` 版本即可。

### 修改点 2：第 332 行区域（上传文件路径）

**当前代码：**
```java
} else if (clazz.equals(UploadFilePathParam.class)) {
    UploadFilePathParam param = (UploadFilePathParam) o;

    String filePath;
    String pre = "";
    if (plotType.isExample()) {
        pre = obtainPlotSamplePath(componentBean, plotGroup.getFolderName(), plotType.getFolderName());
    } else {
        pre = PlotTypeUtils.obtainUserDrawPath(componentBean, plotType.getUserDraw());
    }

    if (ObjectUtils.allNotNull(param.getFileSettings(), param.getFileSettings().getName(), param.getFileSettings().getContentType())) {
        filePath = pre + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator +
                   param.getFileSettings().getName() + ContentTypeConstant.findByType(param.getFileSettings().getContentType()).getExtension();
    } else {
        if (ObjectUtils.isEmpty(param.getValue())) {
            throw new RuntimeException("请选择关联文件或者输入自定义文件名称");
        }
        filePath = pre + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + param.getValue() + ".xlsx";
    }

    if (FileUtil.exist(filePath)) {
        builder.append(filePath);
    } else {
        param.setSingleQuotationMarks(false);
        builder.append("NULL");
    }
}
```

**修改后：**
```java
} else if (clazz.equals(UploadFilePathParam.class)) {
    UploadFilePathParam param = (UploadFilePathParam) o;

    String filePathWindows;  // 用于文件检查
    String filePathLinux;    // 用于传给 R
    String preWindows = "";
    String preLinux = "";

    if (plotType.isExample()) {
        preWindows = obtainPlotSamplePath(componentBean, plotGroup.getFolderName(), plotType.getFolderName());
        preLinux = preWindows;  // 示例路径可能不需要区分
    } else {
        preWindows = PlotTypeUtils.obtainUserDrawPath(componentBean, plotType.getUserDraw());
        preLinux = PlotTypeUtils.obtainUserDrawPathForR(componentBean, plotType.getUserDraw());
    }

    if (ObjectUtils.allNotNull(param.getFileSettings(), param.getFileSettings().getName(), param.getFileSettings().getContentType())) {
        String fileName = param.getFileSettings().getName() +
                         ContentTypeConstant.findByType(param.getFileSettings().getContentType()).getExtension();
        filePathWindows = preWindows + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + fileName;
        filePathLinux = preLinux + "/" + PlotTypeUtils.UPLOAD_FOLDER_NAME + "/" + fileName;
    } else {
        if (ObjectUtils.isEmpty(param.getValue())) {
            throw new RuntimeException("请选择关联文件或者输入自定义文件名称");
        }
        String fileName = param.getValue() + ".xlsx";
        filePathWindows = preWindows + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + fileName;
        filePathLinux = preLinux + "/" + PlotTypeUtils.UPLOAD_FOLDER_NAME + "/" + fileName;
    }

    // 用 Windows 路径检查文件是否存在
    if (FileUtil.exist(filePathWindows)) {
        // 用 Linux 路径传给 R
        builder.append(filePathLinux);
    } else {
        param.setSingleQuotationMarks(false);
        builder.append("NULL");
    }
}
```

**说明：** 这里需要同时维护两个路径，因为需要用 Windows 路径检查文件是否存在，但传给 R 的参数需要使用 Linux 路径。

## 设计要点

### 1. 环境自适应

- **开发环境**：`chartUser` 和 `chartUserForR` 不同，双路径生效
- **生产环境**：两者相同，双路径实际上是同一个值，不会有副作用

### 2. 职责分离

- `preWindows` / `filePathWindows`：用于 Java 本地文件操作
- `preLinux` / `filePathLinux`：用于传递给 R 的参数

### 3. 路径分隔符

- Windows 路径使用 `File.separator`（`\`）
- Linux 路径使用硬编码 `/`

### 4. 示例路径处理

- 示例路径（`obtainPlotSamplePath`）可能不需要区分，两个变量可以使用相同值
- 如果需要，可以后续添加 `obtainPlotSamplePathForR` 方法

## 影响范围

### 修改文件

- `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`

### 影响方法

- `obtainParam` 方法（私有方法）

### 风险评估

- **风险等级**：低
- **原因**：
  1. 只修改私有方法内部实现，不影响公共 API
  2. 在生产环境中，两个路径相同，行为不变
  3. 在开发环境中，修复了路径格式问题

## 测试计划

### 单元测试

1. 测试开发环境下的路径生成
2. 测试生产环境下的路径生成
3. 测试文件存在和不存在的情况

### 集成测试

1. 在开发环境中运行完整的绘图流程
2. 验证 R 能够正确接收和使用 Linux 路径
3. 验证文件检查功能正常工作

## 实施步骤

1. 修改 `obtainParam` 方法中的两处路径拼接逻辑
2. 在开发环境中测试
3. 代码审查
4. 合并到主分支
5. 部署到生产环境

## 后续优化

如果发现示例路径（`obtainPlotSamplePath`）也需要区分 Windows 和 Linux 版本，可以：
1. 添加 `obtainPlotSamplePathForR` 方法
2. 在配置文件中添加 `chartExampleForR` 配置项
3. 更新相关代码使用新方法

## 实施状态

- [x] 设计完成 (2026-03-11)
- [x] 实现完成 (2026-03-11)
- [ ] 开发环境测试
- [ ] 生产环境部署
- [ ] 生产环境验证

## 实施记录

- **修改文件**: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`
- **修改行数**: 第 307-314 行，第 324-364 行
- **提交哈希**: 6c9e528
- **提交信息**: fix: correct path handling in PlotTypeUtils.obtainParam for R execution
- **修改内容**:
  - 使用 obtainUserDrawPathForR 获取结果文件路径（第 312 行）
  - 实现双路径变量方案处理上传文件路径（第 332 行）
  - Windows 路径用于文件存在性检查，Linux 路径用于 R 参数
  - 修复开发环境中的路径格式不匹配问题
