# PRD｜form-view 同步任务系统

## 1. 文档信息

- 文档名称：form-view 同步任务系统 PRD
- 版本：V1.0
- 状态：评审稿
- 面向对象：产品、研发、测试、运维
- 相关范围：`mdl-data-model` → `data-view` 的视图与字段同步能力

---

## 2. 背景与问题

当前 `data-view` 需要将 `mdl-data-model` 中的数据视图定义同步到本地 `form_view / form_view_field` 模型，以支撑应用层的展示、治理、检索与后续业务能力。

现阶段的主要问题有：

1. 缺少一套统一的任务化同步机制。
2. 定时触发、手工补偿、失败重试的链路不统一。
3. 同步过程对“增量进度”的定义不清晰，容易出现重复同步、漏同步或删除感知不完整。
4. 同一 datasource 的并发执行、失败恢复、状态可追踪能力不足。
5. 方案不能过重，当前阶段不适合一开始就引入过多任务表和平台化组件。

因此，需要建设一套 **轻量但可扩展** 的同步任务系统，满足：

- K8S CronJob 优先触发
- 手工补偿可支持
- datasource 级执行
- 失败可重试
- watermark 可续跑
- 架构简单，可快速落地

---

## 3. 产品目标

### 3.1 核心目标

建设一套以 **sync-worker 二合一模式** 为核心的同步任务系统，实现：

1. 支持定时自动创建同步任务。
2. 支持手工创建补偿任务。
3. 支持 `incremental / reconcile / full` 三类同步模式。
4. 支持 datasource 粒度执行、重试与状态追踪。
5. 支持 watermark 长期进度管理，保证增量同步连续、可恢复。
6. 支持将 `t_data_view` 的变化正确同步到本地 `form_view / form_view_field`。

### 3.2 阶段目标

V1 阶段的重点不是做成完整任务平台，而是：

- 先把同步链路跑通
- 先把状态模型稳定下来
- 先把 CronJob 与 worker 跑稳
- 先把失败重试与 watermark 逻辑做正确

---

## 4. 非目标

当前版本不包含以下内容：

1. 不建设独立的 `sync-admin-api` 管理服务。
2. 不建设完整的任务管理前端页面。
3. 不引入独立的 outbox / projector 体系。
4. 不引入独立的 job 主表 + job_ds 子表双层任务模型。
5. 不在 V1 中支持所有复杂视图类型的自动同步策略定制。
6. 不以“实时 CDC”方式替代当前定时+增量模式。

---

## 5. 目标用户

### 5.1 直接用户

1. 平台运维 / SRE
   - 关注定时任务是否生成、是否执行、是否失败。
2. 研发工程师
   - 关注同步逻辑是否正确、是否可追踪、是否可恢复。
3. 数据平台管理员
   - 关注指定 datasource 的补偿同步与全量修复。

### 5.2 间接受益方

1. `data-view` 相关业务模块
2. 后续依赖 `form_view / form_view_field` 的治理与应用能力
3. 搜索、检索、标准绑定等下游能力

---

## 6. 术语定义

### 6.1 Source
源系统，指 `mdl-data-model` 侧的 `t_data_view`。

### 6.2 Local
本地目标模型，指 `data-view` 侧的 `form_view / form_view_field`。

### 6.3 Task
一次 datasource 粒度的同步执行任务。

### 6.4 Batch
同一轮触发生成的一组 datasource 同步任务，共享 `batch_id`。

### 6.5 Watermark
某个 datasource 已安全同步完成的长期进度。

### 6.6 change_time
当前源表下的统一逻辑变更时间：

```text
change_time = GREATEST(f_create_time, f_update_time, f_delete_time)
```

---

## 7. 方案范围与边界

### 7.1 触发方式

支持两类触发：

1. **K8S CronJob**：主路径，优先使用。
2. **手工触发**：补偿路径，用于指定 datasource 或特殊场景。

### 7.2 服务形态

采用一个工程、两个命令模式：

- `sync-worker create-job`
- `sync-worker run-worker`

### 7.3 数据模型

仅保留 2 张核心表：

1. `t_form_view_sync_task`
2. `t_form_view_sync_watermark`

### 7.4 执行粒度

以 **datasource** 为最小执行单元。

---

## 8. 业务流程概览

```text
K8S CronJob / 手工触发
    -> create-job
    -> 按 datasource 生成 sync_task（共享 batch_id）
    -> run-worker 常驻轮询 sync_task
    -> 抢占任务
    -> 读取 watermark
    -> 读取 t_data_view
    -> 与本地 form_view / form_view_field 做 diff
    -> apply 到本地
    -> 更新任务结果
    -> 推进 watermark
    -> 结束 / 重试
```

---

## 9. 功能设计

# 9.1 功能一：任务创建

## 9.1.1 功能描述

支持通过两种方式生成同步任务：

1. K8S CronJob 定时触发
2. 手工命令触发

任务创建后，以 datasource 粒度落到 `t_form_view_sync_task` 中。

## 9.1.2 输入参数

至少支持：

- `job_type`：`incremental | reconcile | full`
- `scope`：`all-active | datasource-list | shard`
- `datasource_ids`：指定 datasource 列表
- `shard_total`
- `shard_index`
- `trigger_source`：`manual | schedule | retry | system`
- `schedule_slot`
- `requested_by`
- `requested_name`
- `reason`

## 9.1.3 创建规则

### 定时触发
- 由 K8S CronJob 执行 `create-job`
- 根据范围解析需要生成任务的 datasource 集合
- 同一轮创建的任务共享 `batch_id`

### 手工触发
- 支持单个或多个 datasource
- 支持 `incremental / reconcile / full`
- `trigger_source = manual`

## 9.1.4 去重规则

使用：

```text
form_view_sync:{job_type}:{datasource_id}:{schedule_slot}
```

作为 `dedupe_key`。

要求：

1. 同一个 `schedule_slot` 下，不重复为相同 datasource + 相同 job_type 创建任务。
2. 如果已存在 active 任务（`pending / running / retry_waiting`），默认不重复创建。

## 9.1.5 输出

返回：

- `batch_id`
- 本次成功创建数量
- 重复跳过数量
- datasource 明细摘要

---

# 9.2 功能二：任务执行

## 9.2.1 功能描述

`run-worker` 作为常驻后台执行器，持续扫描并消费 `sync_task`。

## 9.2.2 主循环逻辑

1. 扫描可执行任务：
   - `status in (pending, retry_waiting)`
   - `next_run_at <= now`
2. 抢占任务执行权
3. 执行 datasource 同步
4. 更新状态、统计、watermark
5. 继续下一轮轮询

## 9.2.3 抢占与租约

任务抢占成功后：

- `status = running`
- 更新 `lock_owner`
- 更新 `lock_until`
- 更新 `heartbeat_at`
- `attempt_no + 1`
- `fencing_token + 1`

## 9.2.4 同 datasource 并发控制

同一时刻，同一个 datasource 只允许一个有效运行中的任务。

如果同 datasource 已存在有效 `running` 任务，则当前任务不执行，回退为：

- `retry_waiting`
- 并设置新的 `next_run_at`

---

# 9.3 功能三：同步执行

## 9.3.1 源数据读取

源表为：`t_data_view`

逻辑变更时间定义为：

```text
change_time = GREATEST(f_create_time, f_update_time, f_delete_time)
```

读取顺序固定为：

```text
ORDER BY change_time ASC, f_view_id ASC
```

## 9.3.2 watermark 使用方式

对于每个 datasource：

- 从 `t_form_view_sync_watermark` 读取当前 `watermark`
- 构造精确游标：
  - `change_time`
  - `view_id`
- 以此作为下一轮增量同步起点

## 9.3.3 本地比较逻辑

同步执行时，需要把源视图与本地对象做两层比较：

### 第一层：视图级比较
源：`t_data_view`
本地：`form_view`

核心比较字段：

- `source_view_id = f_view_id`
- `datasource_id = f_data_source_id`
- `technical_name = f_technical_name`
- `view_name = f_view_name`
- `comment = f_comment`
- `type = f_type`
- `query_type = f_query_type`
- `meta_table_name = f_meta_table_name`
- `status`
- `delete_time`

### 第二层：字段级比较
源：`f_fields` 展开
本地：`form_view_field`

核心比较字段：

- `field_key`
- `display_name`
- `original_name`
- `type`
- `order/index`
- 其他必要属性

## 9.3.4 比较结果分类

每个视图最终归为：

- `NEW`
- `UPDATE`
- `DELETE`
- `UNCHANGED`

字段层面归为：

- `field_created`
- `field_updated`
- `field_deleted`
- `suspected_rename`

## 9.3.5 性能策略

采用：

- datasource 之间可并行
- 单 datasource 内 page 顺序执行
- 单 page 内 diff 可使用 Go 协程 worker pool 并行
- apply 阶段采用小批事务

目标是降低：

- N+1 SQL
- 逐条比较开销
- watermark 推进乱序风险

---

# 9.4 功能四：watermark 推进

## 9.4.1 watermark 的职责

只记录：

- datasource 的长期安全进度
- 最近一次成功任务
- 最近一次 full / reconcile 成功时间

不记录：

- 本次成功多少条
- 本次失败多少条
- 失败原因

这些由 `sync_task` 记录。

## 9.4.2 推进原则

watermark 只能推进到：

> 连续成功前缀的最后一个位置

不能推进到：

- 本次扫描到的最大位置
- 失败洞之后的位置

## 9.4.3 精确游标

watermark 采用：

- `f_watermark_value`
- `f_watermark_cursor_json`

其中 `cursor_json` 结构建议固定为：

```json
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

---

# 9.5 功能五：失败与重试

## 9.5.1 可重试失败

例如：

- 读源失败
- 本地 DB 短暂故障
- 网络超时
- 临时依赖不可用

处理方式：

- `status = retry_waiting`
- `attempt_no + 1`
- `next_run_at = now + backoff`

## 9.5.2 不可重试失败

例如：

- 参数错误
- schema 不兼容
- 数据严重非法

处理方式：

- `status = failed`

## 9.5.3 失败信息记录

建议通过 `f_result_json` 记录：

- `success_count`
- `failed_count`
- `failed_view_ids_sample`
- `last_safe_watermark`
- `last_safe_cursor`
- `fail_stage`
- `fail_reason`
- `retryable`

---

# 9.6 功能六：批次管理

虽然没有主任务表，但仍需支持“这一轮任务”的概念。

通过：

- `f_batch_id`

实现批次归属。

一个 `batch_id` 下可包含多个 datasource 任务。

典型用途：

- 观察本轮 CronJob 创建了多少任务
- 追踪这一轮 incremental / reconcile 的整体情况
- 后续如需补管理页面，可按 `batch_id` 聚合展示

---

## 10. 典型业务场景

### 场景 1：定时 incremental

1. CronJob 每 5 分钟触发一次 `create-job`
2. 按全部活跃 datasource 创建任务
3. `run-worker` 常驻执行
4. 每个 datasource 各自推进 watermark

### 场景 2：夜间 reconcile

1. CronJob 每天凌晨触发一次 reconcile
2. 可按 shard 切分 datasource 范围
3. worker 逐个 datasource 执行对账型同步

### 场景 3：手工 full 修复

1. 运维执行 `create-job --job-type=full --datasource-ids=...`
2. 指定 datasource 生成任务
3. worker 执行全量同步

### 场景 4：部分失败后自动重试

1. 某 datasource 执行中部分成功、部分失败
2. 任务进入 `retry_waiting`
3. watermark 只推进到 `last_safe_cursor`
4. 下次从该安全位置继续

---

## 11. 关键时序

### 11.1 CronJob 创建任务时序

```text
CronJob
  -> 创建 Pod
  -> 执行 sync-worker create-job
  -> 解析 datasource 范围
  -> 生成 batch_id / schedule_slot / dedupe_key
  -> 幂等写入 t_form_view_sync_task
  -> 结束
```

### 11.2 worker 执行任务时序

```text
run-worker 常驻轮询
  -> 扫描 runnable task
  -> 抢占任务
  -> 检查 datasource 并发冲突
  -> 读取 watermark
  -> 分页读取 t_data_view
  -> 批量加载本地 form_view / form_view_field
  -> diff + apply
  -> 更新 task 统计与 cursor
  -> 推进 watermark
  -> success / retry_waiting / failed
```

---

## 12. 关键配置建议

### 12.1 CronJob 侧

- `concurrencyPolicy: Forbid`
- 显式 `timeZone`
- `startingDeadlineSeconds` 合理设置
- 历史 Job 保留低值

### 12.2 worker 侧

- `PollIntervalMs = 2000 ~ 5000`
- `LeaseMs = 90000`
- `HeartbeatMs = 20000`
- `PageSize = 200 ~ 500`
- `MaxDatasourceParallelism = 4 ~ 8`
- `DiffWorkers = min(8, CPU*2)`

---

## 13. 可观测性要求

至少要求记录：

1. 每轮 `create-job` 创建任务数量 / 重复跳过数量
2. `run-worker` 当前轮询频率与可执行任务数
3. 每个 datasource 任务的：
   - 开始 / 结束时间
   - 状态
   - 重试次数
   - view / field 统计
   - watermark 前后值
4. 最近失败原因与失败样本
5. watermark 推进轨迹

---

## 14. 成功标准

### 14.1 功能成功标准

1. CronJob 能按计划创建任务
2. 手工触发可创建指定 datasource 任务
3. worker 能正确消费任务并推进状态
4. datasource 级 watermark 能正确推进
5. 失败任务能重试且不漏同步
6. 相同调度槽位不重复创建相同任务

### 14.2 质量成功标准

1. 无同 datasource 并发写入冲突
2. watermark 不越过失败洞
3. 大多数同步执行可在预期时间内完成
4. 重复创建被正确去重
5. worker 异常退出后可恢复执行

---

## 15. 风险与开放问题

### 15.1 当前风险

1. 当前源表没有物理 `f_change_time` 字段，查询依赖 `GREATEST(...)`，索引利用率一般。
2. 当前版本没有独立 job 主表，批次管理依赖 `batch_id`，聚合展示能力相对较弱。
3. 当前版本没有独立 outbox，若未来要严格保障搜索索引最终一致，需要额外扩展。

### 15.2 建议后续优化

1. 源表补充 `f_change_time` 字段。
2. 视需要补充轻量管理 API。
3. 视需要引入 outbox / projector。
4. 视 datasource 数量增长，引入 shard 调度策略。

---

## 16. 分阶段实施建议

### Phase 1

- 落 2 张表
- 实现 `create-job`
- 实现 `run-worker`
- 跑通单 datasource incremental

### Phase 2

- 接入 K8S CronJob
- 跑通 incremental + reconcile
- 完成失败重试与基本可观测

### Phase 3

- 手工补偿 full
- shard 调度
- 平台化管理与高级治理能力

---

## 17. 最终产品决策

当前版本正式采用：

1. **2 张表模型**
   - `t_form_view_sync_task`
   - `t_form_view_sync_watermark`
2. **sync-worker 二合一模式**
   - `create-job`
   - `run-worker`
3. **K8S CronJob 优先创建任务**
4. **datasource 粒度执行**
5. **batch_id 表达批次**
6. **schedule_slot + dedupe_key 表达定时幂等**
7. **watermark 表达长期安全同步进度**


