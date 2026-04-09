# 2 张表版最终 SQL（`sync_task` + `watermark`）

## 1. 设计目标

当前阶段同步能力的目标是：

* 不引入 `job` 主表
* 不引入 `job_ds` 子表
* 只保留两张核心表：

  * `t_form_view_sync_task`
  * `t_form_view_sync_watermark`
* 任务执行粒度为 **datasource**
* 批次概念通过 `batch_id` 表达
* 定时调度去重通过 `schedule_slot + dedupe_key` 表达
* watermark 独立记录 datasource 的长期同步进度

---

## 2. 表设计概览

### 2.1 `t_form_view_sync_task`

这张表表示：

> 一个 datasource 的一次同步执行任务

它同时承担以下职责：

* 调度任务
* 执行状态
* 重试控制
* 分布式执行租约
* watermark 前后值记录
* 执行结果与统计

### 2.2 `t_form_view_sync_watermark`

这张表表示：

> 一个 datasource 的长期同步进度

它承担：

* 增量同步起点记录
* 精确游标记录
* 最近一次成功 full / reconcile 信息记录

---

## 3. 最终 SQL

## 3.1 同步任务表

```sql
CREATE TABLE IF NOT EXISTS `t_form_view_sync_task` (
  `f_id`                    varchar(64)      NOT NULL COMMENT '任务ID',
  `f_batch_id`              varchar(64)      NOT NULL COMMENT '批次ID，同一轮触发生成的任务共享',
  `f_datasource_id`         varchar(64)      NOT NULL COMMENT '数据源ID',
  `f_datasource_name`       varchar(128)     NULL COMMENT '数据源名称(冗余)',

  `f_job_type`              varchar(32)      NOT NULL COMMENT '任务类型: incremental|reconcile|full',
  `f_trigger_source`        varchar(32)      NOT NULL DEFAULT 'manual' COMMENT '触发来源: manual|schedule|retry|system',
  `f_schedule_slot`         bigint           NULL COMMENT '调度时间槽(ms)，仅 schedule 场景使用',
  `f_dedupe_key`            varchar(255)     NOT NULL COMMENT '去重键，建议 job_type+datasource_id+schedule_slot',

  `f_status`                varchar(32)      NOT NULL DEFAULT 'pending' COMMENT '状态: pending|running|retry_waiting|success|failed|cancelled',

  `f_requested_by`          varchar(64)      NULL COMMENT '触发人ID',
  `f_requested_name`        varchar(128)     NULL COMMENT '触发人名称',
  `f_reason`                varchar(512)     NULL COMMENT '触发原因/备注',

  `f_attempt_no`            int unsigned     NOT NULL DEFAULT 0 COMMENT '当前尝试次数',
  `f_max_attempt`           int unsigned     NOT NULL DEFAULT 3 COMMENT '最大自动重试次数',
  `f_next_run_at`           bigint           NOT NULL COMMENT '下次可执行时间(ms)',

  `f_lock_owner`            varchar(128)     NULL COMMENT '当前执行worker实例ID',
  `f_lock_until`            bigint           NULL COMMENT '执行租约过期时间(ms)',
  `f_heartbeat_at`          bigint           NULL COMMENT '最近心跳时间(ms)',
  `f_fencing_token`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT 'fencing token',

  `f_watermark_type`        varchar(32)      NOT NULL DEFAULT 'change_time' COMMENT '水位类型，当前固定为 change_time',
  `f_watermark_before`      bigint unsigned  NOT NULL DEFAULT 0 COMMENT '执行前水位',
  `f_watermark_after`       bigint unsigned  NOT NULL DEFAULT 0 COMMENT '本次安全提交后的水位',
  `f_cursor_json`           longtext         NULL COMMENT '本次安全游标(JSON)，如 {\"change_time\":1710000000000,\"view_id\":\"dv_xxx\"}',

  `f_page_count`            int unsigned     NOT NULL DEFAULT 0 COMMENT '读取页数',
  `f_batch_count`           int unsigned     NOT NULL DEFAULT 0 COMMENT '处理批次数',

  `f_view_created`          int unsigned     NOT NULL DEFAULT 0 COMMENT '新增视图数',
  `f_view_updated`          int unsigned     NOT NULL DEFAULT 0 COMMENT '更新视图数',
  `f_view_deleted`          int unsigned     NOT NULL DEFAULT 0 COMMENT '删除/下线视图数',

  `f_field_created`         int unsigned     NOT NULL DEFAULT 0 COMMENT '新增字段数',
  `f_field_updated`         int unsigned     NOT NULL DEFAULT 0 COMMENT '更新字段数',
  `f_field_deleted`         int unsigned     NOT NULL DEFAULT 0 COMMENT '删除字段数',

  `f_hash_skipped`          int unsigned     NOT NULL DEFAULT 0 COMMENT '因 hash 相同跳过的对象数',
  `f_suspected_rename`      int unsigned     NOT NULL DEFAULT 0 COMMENT '疑似字段 rename 数',
  `f_audit_revoked`         int unsigned     NOT NULL DEFAULT 0 COMMENT '触发撤销审核次数',

  `f_started_at`            bigint           NULL COMMENT '开始时间(ms)',
  `f_finished_at`           bigint           NULL COMMENT '结束时间(ms)',

  `f_last_error_code`       varchar(64)      NULL COMMENT '最后错误码',
  `f_last_error_msg`        varchar(2000)    NULL COMMENT '最后错误信息',
  `f_result_json`           longtext         NULL COMMENT '执行结果详情(JSON)',

  `f_version`               bigint unsigned  NOT NULL DEFAULT 0 COMMENT '乐观锁版本',
  `f_created_at`            bigint           NOT NULL COMMENT '创建时间(ms)',
  `f_updated_at`            bigint           NOT NULL COMMENT '更新时间(ms)',

  PRIMARY KEY (`f_id`),
  UNIQUE KEY `uk_fvst_dedupe_key` (`f_dedupe_key`),

  KEY `idx_fvst_pick` (`f_status`, `f_next_run_at`, `f_lock_until`),
  KEY `idx_fvst_ds_status` (`f_datasource_id`, `f_status`),
  KEY `idx_fvst_batch_id` (`f_batch_id`),
  KEY `idx_fvst_schedule_slot` (`f_schedule_slot`),
  KEY `idx_fvst_lock_owner` (`f_lock_owner`, `f_lock_until`),
  KEY `idx_fvst_created_at` (`f_created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='form_view 同步任务表（datasource 粒度）';
```

---

## 3.2 watermark 表

```sql
CREATE TABLE IF NOT EXISTS `t_form_view_sync_watermark` (
  `f_datasource_id`           varchar(64)      NOT NULL COMMENT '数据源ID',
  `f_watermark_type`          varchar(32)      NOT NULL DEFAULT 'change_time' COMMENT '水位类型，当前固定为 change_time',
  `f_watermark_value`         bigint unsigned  NOT NULL DEFAULT 0 COMMENT '当前安全水位值，对应 change_time',
  `f_watermark_cursor_json`   longtext         NULL COMMENT '精确游标(JSON)，如 {"change_time":1710000000000,"view_id":"dv_xxx"}',

  `f_last_success_task_id`    varchar(64)      NULL COMMENT '最近一次成功任务ID',
  `f_last_full_sync_at`       bigint           NULL COMMENT '最近一次 full 成功时间(ms)',
  `f_last_reconcile_at`       bigint           NULL COMMENT '最近一次 reconcile 成功时间(ms)',

  `f_updated_at`              bigint           NOT NULL COMMENT '更新时间(ms)',

  PRIMARY KEY (`f_datasource_id`, `f_watermark_type`),
  KEY `idx_fvsw_updated_at` (`f_updated_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='form_view 同步 watermark 表';
```

---

## 4. 字段设计说明

## 4.1 `t_form_view_sync_task`

### 基础标识字段

* `f_id`：任务 ID
* `f_batch_id`：批次 ID，同一轮触发共享
* `f_datasource_id`：数据源 ID
* `f_datasource_name`：数据源名称冗余

### 调度与类型字段

* `f_job_type`：任务类型

  * `incremental`
  * `reconcile`
  * `full`
* `f_trigger_source`：触发来源

  * `manual`
  * `schedule`
  * `retry`
  * `system`
* `f_schedule_slot`：cron 时间槽
* `f_dedupe_key`：任务去重键

### 状态与重试字段

* `f_status`

  * `pending`
  * `running`
  * `retry_waiting`
  * `success`
  * `failed`
  * `cancelled`
* `f_attempt_no`：已尝试次数
* `f_max_attempt`：最大自动重试次数
* `f_next_run_at`：下次可执行时间

### 执行租约字段

* `f_lock_owner`
* `f_lock_until`
* `f_heartbeat_at`
* `f_fencing_token`

### watermark 相关字段

* `f_watermark_type`
* `f_watermark_before`
* `f_watermark_after`
* `f_cursor_json`

### 统计字段

* `f_page_count`
* `f_batch_count`
* `f_view_created`
* `f_view_updated`
* `f_view_deleted`
* `f_field_created`
* `f_field_updated`
* `f_field_deleted`
* `f_hash_skipped`
* `f_suspected_rename`
* `f_audit_revoked`

### 错误与结果字段

* `f_last_error_code`
* `f_last_error_msg`
* `f_result_json`

---

## 4.2 `t_form_view_sync_watermark`

### 核心字段

* `f_datasource_id`
* `f_watermark_type`
* `f_watermark_value`
* `f_watermark_cursor_json`

### 最近成功信息

* `f_last_success_task_id`
* `f_last_full_sync_at`
* `f_last_reconcile_at`

---

## 5. 推荐状态值定义

## 5.1 task 状态

```text
pending
running
retry_waiting
success
failed
cancelled
```

---

## 6. 推荐 JSON 字段格式

## 6.1 `f_cursor_json`

```json
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

---

## 6.2 `f_result_json`

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
  "fail_reason": "parse f_fields error",
  "retryable": true
}
```

---

## 7. 定时调度去重规则

## 7.1 dedupe_key 建议格式

```text
form_view_sync:{job_type}:{datasource_id}:{schedule_slot}
```

例如：

```text
form_view_sync:incremental:ds_1001:1712647500000
```

---

## 7.2 规则

### 同一个 schedule_slot

不允许重复生成相同 datasource、相同 job_type 的 task。

### 同 datasource 已有 active incremental

如果已存在以下状态之一：

* `pending`
* `running`
* `retry_waiting`

则新的 cron incremental 默认不重复生成。

### reconcile / full

也建议同 datasource 下不重复生成 active 任务。

---

## 8. 常用 SQL 示例

## 8.1 定时任务幂等创建

```sql
INSERT INTO `t_form_view_sync_task` (
  `f_id`,
  `f_batch_id`,
  `f_datasource_id`,
  `f_datasource_name`,
  `f_job_type`,
  `f_trigger_source`,
  `f_schedule_slot`,
  `f_dedupe_key`,
  `f_status`,
  `f_attempt_no`,
  `f_max_attempt`,
  `f_next_run_at`,
  `f_watermark_type`,
  `f_watermark_before`,
  `f_watermark_after`,
  `f_created_at`,
  `f_updated_at`
) VALUES (
  ?, ?, ?, ?, ?, 'schedule', ?, ?, 'pending', 0, 3, ?, 'change_time', 0, 0, ?, ?
)
ON DUPLICATE KEY UPDATE
  `f_updated_at` = VALUES(`f_updated_at`);
```

---

## 8.2 抢占执行权

```sql
UPDATE `t_form_view_sync_task`
SET
  `f_status` = 'running',
  `f_lock_owner` = ?,
  `f_lock_until` = ?,
  `f_heartbeat_at` = ?,
  `f_started_at` = IFNULL(`f_started_at`, ?),
  `f_attempt_no` = `f_attempt_no` + 1,
  `f_fencing_token` = `f_fencing_token` + 1,
  `f_updated_at` = ?
WHERE `f_id` = ?
  AND `f_status` IN ('pending', 'retry_waiting')
  AND (`f_lock_until` IS NULL OR `f_lock_until` < ?);
```

---

## 8.3 心跳续租

```sql
UPDATE `t_form_view_sync_task`
SET
  `f_lock_until` = ?,
  `f_heartbeat_at` = ?,
  `f_updated_at` = ?
WHERE `f_id` = ?
  AND `f_status` = 'running'
  AND `f_lock_owner` = ?;
```

---

## 8.4 记录成功

```sql
UPDATE `t_form_view_sync_task`
SET
  `f_status` = 'success',
  `f_watermark_after` = ?,
  `f_cursor_json` = ?,
  `f_page_count` = ?,
  `f_batch_count` = ?,
  `f_view_created` = ?,
  `f_view_updated` = ?,
  `f_view_deleted` = ?,
  `f_field_created` = ?,
  `f_field_updated` = ?,
  `f_field_deleted` = ?,
  `f_hash_skipped` = ?,
  `f_suspected_rename` = ?,
  `f_audit_revoked` = ?,
  `f_result_json` = ?,
  `f_finished_at` = ?,
  `f_updated_at` = ?
WHERE `f_id` = ?;
```

---

## 8.5 记录可重试失败

```sql
UPDATE `t_form_view_sync_task`
SET
  `f_status` = 'retry_waiting',
  `f_next_run_at` = ?,
  `f_watermark_after` = ?,
  `f_cursor_json` = ?,
  `f_page_count` = ?,
  `f_batch_count` = ?,
  `f_view_created` = ?,
  `f_view_updated` = ?,
  `f_view_deleted` = ?,
  `f_field_created` = ?,
  `f_field_updated` = ?,
  `f_field_deleted` = ?,
  `f_hash_skipped` = ?,
  `f_suspected_rename` = ?,
  `f_audit_revoked` = ?,
  `f_last_error_code` = ?,
  `f_last_error_msg` = ?,
  `f_result_json` = ?,
  `f_updated_at` = ?
WHERE `f_id` = ?;
```

---

## 8.6 记录最终失败

```sql
UPDATE `t_form_view_sync_task`
SET
  `f_status` = 'failed',
  `f_watermark_after` = ?,
  `f_cursor_json` = ?,
  `f_page_count` = ?,
  `f_batch_count` = ?,
  `f_view_created` = ?,
  `f_view_updated` = ?,
  `f_view_deleted` = ?,
  `f_field_created` = ?,
  `f_field_updated` = ?,
  `f_field_deleted` = ?,
  `f_hash_skipped` = ?,
  `f_suspected_rename` = ?,
  `f_audit_revoked` = ?,
  `f_last_error_code` = ?,
  `f_last_error_msg` = ?,
  `f_result_json` = ?,
  `f_finished_at` = ?,
  `f_updated_at` = ?
WHERE `f_id` = ?;
```

---

## 8.7 查询 watermark

```sql
SELECT
  `f_datasource_id`,
  `f_watermark_type`,
  `f_watermark_value`,
  `f_watermark_cursor_json`,
  `f_last_success_task_id`,
  `f_last_full_sync_at`,
  `f_last_reconcile_at`,
  `f_updated_at`
FROM `t_form_view_sync_watermark`
WHERE `f_datasource_id` = ?
  AND `f_watermark_type` = 'change_time';
```

---

## 8.8 更新 watermark

```sql
UPDATE `t_form_view_sync_watermark`
SET
  `f_watermark_value` = ?,
  `f_watermark_cursor_json` = ?,
  `f_last_success_task_id` = ?,
  `f_last_full_sync_at` = CASE WHEN ? = 'full' THEN ? ELSE `f_last_full_sync_at` END,
  `f_last_reconcile_at` = CASE WHEN ? = 'reconcile' THEN ? ELSE `f_last_reconcile_at` END,
  `f_updated_at` = ?
WHERE `f_datasource_id` = ?
  AND `f_watermark_type` = 'change_time';
```

---

## 8.9 watermark 不存在时 upsert

```sql
INSERT INTO `t_form_view_sync_watermark` (
  `f_datasource_id`,
  `f_watermark_type`,
  `f_watermark_value`,
  `f_watermark_cursor_json`,
  `f_last_success_task_id`,
  `f_last_full_sync_at`,
  `f_last_reconcile_at`,
  `f_updated_at`
) VALUES (
  ?,
  'change_time',
  ?,
  ?,
  ?,
  CASE WHEN ? = 'full' THEN ? ELSE NULL END,
  CASE WHEN ? = 'reconcile' THEN ? ELSE NULL END,
  ?
)
ON DUPLICATE KEY UPDATE
  `f_watermark_value` = VALUES(`f_watermark_value`),
  `f_watermark_cursor_json` = VALUES(`f_watermark_cursor_json`),
  `f_last_success_task_id` = VALUES(`f_last_success_task_id`),
  `f_last_full_sync_at` = CASE
    WHEN VALUES(`f_last_full_sync_at`) IS NOT NULL THEN VALUES(`f_last_full_sync_at`)
    ELSE `f_last_full_sync_at`
  END,
  `f_last_reconcile_at` = CASE
    WHEN VALUES(`f_last_reconcile_at`) IS NOT NULL THEN VALUES(`f_last_reconcile_at`)
    ELSE `f_last_reconcile_at`
  END,
  `f_updated_at` = VALUES(`f_updated_at`);
```

---

## 9. 最终建议

这版 2 表模型更适合当前阶段，因为它：

* 足够简单
* 执行粒度正确
* 支持 cron 去重
* 支持重试与 watermark 推进
* 后续仍可扩展为更完整的平台模型
