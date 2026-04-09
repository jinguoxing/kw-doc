下面直接给你正式收敛稿：

# 《diff 判定规则 + 本地 apply 规则 最终稿》

这版是基于我们已经对齐的前提来写的：

1. 源表是 `t_data_view`
2. 增量顺序按：

   * `change_time = GREATEST(f_create_time, f_update_time, f_delete_time)`
   * `ORDER BY change_time, f_view_id`
3. `change_time` 只负责：

   * 增量顺序
   * watermark 游标
4. 操作类型由 diff 引擎判定，不由 `change_time` 直接决定
5. 第一版建议：

   * **按顺序处理**
   * **遇错即停**
   * **本地写入必须幂等**

---

# 一、diff 引擎要解决什么问题

diff 引擎不是简单比较两个 JSON。
它要回答 3 个问题：

## 1）这条源记录对应本地哪条记录

即匹配问题。

## 2）它相对本地是什么动作

即：

* CREATE
* UPDATE
* DELETE
* RESTORE
* NOOP

## 3）如果是 UPDATE，到底更新哪些字段

即：

* 视图主字段变了什么
* 字段列表变了什么
* 变化严重度是什么

---

# 二、源记录标准化模型

先不要直接拿数据库行做 diff。
建议先把源记录标准化成统一结构。

---

## 2.1 源视图标准模型

```json id="jgl9vm"
{
  "source_view_id": "dv_xxx",
  "datasource_id": "ds_xxx",
  "group_id": "grp_xxx",
  "type": "meta",
  "query_type": "table",
  "view_name": "订单表",
  "technical_name": "orders",
  "comment": "订单主表",
  "meta_table_name": "ods_orders",
  "status": "published",
  "delete_time": 0,
  "change_time": 1710001234567,
  "fields": [],
  "row_hash": "abc123"
}
```

---

## 2.2 本地视图标准模型

本地建议同步保存 source 影子字段。

```json id="a8z49g"
{
  "local_view_id": "fv_xxx",
  "source_view_id": "dv_xxx",
  "source_datasource_id": "ds_xxx",
  "source_change_time": 1710001230000,
  "source_delete_time": 0,
  "source_row_hash": "abc111",
  "deleted": false,
  "fields": []
}
```

---

# 三、记录匹配规则

---

## 3.1 视图级主匹配键

第一优先级：

```text id="caeipj"
source_view_id = t_data_view.f_view_id
```

也就是本地 form_view 上必须保存：

* `source_view_id`

---

## 3.2 fallback 匹配键

如果历史数据还没有完整 `source_view_id`，则用兜底：

```text id="dlko6y"
datasource_id + technical_name
```

注意：

* 这只是迁移兼容方案
* 不能作为长期主键方案

---

## 3.3 不建议作为主匹配键的字段

不要用：

* `group_id + view_name`

因为你前面已经明确：

* 元数据视图：`group_id = datasource_id`
* 自定义视图：`group_id` 是分组语义

这个字段本身语义不稳定。

---

# 四、源侧状态判定

diff 引擎第一步先判断源记录当前状态。

---

## 4.1 `source_alive`

条件：

```text id="55bav6"
f_delete_time = 0
```

表示这条源视图当前是有效的。

---

## 4.2 `source_deleted`

条件：

```text id="97x6g6"
f_delete_time > 0
```

表示这条源视图当前已经被删除/下线。

---

# 五、本地状态判定

---

## 5.1 `local_missing`

本地找不到对应记录。

---

## 5.2 `local_alive`

本地存在，且未被标记删除/下线。

---

## 5.3 `local_deleted`

本地存在，但已标记删除/下线。

---

# 六、最终动作判定矩阵

这是核心规则，建议直接写进设计文档。

| 源侧状态    | 本地状态    | 动作               | 说明         |
| ------- | ------- | ---------------- | ---------- |
| alive   | missing | CREATE           | 本地新建       |
| alive   | alive   | UPDATE / NOOP    | 比较是否变化     |
| alive   | deleted | RESTORE / UPDATE | 本地恢复并同步    |
| deleted | missing | NOOP             | 已删且本地没有，跳过 |
| deleted | alive   | DELETE           | 本地下线/删除标记  |
| deleted | deleted | NOOP             | 幂等跳过       |

---

# 七、NOOP 判定规则

很多重复扫描最终都要落到 NOOP，所以这个规则很重要。

---

## 7.1 时间比本地旧

如果：

```text id="e9ae7s"
incoming.change_time < local.source_change_time
```

则直接：

```text id="cq8v6u"
NOOP
```

旧数据不能覆盖新数据。

---

## 7.2 时间相同且 hash 相同

如果：

```text id="0vvlty"
incoming.change_time = local.source_change_time
and incoming.row_hash = local.source_row_hash
```

则直接：

```text id="e36e1l"
NOOP
```

说明这是一次重放，不需要再 apply。

---

## 7.3 源已删，本地也已删

如果：

* `source_deleted`
* `local_deleted`

则：

```text id="u75m5d"
NOOP
```

---

# 八、CREATE 判定与 apply 规则

---

## 8.1 CREATE 判定条件

满足：

* `source_alive`
* `local_missing`

---

## 8.2 CREATE 需要写什么

本地要新建：

### 视图主记录

* 基础视图信息
* source 影子字段

### 字段记录

* 从 `f_fields` 展开后批量写入

### source 影子字段建议写入

* `source_view_id`
* `source_datasource_id`
* `source_change_time`
* `source_delete_time = 0`
* `source_row_hash`

---

## 8.3 CREATE 必须是幂等 upsert

第一版不要做“纯 insert 然后报重复”。

应该做成：

* 若本地已存在同 `source_view_id`
* 则降级成 UPDATE 或直接 skip

---

# 九、UPDATE 判定与 apply 规则

---

## 9.1 UPDATE 判定条件

满足：

* `source_alive`
* `local_alive`
* 且不是 NOOP

也就是：

* `incoming.change_time > local.source_change_time`
  或
* `incoming.change_time = local.source_change_time` 且 `hash 不同`

---

## 9.2 UPDATE 分两层

### 第一层：视图主字段比较

比较这些字段：

* `view_name`
* `technical_name`
* `comment`
* `type`
* `query_type`
* `meta_table_name`
* `status`

### 第二层：字段列表比较

展开 `f_fields` 后逐项比较。

---

## 9.3 字段主键规则

字段级比较，第一版建议字段主键这样定：

优先：

```text id="hvjqfq"
original_name
```

兜底：

```text id="sw9jyt"
name
```

并统一做：

* trim
* lower
* 去引号差异

---

## 9.4 字段 diff 结果分类

### `field_create`

源有，本地没有

### `field_update`

源有，本地有，但属性变化

### `field_delete`

本地有，源没有

### `field_order_change`

只有顺序变化

### `suspected_rename`

疑似 rename，第一版只标记，不自动做 rename 合并

---

## 9.5 UPDATE 的 apply 规则

### 视图主记录

只更新变化字段

### 字段记录

* 新增字段：insert
* 更新字段：update
* 删除字段：逻辑删除或标记删除

### 最后统一刷新 source 影子字段

* `source_change_time`
* `source_delete_time`
* `source_row_hash`

---

# 十、DELETE 判定与 apply 规则

---

## 10.1 DELETE 判定条件

满足：

* `source_deleted`
* `local_alive`

---

## 10.2 DELETE 怎么做

第一版不建议物理删除。

建议做：

### 视图级

* 标记 deleted / offline
* 记录删除原因
* 记录 `source_delete_time`

### 字段级

* 可保留历史，不一定要物理删除
* 视具体业务可标记不可用

---

## 10.3 DELETE 必须幂等

如果一条已删除记录又被重复扫描到：

* 只要本地已经 deleted
* 就直接 NOOP

---

# 十一、RESTORE 判定与 apply 规则

这个场景建议你们第一版就定义上。

---

## 11.1 RESTORE 判定条件

满足：

* `source_alive`
* `local_deleted`

这说明：

* 本地之前删过
* 源侧现在重新有效

---

## 11.2 RESTORE 怎么做

建议：

### 第一步

把本地状态从 deleted 恢复成 alive

### 第二步

继续走 UPDATE 流程，把主字段和字段列表补齐

### 第三步

刷新 source 影子字段

---

# 十二、字段级 diff 详细规则

---

## 12.1 字段标准化结构

建议先把源字段和本地字段都标准化成：

```json id="oddoij"
{
  "field_key": "order_id",
  "name": "order_id",
  "original_name": "order_id",
  "display_name": "订单ID",
  "type": "string",
  "index": 1
}
```

---

## 12.2 字段比较字段

第一版建议至少比较：

* `name`
* `original_name`
* `display_name`
* `type`
* `index`

如果本地模型有，还可比较：

* `nullable`
* `comment`
* `length`
* `precision`

---

## 12.3 字段变化严重度

### S1：高影响

* 字段删除
* 字段新增
* 字段类型变化
* `original_name` 变化

### S2：中影响

* `display_name` 变化
* nullable 变化
* comment 变化

### S3：低影响

* index 顺序变化

---

# 十三、本地 apply 的事务边界

这部分非常重要。

---

## 13.1 一个 view 必须一个事务

也就是说：

> **同一条源视图的主记录 + 字段列表 + source 影子字段更新，必须在一个事务内完成。**

不能拆成：

* view 先成功
* fields 后失败

否则本地状态会半残。

---

## 13.2 建议事务包含的内容

### CREATE

* insert local view
* insert local fields
* 写 source 影子字段

### UPDATE

* update local view
* apply field creates
* apply field updates
* apply field deletes
* update source 影子字段

### DELETE

* mark local view deleted
* update source_delete_time
* 记录删除原因

### RESTORE

* mark local view alive
* 再走 UPDATE

---

# 十四、幂等 apply 的最终规则

这是解决“后面已成功，下轮重放怎么办”的关键。

---

## 14.1 旧版本输入不允许覆盖新版本本地状态

如果：

```text id="ytjlwm"
incoming.change_time < local.source_change_time
```

则：

```text id="jo0vdp"
skip
```

---

## 14.2 同版本同内容直接 skip

如果：

```text id="k6r4lk"
incoming.change_time = local.source_change_time
and incoming.row_hash = local.source_row_hash
```

则：

```text id="nycssh"
skip
```

---

## 14.3 同版本不同内容

这个场景理论上不该频繁出现，但第一版建议：

* 记告警
* 仍按 UPDATE 处理
* 最终以源最新内容覆盖

---

## 14.4 DELETE 重放必须幂等

如果本地已经 deleted，再来同一个删除：

* skip

---

## 14.5 CREATE 重放必须幂等

如果本地已存在相同 `source_view_id`：

* 不报重复错误
* 转为 UPDATE/NOOP

---

# 十五、第一版执行策略建议

这里我给你明确建议。

---

## 15.1 最推荐的第一版策略：遇错即停

也就是：

### 规则

按 `(change_time, view_id)` 顺序处理时：

* 当前批/页中某条记录 apply 失败
* 立即停止处理后续记录
* 当前 job_ds 进入 `retry_waiting` 或 `failed`

### 好处

* watermark 最干净
* 不会出现“中间有洞、后面先写成功”
* 一致性最容易保证

---

## 15.2 如果后续一定要支持“失败后继续处理”

那也可以，但前提是：

1. 本地必须保存 source 影子字段
2. apply 必须严格幂等
3. 下轮必须从 `last_safe_cursor` 重新读

第一版我不建议直接上这个复杂模式。

---

# 十六、diff 引擎输出结果建议

建议统一输出一个标准结构，供 apply 层消费。

```json id="4cnm3r"
{
  "action": "UPDATE",
  "severity": "S1",
  "source_view_id": "dv_xxx",
  "local_view_id": "fv_xxx",
  "view_changes": {
    "comment_changed": true,
    "meta_table_name_changed": false
  },
  "field_changes": {
    "created": [],
    "updated": [],
    "deleted": [],
    "suspected_rename": []
  },
  "source_shadow": {
    "source_change_time": 1710001234567,
    "source_delete_time": 0,
    "source_row_hash": "abc123"
  }
}
```

这样：

* diff 层只负责判定
* apply 层只负责执行

边界会很清晰。

---

# 十七、最终定稿规则

建议你们把这几条直接固化进技术方案。

## Rule 1

`change_time` 仅用于增量顺序和 watermark 游标，不直接表示操作类型。

## Rule 2

操作类型由 diff 引擎通过“源侧状态 + 本地状态 + hash 比较”判定。

## Rule 3

视图级动作只允许是：

```text id="c3kvyb"
CREATE
UPDATE
DELETE
RESTORE
NOOP
```

## Rule 4

本地 apply 必须按单视图事务执行。

## Rule 5

本地必须保存 source 影子字段：

* `source_view_id`
* `source_datasource_id`
* `source_change_time`
* `source_delete_time`
* `source_row_hash`

## Rule 6

旧数据不能覆盖新数据；同版本同内容必须跳过。

## Rule 7

第一版推荐按顺序处理、遇错即停。

