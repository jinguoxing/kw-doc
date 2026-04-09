# 03_FIELD_MAPPING_CONTRACT

## 1. 文档目的

本文档定义 **源表 `t_data_view` / `t_data_view.f_fields`** 与 **本地表 `form_view` / `form_view_field`** 的字段映射合同，作为 `sync-worker` 中以下模块的统一输入规范：

- `normalizer`
- `diff_engine`
- `apply_service`
- `runner`

本文档关注：

1. 源对象到目标对象的字段映射
2. 字段比较主键与兜底规则
3. 创建、更新、删除时的落库规则
4. 哪些字段由源驱动，哪些字段由本地业务维护
5. 第一阶段同步范围与边界

---

## 2. 范围与阶段说明

### 2.1 当前阶段范围

当前同步范围限定为：

- **源表**：`t_data_view`
- **本地目标表**：`form_view`
- **本地目标字段表**：`form_view_field`

### 2.2 当前阶段同步对象

当前阶段主流程只同步：

- **元数据视图（Metadata View）**

判定规则：

- 当 `t_data_view.f_group_id = t_data_view.f_data_source_id` 时，视为元数据视图

### 2.3 暂不纳入主链路的对象

以下对象暂不纳入自动同步主链路：

- 自定义视图
- 逻辑实体视图
- 非标准 `f_fields` 结构的特殊对象
- 需要复杂 SQL / 数据范围解析的高级视图

> 说明：后续若需要纳入，将在本合同基础上扩展，不直接修改第一阶段规则。

---

## 3. 源与目标的对象关系

## 3.1 视图级对象关系

源对象：

- `t_data_view` 中一行记录

目标对象：

- `form_view` 中一行记录

### 关系定义

```text
t_data_view (1 row) -> form_view (1 row)
```

---

## 3.2 字段级对象关系

源对象：

- `t_data_view.f_fields` 解析后的字段数组

目标对象：

- `form_view_field` 中多行记录

### 关系定义

```text
t_data_view.f_fields (N items) -> form_view_field (N rows under one form_view)
```

---

## 4. 核心映射主键规则

## 4.1 视图级匹配主键

### 第一优先级
- `form_view.mdl_id = t_data_view.f_view_id`

### 第二优先级（历史兼容兜底）
- `form_view.datasource_id = t_data_view.f_data_source_id`
- `form_view.technical_name = t_data_view.f_technical_name`

### 规则说明
- `mdl_id` 作为源视图主键映射字段
- 如果历史数据里 `mdl_id` 未补齐，可短期用 `(datasource_id, technical_name)` 兜底
- 一旦命中兜底，应在本次同步中补齐 `mdl_id`

---

## 4.2 字段级匹配主键

当前 `form_view_field` 表中没有显式 `source_field_key` 字段，因此第一阶段字段比较采用：

### 第一优先级
- 同一 `form_view_id` 下，按 `technical_name` 匹配

### 第二优先级
- 同一 `form_view_id` 下，按 `original_name` 匹配

### 补充规则
- 比较前统一做标准化：
  - trim 空格
  - lowercase
  - 去掉无意义引号（反引号、双引号）
- 若 `technical_name` 和 `original_name` 都无法稳定匹配，则标记为：
  - `FIELD_SUSPECTED_RENAME`

### 后续增强建议
后续建议给 `form_view_field` 补充字段：

- `source_field_key`
- `source_field_hash`

以提升字段级 diff 的准确性与性能。

---

## 5. 源字段标准化合同

由于 `t_data_view.f_fields` 为长文本 JSON，落库前必须先标准化为统一的 `SourceField` 结构。

## 5.1 标准化后的 SourceField 逻辑字段

| 标准字段 | 说明 | 获取规则 |
|---|---|---|
| `field_key` | 字段比较主键 | 优先 `original_name`，否则 `name`，再做标准化 |
| `name` | 技术名称 | 从 `f_fields` 中 `name` 取值 |
| `original_name` | 原始字段名 | 从 `f_fields` 中 `original_name` 取值；缺失时可回退 `name` |
| `display_name` | 展示名称/业务名称候选 | 优先 `display_name`，否则 `label`，否则 `name` |
| `type` | 规范化后的数据类型 | 优先 `type`，否则 `data_type`，统一转小写 |
| `index` | 字段顺序 | 以 `f_fields` 数组顺序为准 |
| `nullable` | 是否可空 | 从 `nullable` 解析；不存在则记为 `nil` |
| `comment` | 字段注释 | 若 JSON 中有 `comment`/`description` 则读取，否则为空 |
| `data_length` | 数据长度 | 若 JSON 中有则解析，否则默认 `0` |
| `data_accuracy` | 数据精度 | 若 JSON 中有则解析，否则默认 `0` |
| `original_data_type` | 原始数据类型 | 若 JSON 中有则读取，否则回退 `type` |
| `primary_key` | 是否主键 | 若 JSON 中有则解析，否则为空 |
| `hash` | 字段级 hash | 对标准化字段做 canonical json 后 hash |

---

## 5.2 视图级逻辑字段

标准化后的 `SourceView` 必须包含：

| 标准字段 | 说明 | 来源 |
|---|---|---|
| `view_id` | 源视图主键 | `f_view_id` |
| `datasource_id` | 源数据源ID | `f_data_source_id` |
| `group_id` | 分组ID | `f_group_id` |
| `view_name` | 源视图名称 | `f_view_name` |
| `technical_name` | 技术名称 | `f_technical_name` |
| `type` | 源视图类型 | `f_type` |
| `query_type` | 查询类型 | `f_query_type` |
| `comment` | 备注 | `f_comment` |
| `status` | 源状态 | `f_status` |
| `meta_table_name` | 元数据表名 | `f_meta_table_name` |
| `create_time` | 创建时间 | `f_create_time` |
| `update_time` | 更新时间 | `f_update_time` |
| `delete_time` | 删除时间 | `f_delete_time` |
| `change_time` | 统一变更时间 | `max(create_time, update_time, delete_time)` |
| `fields` | 标准化字段数组 | 解析 `f_fields` |
| `payload_hash` | 视图级 hash | 对视图关键字段 + 标准化字段数组做 hash |

---

## 6. `t_data_view` -> `form_view` 字段映射合同

## 6.1 本地自动生成字段

下列字段由本地系统生成，不由源直接映射：

| `form_view` 字段 | 策略 |
|---|---|
| `form_view_id` | 本地雪花ID生成 |
| `id` | 本地 UUID 生成 |
| `created_at` | 本地写入时间 |
| `created_by_uid` | 系统账号或触发账号 |
| `updated_at` | 本地更新时间 |
| `updated_by_uid` | 系统账号或触发账号 |
| `apply_id` | 由本地审核体系生成/维护 |

---

## 6.2 源驱动字段（强同步）

以下字段由源表驱动，源变化后应同步到本地：

| `form_view` 字段 | 源字段/规则 | 创建时策略 | 更新时策略 | 说明 |
|---|---|---|---|---|
| `mdl_id` | `t_data_view.f_view_id` | 必填写入 | 必须保持同步 | 视图级主映射键 |
| `technical_name` | `f_technical_name` | 写入 | 覆盖更新 | 参与兜底匹配 |
| `original_name` | 优先 `f_meta_table_name`，否则 `f_view_name` | 写入 | 覆盖更新 | 原始表名称候选 |
| `datasource_id` | `f_data_source_id` | 写入 | 覆盖更新 | 数据源归属 |
| `comment` | `f_comment` | 写入 | 覆盖更新 | 逻辑视图注释 |
| `status` | 由 diff 计算 | 新建时 `1` | 无变化=0，删除=2，变更=3 | 本地扫描结果状态，不直接等于源状态 |
| `deleted_at` | `f_delete_time` | 写入 | 覆盖更新 | 逻辑删除时间 |

---

## 6.3 条件驱动字段

以下字段需要基于同步阶段或策略决定如何映射：

| `form_view` 字段 | 规则 | 创建时策略 | 更新时策略 | 说明 |
|---|---|---|---|---|
| `business_name` | 优先本地保留；若本地为空，可初始化为 `f_view_name` | 若空则初始化 | 已有值默认保留 | 业务名称通常允许人工维护 |
| `type` | 当前阶段仅同步元数据视图，固定映射为 `1` | 写 `1` | 保持 `1` | 本地枚举：1=元数据视图 |
| `description` | 默认不强制映射 | 可为空 | 默认保留本地 | 后续若明确来源再扩展 |
| `excel_file_name` | `f_file_name`（仅 Excel 类型视图） | 条件写入 | 条件覆盖更新 | 需结合视图来源类型判断 |

---

## 6.4 本地业务保留字段（非源驱动）

以下字段原则上不由同步任务强覆盖，若本地已有值应保留：

| `form_view` 字段 | 原则 |
|---|---|
| `uniform_catalog_code` | 保持本地 |
| `publish_at` | 保持本地 |
| `edit_status` | 保持本地业务状态 |
| `owner_id` | 保持本地 |
| `subject_id` | 保持本地 |
| `department_id` | 保持本地 |
| `info_system_id` | 原则上保持本地；若后续有 datasource -> info_system 映射服务，可单独补充 |
| `scene_analysis_id` | 保持本地 |
| `flow_id` / `flow_name` / `flow_node_id` / `flow_node_name` | 保持本地 |
| `online_status` | 保持本地 |
| `audit_type` | 保持本地 |
| `audit_status` | 保持本地 |
| `proc_def_key` | 保持本地 |
| `audit_advice` | 保持本地 |
| `excel_sheet` / `start_cell` / `end_cell` / `has_headers` / `sheet_as_new_column` | 保持本地，除非后续明确 `f_excel_config` 结构 |
| `source_sign` | 保持本地 |
| `update_cycle` | 保持本地 |
| `shared_type` | 保持本地 |
| `open_type` | 保持本地 |
| `understand_status` | 保持本地 |
| `explore_job_id` / `explore_job_version` / `explore_timestamp_id` / `explore_timestamp_version` | 保持本地 |

---

## 6.5 `form_view.status` 的映射规则

注意：`form_view.status` 不是直接映射 `t_data_view.f_status`，而是由 **diff 结果** 计算：

| diff 结果 | `form_view.status` |
|---|---|
| 本地不存在，源存在 | `1`（新增） |
| 本地存在，源已删除 | `2`（删除） |
| 本地存在，源字段/视图有变化 | `3`（变更） |
| 本地存在，源无变化 | `0`（无变化） |

---

## 7. `SourceField` -> `form_view_field` 字段映射合同

## 7.1 本地自动生成字段

| `form_view_field` 字段 | 策略 |
|---|---|
| `form_view_field_id` | 本地雪花ID生成 |
| `id` | 本地 UUID 生成 |

---

## 7.2 关联字段

| `form_view_field` 字段 | 来源 | 规则 |
|---|---|---|
| `form_view_id` | 本地 `form_view.id` | 通过视图映射得到 |
| `technical_name` | `SourceField.name` | 必填 |
| `original_name` | `SourceField.original_name`，缺失时回退 `name` | 必填候选 |
| `index` | `SourceField.index` | 必填 |

---

## 7.3 源驱动字段（强同步）

| `form_view_field` 字段 | 来源/规则 | 创建时策略 | 更新时策略 | 说明 |
|---|---|---|---|---|
| `technical_name` | `SourceField.name` | 写入 | 覆盖更新 | 字段技术名 |
| `original_name` | `SourceField.original_name` 或 `name` | 写入 | 覆盖更新 | 原始字段名 |
| `comment` | `SourceField.comment` | 写入 | 覆盖更新 | 若源里存在 |
| `status` | 由字段 diff 计算 | 新建=1 | 删除=2，无变化=0 | 本地扫描状态 |
| `primary_key` | `SourceField.primary_key` | 条件写入 | 覆盖更新 | 若源有则写入 |
| `data_type` | `SourceField.type` | 写入 | 覆盖更新 | 规范化后的数据类型 |
| `data_length` | `SourceField.data_length` | 写入 | 覆盖更新 | 缺失时默认 `0` |
| `data_accuracy` | `SourceField.data_accuracy` | 写入 | 覆盖更新 | 缺失时默认 `0` |
| `original_data_type` | `SourceField.original_data_type`，缺失回退 `type` | 写入 | 覆盖更新 | 原始数据类型 |
| `is_nullable` | `SourceField.nullable` | 写入 | 覆盖更新 | 建议统一成字符串：`YES/NO/UNKNOWN` |
| `deleted_at` | 视图删除或字段删除时写入当前删除时间 | 条件写入 | 覆盖更新 | 字段逻辑删除时间 |

---

## 7.4 条件驱动字段

| `form_view_field` 字段 | 规则 | 创建时策略 | 更新时策略 | 说明 |
|---|---|---|---|---|
| `business_name` | 优先保留本地；若本地为空，可初始化为 `display_name` 或 `name` | 若空则初始化 | 默认保留已有值 | 业务名称通常允许人工维护 |
| `field_description` | 若 `SourceField.comment` 为空，可回退 `display_name` | 可初始化 | 默认保留或条件覆盖 | 需团队最终确认 |
| `business_timestamp` | 不由源直接驱动 | 默认空 | 保持本地 | 由探查/规则引擎维护 |
| `reset_before_data_type` / `reset_convert_rules` / `reset_data_length` / `reset_data_accuracy` | 保持本地 | 保持本地 | 保持本地 | 重置类字段不由源同步 |
| `subject_id` | 保持本地 | 保持本地 | 保持本地 | 逻辑实体属性关联 |
| `classify_type` / `match_score` | 保持本地 | 保持本地 | 保持本地 | 理解/归类体系维护 |
| `grade_id` / `grade_type` | 保持本地 | 保持本地 | 保持本地 | 分级体系维护 |

---

## 7.5 本地业务保留字段（非源驱动）

以下字段默认不由源强覆盖：

| `form_view_field` 字段 | 原则 |
|---|---|
| `field_role` | 保持本地 |
| `standard_code` | 保持本地 |
| `standard` | 保持本地 |
| `code_table_id` | 保持本地 |
| `shared_type` | 保持本地 |
| `open_type` | 保持本地 |
| `sensitive_type` | 保持本地 |
| `secret_type` | 保持本地 |

---

## 7.6 `form_view_field.status` 的映射规则

同样，这个字段由 **字段 diff 结果** 计算，而不是直接来自源：

| diff 结果 | `form_view_field.status` |
|---|---|
| 本地不存在，源存在 | `1`（新增） |
| 本地存在，源不存在 | `2`（删除） |
| 本地存在，字段有变化 | `0`（保持无变化） |

### 当前约定
由于本地 DDL 中 `form_view_field.status` 注释定义为：

- `0：无变化`
- `1：新增`
- `2：删除`

当前阶段建议：

- 字段发生更新时，仍记为 `0`
- 具体更新明细通过 `diff_engine` 和 `sync_task` 统计体现
- 如后续需要单独表达字段“变更”，建议扩展枚举，而不是在本合同中强行修改现有语义

---

## 8. 创建、更新、删除的落库规则

## 8.1 创建 `form_view`

### 条件
- 本地按 `mdl_id` 未找到
- 且兜底 `(datasource_id, technical_name)` 也未找到

### 行为
1. 生成本地 `form_view_id`
2. 生成本地 `id`
3. 写入强同步字段
4. 写入条件驱动字段的默认初始化值
5. 保持业务保留字段默认值或空值

---

## 8.2 更新 `form_view`

### 条件
- 已按主键或兜底规则命中本地记录

### 行为
1. 覆盖更新强同步字段
2. 对条件驱动字段执行“空值初始化 / 非空保留”策略
3. 不覆盖本地业务保留字段
4. 根据 diff 结果更新 `status`

---

## 8.3 删除 `form_view`

### 条件
满足任一条件：
- `t_data_view.f_delete_time > 0`
- 源状态明确表示删除/下线

### 行为
1. 本地 `deleted_at = f_delete_time`
2. 本地 `status = 2`
3. 不物理删除记录
4. 是否触发审核撤销，由 `APPLY_RULES_SPEC` 另行定义

---

## 8.4 创建 `form_view_field`

### 条件
- 同一 `form_view_id` 下按字段匹配规则未命中本地字段

### 行为
1. 生成本地 `form_view_field_id`
2. 生成本地 `id`
3. 写入强同步字段
4. 写入条件驱动字段的默认初始化值
5. 保持业务保留字段默认值或空值

---

## 8.5 更新 `form_view_field`

### 条件
- 已按字段匹配规则命中本地字段

### 行为
1. 覆盖更新强同步字段
2. 对条件驱动字段执行“空值初始化 / 非空保留”策略
3. 保留业务保留字段
4. `status` 默认保持 `0`

---

## 8.6 删除 `form_view_field`

### 条件
- 同一 `form_view_id` 下，本地字段存在，源字段不存在

### 行为
1. 本地 `deleted_at = 当前删除时间`
2. 本地 `status = 2`
3. 不物理删除记录

---

## 9. Diff 比较边界

## 9.1 视图级比较字段

下列字段参与视图级 diff：

- `mdl_id`
- `technical_name`
- `original_name`
- `datasource_id`
- `comment`
- `type`
- `query_type`
- `status`（源删除态）
- `meta_table_name`
- `change_time`
- `payload_hash`

---

## 9.2 字段级比较字段

下列字段参与字段级 diff：

- `technical_name`
- `original_name`
- `comment`
- `data_type`
- `data_length`
- `data_accuracy`
- `original_data_type`
- `is_nullable`
- `index`
- `primary_key`

---

## 9.3 不参与 diff 的本地字段

下列字段原则上不参与源驱动比较：

### `form_view`
- `uniform_catalog_code`
- `owner_id`
- `subject_id`
- `department_id`
- `scene_analysis_id`
- 审核流程相关全部字段
- 上下线状态相关字段
- 理解状态相关字段

### `form_view_field`
- `field_role`
- `standard_code`
- `standard`
- `code_table_id`
- `business_timestamp`
- reset 系列字段
- 逻辑实体/分类/分级/共享/开放/敏感/涉密相关字段

---

## 10. 当前阶段的约束与 TODO

## 10.1 当前必须遵守的约束

1. 视图级主键以 `mdl_id = f_view_id` 为准
2. 字段级比较以 `(technical_name -> original_name)` 为当前主规则
3. 当前只同步元数据视图
4. 当前不要求把源 JSON 中所有字段 100% 映射到本地表，只要求满足主链路 diff + apply

---

## 10.2 推荐后续增强

### 推荐增强 1：给 `form_view` 增加影子字段
建议补：
- `source_change_time`
- `source_payload_hash`
- `source_deleted_at`

### 推荐增强 2：给 `form_view_field` 增加影子字段
建议补：
- `source_view_id`
- `source_field_key`
- `source_field_hash`

### 推荐增强 3：补充真实 `f_fields` 样本文件
用于完善：
- type 归一化规则
- nullable 解析规则
- 主键判定规则
- 复杂字段 rename 判定

---

## 11. 本合同对代码生成的约束

本合同将直接约束以下代码生成结果：

### `normalizer`
必须输出：
- `SourceView`
- `SourceField`
- `change_time`
- `payload_hash`

### `diff_engine`
必须使用：
- `mdl_id` 作为视图一级匹配键
- `(technical_name, original_name)` 作为当前字段匹配规则

### `apply_service`
必须区分：
- 强同步字段
- 条件驱动字段
- 本地业务保留字段

### `runner`
必须基于本合同执行：
- view diff
- field diff
- create/update/delete apply

---

## 12. 最终决策摘要

### 视图级主键
- `mdl_id = f_view_id`

### 视图级当前同步类型
- 仅元数据视图
- 本地 `type = 1`

### 字段级主键
- 优先 `technical_name`
- 兜底 `original_name`

### 业务保留字段原则
- 审核、归类、理解、标准、分级、敏感、开放、共享等字段不由源强覆盖

### 本地可初始化但后续保留的字段
- `business_name`
- `field_description`

### status 规则
- `form_view.status` 由 diff 计算
- `form_view_field.status` 当前阶段不单独表达“更新”，更新默认维持 `0`

---

## 13. 需要配套的下一个文档

本合同之后，建议紧接着输出：

1. `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`
2. `05_DIFF_RULES_SPEC.md`
3. `06_APPLY_RULES_SPEC.md`

因为这三份会把本合同的映射关系进一步落到：

- JSON 样本
- 比较规则
- 落库规则
