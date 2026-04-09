
# 一、设计目标

这套 diff 方案的目标，不是“把所有对象逐条查库逐条比”，而是：

> **按 datasource 分页拉源数据 → 批量加载本地视图和字段 → 内存建索引 → 并发 diff → 小批 apply**

必须同时满足：

* **正确性**

  * 不漏比较
  * 不误删除
  * 不错误推进 watermark

* **性能**

  * 避免 N+1 查询
  * 避免逐条大事务
  * page 内用并发加速 CPU 密集阶段

* **可恢复**

  * page 顺序执行
  * watermark 只推进到 `last_safe_cursor`

---

# 二、前置约束与假设

因为你没有把本地 `form_view` / `form_view_field` 的完整表结构贴出来，这里我先按**推荐实现约束**来设计。

如果你们本地表还没有这些字段，我建议补上。

---

## 2.1 本地 `form_view` 建议至少具备这些“源影子字段”

建议字段：

* `source_view_id`
* `source_datasource_id`
* `source_change_time`
* `source_payload_hash`
* `source_deleted_at`

### 用途

* `source_view_id`：源视图主键映射
* `source_datasource_id`：快速按 datasource 过滤
* `source_change_time`：记录最近一次同步源进度
* `source_payload_hash`：快速判断视图级是否变化
* `source_deleted_at`：标记源侧删除时间

---

## 2.2 本地 `form_view_field` 建议至少具备这些“源影子字段”

建议字段：

* `source_view_id`
* `source_field_key`
* `source_field_hash`

### 用途

* `source_view_id`：知道字段属于哪个源视图
* `source_field_key`：字段主键映射
* `source_field_hash`：快速判断字段是否变化

---

# 三、视图与字段的主键映射规则

这块必须先定死，否则 diff 会越来越乱。

---

## 3.1 视图级匹配规则

### 第一优先级

```text id="wz0icv"
source_view_id = t_data_view.f_view_id
```

### 第二优先级（兜底）

```text id="97hqvb"
datasource_id + technical_name
```

也就是：

* `f_data_source_id + f_technical_name`

### 为什么这样定

因为：

* `f_view_id` 是源表稳定主键
* `technical_name` 更稳定，适合历史数据兜底
* 不建议用 `f_view_name` 做主键，因为更容易变化

---

## 3.2 字段级匹配规则

当前源表字段来自：

* `t_data_view.f_fields`（JSON / longtext）

字段主键建议：

### 第一优先级

```text id="mq2d4n"
original_name
```

### 第二优先级

```text id="u97q6h"
name
```

并统一做标准化：

* trim 空白
* lower case
* 去掉无意义引号
* 统一下划线/大小写差异处理（如果你们需要）

### 最终生成

```text id="ti4w5g"
field_key
```

这个 `field_key` 就是：

* 源字段主键
* 本地 `source_field_key`

---

# 四、总体 diff 策略

我们把 diff 分成 **两层**：

---

## 4.1 第一层：视图级 diff

源对象：`t_data_view` 一行
本地对象：`form_view` 一行

输出结果：

* `NEW_VIEW`
* `UPDATE_VIEW`
* `DELETE_VIEW`
* `UNCHANGED_VIEW`

---

## 4.2 第二层：字段级 diff

只有视图需要深入比较时才做字段级比较。

源对象：`f_fields` 解析出的字段列表
本地对象：`form_view_field` 列表

输出结果：

* `FIELD_CREATE`
* `FIELD_UPDATE`
* `FIELD_DELETE`
* `FIELD_ORDER_CHANGE`
* `FIELD_SUSPECTED_RENAME`

---

# 五、源数据标准化（Normalize）

因为 `t_data_view` 的字段很多，而且 `f_fields` / `f_data_scope` 都是长文本，第一步必须先做标准化。

---

## 5.1 标准化后的 SourceView 结构

建议在代码里定义：

```go id="xsvdjr"
type SourceView struct {
    ViewID         string
    DatasourceID   string
    GroupID        string
    ViewName       string
    TechnicalName  string
    Type           string
    QueryType      string
    Comment        string
    Status         string
    MetaTableName  string
    CreateTime     int64
    UpdateTime     int64
    DeleteTime     int64
    ChangeTime     int64
    PayloadHash    string
    Fields         []SourceField
}
```

其中：

```go id="w2zcq3"
ChangeTime = max(CreateTime, UpdateTime, DeleteTime)
```

---

## 5.2 标准化后的 SourceField 结构

```go id="yadp3d"
type SourceField struct {
    FieldKey      string
    Name          string
    OriginalName  string
    DisplayName   string
    Type          string
    Index         int
    Nullable      *bool
    Extra         map[string]any
    Hash          string
}
```

---

## 5.3 PayloadHash 的推荐计算方式

对视图级 hash，建议基于以下字段做 canonical json 后 hash：

* `f_view_name`
* `f_technical_name`
* `f_type`
* `f_query_type`
* `f_comment`
* `f_status`
* `f_meta_table_name`
* 标准化后的 `Fields`

### 这样做的意义

如果 hash 相同，说明这个视图整体没有变化，可以直接跳过深比较。

---

# 六、本地加载方案

这里是性能关键点。

---

## 6.1 不允许逐条查本地表

错误做法：

* 一条源 view 查一次 `form_view`
* 再查一次 `form_view_field`

这样是典型的 N+1。

---

## 6.2 正确做法：批量加载

假设当前 page 有 300 条源 `SourceView`。

### 第一步：批量加载本地 `form_view`

按这一页的：

* `source_view_id IN (...)`

一次查出来。

如果历史数据不完整，再补一轮 fallback：

* `datasource_id = ?`
* `technical_name IN (...)`

### 第二步：批量加载本地 `form_view_field`

对当前 page 所命中的本地 `form_view_id IN (...)` 一次查出来全部字段。

---

## 6.3 本地内存索引

建议建这几个 map：

### 视图级

```go id="ef0ql8"
localViewBySourceID   map[string]LocalView
localViewByFallback   map[string]LocalView
```

其中 fallback key：

```go id="j0nezy"
datasource_id + "#" + technical_name
```

### 字段级

```go id="eu9lv7"
localFieldsByFormViewID map[string][]LocalField
```

以及进一步为单视图生成：

```go id="zr1sud"
map[fieldKey]LocalField
```

---

# 七、视图级比较逻辑

---

## 7.1 判断视图是否存在

对于一条 `SourceView`：

### 先按 `source_view_id` 查

找不到再按：

* `datasource_id + technical_name`

兜底。

---

## 7.2 视图级结果分类

---

### CASE 1：本地不存在

结果：

```text id="hfip8h"
NEW_VIEW
```

动作：

* 新建 `form_view`
* 新建全部 `form_view_field`

---

### CASE 2：源侧已删除

判断依据一般是：

* `f_delete_time > 0`
* 或 `status` 表示删除/下线

结果：

```text id="pe3m5t"
DELETE_VIEW
```

动作：

* 本地标记删除/下线
* 必要时撤销审核

---

### CASE 3：本地存在，且 `source_payload_hash` 相同

结果：

```text id="7lpwk2"
UNCHANGED_VIEW
```

动作：

* 直接跳过
* `hash_skipped + 1`

---

### CASE 4：本地存在，但 hash 不同

结果：

```text id="9ol51j"
UPDATE_VIEW
```

动作：

* 继续做视图字段级深比较

---

# 八、字段级比较逻辑

这部分是核心。

---

## 8.1 先把字段列表 map 化

### 源字段

```go id="mzm34u"
sourceFieldMap[fieldKey] = SourceField
```

### 本地字段

```go id="g2oh35"
localFieldMap[fieldKey] = LocalField
```

---

## 8.2 字段 diff 分类

---

### 1）源有，本地没有

```text id="7ixcdz"
FIELD_CREATE
```

动作：

* 新建本地字段

---

### 2）源没有，本地有

```text id="ku1m2r"
FIELD_DELETE
```

动作：

* 本地字段标记删除/下线

---

### 3）源有，本地也有，但 hash 不同

```text id="n2h6ks"
FIELD_UPDATE
```

需要进一步判断变化项。

---

## 8.3 字段变化项比较

建议至少比较：

* `DisplayName`
* `OriginalName`
* `Type`
* `Index`
* `Nullable`
* 你们业务里其他关键属性

生成：

```go id="d70n30"
type SingleFieldChange struct {
    Key                 string
    DisplayNameChanged  bool
    OriginalNameChanged bool
    TypeChanged         bool
    IndexChanged        bool
    NullableChanged     bool
}
```

---

## 8.4 疑似 rename 判断

在没有稳定 field_id 的前提下，rename 很难做到 100% 自动识别。

### 建议做法

如果出现：

* 一个旧字段删除
* 一个新字段新增
* 两者类型相同
* display_name 高相似
* 索引位置接近

则标记：

```text id="xnr8uk"
FIELD_SUSPECTED_RENAME
```

### 第一版建议

* 只做标记，不自动合并为 rename
* 计数写进 `suspected_rename`

---

# 九、最终 diff 结果结构

建议统一输出为：

```go id="do9ylz"
type DiffResult struct {
    News      []CreateViewDiff
    Updates   []UpdateViewDiff
    Deletes   []DeleteViewDiff
    Unchanged []string
    Stats     SyncStats
}
```

其中：

```go id="tkugqa"
type UpdateViewDiff struct {
    SourceView   SourceView
    LocalView    LocalView
    ViewChanged  bool
    ViewDelta    ViewMetaDelta
    FieldDelta   FieldDelta
    Severity     string // S1/S2/S3
}
```

---

# 十、apply 方案

diff 完不等于结束，还要 apply。

---

## 10.1 建议的 apply 顺序

### 每个 page 内

1. `NEW_VIEW`
2. `UPDATE_VIEW`
3. `DELETE_VIEW`

### 为什么

这样逻辑最清晰，尤其对字段新增/更新/删除更容易控制。

---

## 10.2 事务策略

### 不建议

* 整个 page 一个超大事务

### 推荐

* 每 20~50 个 view 一个小事务
* 或每个 view 一个事务（更稳但更慢）

### 第一版推荐

```text id="7q22v3"
每 20~50 个 view 一组事务
```

---

## 10.3 apply 成功后更新哪些本地字段

### `form_view`

* `source_view_id`
* `source_datasource_id`
* `source_change_time`
* `source_payload_hash`
* `source_deleted_at`

### `form_view_field`

* `source_view_id`
* `source_field_key`
* `source_field_hash`

---

# 十一、Go 协程 + 管道的性能方案

这里是你特别关心的点。

先给结论：

> **可以用协程和管道，但要分层使用，不能为了并发破坏 datasource 内的顺序执行。**

---

## 11.1 总体并发原则

### datasource 之间

可以并行

### 单 datasource 内的 page

必须顺序执行

### 单 page 内的 diff

可以并行

### watermark 推进

必须串行

---

## 11.2 推荐的流水线模型

---

## 外层：datasource worker 并发

`run-worker` 会同时跑多个 datasource task：

```text id="0q0bha"
task(ds1) || task(ds2) || task(ds3)
```

这个并发度由：

* `MaxDatasourceParallelism`

控制。

---

## 内层：单 datasource 的 page 流水线

我建议的 page 级管道是：

```text id="b0w6ju"
Reader -> PageNormalizer -> LocalLoader -> DiffWorkers -> Aggregator -> Apply -> Watermark
```

### 解释

#### Reader

顺序从 `t_data_view` 拉 page

#### PageNormalizer

把当前 page 的源记录解析成 `SourceView`

#### LocalLoader

批量查本地 `form_view / form_view_field`

#### DiffWorkers

page 内并行做单 view diff

#### Aggregator

按原顺序收集 diff 结果，计算 stats 和 `last_safe_cursor`

#### Apply

小批事务写本地

#### Watermark

更新 `job_ds` 进度和长期 watermark

---

## 11.3 Page 内 worker pool 推荐

### 配置建议

* `pageSize = 200 ~ 500`
* `diffWorkers = min(8, CPU*2)`

### 代码示意

```go id="udrjzd"
type CompareTask struct {
    Seq        int
    SourceView SourceView
    LocalView  *LocalView
    LocalFields map[string]LocalField
}

type CompareResult struct {
    Seq   int
    Diff  SingleViewDiff
    Err   error
}
```

### worker pool

```go id="vs8mgz"
tasks := make(chan CompareTask, len(page))
results := make(chan CompareResult, len(page))

for i := 0; i < diffWorkers; i++ {
    go func() {
        for task := range tasks {
            diff, err := compareOneView(task)
            results <- CompareResult{
                Seq:  task.Seq,
                Diff: diff,
                Err:  err,
            }
        }
    }()
}
```

### 为什么要带 `Seq`

因为虽然 compare 可以并行，但最后聚合和安全 cursor 判断需要知道：

* 当前结果在原 page 顺序中的位置

---

## 11.4 Aggregator 的作用

Aggregator 是这套并发设计的关键。

### 它负责

1. 收集所有 `CompareResult`
2. 按 `Seq` 排序/还原顺序
3. 生成 page 级 `DiffResult`
4. 在 apply 后计算 `last_safe_cursor`

### 为什么它重要

如果没有 Aggregator：

* 结果顺序会乱
* watermark 没法判断连续成功前缀

---

## 11.5 为什么单 datasource 内 page 不建议并行

因为一旦 page1/page2/page3 并行：

* 哪个先 apply 成功不确定
* `last_safe_cursor` 会变复杂
* 失败洞会让 watermark 很难正确推进

### 所以建议

```text id="wzr05l"
datasource 内 page 串行
page 内 view diff 并行
```

这就是最平衡的方案。

---

# 十二、推荐的核心伪代码

---

## 12.1 单 datasource 执行主流程

```go id="kf4e8i"
func RunDatasourceTask(ctx context.Context, task SyncTask) error {
    wm, err := watermarkRepo.GetOrInit(ctx, task.DatasourceID)
    if err != nil {
        return err
    }

    cursor := buildCursor(wm)
    stats := NewSyncStats()

    for {
        pageRows, nextCursor, err := sourceReader.ReadPage(ctx, task.DatasourceID, cursor, pageSize)
        if err != nil {
            return failOrRetry(task, err)
        }
        if len(pageRows) == 0 {
            break
        }

        sourceViews := normalizePageConcurrently(pageRows)

        localViews, localFields, err := localLoader.BatchLoad(ctx, sourceViews)
        if err != nil {
            return failOrRetry(task, err)
        }

        diffRes, err := diffPageWithWorkerPool(ctx, sourceViews, localViews, localFields, diffWorkers)
        if err != nil {
            return failOrRetry(task, err)
        }

        applyRes, err := applyService.ApplyInMiniBatches(ctx, diffRes, 20)
        if err != nil {
            return failOrRetry(task, err)
        }

        stats.Merge(applyRes.Stats)

        cursor = applyRes.LastSafeCursor

        if err := taskRepo.UpdateProgress(ctx, task.ID, stats, cursor); err != nil {
            return failOrRetry(task, err)
        }
    }

    if err := watermarkRepo.Advance(ctx, task.DatasourceID, cursor, task.ID, task.JobType); err != nil {
        return failOrRetry(task, err)
    }

    return taskRepo.MarkSuccess(ctx, task.ID, stats, cursor)
}
```

---

# 十三、这套方案的优点

## 1）性能比逐条查快很多

因为是：

* 源分页
* 本地批量加载
* page 内并发 diff

---

## 2）watermark 语义不会乱

因为：

* 单 datasource 内 page 串行
* apply 后再推进 watermark

---

## 3）后续可扩展

后面你们要加：

* `source_payload_hash`
* `source_field_hash`
* 更复杂字段规则

都能平滑长上去。

---

# 十四、我对研发落地的最终建议

如果按你们当前方案落地，我建议研发按下面这几个模块拆：

### 1. `source_reader`

负责按 cursor 读 `t_data_view`

### 2. `normalizer`

负责把 `t_data_view` 转成 `SourceView / SourceField`

### 3. `local_loader`

负责批量读本地 `form_view / form_view_field`

### 4. `diff_engine`

负责 page 内并发比较

### 5. `apply_service`

负责小批事务写本地

### 6. `watermark_service`

负责安全推进 watermark

---

# 十五、最终一句话方案

> **“datasource 级串行、page 级流水、view 级并发、apply 小批、watermark 安全推进”**

