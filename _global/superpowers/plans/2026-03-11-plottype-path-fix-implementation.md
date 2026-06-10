# PlotTypeUtils 路径修复实现计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 修复 PlotTypeUtils.obtainParam 方法中的路径拼接问题，使其在开发环境中能够正确处理 Windows 和 Linux 路径差异

**Architecture:** 在 obtainParam 方法中维护双路径变量（Windows 用于文件检查，Linux 用于传递给 R），利用现有的 obtainUserDrawPath 和 obtainUserDrawPathForR 方法

**Tech Stack:** Java, Spring Boot, Hutool FileUtil

---

## Task 1: 修改第 312 行区域（结果文件路径）

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java:307-314`

**Step 1: 备份当前代码并定位修改位置**

查看当前代码：
```bash
grep -n "FILE_NAME_FIELD_NAME.equals" goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java
```

Expected: 显示第 307 行附近的代码

**Step 2: 修改路径变量名和获取方法**

将第 307-314 行的代码从：
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

修改为：
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

**Step 3: 编译验证**

```bash
cd goto/common
mvn clean compile
```

Expected: BUILD SUCCESS

**Step 4: 提交第一处修改**

```bash
git add goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java
git commit -m "fix: use obtainUserDrawPathForR for result file path in obtainParam"
```

---

## Task 2: 修改第 332 行区域（上传文件路径）

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java:324-353`

**Step 1: 定位修改位置**

查看当前代码：
```bash
grep -n "UploadFilePathParam.class" goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java
```

Expected: 显示第 324 行附近的代码

**Step 2: 修改为双路径变量方案**

将第 324-353 行的代码从：
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
        filePath = pre + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + param.getFileSettings().getName() + ContentTypeConstant.findByType(param.getFileSettings().getContentType()).getExtension();
    } else {
        // 自定义文件名称
        if (ObjectUtils.isEmpty(param.getValue())) {
            throw new RuntimeException("请选择关联文件或者输入自定义文件名称");
        }
        filePath = pre + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + param.getValue() + ".xlsx";
    }

    if (FileUtil.exist(filePath)) {
        builder.append(filePath);
    } else {
        /**
         * 如果是NULL的话，就不加引号了
         */
        param.setSingleQuotationMarks(false);
        builder.append("NULL");
    }
```

修改为：
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
        String fileName = param.getFileSettings().getName() + ContentTypeConstant.findByType(param.getFileSettings().getContentType()).getExtension();
        filePathWindows = preWindows + File.separator + PlotTypeUtils.UPLOAD_FOLDER_NAME + File.separator + fileName;
        filePathLinux = preLinux + "/" + PlotTypeUtils.UPLOAD_FOLDER_NAME + "/" + fileName;
    } else {
        // 自定义文件名称
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
        /**
         * 如果是NULL的话，就不加引号了
         */
        param.setSingleQuotationMarks(false);
        builder.append("NULL");
    }
```

**Step 3: 编译验证**

```bash
cd goto/common
mvn clean compile
```

Expected: BUILD SUCCESS

**Step 4: 提交第二处修改**

```bash
git add goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java
git commit -m "fix: use dual-path approach for upload file path in obtainParam"
```

---

## Task 3: 检查是否有其他使用 obtainUserDrawPath 的地方

**Files:**
- Read: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`

**Step 1: 搜索 obtainParam 方法中的其他路径使用**

```bash
grep -n "obtainUserDrawPath\|obtainPlotSamplePath" goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java | grep -A5 -B5 "obtainParam"
```

Expected: 确认只有两处需要修改（已完成）

**Step 2: 验证修改完整性**

检查 obtainParam 方法中是否还有其他地方拼接路径并传递给 R：
```bash
sed -n '217,400p' goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java | grep -n "builder.append"
```

Expected: 确认所有传递给 R 的路径都已使用正确的方法

**Step 3: 记录检查结果**

如果发现其他需要修改的地方，重复 Task 1 或 Task 2 的步骤。

---

## Task 4: 开发环境测试

**Files:**
- Test: 手动测试绘图功能

**Step 1: 启动开发环境**

确保使用 `application-common-home.yml` 配置：
```bash
# 检查配置
grep -A2 "chartUser:" goto/common/src/main/resources/application-common-home.yml
```

Expected:
```
chartUser: '${properties.config.r.chartRoot}\user'
chartUserForR: '${properties.config.r.chartRootForR}/user'
```

**Step 2: 测试结果文件路径生成**

创建一个测试用例，验证生成的参数字符串中包含正确的 Linux 路径格式：
- 选择一个绘图类型
- 设置文件名参数
- 执行绘图
- 检查传递给 R 的参数字符串

Expected: 参数中的路径应该是 `/root/software/public/ChartRoot/user/...` 格式

**Step 3: 测试上传文件路径生成**

- 上传一个测试文件
- 执行绘图
- 检查文件检查功能正常（使用 Windows 路径）
- 检查传递给 R 的参数使用 Linux 路径

Expected:
- 文件检查成功（FileUtil.exist 返回 true）
- R 接收到的路径是 Linux 格式

**Step 4: 验证 R 执行成功**

- 完整执行一次绘图流程
- 检查 R 是否成功生成结果文件
- 检查结果文件是否保存在正确位置

Expected: 绘图成功，结果文件正常生成

**Step 5: 记录测试结果**

创建测试报告：
```bash
echo "测试日期: $(date)" > docs/plans/2026-03-11-plottype-path-fix-test-report.md
echo "测试环境: 开发环境 (Windows + Docker R)" >> docs/plans/2026-03-11-plottype-path-fix-test-report.md
echo "测试结果: [PASS/FAIL]" >> docs/plans/2026-03-11-plottype-path-fix-test-report.md
```

---

## Task 5: 代码审查和最终提交

**Files:**
- Review: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`

**Step 1: 代码审查检查清单**

检查以下项目：
- [ ] 变量命名清晰（preWindows, preLinux, filePathWindows, filePathLinux）
- [ ] 注释准确（"用于文件检查"、"用于传给 R"）
- [ ] 路径分隔符正确（Windows 用 File.separator，Linux 用 "/"）
- [ ] 逻辑正确（文件检查用 Windows 路径，builder.append 用 Linux 路径）
- [ ] 没有引入新的 bug

**Step 2: 运行完整编译**

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn clean install -DskipTests
```

Expected: BUILD SUCCESS

**Step 3: 查看所有修改**

```bash
git diff HEAD~2 goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java
```

Expected: 显示两处修改，确认符合设计

**Step 4: 创建最终提交（如果需要合并提交）**

如果想要合并之前的两个提交：
```bash
git reset --soft HEAD~2
git commit -m "fix: correct path handling in PlotTypeUtils.obtainParam for R execution

- Use obtainUserDrawPathForR for result file paths (line 312)
- Implement dual-path approach for upload file paths (line 332)
- Windows path for file existence check, Linux path for R parameter
- Fixes path format mismatch in development environment

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

**Step 5: 推送到远程分支**

```bash
git push origin redis-stream-refactor
```

---

## Task 6: 文档更新

**Files:**
- Update: `docs/plans/2026-03-11-plottype-path-fix-design.md`

**Step 1: 在设计文档中添加实施状态**

在设计文档末尾添加：
```markdown
## 实施状态

- [x] 设计完成 (2026-03-11)
- [x] 实现完成 (2026-03-11)
- [x] 开发环境测试 (2026-03-11)
- [ ] 生产环境部署
- [ ] 生产环境验证

## 实施记录

- 修改文件: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`
- 修改行数: 第 307-314 行，第 324-353 行
- 提交哈希: [commit-hash]
```

**Step 2: 提交文档更新**

```bash
git add docs/plans/2026-03-11-plottype-path-fix-design.md
git commit -m "docs: update implementation status in design document"
```

---

## 验收标准

### 功能验收
- [ ] 开发环境中，Java 使用 Windows 路径检查文件存在性
- [ ] 开发环境中，R 接收到正确的 Linux 路径格式参数
- [ ] 生产环境中，路径处理正常（两个路径相同，无副作用）
- [ ] 绘图功能正常，结果文件正确生成

### 代码质量
- [ ] 代码符合项目编码规范
- [ ] 变量命名清晰，注释准确
- [ ] 没有引入新的 bug
- [ ] 编译通过，无警告

### 文档完整性
- [ ] 设计文档完整
- [ ] 实现计划详细
- [ ] 测试报告记录
- [ ] 提交信息清晰

---

## 回滚计划

如果发现问题需要回滚：

```bash
# 回滚到修改前的提交
git revert [commit-hash]

# 或者直接重置
git reset --hard [commit-before-changes]
git push origin redis-stream-refactor --force
```

---

## 后续优化建议

1. **添加单元测试**：为 obtainParam 方法添加单元测试，覆盖不同环境配置
2. **示例路径处理**：如果发现示例路径也需要区分，添加 obtainPlotSamplePathForR 方法
3. **配置验证**：在应用启动时验证 chartUser 和 chartUserForR 配置的一致性
4. **日志增强**：在路径生成时添加调试日志，便于排查问题
