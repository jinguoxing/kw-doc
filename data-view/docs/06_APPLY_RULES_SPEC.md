# 06_APPLY_RULES_SPEC

## 1. 文档目的

本文档定义 `sync-worker` 在拿到 `DiffResult` 之后，如何把变更真正落到本地：

- `form_view`
- `form_view_field`

本文档直接约束：

- `apply_service`
- `runner`
- `repository`
- 事务边界
- 状态联动
- 审核联动的最小实现边界

本文档关注：

1. 视图级 create / update / delete 的落库规则
2. 字段级 create / update / delete 的落库规则
3. 强同步字段、条件驱动字段、本地保留字段的 apply 原则
4. 小批事务策略
5. 失败回滚与 `last_safe_cursor` 的关系
6. 当前阶段审核/状态联动的最小策略

---

## 2. 依赖关系

本文件依赖以下前置文档：

1. `03_FIELD_MAPPING_CONTRACT.md`
2. `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`
3. `05_DIFF_RULES_SPEC.md`

其中：

- `03` 决定字段映射
- `04` 决定标准化
- `05` 决定 diff 输出
- `06` 决定 diff 输出如何落库

---

## 3. 总体 apply 原则

## 3.1 总原则

> **先 diff，后 apply；先视图，后字段；先确保正确，再追求吞吐。**

### 具体含义
1. 任何写本地的动作都必须基于 `DiffResult`
2. 不允许“边比较边写库”
3. 同一个 datasource 内，page 必须顺序 apply
4. page 内可以拆小批事务，但不能破坏顺序语义
5. `watermark` 只能推进到已经成功 apply 的最后安全位置

---

## 3.2 apply 结果必须可追踪

每次 apply 完成后，必须能回写到 `sync_task`：

- 视图新增/更新/删除数量
- 字段新增/更新/删除数量
- `last_safe_cursor`
- 失败阶段
- 失败原因样本

---

## 3.3 apply 不负责做什么

当前阶段，`apply_service` **不负责**：

- 调度下一轮任务
- 自动创建 retry 任务
- 复杂审批流编排
- 外部事件投递
- ES / 缓存 / 搜索索引投影

这些能力留给后续增强，不在第一阶段主链路内。

---

## 4. Diff 到 Apply 的分类映射

## 4.1 视图级

| Diff 类型 | Apply 动作 |
|---|---|
| `NEW_VIEW` | 创建 `form_view`，并创建全部 `form_view_field` |
| `UPDATE_VIEW` | 更新 `form_view`，并按字段 delta 更新 `form_view_field` |
| `DELETE_VIEW` | 逻辑删除 / 下线 `form_view`，并处理关联字段 |
| `UNCHANGED_VIEW` | 不写库 |

---

## 4.2 字段级

| Diff 类型 | Apply 动作 |
|---|---|
| `FIELD_CREATE` | 创建本地 `form_view_field` |
| `FIELD_UPDATE` | 更新本地 `form_view_field` |
| `FIELD_DELETE` | 逻辑删除本地 `form_view_field` |
| `FIELD_SUSPECTED_RENAME` | 仅记录统计和样本，不单独写库 |
| `FIELD_UNCHANGED` | 不写库 |

---

## 5. 视图级 apply 规则

## 5.1 `NEW_VIEW` 规则

### 触发条件
- `DiffResult.News` 中存在该视图

### 落库动作
1. 新建一条 `form_view`
2. 写入源驱动字段
3. 写入条件驱动字段的默认初始化值
4. 保持本地业务保留字段为空或默认值
5. 新建对应全部 `form_view_field`

### 必写字段（建议）
- `form_view_id`
- `id`
- `mdl_id`
- `technical_name`
- `original_name`
- `datasource_id`
- `type`
- `comment`
- `status = 1`
- `deleted_at = 0`
- `created_at`
- `created_by_uid`
- `updated_at`
- `updated_by_uid`

### 默认初始化规则
- `business_name`：若源有 `view_name`，可初始化为源视图名称
- `edit_status`：建议初始化为草稿/新增态
- `understand_status`：建议初始化为未理解（0）
- `audit_type / audit_status / online_status`：使用本地默认值

---

## 5.2 `UPDATE_VIEW` 规则

### 触发条件
- `DiffResult.Updates` 中存在该视图

### 落库动作
1. 更新 `form_view` 的强同步字段
2. 按策略更新条件驱动字段
3. 保留本地业务保留字段
4. 处理字段级 create / update / delete
5. 根据 diff 严重度设置本地 `status`

---

## 5.3 `DELETE_VIEW` 规则

### 触发条件
- `DiffResult.Deletes` 中存在该视图

### 当前阶段策略
**逻辑删除，不物理删除。**

### 落库动作
1. `form_view.deleted_at = source.delete_time`
2. `form_view.status = 2`
3. `updated_at / updated_by_uid` 更新
4. 关联字段统一逻辑删除：
   - `form_view_field.deleted_at = source.delete_time or now`
   - `form_view_field.status = 2`

### 不做的事情
当前阶段默认**不物理删除**：
- `form_view`
- `form_view_field`

---

## 6. 字段级 apply 规则

## 6.1 `FIELD_CREATE` 规则

### 触发条件
- `UpdateViewDiff.FieldDelta.Created`

### 落库动作
1. 新建 `form_view_field`
2. 关联到当前 `form_view.id`
3. 写入强同步字段
4. 条件驱动字段做默认初始化
5. 保持业务保留字段为空或默认值

### 必写字段（建议）
- `form_view_field_id`
- `id`
- `form_view_id`
- `technical_name`
- `original_name`
- `comment`
- `status = 1`
- `primary_key`
- `data_type`
- `data_length`
- `data_accuracy`
- `original_data_type`
- `is_nullable`
- `deleted_at = 0`
- `index`

---

## 6.2 `FIELD_UPDATE` 规则

### 触发条件
- `UpdateViewDiff.FieldDelta.Updated`

### 落库动作
更新以下强同步字段：

- `technical_name`
- `original_name`
- `comment`
- `primary_key`
- `data_type`
- `data_length`
- `data_accuracy`
- `original_data_type`
- `is_nullable`
- `index`

### 状态字段规则
当前阶段：

- `form_view_field.status` **不单独表示“更新”**
- 更新后默认保持 `0`

### 原因
本地表当前枚举定义仅有：
- `0：无变化`
- `1：新增`
- `2：删除`

因此更新通过：
- `sync_task` 统计
- `diff_result`
- `audit/理解联动`
来体现，而不是通过字段表 `status` 扩展表达。

---

## 6.3 `FIELD_DELETE` 规则

### 触发条件
- `UpdateViewDiff.FieldDelta.Deleted`

### 落库动作
1. `deleted_at = now or source_delete_time`
2. `status = 2`
3. 更新 `updated_at / updated_by_uid`（若有）

### 当前阶段策略
**逻辑删除，不物理删除。**

---

## 6.4 `FIELD_SUSPECTED_RENAME` 规则

### 当前阶段策略
只记录：

- `SuspectedRename` 统计
- 可选写入 `f_result_json` 的样本

### 不做的动作
当前阶段默认**不自动把 rename 合并成 update**。

### 原因
当前本地没有稳定 `source_field_key`，自动 rename 风险较高。

---

## 7. 强同步字段、条件驱动字段、本地保留字段的 apply 原则

这是本文件最关键的边界之一。

---

## 7.1 强同步字段

### 定义
源变化后，必须覆盖本地的字段。

### `form_view` 强同步字段
- `mdl_id`
- `technical_name`
- `original_name`
- `datasource_id`
- `comment`
- `deleted_at`
- `status`（由 diff 结果决定）

### `form_view_field` 强同步字段
- `technical_name`
- `original_name`
- `comment`
- `primary_key`
- `data_type`
- `data_length`
- `data_accuracy`
- `original_data_type`
- `is_nullable`
- `index`
- `deleted_at`
- `status`（新增/删除）

---

## 7.2 条件驱动字段

### 定义
这些字段可以在创建时初始化，但更新时不一定强覆盖。

### `form_view`
- `business_name`
- `description`
- `excel_file_name`

### `form_view_field`
- `business_name`
- `field_description`

### 当前阶段策略
#### 创建时
- 若本地为空，可按源做初始化

#### 更新时
- 默认 **非空保留**
- 仅在明确业务要求“源覆盖”时才更新

---

## 7.3 本地业务保留字段

### 定义
这些字段由本地理解、治理、审核、发布、分类体系维护，源同步不得强覆盖。

### `form_view`
- `uniform_catalog_code`
- `publish_at`
- `edit_status`
- `owner_id`
- `subject_id`
- `department_id`
- `info_system_id`
- `scene_analysis_id`
- 审核流相关全部字段
- 上下线状态相关字段
- 理解状态字段
- 探查任务相关字段
- `shared_type / open_type`
- `update_cycle`

### `form_view_field`
- `field_role`
- `standard_code`
- `standard`
- `code_table_id`
- `business_timestamp`
- reset 系列字段
- `subject_id`
- `classify_type / match_score`
- `grade_id / grade_type`
- `shared_type / open_type / sensitive_type / secret_type`

### apply 原则
- **禁止源强覆盖**
- 即使 diff 命中，也默认保留本地已有值

---

## 8. 事务边界规则

## 8.1 总体策略

### 不推荐
- 整个 page 一个超大事务

### 推荐
- **小批事务**
- 每批处理固定数量的视图（例如 20 ~ 50）

---

## 8.2 单批事务范围

一个 mini-batch 中，单个视图的以下动作必须在同一事务内：

### `NEW_VIEW`
- 创建 `form_view`
- 创建全部该视图对应 `form_view_field`

### `UPDATE_VIEW`
- 更新 `form_view`
- 创建字段
- 更新字段
- 删除字段

### `DELETE_VIEW`
- 更新 `form_view.deleted_at / status`
- 批量逻辑删除关联 `form_view_field`

---

## 8.3 为什么要这么设计

### 原因 1：保证单视图的一致性
不能出现：
- view 更新成功
- fields 一半失败

### 原因 2：降低事务体积
一个 page 可能 300 条 view，如果全 page 一个事务，风险过高。

### 原因 3：便于 `last_safe_cursor`
apply 失败时，可以清晰定位到上一个成功视图。

---

## 9. `last_safe_cursor` 与 apply 的关系

## 9.1 基本原则

`last_safe_cursor` 只能在**事务成功提交后**推进。

### 意味着
- diff 完成 ≠ 安全推进
- 只有成功写入本地之后，才能把该视图视为“已提交”

---

## 9.2 单批失败时的规则

假设本批次中按顺序处理视图：

- v1 success
- v2 success
- v3 failed
- v4 未开始

则：

- `last_safe_cursor = v2`
- 不允许推进到 v3 / v4

---

## 9.3 任务状态更新规则

apply 失败时：

- 若错误可重试：
  - `sync_task.status = retry_waiting`
- 若错误不可重试：
  - `sync_task.status = failed`

同时必须把：
- `f_watermark_after`
- `f_cursor_json`
写成 `last_safe_cursor`

---

## 10. 审核与状态联动（当前阶段最小策略）

这一块最容易过度设计，所以当前阶段只定**最小可执行策略**。

---

## 10.1 `form_view.status` 联动规则

| diff 结果 | `form_view.status` |
|---|---|
| `NEW_VIEW` | `1` |
| `DELETE_VIEW` | `2` |
| `UPDATE_VIEW` | `3` |
| `UNCHANGED_VIEW` | `0` |

---

## 10.2 审核联动最小策略

当前阶段建议：

### 对 `S1`
- 记录为高影响变更
- 可选：把 `edit_status` 打回需要确认态（需要团队进一步确认最终枚举）
- 当前阶段**不直接自动操作复杂审核流**

### 对 `S2`
- 记录为变更
- 默认不自动重置审核流

### 对 `S3`
- 仅记录统计和审计

---

## 10.3 为什么当前不直接深度接审核流

因为本地 `form_view` 中审核相关字段很多：

- `flow_id`
- `flow_name`
- `flow_node_id`
- `flow_node_name`
- `audit_type`
- `audit_status`
- `apply_id`
- `proc_def_key`
- `audit_advice`

这些字段说明审核流是一个独立业务域。  
当前阶段同步主链路的目标是：

- 先把数据同步正确
- 先把结构变化识别正确

而不是在第一版就把审核流程自动编排全部打通。

---

## 11. 删除规则补充

## 11.1 视图删除
当前阶段统一采用：

- `form_view.deleted_at > 0`
- `form_view.status = 2`

### 关联字段
同一视图下字段统一：
- `deleted_at > 0`
- `status = 2`

---

## 11.2 字段删除
当前阶段统一采用：

- `form_view_field.deleted_at > 0`
- `form_view_field.status = 2`

---

## 11.3 物理删除策略
当前阶段：

- **禁止物理删除**

原因：
1. 便于追溯与恢复
2. 便于和审核/理解/治理链路兼容
3. 便于后续 reconcile / full 恢复

---

## 12. apply 顺序规则

在一个 page 内，建议统一采用：

1. `NEW_VIEW`
2. `UPDATE_VIEW`
3. `DELETE_VIEW`

### 原因
- 创建新对象逻辑最独立
- 更新对象依赖已有本地数据
- 删除对象放最后，避免影响前面的兜底匹配

---

## 13. 大模型代码生成约束

为了让大模型稳定生成 `apply_service`，本文件要求生成代码时必须遵守：

1. **不得边 diff 边写库**
2. **必须按 `DiffResult` 做分类落库**
3. **单视图的 view + field 变更必须在同一事务内**
4. **不得物理删除**
5. **不得强覆盖本地业务保留字段**
6. **必须区分：强同步字段、条件驱动字段、本地保留字段**
7. **必须把 `last_safe_cursor` 与事务提交绑定**
8. **必须允许 mini-batch apply**
9. **必须为失败输出可重试 / 不可重试信息**
10. **疑似 rename 默认不自动改写本地字段**

---

## 14. 推荐的 `ApplyResult` 结构

建议统一输出：

```go
type ApplyResult struct {
    Stats          SyncStats
    LastSafeCursor Cursor
}
```

其中：

- `Stats`：写回 `sync_task`
- `LastSafeCursor`：用于任务进度和 watermark 推进

---

## 15. 推荐的 `f_result_json` 结构

apply 结束后，建议写入 `sync_task.f_result_json`：

```json
{
  "success_count": 90,
  "failed_count": 10,
  "failed_view_ids_sample": ["dv_101", "dv_102", "dv_103"],
  "last_safe_watermark": 1710001234567,
  "last_safe_cursor": {
    "change_time": 1710001234567,
    "view_id": "dv_000123"
  },
  "fail_stage": "apply",
  "fail_reason": "batch update form_view_field failed",
  "retryable": true
}
```

---

## 16. 测试案例要求

apply 层至少需要准备以下测试集：

### Case 1：新增视图
- 新建 `form_view`
- 新建全部 `form_view_field`

### Case 2：更新视图
- 更新 `form_view`
- 同时字段 create/update/delete

### Case 3：删除视图
- 逻辑删除 `form_view`
- 逻辑删除全部关联字段

### Case 4：字段新增
- 只新增字段，不影响已有字段

### Case 5：字段更新
- 类型 / 长度 / 注释 / index 更新

### Case 6：字段删除
- 逻辑删除单个字段

### Case 7：业务保留字段不被覆盖
- 本地 `business_name` / `field_role` / `standard_code` 保持不变

### Case 8：部分失败
- 第 3 个视图事务失败
- 验证 `last_safe_cursor` 正确停在第 2 个视图

### Case 9：删除后重建
- 源侧先删除，再重新出现
- 验证本地逻辑删除对象的恢复策略（当前阶段如无恢复策略，则按新建/更新规则处理）

---

## 17. 当前阶段的 TODO

以下问题建议在后续版本中继续细化：

1. `business_name` / `field_description` 是否允许 source 强覆盖
2. `S1` 变更是否需要统一重置 `edit_status`
3. 删除后重建时，是否复用原逻辑删除对象
4. 是否需要记录更详细的 apply 审计日志
5. 是否需要拆出 outbox / projector 机制

---

## 18. 最终决策摘要

### 当前阶段 apply 核心原则

1. `NEW_VIEW`：创建 view + 全字段
2. `UPDATE_VIEW`：更新 view + 按字段 delta 落库
3. `DELETE_VIEW`：逻辑删除 view + 字段
4. `FIELD_CREATE / UPDATE / DELETE`：分别做创建 / 覆盖更新 / 逻辑删除
5. `FIELD_SUSPECTED_RENAME`：只记统计，不自动改写
6. 强同步字段必须覆盖
7. 条件驱动字段创建时可初始化、更新时默认保留
8. 本地业务保留字段禁止源强覆盖
9. 小批事务，单视图变更必须在同一事务内
10. `last_safe_cursor` 只能推进到已成功提交的位置

---

## 19. 下一份文档建议

本文件之后，建议继续补：

- `07_PROJECT_STRUCTURE_AND_LAYERING.md`

因为到这一步为止，业务逻辑合同已经基本齐了：

- 字段映射
- 字段标准化
- diff 规则
- apply 规则

接下来就应该把这些规则落到：

- 工程目录
- 包职责
- 层间依赖
- 仓储接口
- 事务放置位置

上，避免大模型把代码写散。
