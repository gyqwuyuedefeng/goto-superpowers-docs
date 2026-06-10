# UserDraw page_list 图片格式参数设计

## 背景

当前 `page_list` 接口的图片返回逻辑固定读取用户结果目录下的 `Result/<fileName>.svg`，再转成 base64 写入返回结果中的 `image` 字段。

现状链路如下：

- `UserDrawController.pageList`
- `UserDrawServiceImpl.pageList`
- `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1`

当前问题：

- 接口调用方无法指定返回图片格式
- 后端实现将结果文件后缀写死为 `.svg`
- 即使结果目录中存在 `.png` 文件，`page_list` 也不会读取

## 目标

为 `POST /api/userDraw/page_list` 增加一个随 `UserDrawQueryCriteria` 一起传递的图片格式参数，使接口支持按参数读取已有的结果图片并返回 base64。

明确目标如下：

- 新增 `imageType` 请求参数
- 支持调用方传 `png`
- 不传时默认按 `svg` 处理
- `png` 文件存在时，返回其 base64
- `png` 文件不存在时，返回空图片值
- 保持现有返回结构不变，继续复用 `image` 字段

## 非目标

本设计不包含以下内容：

- 不改造 `page_list` 为“请求时实时生成 png”
- 不在本次需求中修改分析主链路的图片产出逻辑
- 不改造下载链路中的 `svg/png` 转换逻辑
- 不新增独立图片接口
- 不修改分页、权限、筛选和响应包装结构

## 现状分析

当前 `page_list` 的实现特点：

- `UserDrawController.pageList` 仅透传 `UserDrawQueryCriteria`
- `UserDrawServiceImpl.pageList` 查询分页数据后，调用 `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1`
- `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1` 通过 `paramSettings` 中的 `fileName` 参数拼接结果文件路径
- 该路径当前被写死为：

```text
<userDrawPath>/Result/<fileName>.svg
```

- 如果文件存在，则通过现有工具转为 base64；不存在则保持空值

因此，默认返回 `svg` 不是前端行为，而是后端文件后缀硬编码导致的。

## 方案决策

采用“最小改动扩展现有接口”的方案。

### 推荐方案

在 `UserDrawQueryCriteria` 中新增 `imageType` 字段，并将其透传至图片填充工具方法。图片填充逻辑根据 `imageType` 动态拼接结果文件后缀，读取已有文件并转为 base64。

选择原因：

- 与现有 `POST + JSON body` 设计一致
- 接口兼容性最好，不需要调整 controller 签名
- 只影响 `page_list` 查询链路，改动边界清晰
- 满足“存在则按该格式返回，不存在则空”的需求
- 不把图片生成责任混入分页查询接口

### 放弃方案 1：严格校验非法值并直接报错

不采用原因：

- 会改变当前接口的宽松容错风格
- 老调用方如果传错值会直接失败
- 与本次“默认兼容 svg”目标不完全一致

### 放弃方案 2：传 `png` 时若文件不存在则实时从 `svg` 转换

不采用原因：

- 会把脚本转换、性能开销和失败重试引入分页接口
- 会扩大改动范围到结果生成和文件转换链路
- 超出本次需求边界

## 接口设计

### 请求体扩展

在 `UserDrawQueryCriteria` 中新增：

- `String imageType`

请求示例：

```json
{
  "page": 0,
  "size": 10,
  "imageType": "png"
}
```

### 参数规则

- 不传 `imageType` 时，默认使用 `svg`
- 传空串时，按 `svg` 处理
- 传入大小写混合值时，统一转小写后处理
- 仅识别 `svg` 和 `png`
- 对于非法值，例如 `jpg`、`jpeg`、`pdf`，统一兜底为 `svg`

### 响应设计

响应结构不变：

- 不新增字段
- 仍使用原有 `image` 字段
- `image` 的值仍是文件内容对应的 base64 字符串

## 改动边界

本次设计建议控制在以下 4 个位置：

1. `goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java`
2. `goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java`
3. `goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java`
4. `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`

其中：

- `UserDrawQueryCriteria` 负责承接请求参数
- `UserDrawServiceImpl.pageList` 负责解析并透传 `imageType`
- `UserDrawCommonUtils` 负责按图片类型构建目标文件路径
- `PlotTypeUtils` 继续负责文件存在判断和 base64 读取，必要时补充小型辅助方法

不建议修改：

- `UserDrawController`
- 图片生成流程
- 下载流程
- 返回模型结构

## 详细设计

### 1. QueryCriteria 扩展

给 `UserDrawQueryCriteria` 增加 `imageType` 字段，作为 `page_list` 接口的唯一新增输入口。

该字段仅参与图片读取逻辑：

- 不参与数据库查询条件
- 不影响分页参数
- 不影响权限和筛选逻辑

### 2. Service 层处理

`UserDrawServiceImpl.pageList` 在查询到分页数据后：

1. 读取 `criteria.getImageType()`
2. 对值进行标准化处理
3. 得到最终使用的图片类型
4. 调用图片填充方法时将该值透传下去

建议标准化规则：

```text
null / 空串 / 非法值 -> svg
png -> png
svg -> svg
```

### 3. 图片填充逻辑调整

当前 `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1` 中写死了 `.svg`。

建议改为：

- 保留现有方法作为默认入口，内部默认调用 `svg`
- 新增或重载支持 `imageType` 的方法
- 文件路径从：

```text
Result/<fileName>.svg
```

改为：

```text
Result/<fileName>.<imageType>
```

这样可以最小化影响已有调用者，同时为 `page_list` 增加可控行为。

### 4. 文件读取行为

文件读取行为保持当前宽松语义：

- 文件存在：转 base64，写入 `image`
- 文件不存在：`image` 为空
- 不抛出异常影响整页返回

这与当前 `svg` 的行为保持一致，只是把后缀从硬编码变成参数控制。

## 容错设计

### 非法图片类型

对于调用方传入的非法 `imageType`：

- 不返回 400
- 不抛业务异常
- 统一回退到 `svg`

原因：

- 用户已确认接受该兼容策略
- 可避免老调用方或前端传错值时影响分页主功能

### 文件不存在

当目标文件不存在时：

- `image` 返回空值
- 不影响分页数据的其他字段
- 不影响整页状态码

例如：

- 请求 `png`
- 目录下没有 `Result/<fileName>.png`
- 则该条记录的 `image = null` 或保持空值

### 缺少 `fileName` 参数

如果某条 `UserDrawSimple1` 的 `paramSettings` 中无法解析出 `fileName`：

- 沿用当前宽松逻辑
- 不报错
- `image` 为空

## 数据流

调整后的调用链如下：

1. 前端请求 `POST /api/userDraw/page_list`
2. 请求体携带 `imageType`
3. `UserDrawController.pageList` 接收 `UserDrawQueryCriteria`
4. `UserDrawServiceImpl.pageList` 标准化 `imageType`
5. `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(..., imageType)` 按类型拼接文件路径
6. 若文件存在，则调用现有 base64 工具读取
7. 将结果写入分页列表中每条记录的 `image`

## 测试设计

至少覆盖以下场景：

### 单元/服务测试

- `imageType = null` 时，默认读取 `.svg`
- `imageType = ""` 时，默认读取 `.svg`
- `imageType = "svg"` 时，读取 `.svg`
- `imageType = "png"` 时，读取 `.png`
- `imageType = "PNG"` 时，能正常标准化为 `png`
- `imageType = "jpg"` 时，回退读取 `.svg`

### 文件缺失测试

- 请求 `png` 且对应 `.png` 文件不存在时，`image` 为空
- 请求 `svg` 且对应 `.svg` 文件不存在时，`image` 为空
- `paramSettings` 中缺少 `fileName` 时，`image` 为空

### 回归测试

- 分页查询总数和列表内容不受影响
- 权限校验不受影响
- 现有不传 `imageType` 的前端调用保持原行为

## 验收标准

满足以下条件视为完成：

1. `page_list` 支持在请求体中传 `imageType`
2. 不传时默认返回 `svg` 的 base64
3. 传 `png` 且文件存在时，返回 `.png` 的 base64
4. 传 `png` 且文件不存在时，`image` 为空，不报错
5. 传非法值时自动回退为 `svg`
6. 返回结构、分页行为和权限逻辑保持不变

## 风险与边界说明

风险较低，主要原因：

- 改动只发生在参数透传和文件后缀选择
- 不涉及数据库结构
- 不涉及分析计算逻辑
- 不涉及下载和转换链路

需要明确的边界：

- 本次只负责“按参数读取已有文件”
- 如果业务后续要求“分析完成后自动生成 png”，应另起设计
- 如果后续要求支持更多格式，应补充统一枚举或常量定义，而不是继续散落硬编码

## 实施建议

推荐实施顺序：

1. 扩展 `UserDrawQueryCriteria.imageType`
2. 在 `UserDrawServiceImpl.pageList` 中补标准化逻辑
3. 调整 `UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1`
4. 补充必要测试
5. 做接口回归验证
