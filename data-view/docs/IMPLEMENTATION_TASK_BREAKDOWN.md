# IMPLEMENTATION_TASK_BREAKDOWN

## 1. 文档目的

本文档把 `sync-worker` 的实现工作拆解成一组**可独立开发、可独立验收、可并行分配给不同大模型**的任务包（Package）。

目标：

1. 让不同模型开发时遵循同一套边界
2. 避免多个模型同时改同一层、同一文件
3. 降低“大模型自由发挥”导致的实现漂移
4. 让每个 Package 都有明确输入、输出、依赖、验收条件

---

## 2. 总体拆包原则

### 2.1 拆包原则

每个 Package 必须满足：

- 职责单一
- 依赖清晰
- 文件范围明确
- 可独立测试
- 可独立 review
- 尽量减少跨层改动

### 2.2 拆包边界

拆包按以下顺序组织：

1. 工程骨架
2. 数据访问层
3. 源读取与标准化
4. diff 逻辑
5. apply 逻辑
6. 任务状态机
7. worker 运行时
8. 测试与 fixtures

---

## 3. Package 总览

| Package | 名称 | 目标 | 依赖 |
|---|---|---|---|
| P0 | Skeleton & Bootstrap | 建立工程骨架与程序入口 | 无 |
| P1 | Config & ServiceContext | 配置与依赖注入 | P0 |
| P2 | Model & Repository | 表模型与仓储层 | P0, P1 |
| P3 | Source Reader & Normalizer | 读取源 `t_data_view` 并标准化 | P1, P2 |
| P4 | Diff Engine | 实现 view/field diff 规则 | P2, P3 |
| P5 | Apply Service | 实现 create/update/delete apply | P2, P4 |
| P6 | Task State Machine | 实现任务状态流转与 retry/backoff | P2 |
| P7 | Watermark Service | 实现长期进度读取与推进 | P2, P6 |
| P8 | Worker Runtime | 实现 poller/dispatcher/heartbeat/runner | P2, P3, P4, P5, P6, P7 |
| P9 | Create-Job Command | 实现任务生成器 | P2, P6 |
| P10 | Tests & Fixtures | 单测、集成测试、fixtures | P2~P9 |

---

## 4. Package 详细定义

# P0 - Skeleton & Bootstrap

## 目标
建立可编译、可运行的最小工程骨架。

## 范围
- `cmd/sync-worker/main.go`
- `go.mod`
- 基础目录结构
- 最小日志初始化
- 命令路由骨架

## 输入
- `PROJECT_STRUCTURE_AND_LAYERING.md`

## 输出
- 工程可编译
- 可识别 `create-job` / `run-worker` 命令入口

## 不允许做
- 不实现任何业务逻辑
- 不直接写 SQL
- 不实现 diff / apply / watermark

## 验收标准
- `go build ./...` 通过
- `./sync-worker create-job --help`
- `./sync-worker run-worker --help`

---

# P1 - Config & ServiceContext

## 目标
定义配置结构，完成依赖容器初始化。

## 范围
- `internal/config/config.go`
- `internal/svc/service_context.go`
- `etc/sync-worker.yaml`

## 输入
- `PROJECT_STRUCTURE_AND_LAYERING.md`

## 输出
- 配置结构体
- DB / logger / worker 参数注入
- service context 可供 command/domain 使用

## 不允许做
- 不写业务逻辑
- 不做任务状态更新

## 验收标准
- 配置可成功加载
- ServiceContext 初始化通过
- 单测覆盖配置解析

---

# P2 - Model & Repository

## 目标
实现 2 张表与本地目标表的 model/repository。

## 范围
- `internal/model/sync_task_model.go`
- `internal/model/sync_watermark_model.go`
- `internal/model/form_view_model.go`
- `internal/model/form_view_field_model.go`
- `internal/repository/*`

## 输入
- `03_FIELD_MAPPING_CONTRACT.md`
- `SYNC_TASK_STATE_MACHINE.md`
- 本地表 DDL / task 表 DDL

## 输出
- SyncTaskRepo
- SyncWatermarkRepo
- FormViewRepo
- FormViewFieldRepo
- DatasourceRepo（若需要）

## 必须实现的方法
- `ListRunnableTasks`
- `TryAcquireTask`
- `UpdateTaskProgress`
- `MarkTaskSuccess`
- `MarkTaskRetry`
- `MarkTaskFailed`
- `GetOrInitWatermark`
- `AdvanceWatermark`
- `BatchLoadLocalViews`
- `BatchLoadLocalFields`

## 不允许做
- 不判断 retryable
- 不做 diff 逻辑
- 不做状态机决策

## 验收标准
- Repository 集成测试通过
- SQL 不出现在 domain/command

---

# P3 - Source Reader & Normalizer

## 目标
实现源读取与 `f_fields` 标准化。

## 范围
- `internal/domain/source/reader.go`
- `internal/domain/source/normalizer.go`

## 输入
- `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`

## 输出
- `ReadPage(datasourceID, cursor, pageSize)`
- `NormalizeView(row)`
- `NormalizeField(item)`

## 必须实现
- `change_time = max(create, update, delete)`
- `FieldKey` 生成
- `Type` / `Nullable` 归一化
- field hash / payload hash
- 非法 JSON 错误处理

## 不允许做
- 不比较本地数据
- 不写本地表

## 验收标准
- Normalizer 单测全部通过
- 能处理 fixtures 中标准 / 异常样本

---

# P4 - Diff Engine

## 目标
实现视图级与字段级 diff。

## 范围
- `internal/domain/diff/engine.go`
- `internal/domain/diff/rules.go`

## 输入
- `03_FIELD_MAPPING_CONTRACT.md`
- `05_DIFF_RULES_SPEC.md`

## 输出
- `DiffResult`
- `CreateViewDiff`
- `UpdateViewDiff`
- `DeleteViewDiff`
- `Severity`

## 必须实现
- `NEW_VIEW / UPDATE_VIEW / DELETE_VIEW / UNCHANGED_VIEW`
- `FIELD_CREATE / UPDATE / DELETE / SUSPECTED_RENAME`
- hash 快速路径
- page 内并发 diff
- 顺序恢复

## 不允许做
- 不写库
- 不更新 task/watermark

## 验收标准
- DiffEngine 单测通过
- fixture 的 diff 结果与 expected 一致

---

# P5 - Apply Service

## 目标
把 DiffResult 正确落到本地表。

## 范围
- `internal/domain/apply/service.go`
- `internal/domain/apply/result.go`

## 输入
- `03_FIELD_MAPPING_CONTRACT.md`
- `06_APPLY_RULES_SPEC.md`

## 输出
- `ApplyInMiniBatches`
- `ApplyResult`
- `LastSafeCursor`

## 必须实现
- `NEW_VIEW` create
- `UPDATE_VIEW` update
- `DELETE_VIEW` 逻辑删除
- field create/update/delete
- 强同步 / 条件驱动 / 本地保留字段策略
- mini-batch 事务

## 不允许做
- 不直接生成任务
- 不直接推进长期 watermark

## 验收标准
- ApplyService 单测通过
- 本地保留字段不被覆盖
- `last_safe_cursor` 计算正确

---

# P6 - Task State Machine

## 目标
实现任务状态流转与 retry/backoff 决策。

## 范围
- `internal/domain/task/service.go`
- `internal/domain/task/state_machine.go`
- `internal/domain/task/retry_policy.go`

## 输入
- `SYNC_TASK_STATE_MACHINE.md`

## 输出
- 状态流转函数
- retryable 判定
- backoff 计算
- dedupe key 生成规则

## 必须实现
- `pending/running/retry_waiting/success/failed/cancelled`
- `running -> retry_waiting`
- `running -> failed`
- datasource 冲突短退避

## 不允许做
- 不直接读源
- 不做 diff/apply

## 验收标准
- 状态机单测通过
- retry/backoff 与合同一致

---

# P7 - Watermark Service

## 目标
实现 datasource 长期进度服务。

## 范围
- `internal/domain/watermark/service.go`

## 输入
- `SYNC_TASK_STATE_MACHINE.md`
- `05_DIFF_RULES_SPEC.md`

## 输出
- `GetOrInit`
- `Advance`
- `CursorGreater`
- 只进不退校验

## 必须实现
- `change_time` cursor
- `last_safe_cursor` 推进
- `last_full_sync_at`
- `last_reconcile_at`

## 不允许做
- 不做任务调度
- 不做 diff

## 验收标准
- Watermark 单测通过
- 不允许回退
- 部分成功只推进到安全断点

---

# P8 - Worker Runtime

## 目标
实现常驻 worker 主循环。

## 范围
- `internal/domain/runner/poller.go`
- `internal/domain/runner/dispatcher.go`
- `internal/domain/runner/heartbeat.go`
- `internal/domain/runner/service.go`

## 输入
- `SYNC_TASK_STATE_MACHINE.md`
- `PROJECT_STRUCTURE_AND_LAYERING.md`

## 输出
- poll -> acquire -> heartbeat -> run -> finish/retry

## 必须实现
- 扫 runnable tasks
- 任务抢占
- datasource 互斥检查
- 心跳续租
- 串联 source/diff/apply/watermark

## 不允许做
- 不直接生成任务
- 不绕过状态机直接写成功/失败

## 验收标准
- runner 集成测试通过
- 多任务并发下行为正确

---

# P9 - Create-Job Command

## 目标
实现任务生成器。

## 范围
- `internal/command/createjob/create_job_command.go`
- `internal/domain/task/create_service.go`

## 输入
- `SYNC_TASK_STATE_MACHINE.md`
- `PROJECT_STRUCTURE_AND_LAYERING.md`

## 输出
- 创建 `sync_task`
- 生成 `batch_id`
- 生成 `schedule_slot`
- 生成 `dedupe_key`

## 必须实现
- `scope=all-active`
- `scope=datasource-list`
- `scope=shard`
- 幂等创建 / duplicated 返回

## 不允许做
- 不执行同步
- 不读取源视图

## 验收标准
- create-job 单测 / 集成测试通过
- 相同 dedupe_key 不重复创建

---

# P10 - Tests & Fixtures

## 目标
补齐测试与夹具。

## 范围
- `tests/unit/*`
- `tests/integration/*`
- `fixtures/*`

## 输入
- `TEST_PLAN_AND_FIXTURES.md`

## 输出
- 单测
- 集成测试
- e2e fixtures
- golden expected

## 必须实现
- normalizer tests
- diff tests
- apply tests
- state machine tests
- runner 最小 E2E
- 部分成功 / 部分失败案例

## 验收标准
- CI 可执行
- fixture 输出稳定
- expected 可回归对比

---

## 5. 推荐执行顺序

### 5.1 串行依赖顺序

推荐顺序：

```text
P0 -> P1 -> P2 -> P3 -> P4 -> P5 -> P6 -> P7 -> P8 -> P9 -> P10
```

### 5.2 可并行的部分

以下在前置满足后可以并行：

- `P4 Diff Engine`
- `P6 Task State Machine`
- `P7 Watermark Service`

以及稍后：

- `P8 Worker Runtime`
- `P9 Create-Job`

---

## 6. 多模型分工建议

### 模型 A
- P0
- P1
- P2

### 模型 B
- P3
- P4

### 模型 C
- P5
- P6
- P7

### 模型 D
- P8
- P9
- P10

### Reviewer 模型
- 不开发
- 专门做包级 review 与一致性检查

---

## 7. Package 交付格式要求

每个 Package 必须交付：

1. 变更计划
2. 文件清单
3. 代码
4. 自检结果
5. 风险与待确认点

### 标准格式
```text
Package:
Goal:
Allowed files:
Blocked files:
Specs:
Implementation plan:
Code:
Self-check:
Open questions:
```

---

## 8. 一致性控制要求

为了让不同模型实现基本一致，每个 Package 必须满足：

1. 文件范围固定
2. 不得修改其他 Package 的合同文档
3. 不得引入未批准依赖
4. 不得发明新状态枚举
5. 不得自创新的 repository 分层
6. 必须复用统一的 domain type / cursor / diff result 结构
7. 必须通过统一 fixtures 和 tests

---

## 9. 交付完成定义（Definition of Done）

一个 Package 被认为完成，必须满足：

1. 编译通过
2. 对应测试通过
3. 不违反分层约束
4. 不修改禁止变更文件
5. 满足对应 spec 文档要求
6. reviewer 检查通过

---

## 10. 最终建议

当前阶段最适合按 Package 逐个推进，而不是一次让模型生成整个工程。

### 先做
- P0 ~ P4

### 再做
- P5 ~ P7

### 最后做
- P8 ~ P10
