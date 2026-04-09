
# 《`t_form_view_sync_job_ds` 表 SQL 修订稿》

这版是基于我们前面已经对齐的几件事来定的：

1. 只保留 **3 张表**
2. `job_ds` 是 **datasource 级执行单元**
3. watermark 采用：

   * `change_time = GREATEST(f_create_time, f_update_time, f_delete_time)`
   * `watermark_value + cursor_json`
4. 部分成功、部分失败时：

   * 结果明细记录在 `job_ds`
   * `watermark` 只推进到 `last_safe_cursor`
5. 第一版不单独拆：

   * `lease` 表
   * `job_log` 表
   * `outbox` 表

所以这张 `job_ds` 表需要同时承担 5 类职责：

* 子任务状态
* 执行租约
* 重试控制
* 执行统计
* 结果明细与安全水位记录

---

# 一、最终 SQL

```sql id="q1cm1u"
CREATE TABLE IF NOT EXISTS `t_form_view_sync_job_ds` (
  `f_id`                    varchar(64)    NOT NULL COMMENT '子任务ID',
  `f_job_id`                varchar(64)    NOT NULL COMMENT '主任务ID',
  `f_datasource_id`         varchar(64)    NOT NULL COMMENT '数据源ID',
  `f_datasource_name`       varchar(128)   NULL COMMENT '数据源名称(冗余)',

  `f_job_type`              varchar(32)    NOT NULL COMMENT '任务类型: incremental|reconcile|full',
  `f_status`                varchar(32)    NOT NULL DEFAULT 'pending' COMMENT '状态: pending|running|retry_waiting|success|failed|cancelled',

  `f_attempt_no`            int unsigned   NOT NULL DEFAULT 0 COMMENT '当前尝试次数',
  `f_max_attempt`           int unsigned   NOT NULL DEFAULT 3 COMMENT '最大自动重试次数',
  `f_next_run_at`           bigint         NOT NULL COMMENT '下次可执行时间(ms)',

  `f_lock_owner`            varchar(128)   NULL COMMENT '当前执行worker实例ID',
  `f_lock_until`            bigint         NULL COMMENT '执行租约过期时间(ms)',
  `f_heartbeat_at`          bigint         NULL COMMENT '最近一次心跳时间(ms)',
  `f_fencing_token`         bigint unsigned NOT NULL DEFAULT 0 COMMENT 'fencing token',

  `f_watermark_type`        varchar(32)    NOT NULL DEFAULT 'change_time' COMMENT '水位类型，当前固定为 change_time',
  `f_watermark_before`      bigint unsigned NOT NULL DEFAULT 0 COMMENT '执行前 watermark 值，对应 change_time',
  `f_watermark_after`       bigint unsigned NOT NULL DEFAULT 0 COMMENT '本次执行后可安全提交的 watermark 值',
  `f_cursor_json`           longtext       NULL COMMENT '本次执行断点/安全游标(JSON)，如 {"change_time":1710000000000,"view_id":"dv_xxx"}',

  `f_page_count`            int unsigned   NOT NULL DEFAULT 0 COMMENT '读取页数',
  `f_batch_count`           int unsigned   NOT NULL DEFAULT 0 COMMENT '处理批次数',

  `f_view_created`          int unsigned   NOT NULL DEFAULT 0 COMMENT '新增视图数',
  `f_view_updated`          int unsigned   NOT NULL DEFAULT 0 COMMENT '更新视图数',
  `f_view_deleted`          int unsigned   NOT NULL DEFAULT 0 COMMENT '删除/下线视图数',

  `f_field_created`         int unsigned   NOT NULL DEFAULT 0 COMMENT '新增字段数',
  `f_field_updated`         int unsigned   NOT NULL DEFAULT 0 COMMENT '更新字段数',
  `f_field_deleted`         int unsigned   NOT NULL DEFAULT 0 COMMENT '删除字段数',

  `f_hash_skipped`          int unsigned   NOT NULL DEFAULT 0 COMMENT '因 hash 相同跳过的对象数',
  `f_suspected_rename`      int unsigned   NOT NULL DEFAULT 0 COMMENT '疑似字段 rename 数',
  `f_audit_revoked`         int unsigned   NOT NULL DEFAULT 0 COMMENT '触发撤销审核次数',

  `f_started_at`            bigint         NULL COMMENT '开始时间(ms)',
  `f_finished_at`           bigint         NULL COMMENT '结束时间(ms)',

  `f_last_error_code`       varchar(64)    NULL COMMENT '最后错误码',
  `f_last_error_msg`        varchar(2000)  NULL COMMENT '最后错误信息',
  `f_result_json`           longtext       NULL COMMENT '执行结果详情(JSON)',

  `f_version`               bigint unsigned NOT NULL DEFAULT 0 COMMENT '乐观锁版本',
  `f_created_at`            bigint         NOT NULL COMMENT '创建时间(ms)',
  `f_updated_at`            bigint         NOT NULL COMMENT '更新时间(ms)',

  PRIMARY KEY (`f_id`),
  UNIQUE KEY `uk_fvsjd_job_ds` (`f_job_id`, `f_datasource_id`),

  KEY `idx_fvsjd_pick` (`f_status`, `f_next_run_at`, `f_lock_until`),
  KEY `idx_fvsjd_job_id` (`f_job_id`),
  KEY `idx_fvsjd_ds_status` (`f_datasource_id`, `f_status`),
  KEY `idx_fvsjd_lock_owner` (`f_lock_owner`, `f_lock_until`),
  KEY `idx_fvsjd_created_at` (`f_created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='form_view 同步 datasource 子任务表';
```

---

# 二、这张表里每类字段的职责

---

## 2.1 任务归属字段

### 字段

* `f_id`
* `f_job_id`
* `f_datasource_id`
* `f_datasource_name`
* `f_job_type`

### 含义

这部分回答的是：

* 这条子任务属于哪个主任务
* 它同步的是哪个 datasource
* 这是增量、对账还是全量

---

## 2.2 执行状态字段

### 字段

* `f_status`
* `f_started_at`
* `f_finished_at`

### 含义

这部分回答的是：

* 当前子任务是不是还在排队
* 是否正在执行
* 是否进入重试
* 是否最终成功或失败

---

## 2.3 重试控制字段

### 字段

* `f_attempt_no`
* `f_max_attempt`
* `f_next_run_at`

### 含义

这部分控制：

* 当前是第几次跑
* 最多还能自动重试几次
* 什么时候允许下一次调度

---

## 2.4 简化版租约字段

### 字段

* `f_lock_owner`
* `f_lock_until`
* `f_heartbeat_at`
* `f_fencing_token`

### 含义

这部分承担的是第一版简化租约能力。

因为我们不单独建 `lease` 表，所以：

* 哪个 worker 抢到了这条子任务
* 租约什么时候失效
* 最近一次心跳是什么时候
* 当前是不是最新执行者

都记录在这里。

---

## 2.5 watermark 与断点字段

### 字段

* `f_watermark_type`
* `f_watermark_before`
* `f_watermark_after`
* `f_cursor_json`

### 含义

这是这张表最关键的一组字段。

#### `f_watermark_before`

表示：

> 本次子任务开始执行前，datasource 的当前安全水位是多少

#### `f_watermark_after`

表示：

> 本次子任务执行过程中，最终可安全提交到哪里

注意它不是“本次扫描到哪里”，而是：

> **本次能够连续成功、安全推进的位置**

#### `f_cursor_json`

记录精确断点，例如：

```json id="7v9k1p"
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

这就是：

* `last_safe_cursor`
* 或本次执行断点

---

## 2.6 统计字段

### 字段

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

### 含义

这部分是为了让你们：

* 不用扫日志
* 不用去所有业务表反推
* 直接从 `job_ds` 看出这次 datasource 执行的结果规模

---

## 2.7 错误与明细字段

### 字段

* `f_last_error_code`
* `f_last_error_msg`
* `f_result_json`

### 含义

这是为了记录：

* 最后一个错误是什么
* 样本失败对象有哪些
* 本次安全提交的边界是什么
* 是否部分成功部分失败

---

# 三、状态值最终约定

## 3.1 `f_status` 可选值

```text id="pxfew8"
pending
running
retry_waiting
success
failed
cancelled
```

---

## 3.2 各状态的业务定义

### `pending`

已创建，等待 worker 执行

### `running`

已被某个 worker 抢占，正在执行

### `retry_waiting`

执行失败，但仍可自动重试，等待到 `f_next_run_at`

### `success`

本次 datasource 子任务执行成功结束

### `failed`

本次 datasource 子任务最终失败，不再自动重试

### `cancelled`

人工取消，或主任务被取消后传播到子任务

---

# 四、`f_cursor_json` 的标准结构

建议直接标准化，不要每个逻辑自己拼。

## 4.1 标准 JSON 结构

```json id="1x16cr"
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

---

## 4.2 字段含义

### `change_time`

对应当前 `t_data_view` 的逻辑变更时间：

```text id="0lfc00"
GREATEST(f_create_time, f_update_time, f_delete_time)
```

### `view_id`

对应：

```text id="nnfbni"
f_view_id
```

用于在同一个 `change_time` 下做精确断点区分。

---

# 五、`f_result_json` 的标准结构

这个字段我建议你们一开始就规范，否则后面很容易变成一锅粥。

---

## 5.1 推荐结构

```json id="jlwm3w"
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

## 5.2 字段解释

### `success_count`

本次成功处理的对象数量

### `failed_count`

本次失败的对象数量

### `failed_view_ids_sample`

失败对象样本，不建议无限存全量

### `last_safe_watermark`

本次安全提交到的最大 `change_time`

### `last_safe_cursor`

本次安全提交到的精确游标

### `fail_stage`

失败阶段，建议固定枚举：

```text id="gjq0t5"
fetch
diff
apply
watermark
```

### `fail_reason`

失败原因摘要

### `retryable`

是否可自动重试

---

# 六、部分成功、部分失败时怎么记录

这部分你前面特别关心，我这里直接定稿。

---

## 6.1 场景

某个 datasource 这次处理 100 张视图：

* 成功 90
* 失败 10

### 记录方式

#### `job_ds`

记录：

* `f_status = retry_waiting` 或 `failed`
* `f_view_updated = 90`
* `f_last_error_msg = ...`
* `f_watermark_after = last_safe_change_time`
* `f_cursor_json = last_safe_cursor`
* `f_result_json` 放失败样本与失败阶段

#### `watermark`

只更新到：

* `last_safe_change_time`
* `last_safe_cursor`

### 关键点

**不是处理到哪里就推进到哪里，而是只推进到连续成功前缀的最后位置。**

---

## 6.2 例子

按 `(change_time, view_id)` 排序：

| 顺序 | view_id | change_time | 结果      |
| -: | ------- | ----------: | ------- |
|  1 | v1      |        1001 | success |
|  2 | v2      |        1002 | success |
|  3 | v3      |        1003 | failed  |
|  4 | v4      |        1004 | success |

那么：

### `f_watermark_after`

```text id="6d27uu"
1002
```

### `f_cursor_json`

```json id="e0cwen"
{
  "change_time": 1002,
  "view_id": "v2"
}
```

### `f_result_json`

```json id="qv3jpz"
{
  "success_count": 3,
  "failed_count": 1,
  "failed_view_ids_sample": ["v3"],
  "last_safe_watermark": 1002,
  "last_safe_cursor": {
    "change_time": 1002,
    "view_id": "v2"
  },
  "fail_stage": "apply",
  "fail_reason": "update local fields failed",
  "retryable": true
}
```

---

# 七、推荐的 SQL 操作语句

---

## 7.1 创建子任务

```sql id="jlwm6z"
INSERT INTO `t_form_view_sync_job_ds` (
  `f_id`,
  `f_job_id`,
  `f_datasource_id`,
  `f_datasource_name`,
  `f_job_type`,
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
  ?, ?, ?, ?, ?, 'pending', 0, 3, ?, 'change_time', 0, 0, ?, ?
);
```

---

## 7.2 抢占执行权

```sql id="wwl4ma"
UPDATE `t_form_view_sync_job_ds`
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

## 7.3 心跳续租

```sql id="m2z3mf"
UPDATE `t_form_view_sync_job_ds`
SET
  `f_lock_until` = ?,
  `f_heartbeat_at` = ?,
  `f_updated_at` = ?
WHERE `f_id` = ?
  AND `f_status` = 'running'
  AND `f_lock_owner` = ?;
```

---

## 7.4 记录成功

```sql id="2k47ee"
UPDATE `t_form_view_sync_job_ds`
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

## 7.5 记录可重试失败

```sql id="yhk1lv"
UPDATE `t_form_view_sync_job_ds`
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

## 7.6 记录最终失败

```sql id="43im70"
UPDATE `t_form_view_sync_job_ds`
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

# 八、建议补的业务约束

虽然我们只保留 3 张表，但 `job_ds` 这张表最好配几条代码层约束。

---

## 8.1 同一个 datasource 同时只允许一个有效 running 子任务

因为没独立 lease 表，这条必须在 worker 抢任务时保证。

---

## 8.2 只有拿到 `last_safe_cursor`，才能更新 `f_watermark_after`

不能把“扫描到哪里”误写成“安全推进到哪里”。

---

## 8.3 `f_watermark_after` 和 `f_cursor_json` 必须成对更新

不能只更新一个。

---

## 8.4 `f_result_json` 只放摘要，不放大体量明细

失败对象过多时只存 sample，避免一条记录无限膨胀。

---

# 九、我对这张表的最终建议

如果按当前阶段上线，这版已经够用了。
它做到了这几点：

1. **能表达 datasource 子任务**
2. **能表达执行锁和心跳**
3. **能表达重试**
4. **能表达部分成功、部分失败**
5. **能表达安全水位推进**
6. **能表达断点恢复**

而且没有引入额外表复杂度。

