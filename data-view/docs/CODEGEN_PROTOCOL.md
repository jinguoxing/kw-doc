# CODEGEN_PROTOCOL

## 1. 文档目的

本文档定义不同大模型在实现 `sync-worker` 时必须遵守的**统一执行协议**。

目标：

1. 让不同模型按同一流程工作
2. 减少实现风格与行为漂移
3. 限制模型“自由发挥”范围
4. 让输出结构标准化，便于 review 与自动化验收

---

## 2. 协议适用范围

本协议适用于以下所有代码生成任务：

- 工程骨架
- repository
- source reader / normalizer
- diff engine
- apply service
- task state machine
- watermark service
- worker runtime
- tests / fixtures

---

## 3. 通用原则

## 3.1 模型不是设计者，而是实现者

模型必须：

- **执行现有 spec**
- **不得自行改设计**
- **不得改合同含义**
- **不得用“更优雅”为由重构架构**

### 明确禁止
- 自创新状态机
- 自创新分层
- 修改字段映射语义
- 修改 watermark 语义
- 修改 diff 分类定义

---

## 3.2 一次只做一个 Package

模型每次只允许实现：

- 一个明确的 Package
- 或一个极小的子任务

### 禁止
- 一次实现整个工程
- 跨多个 Package 大范围改动
- 同时修改业务 spec 与代码

---

## 3.3 必须遵守的 spec 文档优先级

若文档存在冲突，优先级按以下顺序：

1. `03_FIELD_MAPPING_CONTRACT.md`
2. `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`
3. `05_DIFF_RULES_SPEC.md`
4. `06_APPLY_RULES_SPEC.md`
5. `SYNC_TASK_STATE_MACHINE.md`
6. `PROJECT_STRUCTURE_AND_LAYERING.md`
7. `TEST_PLAN_AND_FIXTURES.md`
8. `IMPLEMENTATION_TASK_BREAKDOWN.md`

### 说明
- 字段映射优先于实现习惯
- 状态机优先于“简化逻辑”
- 分层约束优先于模型个人偏好

---

## 4. 标准输入模板

发给模型的任务必须使用统一模板。

---

## 4.1 任务卡模板

```text
Task Package:
Goal:
Scope:
Allowed files:
Blocked files:
Required specs:
Required fixtures/tests:
Constraints:
Output format:
```

---

## 4.2 示例

```text
Task Package: P4 - Diff Engine
Goal:
实现视图级与字段级 diff 引擎

Scope:
- internal/domain/diff/engine.go
- internal/domain/diff/rules.go
- tests/unit/diff_engine_test.go

Allowed files:
- internal/domain/diff/engine.go
- internal/domain/diff/rules.go
- tests/unit/diff_engine_test.go

Blocked files:
- 03_FIELD_MAPPING_CONTRACT.md
- 06_APPLY_RULES_SPEC.md
- internal/repository/*
- internal/domain/apply/*
- internal/domain/runner/*

Required specs:
- 03_FIELD_MAPPING_CONTRACT.md
- 04_FIELDS_SCHEMA_AND_NORMALIZATION.md
- 05_DIFF_RULES_SPEC.md
- PROJECT_STRUCTURE_AND_LAYERING.md

Required fixtures/tests:
- fixtures/source/source_case_01.json
- fixtures/local/local_views_case_01.json
- fixtures/expected/diff_case_01.json

Constraints:
- 不允许写 SQL
- 不允许写 apply
- page 内可并发 compare，输出必须恢复顺序
- 必须输出 Severity
- 不允许把 rename 自动合并成 update

Output format:
- Plan
- Files changed
- Code
- Self-check
- Open questions
```

---

## 5. 标准输出格式

所有模型必须按统一格式输出。

## 5.1 输出顺序

### 1）Plan
必须先说明：

- 自己准备怎么实现
- 哪些文件会改
- 为什么这些文件足够

### 2）Files changed
列出会修改的文件

### 3）Code
给出代码

### 4）Self-check
逐条对照 spec 做自检

### 5）Open questions
如果仍有不确定点，必须显式列出

---

## 5.2 不允许的输出方式

### 禁止
- 直接一大段代码，不给计划
- 不列文件范围
- 不做自检
- 不说明不确定点
- 用自然语言“我顺手优化了架构”

---

## 6. 文件修改范围协议

## 6.1 只能修改允许文件

每次任务只能修改 `Allowed files` 中列出的文件。

### 禁止
- “顺手修一下”其他包
- “顺手补一下” spec 文档
- “顺手重构” repository
- “顺手统一”命名

---

## 6.2 若确实需要新增文件

只有在以下条件同时满足时，允许新增文件：

1. 新文件属于当前 Package
2. 新文件不违反 `PROJECT_STRUCTURE_AND_LAYERING.md`
3. 新文件被明确列入输出说明
4. 不改变既有分层

---

## 7. 依赖与库协议

## 7.1 默认不允许新增依赖

模型不得引入新的第三方库，除非任务卡明确允许。

### 说明
当前阶段优先：
- Go 标准库
- 已批准的 go-zero 基础设施
- 已在工程中的数据库库

---

## 7.2 禁止引入的依赖类型

- 新 ORM
- 新任务调度框架
- 新状态机框架
- 新 diff 库
- 新配置系统
- 新日志系统

---

## 8. 分层约束协议

模型必须严格遵守：

- `command` 不写业务逻辑
- `domain` 不写 SQL
- `repository` 不做业务判断
- `model` 不承载业务方法

### 禁止
- 在 `command` 里直接更新任务状态
- 在 `domain` 里直接拼 SQL
- 在 `repository` 里做 retryable 判定
- 在 `model` 里放 diff 方法

---

## 9. 状态机约束协议

模型不得发明任何新的任务状态。

### 当前合法状态
- `pending`
- `running`
- `retry_waiting`
- `success`
- `failed`
- `cancelled`

### 明确禁止
- `queued`
- `done`
- `partial_success`
- `aborted`
- `stopped`

---

## 10. watermark 协议

模型必须遵守：

1. `change_time = max(create, update, delete)`
2. 排序固定为 `(change_time ASC, view_id ASC)`
3. `last_safe_cursor` 只推进到连续成功前缀
4. watermark 只进不退

### 禁止
- 直接推进到扫描末尾
- 忽略失败洞
- 用成功数量估算 cursor

---

## 11. diff / apply 协议

## 11.1 diff
必须遵守：

- 先视图级后字段级
- hash 快速路径
- rename 只标记不自动合并
- page 内可并发，输出必须恢复顺序

## 11.2 apply
必须遵守：

- 不物理删除
- 单视图 view + field 变更在同一事务内
- 强同步字段覆盖
- 条件驱动字段创建时可初始化、更新默认保留
- 本地保留字段禁止源强覆盖

---

## 12. 测试协议

## 12.1 每个 Package 必须带测试

除极小入口包装外，每个业务 Package 都必须生成对应测试。

### 最少要求
- 单测
- fixture 使用
- expected 断言

---

## 12.2 测试不得偷懒

### 禁止
- 只写 happy path
- 只 mock，不测真实 repo SQL
- 不验证 `last_safe_cursor`
- 不验证本地保留字段
- 不验证 datasource 冲突 / retry

---

## 13. 自检协议

模型输出时必须显式自检：

### 自检模板
```text
Self-check:
- [ ] 是否只修改允许文件
- [ ] 是否遵守字段映射合同
- [ ] 是否遵守状态机合同
- [ ] 是否遵守分层约束
- [ ] 是否引入新依赖
- [ ] 是否补充测试
- [ ] 是否影响其他 Package
```

---

## 14. Review 协议

第二个模型做 review 时，必须采用固定检查模板。

### Review 模板
```text
Review against specs:
- blocking issues
- non-blocking issues
- suggestions

Checkpoints:
1. 字段映射是否违反 03
2. 标准化是否违反 04
3. diff 是否违反 05
4. apply 是否违反 06
5. 状态机是否违反 SYNC_TASK_STATE_MACHINE
6. 分层是否违反 PROJECT_STRUCTURE_AND_LAYERING
7. 测试是否覆盖 TEST_PLAN_AND_FIXTURES
```

---

## 15. 不确定问题处理协议

模型不能自己悄悄假设。

### 若存在不确定点
必须：
1. 先按保守策略实现
2. 显式列出假设
3. 在 `Open questions` 中说明

### 禁止
- 自己 silently 决定并改合同逻辑

---

## 16. 失败输出协议

如果模型无法完成任务，也不能空泛地说“实现有困难”。

必须明确输出：

1. 哪个 spec 信息不足
2. 哪个文件范围阻止了实现
3. 当前可交付的部分实现
4. 需要补充的最小信息

---

## 17. 一致性目标定义

不同模型“实现基本一致”不意味着代码逐行一致，而意味着：

1. 目录结构一致
2. 接口签名一致
3. 状态机一致
4. 行为结果一致
5. fixtures 结果一致
6. 测试通过标准一致

---

## 18. 最终建议

每次给模型的任务都必须附带：

- 对应 Package 任务卡
- 允许修改文件范围
- 必须遵守的 spec 文档
- 必须通过的测试与 fixtures
- 本协议

### 否则
不同模型一定会朝不同方向“合理发挥”。
