# PROJECT_STRUCTURE_AND_LAYERING

## 1. 文档目的

本文档定义 2 表版 `sync-worker` 工程在 go-zero 风格下的：

- 目录结构
- 分层边界
- 包职责
- 依赖方向
- 事务边界放置位置
- SQL 放置位置
- 命令模式边界

本文档直接约束：

- 工程脚手架
- 包划分
- repository 设计
- domain service 设计
- create-job / run-worker 的代码组织方式
- 大模型生成代码时的文件落点与依赖边界

---

## 2. 总体原则

## 2.1 单工程、双命令模式

当前阶段采用：

> **一个工程，一个镜像，两个命令模式**

- `create-job`
- `run-worker`

### 说明
- 不单独拆 `sync-admin-api`
- 但不等于创建与执行逻辑混在一起
- 仍然要求代码层明确分层

---

## 2.2 核心原则

1. **命令层不写业务逻辑**
2. **业务逻辑放在 domain/service 层**
3. **SQL 只放在 repository/model 层**
4. **事务只由 application/domain service 控制**
5. **normalizer / diff / apply 属于纯业务逻辑，不得依赖 HTTP / CLI**
6. **watermark / task 状态流转属于业务核心，不得散落在各 command 中**

---

## 3. 推荐目录结构

推荐工程目录如下：

```text
sync-worker/
  cmd/
    sync-worker/
      main.go

  etc/
    sync-worker.yaml

  internal/
    config/
      config.go

    svc/
      service_context.go

    command/
      createjob/
        create_job_command.go
      worker/
        run_worker_command.go

    domain/
      contracts/
        types.go

      task/
        service.go
        create_service.go
        state_machine.go
        retry_policy.go

      watermark/
        service.go

      source/
        reader.go
        normalizer.go

      diff/
        engine.go
        rules.go

      apply/
        service.go
        result.go

      runner/
        service.go
        heartbeat.go
        dispatcher.go
        poller.go

    repository/
      sync_task_repo.go
      sync_watermark_repo.go
      form_view_repo.go
      form_view_field_repo.go
      datasource_repo.go
      tx_manager.go

    model/
      sync_task_model.go
      sync_watermark_model.go
      form_view_model.go
      form_view_field_model.go
      datasource_model.go

  docs/
  fixtures/
```

---

## 4. 各层职责定义

## 4.1 `cmd/`
### 作用
程序入口。

### 职责
- 解析命令行参数
- 加载配置
- 初始化 service context
- 路由到对应 command

### 不允许做的事
- 不允许直接写 SQL
- 不允许直接改任务状态
- 不允许包含 diff / apply 逻辑

---

## 4.2 `internal/config/`
### 作用
配置结构定义。

### 职责
- 定义 DB / logger / worker / retry / pageSize / diffWorkers 等配置结构
- 承载 YAML -> Go struct 的映射

### 不允许做的事
- 不写业务逻辑

---

## 4.3 `internal/svc/`
### 作用
依赖容器 / 运行时上下文。

### 职责
- 初始化：
  - DB
  - logger
  - model/repository
  - domain service
- 注入共享依赖

### 不允许做的事
- 不承载业务流程
- 不直接调度状态机

---

## 4.4 `internal/command/`
### 作用
命令模式的薄入口层。

### 需要区分：
- `createjob`
- `worker`

### 职责
#### createjob command
- 接收 CLI 参数
- 构造 `CreateJobRequest`
- 调用 `domain/task.CreateService`

#### worker command
- 启动常驻循环
- 调用 `domain/runner.Poller/Dispatcher/Service`

### 不允许做的事
- 不直接操作 repo
- 不直接写任务状态 SQL
- 不直接做 diff / apply

---

## 4.5 `internal/domain/`
### 作用
核心业务逻辑层。

这是整个工程最重要的一层。

### 职责
- task 状态机
- watermark 服务
- source 标准化
- diff
- apply
- worker 运行编排

### 原则
- 可以依赖 repository 抽象
- 不依赖具体 command
- 不依赖具体 DB model 细节
- 不依赖 CLI / HTTP 输入结构

---

## 4.6 `internal/repository/`
### 作用
仓储层，封装数据库访问与持久化操作。

### 职责
- 查询 / 插入 / 更新 SQL
- 批量加载本地视图与字段
- task 抢占 / 状态更新
- watermark 读取与推进

### 原则
- 只做数据访问
- 不写业务判断
- 不做 diff 分类
- 不决定状态机规则

---

## 4.7 `internal/model/`
### 作用
最贴近表结构的数据对象层。

### 职责
- 与数据库表字段一一对应
- 用于 repository 层读写

### 原则
- 尽量接近 DDL
- 不承载业务方法
- 不混入 diff / apply 逻辑

---

## 5. 命令模式边界

## 5.1 `create-job` 的边界

### 允许做
- 解析输入
- 选择 datasource scope
- 计算 batch_id / dedupe_key / schedule_slot
- 创建 `sync_task`

### 不允许做
- 不执行数据同步
- 不读取 `t_data_view`
- 不推进 watermark
- 不进行 diff / apply

---

## 5.2 `run-worker` 的边界

### 允许做
- poll task
- acquire task
- 启动 heartbeat
- 读取 watermark
- 读源数据
- diff
- apply
- 更新 task
- 推进 watermark

### 不允许做
- 不主动决定要创建哪些任务
- 不直接实现 CronJob 语义
- 不跳过状态机直接写成功/失败

---

## 6. domain 层子模块划分

---

## 6.1 `domain/task`
### 职责
任务相关核心逻辑：

- `CreateService`
- `StateMachine`
- `RetryPolicy`

### 包含内容
- request -> task 的生成逻辑
- 状态流转校验
- dedupe 规则
- backoff 规则

---

## 6.2 `domain/watermark`
### 职责
管理 datasource 长期进度。

### 包含内容
- `GetOrInit`
- `Advance`
- Cursor 比较
- 只进不退校验

---

## 6.3 `domain/source`
### 职责
源表读取与标准化。

### 包含内容
- `ReadPage`
- `NormalizeView`
- `NormalizeField`
- `PayloadHash`

---

## 6.4 `domain/diff`
### 职责
把 source objects 与 local objects 做比较。

### 包含内容
- `ComparePage`
- `CompareOne`
- `compareFields`
- severity 计算
- `last_safe_cursor` 所依赖的顺序信息

---

## 6.5 `domain/apply`
### 职责
把 diff 结果落到本地表。

### 包含内容
- `ApplyInMiniBatches`
- 单视图事务
- create / update / delete 规则
- 本地保留字段策略

---

## 6.6 `domain/runner`
### 职责
后台 worker 编排。

### 包含内容
- `Poller`
- `Dispatcher`
- `RunOneTask`
- `Heartbeat`

---

## 7. repository 层职责与边界

## 7.1 repository 必须做的事

- 提供查询与更新接口
- 封装 SQL
- 对外暴露明确方法语义
- 隐藏数据库实现细节

### 例子
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

---

## 7.2 repository 不允许做的事

- 不得决定状态机规则
- 不得直接判断 retryable / non-retryable
- 不得计算 diff
- 不得计算 `last_safe_cursor`
- 不得直接拼接复杂业务 DTO

---

## 8. SQL 放置规则

## 8.1 SQL 应放哪里
所有 SQL 统一放在：

- `internal/repository/*`

### 原因
- 便于替换数据源实现
- 避免 command / domain 混 SQL
- 便于测试 mock

---

## 8.2 不允许 SQL 出现的位置

以下层禁止直接写 SQL：

- `command`
- `domain`
- `svc`
- `cmd/main`

---

## 9. 事务边界放置规则

## 9.1 事务由谁控制

### 原则
事务由 **application/domain service** 控制，而不是由 repository 自己内部随意开启。

### 当前推荐
- `apply_service` 控制 mini-batch 事务
- 单视图的 view + field 变更必须在同一事务内

---

## 9.2 repository 的事务接口

建议 repository 支持：

- `WithTx(ctx, fn)`
- 或显式 `tx` 参数注入

### 原则
- repository 可以接受 tx
- 但不主动在内部开启不透明事务

---

## 10. 依赖方向规则

必须保持以下依赖方向：

```text
cmd -> command -> domain -> repository -> model
                    \-> svc
```

### 说明
- `command` 依赖 `domain`
- `domain` 依赖 repository 抽象
- repository 依赖 model
- `domain` 不得依赖 command
- `repository` 不得依赖 domain

---

## 11. DTO / Model / Domain Object 区分

为了避免大模型把所有 struct 混在一起，必须明确三类对象：

### 11.1 Input DTO
来源于 CLI / 配置 / 上层请求。

例子：
- `CreateJobRequest`

### 11.2 Model
对应表结构。

例子：
- `SyncTaskModel`
- `SyncWatermarkModel`
- `FormViewModel`

### 11.3 Domain Object
业务语义对象。

例子：
- `SourceView`
- `SourceField`
- `DiffResult`
- `Cursor`
- `ApplyResult`

### 规则
- 不允许把 model 直接拿来做 diff
- 不允许把 CLI request 直接传给 repository

---

## 12. create-job 的代码组织建议

推荐流程：

```text
command/createjob
  -> domain/task.CreateService
      -> repository.DatasourceRepo
      -> repository.SyncTaskRepo
```

### 职责分配
- `command/createjob`：参数解析
- `task.CreateService`：batch_id / schedule_slot / dedupe_key / scope 逻辑
- `SyncTaskRepo`：插入任务

---

## 13. run-worker 的代码组织建议

推荐流程：

```text
command/worker
  -> domain/runner.Poller
  -> domain/runner.Dispatcher
  -> domain/runner.RunOneTask
      -> domain/watermark
      -> domain/source
      -> domain/diff
      -> domain/apply
```

### 职责分配
- `command/worker`：常驻进程启动
- `Poller`：取待执行任务
- `Dispatcher`：限流 + acquire
- `RunOneTask`：单 datasource 主流程
- `source/diff/apply/watermark`：独立业务模块

---

## 14. 配置文件边界

配置文件只允许承载：

- DB 连接
- logger
- worker 并发
- page size
- diffWorkers
- lease_ms
- heartbeat_ms
- retry_backoff
- retry_max_attempt

### 不允许放的内容
- 复杂业务逻辑表达式
- diff 规则
- apply 规则

这些必须固化在文档和代码中，而不是配置里临时拼。

---

## 15. 大模型代码生成约束

为了让大模型稳定生成工程代码，本文件要求：

1. 严格按目录落文件
2. 不得把所有代码写进一个 `service.go`
3. `create-job` 和 `run-worker` 必须分 command
4. domain / repository / model 必须分层
5. SQL 必须只在 repository
6. 事务控制不得藏在 repository 内部
7. `diff_engine` 与 `apply_service` 必须独立
8. `watermark_service` 必须独立
9. `state_machine` 必须独立
10. 不得把 CLI 参数结构直接传进 repository

---

## 16. 推荐的接口清单

## 16.1 task service
- `CreateTasks(req)`
- `BuildBatchID()`
- `BuildScheduleSlot()`
- `BuildDedupeKey()`

## 16.2 runner service
- `Run()`
- `Poll()`
- `Dispatch()`
- `RunOneTask(taskID)`

## 16.3 source service
- `ReadPage(datasourceID, cursor, pageSize)`
- `NormalizeView(row)`

## 16.4 diff service
- `ComparePage(sourceViews, localViews, localFields, workers)`

## 16.5 apply service
- `ApplyInMiniBatches(diffResult, batchSize)`

## 16.6 watermark service
- `GetOrInit(datasourceID)`
- `Advance(datasourceID, cursor, taskID, jobType)`

---

## 17. 最终决策摘要

### 工程架构核心决策
1. 单工程、双命令模式
2. `create-job` 与 `run-worker` 逻辑分离
3. `command / domain / repository / model` 明确分层
4. SQL 只放 repository
5. 事务控制放 apply/domain service
6. `state_machine / watermark / diff / apply` 必须独立模块
7. 不允许把仓储层写成业务层
8. 不允许把 domain 写成脚本式大文件

---

## 18. 下一份文档建议

本文件之后，建议继续补：

- `TEST_PLAN_AND_FIXTURES.md`

因为到这一步，工程化产代码已经具备了：

- 业务规则
- 状态机
- 目录结构
- 分层边界

下一步就需要把：

- 单测矩阵
- fixture 样本
- 集成测试方案
- 回归检查点

固定下来。
