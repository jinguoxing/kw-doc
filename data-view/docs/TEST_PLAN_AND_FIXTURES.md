# TEST_PLAN_AND_FIXTURES

## 1. 文档目的

本文档定义 2 表版 `sync-worker` 的测试方案与夹具（fixtures）规范，目的是让：

- 研发
- 大模型生成代码
- 自动化测试
- 回归测试

都能基于统一的样本和断言来验证：

1. 字段标准化是否正确
2. diff 规则是否正确
3. apply 结果是否正确
4. 状态机流转是否正确
5. watermark 是否正确推进
6. retry / lease / datasource 互斥是否正确

---

## 2. 测试分层

本项目测试分 4 层：

1. **单元测试**
2. **模块集成测试**
3. **端到端流程测试**
4. **回归测试**

---

## 3. 单元测试范围

## 3.1 Normalizer 单元测试

### 覆盖点
1. `f_fields` 解析
2. `FieldKey` 生成
3. `Type` 归一化
4. `Nullable` 归一化
5. 字段级 hash
6. 视图级 payload hash
7. 异常 JSON 处理

### 必测案例
- 标准字段数组
- 字段别名兼容
- 空数组
- 非法 JSON
- 缺少关键字段
- 类型带长度精度
- nullable 多种输入格式

---

## 3.2 DiffEngine 单元测试

### 覆盖点
1. `NEW_VIEW`
2. `DELETE_VIEW`
3. `UNCHANGED_VIEW`
4. `UPDATE_VIEW`
5. `FIELD_CREATE`
6. `FIELD_UPDATE`
7. `FIELD_DELETE`
8. `FIELD_SUSPECTED_RENAME`
9. `Severity` 判断

### 必测案例
- 命中 `mdl_id`
- 命中 fallback `(datasource_id, technical_name)`
- 视图 hash 一致直接跳过
- 字段新增
- 字段删除
- 字段类型变化
- 字段顺序变化
- 疑似 rename

---

## 3.3 ApplyService 单元测试

### 覆盖点
1. 创建 view + 全字段
2. 更新 view + create/update/delete fields
3. 删除 view + 级联逻辑删除 field
4. mini-batch 事务
5. `last_safe_cursor`
6. 本地保留字段不被覆盖

### 必测案例
- `NEW_VIEW`
- `UPDATE_VIEW`
- `DELETE_VIEW`
- 第 3 个视图 apply 失败
- 业务保留字段保留
- 条件驱动字段初始化

---

## 3.4 State Machine 单元测试

### 覆盖点
1. `pending -> running`
2. `running -> success`
3. `running -> retry_waiting`
4. `retry_waiting -> running`
5. `running -> failed`
6. datasource 冲突短退避
7. 取消状态

---

## 4. 集成测试范围

## 4.1 repository 集成测试

### 覆盖点
- `TryAcquireTask`
- `ListRunnableTasks`
- `UpdateTaskProgress`
- `MarkTaskSuccess`
- `MarkTaskRetry`
- `MarkTaskFailed`
- `GetOrInitWatermark`
- `AdvanceWatermark`
- `BatchLoadLocalViews`
- `BatchLoadLocalFields`

### 要求
- 使用真实 MySQL / MariaDB
- 不建议只 mock repository

---

## 4.2 source reader 集成测试

### 覆盖点
- 按 datasource 读取
- 按 cursor 读取
- `(change_time, view_id)` 排序
- page 分页
- 同 change_time 场景

---

## 4.3 runner 集成测试

### 覆盖点
- 单 datasource 正常执行
- retry 场景
- `last_safe_cursor`
- watermark 推进
- 多 page 场景
- datasource 冲突场景

---

## 5. 端到端测试范围

端到端流程应覆盖：

1. `create-job` 创建 task
2. `run-worker` 轮询 task
3. 抢占执行
4. 读取源 `t_data_view`
5. diff
6. apply
7. 推进 watermark
8. 写回 `sync_task`

---

## 6. Fixtures 目录规范

推荐目录：

```text
fixtures/
  source/
    data_view_page_01.json
    data_view_page_02.json
    data_view_invalid_fields.json

  local/
    form_view_before.json
    form_view_field_before.json

  expected/
    diff_result_case_01.json
    form_view_after_case_01.json
    form_view_field_after_case_01.json
    sync_task_after_case_01.json
    watermark_after_case_01.json

  edge_cases/
    duplicate_local_view.json
    duplicate_local_field.json
    rename_case.json
    partial_success_case.json
```

---

## 7. Fixtures 内容要求

## 7.1 `source/`
保存源侧样本：

- 原始 `t_data_view` 行数组
- 尽量接近真实结构
- 包含 `f_fields` 原始文本

### 至少准备
- 一个标准元数据视图样本
- 一个多字段视图样本
- 一个删除态样本
- 一个非法 `f_fields` 样本
- 一个同 `change_time` 多对象样本

---

## 7.2 `local/`
保存本地同步前数据：

- `form_view_before.json`
- `form_view_field_before.json`

### 作用
用于在测试中构造本地已有状态。

---

## 7.3 `expected/`
保存期望输出：

- `DiffResult`
- apply 后的本地表快照
- `sync_task` 更新结果
- `watermark` 更新结果

---

## 7.4 `edge_cases/`
保存特殊与异常场景：

- 重复命中
- rename 疑似
- 部分成功部分失败
- datasource 冲突
- retry 场景

---

## 8. 测试案例矩阵

下面给出建议的核心测试矩阵。

---

## 8.1 Normalizer 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| N1 | 标准字段数组 | 成功解析 |
| N2 | `field_name` / `label` 兼容 | 成功解析 |
| N3 | `nullable` 多种格式 | 正确归一化 |
| N4 | `varchar(255)` / `decimal(18,2)` | 长度精度正确 |
| N5 | 空字段数组 | 解析为空 |
| N6 | 非法 JSON | 返回错误 |
| N7 | 缺失 `name` / `original_name` | 返回错误 |

---

## 8.2 View-level Diff 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| V1 | 本地无视图，源有视图 | `NEW_VIEW` |
| V2 | 本地有视图，源删除态 | `DELETE_VIEW` |
| V3 | hash 一致 | `UNCHANGED_VIEW` |
| V4 | comment 变化 | `UPDATE_VIEW` + S2 |
| V5 | meta_table_name 变化 | `UPDATE_VIEW` + S1 |
| V6 | technical_name 变化 | `UPDATE_VIEW` + S1 |

---

## 8.3 Field-level Diff 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| F1 | 本地无字段，源有字段 | `FIELD_CREATE` |
| F2 | 本地有字段，源无字段 | `FIELD_DELETE` |
| F3 | type 变化 | `FIELD_UPDATE` + S1 |
| F4 | display_name 变化 | `FIELD_UPDATE` + S2 |
| F5 | index 变化 | `FIELD_UPDATE` + S3 |
| F6 | 疑似 rename | `FIELD_SUSPECTED_RENAME` |

---

## 8.4 Apply 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| A1 | 新建视图 | 创建 view + 所有 field |
| A2 | 更新视图 | 更新 view + 字段增删改 |
| A3 | 删除视图 | 逻辑删除 view + 字段 |
| A4 | 字段更新 | 本地字段正确覆盖 |
| A5 | 本地保留字段存在 | 不被覆盖 |
| A6 | 条件驱动字段为空 | 初始化 |
| A7 | 第 3 个视图 apply 失败 | `last_safe_cursor` = 第 2 个视图 |

---

## 8.5 State Machine 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| S1 | 新任务创建 | `pending` |
| S2 | worker 抢占成功 | `running` |
| S3 | 可重试失败 | `retry_waiting` |
| S4 | 重试成功 | `success` |
| S5 | 超过最大重试 | `failed` |
| S6 | datasource 冲突 | 短退避 + `retry_waiting` |
| S7 | cancelled | worker 安全退出 |

---

## 8.6 Watermark 测试矩阵

| Case | 场景 | 预期 |
|---|---|---|
| W1 | 首次同步 | watermark 初始化 |
| W2 | 正常推进 | 推进到最后成功视图 |
| W3 | 部分成功部分失败 | 只推进到连续成功前缀 |
| W4 | 同 change_time 多视图 | cursor 精确到 `view_id` |
| W5 | 不允许回退 | 旧值保持 |

---

## 9. 最小端到端样本组合

为了让大模型和研发都能快速起步，建议至少准备以下 5 组样本。

---

## 9.1 Case E2E-01：首次全量同步
### 输入
- source: 2 个元数据视图，4 个字段 / 视图
- local: 空

### 期望
- 创建 2 条 `form_view`
- 创建 8 条 `form_view_field`
- task success
- watermark 推进到第 2 个视图

---

## 9.2 Case E2E-02：增量无变化
### 输入
- source 与本地完全一致
- hash 一致

### 期望
- 不写本地
- `hash_skipped > 0`
- watermark 可推进到最后对象

---

## 9.3 Case E2E-03：字段结构变化
### 输入
- 一个视图新增 1 个字段，更新 1 个字段，删除 1 个字段

### 期望
- `FIELD_CREATE = 1`
- `FIELD_UPDATE = 1`
- `FIELD_DELETE = 1`
- task success

---

## 9.4 Case E2E-04：源删除
### 输入
- 源视图 `delete_time > 0`
- 本地已有对应 view 和 fields

### 期望
- 本地逻辑删除
- task success
- watermark 推进

---

## 9.5 Case E2E-05：部分成功部分失败
### 输入
- 同 page 中第 1、2 个视图成功
- 第 3 个视图 apply 失败
- 第 4 个视图未执行

### 期望
- `last_safe_cursor = 第2个视图`
- task `retry_waiting` 或 `failed`
- watermark 只推进到第 2 个视图

---

## 10. 失败样本记录规则

在测试中，`sync_task.f_result_json` 应至少校验以下结构：

```json
{
  "success_count": 90,
  "failed_count": 10,
  "failed_view_ids_sample": ["dv_101", "dv_102"],
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

## 11. Mock 与真实依赖的使用原则

## 11.1 单元测试
允许 mock：
- repository
- source reader

### 目标
验证纯逻辑：
- normalizer
- diff
- apply
- state machine

---

## 11.2 集成测试
不建议全部 mock。

### 至少真实连接：
- MySQL / MariaDB
- 真实 repo SQL

### 原因
很多问题只会在：
- SQL 条件
- 事务边界
- 索引 / 唯一键
- 并发抢占
中暴露。

---

## 12. 自动化回归建议

建议在 CI 中至少跑：

1. Normalizer 单测
2. DiffEngine 单测
3. ApplyService 单测
4. StateMachine 单测
5. 一个最小 E2E 场景
6. 一个部分成功 / 部分失败场景

---

## 13. 大模型代码生成约束

为了让大模型稳定生成测试代码，本文件要求：

1. 每个核心模块都必须有单测
2. fixture 必须放到固定目录
3. 不允许测试里硬编码超长 JSON
4. 必须把 expected output 单独落文件
5. 必须覆盖异常场景，不只覆盖 happy path
6. 必须验证 `last_safe_cursor`
7. 必须验证业务保留字段不被覆盖
8. 必须验证 datasource 冲突和 retry 状态

---

## 14. 推荐文件清单

最终建议至少有这些测试文件：

```text
tests/
  unit/
    normalizer_test.go
    diff_engine_test.go
    apply_service_test.go
    state_machine_test.go

  integration/
    sync_task_repo_test.go
    sync_watermark_repo_test.go
    source_reader_test.go
    runner_test.go

fixtures/
  source/
    source_case_01.json
    source_case_02.json
    source_invalid_fields.json
  local/
    local_views_case_01.json
    local_fields_case_01.json
  expected/
    diff_case_01.json
    apply_case_01_views.json
    apply_case_01_fields.json
    task_case_01.json
    watermark_case_01.json
  edge_cases/
    duplicate_local_view.json
    duplicate_local_field.json
    rename_case.json
    partial_success_case.json
    datasource_conflict_case.json
```

---

## 15. 最终决策摘要

### 测试体系核心决策
1. 测试分为：单测、集成、E2E、回归
2. fixture 必须文件化、结构化
3. 必须覆盖正常场景与异常场景
4. 必须覆盖 watermark 与 `last_safe_cursor`
5. 必须覆盖 datasource 互斥与 retry
6. 大模型生成代码时，测试文件也必须同步生成

---

## 16. 下一步建议

到这一步，工程化产代码最关键的文档已经基本齐了：

- `03_FIELD_MAPPING_CONTRACT.md`
- `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`
- `05_DIFF_RULES_SPEC.md`
- `06_APPLY_RULES_SPEC.md`
- `SYNC_TASK_STATE_MACHINE.md`
- `PROJECT_STRUCTURE_AND_LAYERING.md`
- `TEST_PLAN_AND_FIXTURES.md`

下一步最适合做的是：

1. 基于这些文档生成：
   - `sync-worker` 工程骨架
2. 输出：
   - `command/createjob`
   - `command/worker`
   - `domain/source`
   - `domain/diff`
   - `domain/apply`
   - `domain/task`
   - `domain/watermark`
   - `repository`
3. 同步生成最小测试夹具与样例
