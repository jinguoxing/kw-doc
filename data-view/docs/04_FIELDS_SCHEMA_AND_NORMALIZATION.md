# 04_FIELDS_SCHEMA_AND_NORMALIZATION

## 1. 文档目的

本文档定义源表 `t_data_view.f_fields` 的：

1. 数据形态假设
2. 标准化输出结构
3. 字段主键生成规则
4. 类型归一化规则
5. 可空值归一化规则
6. 字段级 hash 规则
7. 异常数据处理规则
8. 单元测试与样本准备要求

本文档是以下模块的直接实现依据：

- `normalizer`
- `diff_engine`
- `apply_service`

---

## 2. 当前前提与边界

## 2.1 当前已知事实

源表 `t_data_view` 中字段：

- `f_fields longtext DEFAULT NULL COMMENT '字段列表'`

说明源字段列表存储在 JSON / 文本中，需要在同步前完成解析与标准化。

## 2.2 当前未完全确定的部分

目前**还没有提供真实 `f_fields` 样本**，因此本文件中的 JSON 结构是：

- 基于行业常见结构的**标准化假设**
- 用于指导大模型和研发先产出第一版代码
- 后续拿到真实样本后需要再校正

## 2.3 当前策略

在没有真实样本前，标准化模块必须：

1. 尽量兼容多种字段命名
2. 对缺失字段采用保守默认值
3. 对无法识别的数据结构明确报错
4. 将异常样本纳入测试夹具

---

## 3. 标准化目标输出结构

## 3.1 SourceField 标准结构

`f_fields` 中每个字段对象最终都必须归一化为：

```go
type SourceField struct {
    FieldKey         string
    Name             string
    OriginalName     string
    DisplayName      string
    Type             string
    Index            int
    Nullable         *bool
    Comment          string
    DataLength       int
    DataAccuracy     int
    OriginalDataType string
    PrimaryKey       *bool
    Hash             string
    Raw              map[string]any
}
```

---

## 3.2 字段说明

| 字段 | 含义 | 备注 |
|---|---|---|
| `FieldKey` | 字段比较主键 | 当前阶段最关键字段 |
| `Name` | 技术名称 | 用于写入 `form_view_field.technical_name` |
| `OriginalName` | 原始字段名 | 用于兜底匹配与写入 `original_name` |
| `DisplayName` | 展示名称 / 业务名称候选 | 用于初始化 `business_name` |
| `Type` | 规范化后数据类型 | 用于写入 `data_type` |
| `Index` | 字段顺序 | 用于写入 `index` |
| `Nullable` | 是否可空 | 最终会转成 `YES/NO/UNKNOWN` |
| `Comment` | 字段注释 | 用于 `comment` / `field_description` |
| `DataLength` | 数据长度 | 缺失时默认为 `0` |
| `DataAccuracy` | 数据精度 | 缺失时默认为 `0` |
| `OriginalDataType` | 原始数据类型 | 缺失时回退 `Type` |
| `PrimaryKey` | 是否主键 | 缺失时保持 `nil` |
| `Hash` | 字段级 hash | 用于快速比较 |
| `Raw` | 原始对象 | 用于调试与扩展 |

---

## 4. `f_fields` 的兼容输入格式假设

当前阶段假设 `f_fields` 至少支持以下几类格式：

---

## 4.1 格式 A：标准对象数组

```json
[
  {
    "name": "order_id",
    "original_name": "order_id",
    "display_name": "订单ID",
    "type": "bigint",
    "nullable": false,
    "comment": "订单主键",
    "data_length": 20,
    "data_accuracy": 0,
    "primary_key": true
  },
  {
    "name": "order_amount",
    "original_name": "order_amount",
    "display_name": "订单金额",
    "type": "decimal",
    "nullable": true,
    "comment": "订单金额",
    "data_length": 18,
    "data_accuracy": 2
  }
]
```

这是推荐和最理想的结构。

---

## 4.2 格式 B：字段名不一致但语义相近

```json
[
  {
    "field_name": "order_id",
    "label": "订单ID",
    "data_type": "BIGINT",
    "is_nullable": "NO",
    "description": "订单主键"
  }
]
```

当前阶段 normalizer 应尽量兼容这类字段名差异。

---

## 4.3 格式 C：字段信息不全

```json
[
  {
    "name": "dt"
  }
]
```

这种情况下要补默认值，但不能让程序崩掉。

---

## 4.4 不支持的格式

以下输入当前阶段建议直接视为非法：

1. 根节点不是 JSON 数组
2. 数组元素不是对象
3. `name` / `field_name` / `original_name` 这类关键字段全部缺失
4. 完全不可反序列化

---

## 5. 字段提取规则

## 5.1 `Name` 提取规则

优先级如下：

1. `name`
2. `field_name`
3. `technical_name`

### 规则
- 取到后做 `trim`
- 保留原始大小写用于展示
- 另生成标准化小写副本用于比较

### 如果为空
- 视为非法字段
- 当前字段解析失败
- 当前视图可以选择：
  - 整体失败（推荐）
  - 或把该字段记录为异常并跳过（不推荐）

---

## 5.2 `OriginalName` 提取规则

优先级如下：

1. `original_name`
2. `source_name`
3. `name`
4. `field_name`

### 规则
- 若源里没有显式原始字段名，则回退到技术名称

---

## 5.3 `DisplayName` 提取规则

优先级如下：

1. `display_name`
2. `label`
3. `business_name`
4. `comment_name`
5. `name`

### 规则
- 若无展示名称，则回退 `name`

---

## 5.4 `Comment` 提取规则

优先级如下：

1. `comment`
2. `description`
3. `remark`

### 规则
- 若都没有，则为空字符串

---

## 5.5 `Type` 提取规则

优先级如下：

1. `type`
2. `data_type`
3. `column_type`

### 规则
- 去前后空格
- 转小写
- 去掉长度部分后得到逻辑类型（如有需要）

例子：

| 原值 | 归一化结果 |
|---|---|
| `BIGINT` | `bigint` |
| `varchar` | `varchar` |
| `DECIMAL(18,2)` | `decimal` |
| `timestamp` | `timestamp` |

---

## 5.6 `OriginalDataType` 提取规则

优先级如下：

1. `original_data_type`
2. `column_type`
3. 原始 `type/data_type` 字段值

### 规则
- 若仍为空，回退使用规范化后的 `Type`

---

## 5.7 `Nullable` 提取规则

兼容以下输入：

- `true / false`
- `"true" / "false"`
- `"YES" / "NO"`
- `"Y" / "N"`
- `"1" / "0"`
- `1 / 0`

### 规则
- 能识别则输出 `*bool`
- 无法识别则输出 `nil`

---

## 5.8 `DataLength` 提取规则

优先级如下：

1. `data_length`
2. `length`
3. 从 `column_type` 中解析长度
4. 默认 `0`

例子：

- `varchar(255)` -> `255`
- `decimal(18,2)` -> `18`

---

## 5.9 `DataAccuracy` 提取规则

优先级如下：

1. `data_accuracy`
2. `scale`
3. 从 `column_type` 中解析精度
4. 默认 `0`

例子：

- `decimal(18,2)` -> `2`

---

## 5.10 `PrimaryKey` 提取规则

兼容以下字段：

- `primary_key`
- `is_primary_key`
- `pk`

兼容值：

- `true/false`
- `1/0`
- `YES/NO`

若无法识别，则为 `nil`。

---

## 6. `FieldKey` 生成规则

这是本文件最关键的规则之一。

## 6.1 当前阶段最终规则

### 优先级
1. `OriginalName`
2. `Name`

### 标准化处理
生成前统一做：

1. `trim` 前后空格
2. 转小写
3. 去掉包裹引号：
   - `` ` ``
   - `"`
   - `'`

### 例子

| OriginalName | Name | FieldKey |
|---|---|---|
| `ORDER_ID` | `order_id` | `order_id` |
| `` `user_id` `` | `user_id` | `user_id` |
| 空 | `dt` | `dt` |

---

## 6.2 为什么这样设计

因为当前本地 `form_view_field` 没有独立的 `source_field_key` 字段，  
而字段 rename 场景又很难 100% 自动识别，所以第一阶段必须用一个稳定、简单、可推导的规则。

---

## 6.3 后续增强建议

后续建议把 `FieldKey` 显式落到本地字段表：

- `source_field_key`

这样字段级匹配和性能都会更稳定。

---

## 7. 类型归一化规则

## 7.1 逻辑类型归一化表（第一阶段）

| 输入类型示例 | 归一化类型 |
|---|---|
| `bigint` / `BIGINT` | `bigint` |
| `int` / `integer` | `int` |
| `smallint` | `smallint` |
| `tinyint` | `tinyint` |
| `varchar` / `varchar(255)` | `varchar` |
| `char` / `char(32)` | `char` |
| `text` / `longtext` / `mediumtext` | `text` |
| `decimal` / `decimal(18,2)` / `numeric` | `decimal` |
| `double` | `double` |
| `float` | `float` |
| `date` | `date` |
| `datetime` | `datetime` |
| `timestamp` | `timestamp` |
| `json` | `json` |
| `boolean` / `bool` | `bool` |

---

## 7.2 未识别类型

如果遇到未识别类型：

- `Type` 保留规范化后的原始值（小写）
- `OriginalDataType` 保留源值
- 不直接报错

### 原因
第一阶段优先保证流程能跑通，不因为某个新类型直接阻断整批任务。

---

## 8. 可空值归一化规则

## 8.1 标准输出

在 `SourceField.Nullable` 中统一表示为：

- `true`
- `false`
- `nil`

## 8.2 落到本地 `form_view_field.is_nullable` 时的映射

建议统一为字符串：

| Nullable | `is_nullable` |
|---|---|
| `true` | `YES` |
| `false` | `NO` |
| `nil` | `UNKNOWN` |

---

## 9. Hash 规则

## 9.1 字段级 Hash

字段级 hash 用于快速判断字段是否变化。

### 输入字段
建议对以下标准化字段做 canonical json：

- `FieldKey`
- `Name`
- `OriginalName`
- `DisplayName`
- `Type`
- `Index`
- `Nullable`
- `Comment`
- `DataLength`
- `DataAccuracy`
- `OriginalDataType`
- `PrimaryKey`

### 输出
- SHA1 或 MD5 均可
- 当前阶段建议：
  - **实现简单优先：SHA1**

---

## 9.2 视图级 Payload Hash

视图级 hash 用于快速判断整个视图是否变化。

### 输入字段
建议包含：

- `view_name`
- `technical_name`
- `type`
- `query_type`
- `comment`
- `status`
- `meta_table_name`
- **标准化后的字段数组**

### 字段数组要求
计算 hash 前必须先：

1. 对字段按 `FieldKey` 排序
2. 再序列化

这样才能避免数组顺序噪音带来的误报。

---

## 10. 异常数据处理规则

## 10.1 `f_fields` 为空

### 情况
- `NULL`
- 空字符串
- `"[]"`

### 处理
- 解析为空字段列表
- 不直接报错

---

## 10.2 `f_fields` 非法 JSON

### 处理建议
当前阶段推荐：

- 视图级标准化失败
- 当前视图同步失败
- 任务进入 `retry_waiting` 或 `failed`

### 原因
字段结构都无法解析时，继续同步很容易生成不完整本地数据。

---

## 10.3 单个字段关键字段缺失

### 情况
- `name` 和 `original_name` 都缺失

### 处理建议
- 当前视图标准化失败
- 不建议静默跳过该字段

---

## 10.4 长度/精度解析失败

### 处理建议
- `DataLength = 0`
- `DataAccuracy = 0`
- 保持 `OriginalDataType`

---

## 11. 建议的大模型代码实现约束

为了让大模型生成代码更稳定，本文件要求生成的 `normalizer` 至少满足：

1. **不得直接依赖固定单一字段名**
   - 必须支持 `name / field_name / technical_name` 等兼容提取
2. **必须保留 Raw**
   - 便于调试和后续增强
3. **必须输出 FieldKey**
4. **必须输出 Hash**
5. **必须显式处理错误**
   - 非法 JSON
   - 关键字段缺失
6. **不得把源数组顺序直接当作唯一逻辑主键**
   - 顺序只是 `Index`
   - 主键仍然是 `FieldKey`

---

## 12. 测试样本要求

在真实 `f_fields` 样本到位前，建议先准备以下测试用例：

### Case 1：标准字段数组
- name/original_name/display_name/type/nullable 全部存在

### Case 2：字段名别名
- 用 `field_name`、`label`、`data_type`

### Case 3：类型带长度精度
- `varchar(255)`
- `decimal(18,2)`

### Case 4：nullable 为多种格式
- `true`
- `"YES"`
- `1`
- `"0"`

### Case 5：空字段数组
- `[]`

### Case 6：非法 JSON
- `{`
- `not-json`

### Case 7：关键字段缺失
- 只有 `display_name`，没有 `name` / `original_name`

---

## 13. 当前阶段的 TODO

当前文件仍有以下待确认项：

1. 真实 `f_fields` 样本中到底使用哪些字段名
2. 是否存在嵌套结构
3. 是否存在字段级别特殊属性（例如 code table、masking、semantic type）
4. 主键标识的真实来源字段名
5. `nullable` 的真实表示格式
6. `column_type` 的真实样例

---

## 14. 与后续文档的关系

本文件输出之后，下一步应继续补：

1. `05_DIFF_RULES_SPEC.md`
2. `06_APPLY_RULES_SPEC.md`

因为本文件解决的是：

- **源字段如何被标准化**

后面两份解决的是：

- **标准化后的对象怎么比较**
- **比较后的结果怎么落库**

---

## 15. 最终决策摘要

### 当前 `f_fields` 标准化的核心规则

1. 根节点必须是字段对象数组
2. `Name` 优先取 `name`，兼容 `field_name`
3. `OriginalName` 优先取 `original_name`，缺失回退 `name`
4. `DisplayName` 优先取 `display_name`，缺失回退 `label` / `name`
5. `Type` 优先取 `type`，缺失回退 `data_type`
6. `FieldKey = normalize(original_name or name)`
7. 字段级 hash 基于标准化字段生成
8. 非法 JSON 或关键字段缺失视为当前视图标准化失败
9. 当前阶段优先兼容多种字段名，不追求一次性穷尽所有源结构
