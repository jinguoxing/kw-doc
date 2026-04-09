# 《研发实现骨架（go-zero 目录结构 + 关键伪代码）》

## 1. 目标

本文档给出 **2 张表版 form-view/sync** 的研发实现骨架，目标是让研发团队可以直接按模块开工。

当前约束：

- 使用 **go-zero** 作为工程框架
- 使用 **2 张表**：
  - `t_form_view_sync_task`
  - `t_form_view_sync_watermark`
- 使用 **sync-worker 二合一模式**：
  - `create-job`
  - `run-worker`
- `mdl` 作为 core 数据源，`data-view` 作为 application
- 读取源表 `t_data_view`
- 同步写入本地 `form_view / form_view_field`

---

## 2. 总体实现思路

一个二进制，两个命令入口：

- `create-job`：只负责生成 datasource 粒度任务
- `run-worker`：常驻执行任务

共享以下能力：

- 配置加载
- 数据库连接
- model / repo
- watermark 服务
- 源表读取服务
- diff 引擎
- apply 服务

---

## 3. 推荐目录结构

```text
sync-worker/
├── go.mod
├── go.sum
├── README.md
├── cmd/
│   └── sync-worker/
│       └── main.go
├── etc/
│   ├── sync-worker.yaml
│   ├── sync-worker-dev.yaml
│   └── sync-worker-prod.yaml
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── svc/
│   │   └── service_context.go
│   ├── model/
│   │   ├── formviewsynctaskmodel.go
│   │   ├── formviewsyncwatermarkmodel.go
│   │   └── types.go
│   ├── command/
│   │   ├── createjob/
│   │   │   ├── command.go
│   │   │   ├── request.go
│   │   │   └── service.go
│   │   └── worker/
│   │       ├── command.go
│   │       ├── poller.go
│   │       ├── dispatcher.go
│   │       ├── runner.go
│   │       ├── heartbeat.go
│   │       └── retry.go
│   ├── domain/
│   │   ├── entity/
│   │   │   ├── sync_task.go
│   │   │   ├── watermark.go
│   │   │   ├── source_view.go
│   │   │   └── local_view.go
│   │   ├── repo/
│   │   │   ├── sync_task_repo.go
│   │   │   ├── watermark_repo.go
│   │   │   ├── local_form_view_repo.go
│   │   │   └── local_form_view_field_repo.go
│   │   ├── service/
│   │   │   ├── datasource_scope_service.go
│   │   │   ├── dedupe_service.go
│   │   │   ├── watermark_service.go
│   │   │   ├── mdl_reader_service.go
│   │   │   ├── diff_engine.go
│   │   │   ├── apply_service.go
│   │   │   ├── sync_task_service.go
│   │   │   └── task_result_service.go
│   │   └── types/
│   │       ├── cursor.go
│   │       ├── diff.go
│   │       ├── apply_result.go
│   │       └── stats.go
│   ├── infra/
│   │   ├── repoimpl/
│   │   │   ├── sync_task_repo_impl.go
│   │   │   ├── watermark_repo_impl.go
│   │   │   ├── local_form_view_repo_impl.go
│   │   │   └── local_form_view_field_repo_impl.go
│   │   ├── datasource/
│   │   │   └── mdl_reader_mysql.go
│   │   └── lock/
│   │       └── lease.go
│   └── pkg/
│       ├── hash/
│       │   └── hash.go
│       ├── jsonx/
│       │   └── json.go
│       ├── timex/
│       │   └── time.go
│       └── errx/
│           └── error.go
└── scripts/
    ├── run-create-job.sh
    └── run-worker.sh
```

---

## 4. 各目录职责说明

### 4.1 `cmd/sync-worker/main.go`

作用：

- 解析命令行参数
- 识别是 `create-job` 还是 `run-worker`
- 初始化 `ServiceContext`
- 分派到对应命令

---

### 4.2 `internal/config`

作用：

- 定义配置结构体
- 对接 `conf.MustLoad`
- 管理：
  - MySQL
  - 源库只读连接
  - PollInterval
  - LeaseMs
  - HeartbeatMs
  - PageSize
  - DiffWorkers
  - RetryBackoff

---

### 4.3 `internal/svc`

作用：

- 聚合全局依赖
- 统一注入：
  - DB
  - 源库读连接
  - Model
  - Logger
  - Metrics
  - Repo
  - Domain Service

---

### 4.4 `internal/model`

作用：

- 对接 2 张表 model
- 建议用 `goctl model mysql ddl` 生成基础 model
- 自定义扩展查询

建议至少包含：

- `FormViewSyncTaskModel`
- `FormViewSyncWatermarkModel`

---

### 4.5 `internal/command/createjob`

作用：

- 实现命令模式下的“任务创建”
- 参数解析后转业务请求
- 调用 `sync_task_service.CreateBatchTasks()`

---

### 4.6 `internal/command/worker`

作用：

- 常驻轮询任务
- 抢占任务
- 启动单 datasource 执行
- 心跳续租
- 失败重试回退

---

### 4.7 `internal/domain/service`

作用：

- 放核心业务逻辑
- 不依赖 go-zero handler
- 是未来升级为 API 服务时最可复用的层

---

### 4.8 `internal/infra`

作用：

- 仓储实现
- MySQL 读取实现
- 其他基础设施适配

---

## 5. 配置结构建议

### 5.1 `config.go`

```go
package config

import "github.com/zeromicro/go-zero/rest"

type Config struct {
    rest.RestConf

    MetaDB struct {
        DSN string
    }

    MdlReadDB struct {
        DSN string
    }

    Worker struct {
        PollIntervalMs           int
        PickLimit                int
        LeaseMs                  int64
        HeartbeatMs              int64
        MaxDatasourceParallelism int
        PageSize                 int
        DiffWorkers              int
    }

    Retry struct {
        BackoffMs []int64
    }
}
```

说明：

- 即使不暴露 API，也可以沿用 go-zero 的配置风格
- 如果你们最终完全不需要 `rest.RestConf`，也可以改成纯配置结构

---

## 6. `ServiceContext` 骨架

```go
package svc

import (
    "database/sql"

    "sync-worker/internal/config"
    "sync-worker/internal/model"
)

type ServiceContext struct {
    Config config.Config

    MetaDB *sql.DB
    MdlReadDB *sql.DB

    SyncTaskModel      model.FormViewSyncTaskModel
    SyncWatermarkModel model.FormViewSyncWatermarkModel
}

func NewServiceContext(c config.Config) *ServiceContext {
    metaDB := mustOpen(c.MetaDB.DSN)
    mdlReadDB := mustOpen(c.MdlReadDB.DSN)

    return &ServiceContext{
        Config:            c,
        MetaDB:            metaDB,
        MdlReadDB:         mdlReadDB,
        SyncTaskModel:     model.NewFormViewSyncTaskModel(metaDB),
        SyncWatermarkModel:model.NewFormViewSyncWatermarkModel(metaDB),
    }
}
```

---

## 7. `main.go` 骨架

```go
package main

import (
    "flag"
    "fmt"

    "github.com/zeromicro/go-zero/core/conf"
    "sync-worker/internal/command/createjob"
    workercmd "sync-worker/internal/command/worker"
    "sync-worker/internal/config"
    "sync-worker/internal/svc"
)

func main() {
    var configFile string
    flag.StringVar(&configFile, "f", "etc/sync-worker.yaml", "config file")
    flag.Parse()

    if flag.NArg() < 1 {
        panic("missing subcommand: create-job | run-worker")
    }

    var c config.Config
    conf.MustLoad(configFile, &c)
    svcCtx := svc.NewServiceContext(c)

    switch flag.Arg(0) {
    case "create-job":
        createjob.MustRunFromFlags(svcCtx, flag.Args()[1:])
    case "run-worker":
        workercmd.MustRun(svcCtx)
    default:
        panic(fmt.Sprintf("unknown subcommand: %s", flag.Arg(0)))
    }
}
```

---

## 8. `create-job` 实现骨架

## 8.1 参数结构

```go
package createjob

type Request struct {
    JobType       string   // incremental|reconcile|full
    Scope         string   // all-active|datasource-list|shard
    DatasourceIDs []string
    ShardTotal    int
    ShardIndex    int
    TriggerSource string   // manual|schedule|retry|system
    ScheduleSlot  int64
    RequestedBy   string
    RequestedName string
    Reason        string
}
```

---

## 8.2 命令入口骨架

```go
package createjob

import "sync-worker/internal/svc"

func MustRunFromFlags(svcCtx *svc.ServiceContext, args []string) {
    req := ParseFlags(args)
    service := NewCreateJobService(svcCtx)
    if err := service.Run(req); err != nil {
        panic(err)
    }
}
```

---

## 8.3 核心服务骨架

```go
package createjob

import (
    "fmt"
    "time"

    "sync-worker/internal/svc"
)

type Service struct {
    svcCtx *svc.ServiceContext
}

func NewCreateJobService(svcCtx *svc.ServiceContext) *Service {
    return &Service{svcCtx: svcCtx}
}

func (s *Service) Run(req Request) error {
    dsIDs, err := s.resolveDatasourceScope(req)
    if err != nil {
        return err
    }

    batchID := newBatchID()
    now := time.Now().UnixMilli()

    for _, dsID := range dsIDs {
        dedupeKey := fmt.Sprintf("form_view_sync:%s:%s:%d", req.JobType, dsID, req.ScheduleSlot)
        task := BuildSyncTask(batchID, dsID, req, dedupeKey, now)
        if err := s.insertTaskWithDedupe(task); err != nil {
            return err
        }
    }
    return nil
}
```

---

## 8.4 `BuildSyncTask` 规则

建议把下面字段统一构建：

- `f_id`
- `f_batch_id`
- `f_datasource_id`
- `f_job_type`
- `f_trigger_source`
- `f_schedule_slot`
- `f_dedupe_key`
- `f_status = pending`
- `f_attempt_no = 0`
- `f_max_attempt = 3`
- `f_next_run_at = now`
- `f_watermark_type = change_time`
- `f_watermark_before = 0`
- `f_watermark_after = 0`

---

## 9. `run-worker` 实现骨架

## 9.1 主循环骨架

```go
package worker

import (
    "context"
    "time"

    "sync-worker/internal/svc"
)

type App struct {
    svcCtx *svc.ServiceContext
}

func MustRun(svcCtx *svc.ServiceContext) {
    app := &App{svcCtx: svcCtx}
    if err := app.Run(context.Background()); err != nil {
        panic(err)
    }
}

func (a *App) Run(ctx context.Context) error {
    ticker := time.NewTicker(time.Duration(a.svcCtx.Config.Worker.PollIntervalMs) * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            a.pollAndDispatch(ctx)
        }
    }
}
```

---

## 9.2 轮询任务骨架

```go
func (a *App) pollAndDispatch(ctx context.Context) {
    tasks, err := a.listRunnableTasks(ctx)
    if err != nil {
        return
    }

    limiter := make(chan struct{}, a.svcCtx.Config.Worker.MaxDatasourceParallelism)

    for _, task := range tasks {
        limiter <- struct{}{}
        go func(t SyncTask) {
            defer func() { <-limiter }()
            _ = a.tryRunOneTask(ctx, t)
        }(task)
    }
}
```

---

## 9.3 查询可执行任务

查询条件建议与表结构文档保持一致：

- `f_status in ('pending', 'retry_waiting')`
- `f_next_run_at <= now`
- 按 `f_status, f_next_run_at, f_lock_until` 查询。fileciteturn55file0

伪代码：

```go
func (a *App) listRunnableTasks(ctx context.Context) ([]SyncTask, error) {
    return a.repo.ListRunnable(ctx, time.Now().UnixMilli(), a.svcCtx.Config.Worker.PickLimit)
}
```

---

## 9.4 抢占任务骨架

文档里已经给出推荐 SQL：

- `f_status -> running`
- `f_lock_owner`
- `f_lock_until`
- `f_heartbeat_at`
- `f_attempt_no + 1`
- `f_fencing_token + 1`。fileciteturn55file0

伪代码：

```go
func (a *App) tryRunOneTask(ctx context.Context, task SyncTask) error {
    ok, err := a.repo.TryAcquire(ctx, task.ID, workerID(), leaseUntil())
    if err != nil || !ok {
        return err
    }

    busy, err := a.repo.ExistsOtherRunningSameDatasource(ctx, task.DatasourceID, task.ID)
    if err != nil {
        return err
    }
    if busy {
        return a.repo.MarkRetryWaiting(ctx, task.ID, shortBackoffMs(), "datasource busy")
    }

    runner := NewRunner(a.svcCtx)
    return runner.RunDatasourceTask(ctx, task.ID)
}
```

---

## 10. Heartbeat 骨架

```go
package worker

import (
    "context"
    "time"
)

type Heartbeat struct {
    repo SyncTaskRepo
}

func (h *Heartbeat) Renew(ctx context.Context, taskID string, lockOwner string, heartbeatMs int64, leaseMs int64) {
    ticker := time.NewTicker(time.Duration(heartbeatMs) * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            _ = h.repo.RenewLease(ctx, taskID, lockOwner, time.Now().Add(time.Duration(leaseMs)*time.Millisecond).UnixMilli())
        }
    }
}
```

---

## 11. 单 datasource 执行骨架

## 11.1 Runner 主流程

```go
package worker

import (
    "context"
)

type Runner struct {
    svcCtx *svc.ServiceContext
}

func NewRunner(svcCtx *svc.ServiceContext) *Runner {
    return &Runner{svcCtx: svcCtx}
}

func (r *Runner) RunDatasourceTask(ctx context.Context, taskID string) error {
    task, err := r.repo.FindOne(ctx, taskID)
    if err != nil {
        return err
    }

    hbCtx, cancel := context.WithCancel(ctx)
    defer cancel()
    go r.heartbeat.Renew(hbCtx, task.ID, task.LockOwner, r.cfg.HeartbeatMs, r.cfg.LeaseMs)

    wm, err := r.watermarkService.GetOrInit(ctx, task.DatasourceID)
    if err != nil {
        return r.failOrRetry(ctx, task, err)
    }

    cursor := BuildCursorFromWatermark(wm)
    stats := NewSyncStats()

    for {
        sourcePage, nextCursor, err := r.mdlReader.ListPage(ctx, task.DatasourceID, cursor, r.cfg.PageSize)
        if err != nil {
            return r.failOrRetry(ctx, task, err)
        }
        if len(sourcePage) == 0 {
            break
        }

        localViews, localFields, err := r.localLoader.BatchLoad(ctx, sourcePage)
        if err != nil {
            return r.failOrRetry(ctx, task, err)
        }

        diffRes, err := r.diffEngine.ComparePage(ctx, sourcePage, localViews, localFields, r.cfg.DiffWorkers)
        if err != nil {
            return r.failOrRetry(ctx, task, err)
        }

        applyRes, err := r.applyService.ApplyPage(ctx, diffRes)
        if err != nil {
            return r.failOrRetry(ctx, task, err)
        }

        stats.Merge(applyRes.Stats)
        cursor = applyRes.LastSafeCursor

        if err := r.repo.UpdateProgress(ctx, task.ID, stats, cursor); err != nil {
            return r.failOrRetry(ctx, task, err)
        }

        _ = nextCursor // 如采用精确游标推进，可视设计保留
    }

    if err := r.watermarkService.Advance(ctx, task.DatasourceID, task.ID, cursor, task.JobType); err != nil {
        return r.failOrRetry(ctx, task, err)
    }

    return r.repo.MarkSuccess(ctx, task.ID, stats, cursor)
}
```

---

## 11.2 `mdlReader` 骨架

当前源表 `t_data_view` 没有 `f_change_time` 字段，因此逻辑上定义：

```text
change_time = GREATEST(f_create_time, f_update_time, f_delete_time)
```

查询建议按：

```text
ORDER BY change_time ASC, f_view_id ASC
```

伪代码：

```go
func (r *MdlReaderService) ListPage(ctx context.Context, datasourceID string, cursor Cursor, pageSize int) ([]SourceView, Cursor, error) {
    rows, err := r.repo.QuerySourceViews(ctx, datasourceID, cursor.ChangeTime, cursor.ViewID, pageSize)
    if err != nil {
        return nil, Cursor{}, err
    }

    views := make([]SourceView, 0, len(rows))
    for _, row := range rows {
        views = append(views, NormalizeSourceView(row))
    }

    nextCursor := cursor
    if len(views) > 0 {
        last := views[len(views)-1]
        nextCursor = Cursor{ChangeTime: last.ChangeTime, ViewID: last.ViewID}
    }
    return views, nextCursor, nil
}
```

---

## 12. `diff_engine` 骨架

## 12.1 page 内并行 diff

策略：

- datasource 内 page 顺序执行
- page 内 view 级比较并行

```go
func (e *DiffEngine) ComparePage(ctx context.Context, source []SourceView, localViews map[string]LocalView, localFields map[string][]LocalField, workers int) (DiffResult, error) {
    jobs := make(chan SourceView)
    results := make(chan SingleViewDiff)

    for i := 0; i < workers; i++ {
        go func() {
            for sv := range jobs {
                results <- e.compareOne(sv, localViews, localFields)
            }
        }()
    }

    go func() {
        defer close(jobs)
        for _, sv := range source {
            jobs <- sv
        }
    }()

    out := NewDiffResult()
    for i := 0; i < len(source); i++ {
        out.Merge(<-results)
    }
    return out, nil
}
```

---

## 12.2 `compareOne` 核心规则

### 视图级

- 优先匹配 `source_view_id = f_view_id`
- fallback：`datasource_id + technical_name`

### 字段级

- 优先匹配 `original_name`
- fallback：`name`

### 结果分类

- `NEW`
- `UPDATE`
- `DELETE`
- `UNCHANGED`

---

## 13. `apply_service` 骨架

```go
func (a *ApplyService) ApplyPage(ctx context.Context, diff DiffResult) (ApplyResult, error) {
    stats := NewSyncStats()
    lastSafe := Cursor{}

    err := a.withTx(ctx, func(tx Tx) error {
        for _, d := range diff.News {
            if err := a.localRepo.CreateView(tx, d); err != nil {
                return err
            }
            stats.ViewCreated++
            lastSafe = d.Cursor
        }

        for _, d := range diff.Updates {
            if err := a.localRepo.UpdateView(tx, d); err != nil {
                return err
            }
            stats.ViewUpdated++
            lastSafe = d.Cursor
        }

        for _, d := range diff.Deletes {
            if err := a.localRepo.MarkViewDeleted(tx, d); err != nil {
                return err
            }
            stats.ViewDeleted++
            lastSafe = d.Cursor
        }
        return nil
    })
    if err != nil {
        return ApplyResult{}, err
    }

    return ApplyResult{Stats: stats, LastSafeCursor: lastSafe}, nil
}
```

说明：

- 第一版建议“小批事务 apply”
- watermark 只推进到 `LastSafeCursor`

---

## 14. 失败与重试骨架

```go
func (r *Runner) failOrRetry(ctx context.Context, task SyncTask, err error) error {
    retryable := IsRetryable(err)
    backoff := CalcBackoff(task.AttemptNo, r.svcCtx.Config.Retry.BackoffMs)

    resultJSON := BuildResultJSON(task, err, retryable)

    if retryable && task.AttemptNo < task.MaxAttempt {
        return r.repo.MarkRetryWaiting(ctx, task.ID, backoff, resultJSON, err)
    }
    return r.repo.MarkFailed(ctx, task.ID, resultJSON, err)
}
```

---

## 15. 核心 SQL 对应实现点

建议直接把文档里的 SQL 封装到 repo：

### `sync_task_repo`
- `InsertWithDedupe()`
- `ListRunnable()`
- `TryAcquire()`
- `RenewLease()`
- `ExistsOtherRunningSameDatasource()`
- `UpdateProgress()`
- `MarkSuccess()`
- `MarkRetryWaiting()`
- `MarkFailed()`

### `watermark_repo`
- `GetOrInit()`
- `Advance()`
- `Upsert()`

这些 SQL 已在 2 表结构文档中给出推荐模板。fileciteturn55file0

---

## 16. 推荐命令行参数规范

## 16.1 `create-job`

```bash
./sync-worker create-job \
  --job-type=incremental \
  --scope=all-active \
  --trigger-source=schedule \
  --schedule-slot=1712647500000
```

支持参数：

- `--job-type`
- `--scope`
- `--datasource-ids`
- `--shard-total`
- `--shard-index`
- `--trigger-source`
- `--schedule-slot`
- `--requested-by`
- `--requested-name`
- `--reason`

---

## 16.2 `run-worker`

```bash
./sync-worker run-worker
```

可选参数：

- `--worker-id`
- `--poll-interval-ms`
- `--pick-limit`
- `--max-parallelism`

---

## 17. 最终建议

### 当前最适合落地的方式

- 一个工程：`sync-worker`
- 两个命令：`create-job` / `run-worker`
- 两张表：`sync_task` / `sync_watermark`
- 调度优先走 K8S CronJob
- 手工补偿走 one-off 命令

### 研发实现顺序

1. 先把 2 张表 model 生成
2. 先做 `create-job`
3. 再做 `run-worker` 常驻轮询
4. 再做 `mdlReader + diffEngine + applyService`
5. 最后接入 K8S CronJob

