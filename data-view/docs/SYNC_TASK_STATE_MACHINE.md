# SYNC_TASK_STATE_MACHINE

## 1. 文档目的

本文档定义 2 表版 `sync-worker` 体系中：

- `t_form_view_sync_task`
- `t_form_view_sync_watermark`

所对应的**任务状态机、调度状态机、watermark 推进规则、错误重试规则**。

本文档直接约束：

- `create-job`
- `run-worker`
- `poller`
- `dispatcher`
- `heartbeat`
- `runner`
- `retry/backoff`
- `watermark_service`

---

## 2. 设计目标

本状态机要解决以下问题：

1. 一个 datasource 的同步任务在生命周期内如何流转
2. 定时调度与手工触发如何生成任务
3. 如何避免同一 datasource 同一时刻并发执行
4. 如何处理任务失败与自动重试
5. 如何在部分成功 / 部分失败时正确推进 watermark
6. 如何保证 `run-worker` 多实例部署时不会重复执行同一任务

---

## 3. 核心对象

## 3.1 `sync_task`

`sync_task` 表示：

> 一个 datasource 的一次同步执行任务

它同时承担：

- 调度任务
- 执行任务
- 重试状态
- 简化租约
- watermark 前后值
- 统计结果
- 错误结果

### 关键标识
- `f_id`
- `f_batch_id`
- `f_job_type`
- `f_datasource_id`
- `f_trigger_source`
- `f_schedule_slot`
- `f_dedupe_key`

---

## 3.2 `sync_watermark`

`sync_watermark` 表示：

> 某个 datasource 的长期安全同步进度

它不表示“本次任务做了多少”，只表示：

- 下次增量从哪里继续
- 最近一次安全推进到哪里

---

## 4. 任务状态定义

当前阶段 `sync_task.f_status` 固定采用以下枚举：

```text
pending
running
retry_waiting
success
failed
cancelled
```

---

## 4.1 `pending`

### 含义
任务已创建，尚未被 worker 抢占执行。

### 典型来源
- `create-job` 新建任务
- 手工创建任务
- 定时 CronJob 创建任务

### 可流转到
- `running`
- `cancelled`

---

## 4.2 `running`

### 含义
任务已经被某个 `run-worker` 抢占，正在执行中。

### 特征
- `f_lock_owner` 非空
- `f_lock_until` > now
- `f_heartbeat_at` 持续刷新

### 可流转到
- `success`
- `retry_waiting`
- `failed`
- `cancelled`

---

## 4.3 `retry_waiting`

### 含义
任务执行失败，但被判定为**可自动重试**，等待下一次调度时间到达。

### 特征
- `f_next_run_at` > now
- `attempt_no < max_attempt`

### 可流转到
- `running`
- `failed`
- `cancelled`

---

## 4.4 `success`

### 含义
任务成功执行结束，并且对应的 `last_safe_cursor` 已正确推进。

### 特征
- 本次任务最终成功
- `f_watermark_after` 与 `f_cursor_json` 对应当前安全断点
- 可被用于更新长期 watermark

### 可流转到
- 无

---

## 4.5 `failed`

### 含义
任务最终失败，不再自动重试。

### 触发原因
- 不可重试错误
- 重试次数达到上限
- 数据异常不可恢复
- 明确配置为不自动重试

### 可流转到
- 无（手工补偿时应新建任务，而不是直接复活旧任务）

---

## 4.6 `cancelled`

### 含义
任务被人工取消或系统明确放弃。

### 使用边界
当前阶段只做最小支持：
- 允许未来手工取消
- `run-worker` 在执行前 / 执行中都应检查取消状态

### 可流转到
- 无

---

## 5. 状态流转图

```text
pending -> running -> success
                 -> retry_waiting -> running -> success
                 -> retry_waiting -> running -> failed
                 -> failed
                 -> cancelled
pending -> cancelled
retry_waiting -> cancelled
```

---

## 6. 状态流转规则

## 6.1 `pending -> running`

### 触发条件
同时满足：

1. `run-worker` 轮询到该任务
2. `f_status in ('pending', 'retry_waiting')`
3. `f_next_run_at <= now`
4. `f_lock_until is null OR f_lock_until < now`
5. 同 datasource 没有其他有效 `running` 任务

### 动作
原子更新：

- `f_status = 'running'`
- `f_lock_owner = worker_id`
- `f_lock_until = now + lease_ms`
- `f_heartbeat_at = now`
- `f_started_at = if null then now`
- `f_attempt_no = f_attempt_no + 1`
- `f_fencing_token = f_fencing_token + 1`

---

## 6.2 `running -> success`

### 触发条件
同时满足：

1. 本次 datasource 全流程执行完成
2. 所有已处理对象成功 apply 到本地
3. `last_safe_cursor` 已计算完成
4. watermark 已成功推进或可安全记录
5. 本任务没有未处理失败洞

### 动作
更新：

- `f_status = 'success'`
- `f_finished_at = now`
- `f_watermark_after`
- `f_cursor_json`
- `f_result_json`
- 统计字段

---

## 6.3 `running -> retry_waiting`

### 触发条件
满足任一：

1. 读源失败，但为临时故障
2. 本地数据库写失败，但为临时故障
3. 网络 / 连接超时
4. 事务死锁 / 临时资源争抢
5. 当前 datasource 被其他任务占用，需要短退避
6. 其他被判定为 `retryable=true` 的错误

### 前提
- `attempt_no < max_attempt`

### 动作
更新：

- `f_status = 'retry_waiting'`
- `f_next_run_at = now + backoff`
- `f_last_error_code`
- `f_last_error_msg`
- `f_result_json`
- `f_watermark_after = last_safe_cursor.change_time`
- `f_cursor_json = last_safe_cursor`

---

## 6.4 `running -> failed`

### 触发条件
满足任一：

1. 错误被判定为不可重试
2. `attempt_no >= max_attempt`
3. 数据损坏 / 结构错误 / 关键字段缺失
4. 明确配置为失败后不自动重试

### 动作
更新：

- `f_status = 'failed'`
- `f_finished_at = now`
- `f_last_error_code`
- `f_last_error_msg`
- `f_result_json`
- 保留 `last_safe_cursor`

---

## 6.5 `pending/running/retry_waiting -> cancelled`

### 触发条件
- 人工取消
- 上层系统明确终止

### 当前阶段策略
- 若未开始执行，直接进入 `cancelled`
- 若执行中，worker 应在安全点检查并尽快退出

---

## 7. `create-job` 状态机责任

`create-job` 不执行任务，它只负责：

1. 解析输入参数
2. 解析 datasource 范围
3. 计算 `schedule_slot`
4. 计算 `batch_id`
5. 生成 `dedupe_key`
6. 插入 `sync_task(status=pending)`

### 注意
`create-job` 不负责：
- 抢占执行权
- watermark 推进
- diff / apply
- retry

---

## 8. `run-worker` 状态机责任

`run-worker` 负责消费 `sync_task` 并驱动状态流转：

1. 轮询任务
2. 抢占任务
3. 续租
4. 执行 datasource 同步
5. 推进 `last_safe_cursor`
6. 写回任务状态
7. 推进长期 watermark

---

## 9. 调度去重模型

## 9.1 `schedule_slot`

`f_schedule_slot` 表示：

> 一次调度时间桶

例子：

- incremental 每 5 分钟一轮
- 2026-04-10 10:05 对应一个 `schedule_slot`

### 用途
- 调度幂等
- 批次识别
- 可观测性

---

## 9.2 `dedupe_key`

推荐格式：

```text
form_view_sync:{job_type}:{datasource_id}:{schedule_slot}
```

例子：

```text
form_view_sync:incremental:ds_1001:2026-04-10T10:05
```

### 规则
同一个：
- `job_type`
- `datasource_id`
- `schedule_slot`

只能存在一条 active 任务：

- `pending`
- `running`
- `retry_waiting`

### 创建结果
若已存在 active 任务，则：
- `create-job` 不重复创建
- 返回 duplicated 结果

---

## 10. `batch_id` 规则

`f_batch_id` 表示：

> 一次 `create-job` 执行所生成的同批任务集合

### 作用
- 让研发 / 运维知道“这一轮创建了哪些 datasource 任务”
- 便于日志、统计、查询

### 规则
- 一次 `create-job` 命令执行生成一个 `batch_id`
- 同一 `batch_id` 下可包含多个 datasource task

---

## 11. datasource 互斥规则

当前 2 表模型里没有独立 lease 表，因此必须采用 **任务表 + 业务检查** 的方式保证互斥。

## 11.1 规则
同一个 `datasource_id` 同一时刻只允许一个有效 `running` 任务。

## 11.2 检查方式
在任务抢占成功后，再检查：

- 是否存在同 datasource 的其他 `running`
- 且 `lock_until > now`
- 且任务 id != 当前任务

### 若存在
当前任务应：
- 回退到 `retry_waiting`
- `next_run_at = now + short_backoff`

---

## 12. 租约与心跳规则

## 12.1 租约字段

- `f_lock_owner`
- `f_lock_until`
- `f_heartbeat_at`
- `f_fencing_token`

---

## 12.2 租约规则

### 抢占时
- 设置 `lock_owner`
- 设置 `lock_until = now + lease_ms`
- 设置 `heartbeat_at = now`
- `fencing_token + 1`

### 执行中
- 周期性续租
- 每次刷新：
  - `lock_until`
  - `heartbeat_at`

### 失效判定
如果：
- `lock_until < now`

则视为租约过期，可被其他 worker 重新抢占。

---

## 12.3 fencing token 规则

`f_fencing_token` 的作用是：

> 避免旧 worker 恢复后继续写入，覆盖新执行器结果

### 原则
- 每次成功抢占任务时 `+1`
- 后续关键写入可校验当前 token 是否仍为最新

### 当前阶段
第一阶段可先：
- 写入 token
- 保留接口与结构
- 关键路径逐步增强校验

---

## 13. 重试规则

## 13.1 可重试错误

推荐判定为可重试的错误：

- 源库读取超时
- 本地 DB 连接异常
- 事务死锁
- 短暂网络异常
- datasource 冲突
- 短暂资源不足

### 状态迁移
- `running -> retry_waiting`

---

## 13.2 不可重试错误

推荐判定为不可重试的错误：

- `f_fields` 非法 JSON
- 关键字段缺失
- 本地对象重复命中
- 主键冲突且无法自动修复
- 不满足字段映射合同
- 程序逻辑断言失败

### 状态迁移
- `running -> failed`

---

## 13.3 backoff 规则

推荐指数退避或固定退避：

### 建议值
- 第 1 次：1 分钟
- 第 2 次：5 分钟
- 第 3 次：15 分钟

### 可配置项
- `retry_backoff_seconds[]`

---

## 13.4 最大重试次数

由：

- `f_max_attempt`

控制。

### 当前建议
默认：

```text
3
```

---

## 14. watermark 状态机

watermark 不是任务状态机，而是 datasource 的长期进度状态。

## 14.1 watermark 核心字段

- `f_watermark_type = change_time`
- `f_watermark_value`
- `f_watermark_cursor_json`
- `f_last_success_task_id`
- `f_last_full_sync_at`
- `f_last_reconcile_at`

---

## 14.2 watermark 推进规则

### 规则 1
只允许成功推进，不允许回退。

### 规则 2
推进依据不是“扫描到哪里”，而是：
- `last_safe_cursor`

### 规则 3
`last_safe_cursor` 必须来自：
- 已经成功 apply 到本地的最后一个对象

### 规则 4
部分成功 / 部分失败时，只推进到连续成功前缀。

---

## 14.3 `change_time` 定义

在当前 `t_data_view` 结构下：

```text
change_time = max(f_create_time, f_update_time, f_delete_time)
```

### 当前阶段排序规则
固定为：

```text
(change_time ASC, view_id ASC)
```

这是 `last_safe_cursor` 计算的前提。

---

## 15. `last_safe_cursor` 规则

## 15.1 定义
表示：

> 当前任务已成功 apply 的最后一个安全断点

结构：

```json
{
  "change_time": 1710001234567,
  "view_id": "dv_000123"
}
```

---

## 15.2 原则
只能推进到：
- 连续成功前缀的最后一个视图

### 不能推进到
- 失败洞之后
- 扫描到的最大记录
- 仅完成 diff 未完成 apply 的对象

---

## 16. 部分成功 / 部分失败处理规则

## 16.1 任务层
若本次任务执行中：
- 前 90 个视图成功
- 后 10 个视图失败

则：

### `sync_task`
必须记录：
- `success_count = 90`
- `failed_count = 10`
- `last_safe_cursor`
- `fail_stage`
- `fail_reason`
- `failed_view_ids_sample`

---

## 16.2 watermark 层
只推进到：
- `last_safe_cursor`

---

## 16.3 状态
- 若可重试：`retry_waiting`
- 若不可重试：`failed`

---

## 17. 取消规则

## 17.1 当前阶段支持
- 允许未来接手工取消
- worker 在关键安全点检查当前状态是否已被改为 `cancelled`

## 17.2 安全点
建议检查点：
1. page 读取前
2. page diff 完成后
3. mini-batch apply 前
4. watermark 推进前

---

## 18. 多实例部署规则

`run-worker` 支持多实例 Deployment，但必须遵守：

1. 任务抢占依赖数据库原子更新
2. datasource 互斥检查必须保留
3. 心跳续租必须开启
4. `fencing_token` 保留，便于后续增强

---

## 19. 研发实现约束

大模型或研发生成代码时，必须遵守：

1. `create-job` 与 `run-worker` 逻辑分离
2. `create-job` 只负责生成任务
3. `run-worker` 只负责消费任务
4. 不允许 `run-worker` 自己顺手创建任务
5. 任务状态流转必须显式，不得隐式跳状态
6. `retry_waiting` 必须显式记录 `next_run_at`
7. `last_safe_cursor` 必须和 apply 成功绑定
8. watermark 推进必须单向、不可回退
9. 同 datasource 不允许并发执行
10. page 内可以并发 diff，但 datasource 内 page 必须顺序执行

---

## 20. 状态机测试矩阵

至少应覆盖以下案例：

### Case 1
- `pending -> running -> success`

### Case 2
- `pending -> running -> retry_waiting -> running -> success`

### Case 3
- `pending -> running -> retry_waiting -> running -> failed`

### Case 4
- datasource 冲突导致 `retry_waiting`

### Case 5
- 心跳丢失导致租约过期，任务被重新抢占

### Case 6
- 部分成功 / 部分失败，验证 `last_safe_cursor`

### Case 7
- 幂等创建：相同 `dedupe_key` 不重复创建

### Case 8
- `cancelled` 状态下 worker 正确退出

---

## 21. 最终决策摘要

### 当前阶段状态机核心决策
1. 任务状态固定为：
   - `pending / running / retry_waiting / success / failed / cancelled`
2. `create-job` 只生成任务，不执行任务
3. `run-worker` 只消费任务，不决定生成任务
4. `dedupe_key` 控制调度去重
5. `batch_id` 表示一次创建批次
6. datasource 同时只允许一个 `running`
7. `last_safe_cursor` 只推进到连续成功前缀
8. watermark 单向推进，不可回退
9. 可重试错误进 `retry_waiting`
10. 不可重试错误进 `failed`
