# 05_DIFF_RULES_SPEC

## 1. 文档目的

本文档定义 `sync-worker` 在执行 `mdl.t_data_view -> 本地 form_view / form_view_field` 同步时的 **diff 比较规则合同**。

本文档直接约束：

- `diff_engine`
- `runner`
- `apply_service`
- `watermark` 推进逻辑

本文档关注：

1. 视图级 diff 分类规则
2. 字段级 diff 分类规则
3. 严重度分级规则
4. `last_safe_cursor` 的计算规则
5. 部分成功 / 部分失败的处理原则
6. 并发比较时的顺序约束

---

## 2. 依赖关系

本文件依赖以下前置文档：

1. `03_FIELD_MAPPING_CONTRACT.md`
2. `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`

其中：

- `03` 解决“源字段怎么映射到本地表”
- `04` 解决“`f_fields` 怎么标准化成 `SourceField`”
- `05` 解决“标准化后的对象怎么比较”

---

## 3. 术语定义

## 3.1 SourceView
由 `t_data_view` 一行标准化后得到的视图对象。

## 3.2 LocalView
由本地 `form_view` 一行构造的视图对象。

## 3.3 SourceField
由 `t_data_view.f_fields` 中一个字段标准化后得到的字段对象。

## 3.4 LocalField
由本地 `form_view_field` 一行构造的字段对象。

## 3.5 View-level Diff
视图级比较，决定该视图是：

- 新增
- 更新
- 删除
- 无变化

## 3.6 Field-level Diff
字段级比较，决定该字段是：

- 新增
- 更新
- 删除
- 疑似 rename
- 无变化

## 3.7 Safe Cursor / `last_safe_cursor`
在当前 datasource 的有序处理过程中，**已经连续成功完成 apply** 的最后一个安全断点。

---

## 4. 视图级匹配规则

## 4.1 一级匹配主键

视图级匹配采用以下优先级：

### 第一优先级
- `form_view.mdl_id == t_data_view.f_view_id`

### 第二优先级（历史兜底）
- `form_view.datasource_id == t_data_view.f_data_source_id`
- `form_view.technical_name == t_data_view.f_technical_name`

---

## 4.2 兜底命中后的补齐规则

如果通过第二优先级命中了本地视图，则：

1. 本次同步仍视为命中同一视图
2. apply 时必须回填：
   - `form_view.mdl_id = t_data_view.f_view_id`

### 原因
避免后续继续走 fallback，提升匹配稳定性与性能。

---

## 4.3 当前阶段不允许的匹配方式

当前阶段不允许把下面字段当作视图主键：

- `f_view_name`
- `business_name`
- `uniform_catalog_code`

### 原因
这些字段业务变化概率高，不适合作为稳定主键。

---

## 5. 视图级 diff 分类规则

## 5.1 总体分类

每个 `SourceView` 在视图级必须被归类到以下四类之一：

1. `NEW_VIEW`
2. `UPDATE_VIEW`
3. `DELETE_VIEW`
4. `UNCHANGED_VIEW`

---

## 5.2 `NEW_VIEW` 判定规则

满足以下条件时判定为 `NEW_VIEW`：

1. 按一级主键未命中本地 `form_view`
2. 按兜底规则也未命中本地 `form_view`
3. 当前源对象不是删除态

### 输出动作
- 生成 `CreateViewDiff`
- 后续 apply 时创建：
  - `form_view`
  - 对应全部 `form_view_field`

---

## 5.3 `DELETE_VIEW` 判定规则

满足以下任一条件时，判定为 `DELETE_VIEW`：

1. `t_data_view.f_delete_time > 0`
2. `SourceView.Status` 明确表示删除/下线（如后续源语义确认）

### 额外约束
仅当本地视图已命中时，才生成 `DeleteViewDiff`。  
如果源对象是删除态，但本地不存在，则：

- 当前阶段默认忽略
- 不生成新对象
- 不作为错误

---

## 5.4 `UNCHANGED_VIEW` 判定规则

满足以下条件时判定为 `UNCHANGED_VIEW`：

1. 已命中本地视图
2. 源对象不是删除态
3. 本地已保存 `source_payload_hash`
4. `source_payload_hash == SourceView.PayloadHash`

### 输出动作
- 直接跳过字段级深比较
- 统计：
  - `hash_skipped + 1`

---

## 5.5 `UPDATE_VIEW` 判定规则

满足以下条件时判定为 `UPDATE_VIEW`：

1. 已命中本地视图
2. 源对象不是删除态
3. 不满足 `UNCHANGED_VIEW`
4. 视图级字段或字段级对象有变化

### 输出动作
- 继续做字段级比较
- 生成 `UpdateViewDiff`

---

## 6. 视图级比较字段

以下字段参与视图级比较。

---

## 6.1 强比较字段

| SourceView 字段 | LocalView 字段 | 说明 |
|---|---|---|
| `ViewID` | `mdl_id` | 一级主键 |
| `TechnicalName` | `technical_name` | 技术名称 |
| `Comment` | `comment` | 注释 |
| `Type` | `type` | 当前阶段本地固定 1，若源不满足元数据视图条件则不进入主链路 |
| `QueryType` | 暂不直接落本地，作为视图级变化信息参与比较 | 查询类型 |
| `MetaTableName` | `original_name`（或视图侧映射候选） | 原始表名候选 |
| `DeleteTime` | `deleted_at` | 删除态 |
| `PayloadHash` | `source_payload_hash`（推荐增强字段） | 快速跳过深比较 |
| `ChangeTime` | `source_change_time`（推荐增强字段） | 最近源变更时间 |

---

## 6.2 条件比较字段

以下字段是否比较，取决于本地是否决定由源驱动：

| SourceView 字段 | LocalView 字段 | 当前阶段策略 |
|---|---|---|
| `ViewName` | `business_name` | 仅在本地为空时用于初始化，不作为稳定更新比较主字段 |
| `Status` | `status` | 本地 `status` 由 diff 结果计算，不直接等于源状态 |

---

## 6.3 视图级 delta 输出结构

建议 `diff_engine` 统一输出：

```go
type ViewMetaDelta struct {
    ViewNameChanged      bool
    TechnicalNameChanged bool
    CommentChanged       bool
    TypeChanged          bool
    QueryTypeChanged     bool
    StatusChanged        bool
    MetaTableNameChanged bool
    DeletedChanged       bool
}
```

### 说明
- `ViewNameChanged` 在当前阶段主要用于记录变化，不一定直接触发覆盖写 `business_name`
- `StatusChanged` 指源侧状态/删除态变化，不等价于本地 `status` 字段枚举变化

---

## 7. 字段级匹配规则

## 7.1 当前阶段字段匹配优先级

在同一个视图下，字段匹配采用：

### 第一优先级
- `technical_name`

### 第二优先级
- `original_name`

### 比较前标准化规则
必须统一做：

1. trim 空格
2. lowercase
3. 去掉反引号/双引号/单引号

---

## 7.2 为什么当前不直接要求 `source_field_key`

因为当前本地表结构中没有显式：

- `source_field_key`

所以第一阶段必须基于现有字段完成匹配。

### 后续增强建议
建议后续给 `form_view_field` 增加：

- `source_field_key`
- `source_field_hash`

这样可把字段比较从“弱匹配”升级到“强匹配”。

---

## 8. 字段级 diff 分类规则

每个字段最终必须落入以下类别之一：

1. `FIELD_CREATE`
2. `FIELD_UPDATE`
3. `FIELD_DELETE`
4. `FIELD_SUSPECTED_RENAME`
5. `FIELD_UNCHANGED`

---

## 8.1 `FIELD_CREATE` 判定规则

满足以下条件时判定：

1. 源字段存在
2. 本地字段按匹配规则未命中

### 输出动作
- 将该字段加入 `FieldDelta.Created`

---

## 8.2 `FIELD_DELETE` 判定规则

满足以下条件时判定：

1. 本地字段存在
2. 源字段按匹配规则未命中

### 输出动作
- 将该字段加入 `FieldDelta.Deleted`

---

## 8.3 `FIELD_UPDATE` 判定规则

满足以下条件时判定：

1. 源字段与本地字段匹配命中
2. 以下任一字段发生变化：

- `DisplayName`
- `OriginalName`
- `Type`
- `Index`
- `Nullable`
- `Comment`
- `DataLength`
- `DataAccuracy`
- `OriginalDataType`
- `PrimaryKey`

### 输出动作
- 将该字段加入 `FieldDelta.Updated`

---

## 8.4 `FIELD_UNCHANGED` 判定规则

满足以下条件时判定：

1. 源字段与本地字段匹配命中
2. 所有参与比较字段均无变化

### 输出动作
- 不进入任何变更集合

---

## 8.5 `FIELD_SUSPECTED_RENAME` 判定规则

当前阶段不自动把 rename 直接当作“更新”，而是采用保守策略：

### 场景
当同时出现：

- 一个 `FIELD_DELETE`
- 一个 `FIELD_CREATE`

且满足以下条件中的大部分：

1. `Type` 相同
2. `Index` 接近（差值 <= 1）
3. `DisplayName` 相同或高度相似
4. 其他属性（长度、精度、nullable）相近

则可标记为：

- `FIELD_SUSPECTED_RENAME`

### 输出动作
- 仅记录为疑似 rename
- 不自动合并成 update
- 计入 `suspected_rename`

### 原因
当前阶段没有稳定 field_id，不能 100% 安全自动认定 rename。

---

## 9. 字段级比较字段

当前阶段字段级参与比较的字段如下：

| SourceField 字段 | LocalField 字段 | 是否参与 diff |
|---|---|---|
| `Name` | `technical_name` | 是 |
| `OriginalName` | `original_name` | 是 |
| `DisplayName` | `business_name` / `field_description` 初始化参考 | 是，但需注意当前阶段通常不强覆盖 |
| `Comment` | `comment` | 是 |
| `Type` | `data_type` | 是 |
| `DataLength` | `data_length` | 是 |
| `DataAccuracy` | `data_accuracy` | 是 |
| `OriginalDataType` | `original_data_type` | 是 |
| `Nullable` | `is_nullable` | 是 |
| `PrimaryKey` | `primary_key` | 是 |
| `Index` | `index` | 是 |

---

## 10. 当前阶段不参与字段 diff 的本地字段

以下字段默认不由源驱动，因此不参与字段级 diff：

- `field_role`
- `standard_code`
- `standard`
- `code_table_id`
- `business_timestamp`
- `reset_before_data_type`
- `reset_convert_rules`
- `reset_data_length`
- `reset_data_accuracy`
- `subject_id`
- `classify_type`
- `match_score`
- `grade_id`
- `grade_type`
- `shared_type`
- `open_type`
- `sensitive_type`
- `secret_type`

---

## 11. 严重度分级规则

视图与字段 diff 最终要统一映射为严重度，供：

- `apply_service`
- `sync_task` 统计
- 审核联动逻辑

使用。

---

## 11.1 `S1`：高影响结构变更

满足以下任一条件时归类为 `S1`：

### 视图级
- `DELETE_VIEW`
- `TechnicalNameChanged`
- `TypeChanged`
- `QueryTypeChanged`
- `MetaTableNameChanged`
- `DeletedChanged`

### 字段级
- 存在 `FIELD_CREATE`
- 存在 `FIELD_DELETE`
- 字段 `TypeChanged`
- 字段 `PrimaryKey` 变化
- 字段 `Nullable` 变化（可视业务调成 S2）

### 业务含义
- 结构发生变化
- 可能影响下游模型、SQL、审核状态

---

## 11.2 `S2`：中影响语义变更

满足以下条件时归类为 `S2`：

### 视图级
- `ViewNameChanged`
- `CommentChanged`
- `StatusChanged`

### 字段级
- `DisplayNameChanged`
- `OriginalNameChanged`
- `CommentChanged`
- `DataLengthChanged`
- `DataAccuracyChanged`
- `OriginalDataTypeChanged`

### 业务含义
- 语义发生变化
- 结构未发生根本性变化
- 可能需要重新理解 / 重新确认，但不一定影响所有下游

---

## 11.3 `S3`：低影响变化

满足以下条件时归类为 `S3`：

- 仅字段顺序变化 `IndexChanged`
- 仅补齐一些缺省源值
- 仅有不影响语义与结构的次要变更

### 业务含义
- 可记录
- 默认不触发强副作用

---

## 12. `DiffResult` 输出结构合同

建议统一输出以下结构：

```go
type DiffResult struct {
    News      []CreateViewDiff
    Updates   []UpdateViewDiff
    Deletes   []DeleteViewDiff
    Unchanged []string
    Stats     SyncStats
}
```

其中：

```go
type UpdateViewDiff struct {
    SourceView SourceView
    LocalView  LocalView
    ViewDelta  ViewMetaDelta
    FieldDelta FieldDelta
    Severity   string // S1 / S2 / S3
}
```

---

## 13. `SyncStats` 统计规则

diff 结束后必须产出统计，用于写入 `sync_task`。

建议最少统计：

- `ViewCreated`
- `ViewUpdated`
- `ViewDeleted`
- `FieldCreated`
- `FieldUpdated`
- `FieldDeleted`
- `HashSkipped`
- `SuspectedRename`
- `AuditRevoked`（当前阶段可先保留为 0，由 apply 决定）

---

## 14. 顺序与并发规则

这是本文件最关键的实现约束之一。

## 14.1 datasource 之间
- 可以并行执行

## 14.2 单 datasource 内
- page 必须顺序执行

## 14.3 单 page 内
- 单 view 的标准化 / hash / diff 可以并行

## 14.4 输出聚合
- 必须按原始顺序（`change_time ASC, view_id ASC`）聚合结果

### 原因
`last_safe_cursor` 只能基于有序结果来计算。

---

## 15. `last_safe_cursor` 规则

## 15.1 定义

`last_safe_cursor` 表示：

> 当前 datasource 在本次执行中，已经连续成功完成 apply 的最后一个断点。

其结构为：

```json
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

---

## 15.2 计算前提

源记录顺序固定为：

```text
(change_time ASC, view_id ASC)
```

其中：

```text
change_time = max(f_create_time, f_update_time, f_delete_time)
```

---

## 15.3 推进原则

### 原则
只能推进到：

- **连续成功前缀** 的最后一条

### 不能推进到
- 本次扫描到的最大 `change_time`
- 本次成功条数最多的位置
- 失败洞之后的记录

---

## 15.4 例子

### 输入顺序
| 顺序 | view_id | change_time | 结果 |
|---:|---|---:|---|
| 1 | v1 | 1001 | success |
| 2 | v2 | 1002 | success |
| 3 | v3 | 1003 | failed |
| 4 | v4 | 1004 | success |

### 结果
`last_safe_cursor` 只能是：

```json
{
  "change_time": 1002,
  "view_id": "v2"
}
```

而不是 `v4`。

---

## 16. 部分成功 / 部分失败处理规则

## 16.1 原则

如果一个 datasource 任务出现：

- 部分视图 apply 成功
- 部分视图 apply 失败

则：

1. `sync_task` 必须记录：
   - 成功数量
   - 失败数量
   - `last_safe_cursor`
   - 失败样本与失败阶段
2. `watermark` 只推进到 `last_safe_cursor`
3. 不允许越过失败洞推进

---

## 16.2 任务状态建议

- 若错误可重试：
  - `sync_task.status = retry_waiting`
- 若错误不可重试：
  - `sync_task.status = failed`

---

## 17. Hash 快速路径规则

## 17.1 视图级
若本地存在 `source_payload_hash` 且：

```text
LocalView.SourcePayloadHash == SourceView.PayloadHash
```

则可直接判定：

- `UNCHANGED_VIEW`

无需再做字段级深比较。

---

## 17.2 字段级
若后续本地表增加：

- `source_field_hash`

则字段级也可先走 hash 快速跳过。

### 当前阶段
由于本地还没有 `source_field_hash`，第一阶段字段级仍以逐字段比较为主。

---

## 18. 异常与边界处理规则

## 18.1 源对象标准化失败
例如：

- `f_fields` 非法 JSON
- 关键字段缺失

### 处理
- 当前视图 diff 失败
- 当前 page / datasource 任务进入失败流程
- 不应 silently skip

---

## 18.2 本地对象重复命中
如果同一 `SourceView` 命中多个本地 `form_view`：

### 处理
- 视为数据异常
- 当前视图失败
- 交给任务失败 / 重试逻辑

---

## 18.3 字段重复命中
如果同一 `FieldKey` 命中多个本地字段：

### 处理
- 视为数据异常
- 当前视图失败
- 交给任务失败 / 重试逻辑

---

## 19. 大模型代码生成约束

为了让大模型稳定生成 `diff_engine`，本文件要求生成代码时必须遵守：

1. **先视图级后字段级**
2. **先批量加载本地对象，再做内存 diff**
3. **不得逐条源记录打本地 SQL**
4. **page 内可以并发 diff，但聚合结果必须恢复原顺序**
5. **必须显式输出 `Severity`**
6. **必须显式输出 `last_safe_cursor` 所依赖的顺序信息**
7. **不得直接把 rename 当 update 自动合并**
8. **必须支持 hash 快速路径**
9. **必须把失败样本和失败阶段留给 `sync_task` 写入**

---

## 20. 测试案例要求

本文件要求至少准备以下测试集：

### Case 1：新视图
- 本地无视图
- 源有视图

### Case 2：删除视图
- 本地有视图
- 源 `delete_time > 0`

### Case 3：hash 无变化
- 本地 `source_payload_hash` 与源一致

### Case 4：视图元信息变化
- 注释变化
- meta_table_name 变化

### Case 5：字段新增
- 本地少字段

### Case 6：字段删除
- 本地多字段

### Case 7：字段更新
- 类型 / 长度 / 注释 / index 变化

### Case 8：疑似 rename
- 一删一增，类型相同、位置接近、display_name 相似

### Case 9：部分成功部分失败
- 验证 `last_safe_cursor`

### Case 10：重复命中异常
- 同一 `mdl_id` 命中多个本地视图
- 同一字段 key 命中多个本地字段

---

## 21. 最终决策摘要

### 视图级分类
- `NEW_VIEW`
- `UPDATE_VIEW`
- `DELETE_VIEW`
- `UNCHANGED_VIEW`

### 字段级分类
- `FIELD_CREATE`
- `FIELD_UPDATE`
- `FIELD_DELETE`
- `FIELD_SUSPECTED_RENAME`
- `FIELD_UNCHANGED`

### 严重度
- `S1` 高影响结构变更
- `S2` 中影响语义变更
- `S3` 低影响变化

### watermark 相关
- 顺序固定：`(change_time ASC, view_id ASC)`
- `last_safe_cursor` 只推进到连续成功前缀

### 并发规则
- datasource 间并行
- 单 datasource page 串行
- 单 page diff 可并行

---

## 22. 下一份文档建议

本文件之后，建议继续补：

- `06_APPLY_RULES_SPEC.md`

因为当前已经明确：

- 字段映射
- 标准化规则
- diff 规则

下一步就应该把：

- create / update / delete 的落库逻辑
- 事务边界
- 审核联动
- 本地保留字段策略

彻底定死。
