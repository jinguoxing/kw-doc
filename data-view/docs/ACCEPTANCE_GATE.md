# ACCEPTANCE_GATE

## 1. 文档目的

本文档定义 `sync-worker` 多模型协同开发时的**统一验收门禁**。

目标：

1. 让不同模型虽然代码风格不同，但最终行为一致
2. 建立统一的“完成定义”
3. 防止实现看似完成、实则偏离 spec
4. 让评审和自动化流水线都有明确判断标准

---

## 2. 适用范围

本验收门禁适用于：

- 每个 Package 的增量交付
- 合并前 review
- 自动化 CI
- 回归检查
- 多模型实现对比

---

## 3. 验收层级

验收分 5 层：

1. **结构验收**
2. **编译验收**
3. **规则验收**
4. **测试验收**
5. **行为一致性验收**

---

## 4. 结构验收

## 4.1 目录结构门禁

代码必须符合 `PROJECT_STRUCTURE_AND_LAYERING.md`。

### 必须满足
- `command/createjob`
- `command/worker`
- `domain/source`
- `domain/diff`
- `domain/apply`
- `domain/task`
- `domain/watermark`
- `domain/runner`
- `repository`
- `model`

### 不允许
- 把大量业务代码堆在一个 `service.go`
- 把 SQL 写进 `domain`
- 把状态机写进 `command`
- 把 diff/apply 混进 `runner`

---

## 4.2 文件修改范围门禁

每个任务只能修改 `Allowed files`。

### 必须满足
- 不修改 blocked files
- 不越界改其他 Package

### 验收方法
- Review 文件清单
- Git diff 检查

---

## 5. 编译验收

## 5.1 最低标准
必须通过：

```bash
go build ./...
```

### 不允许
- 编译不过也提交
- 只在局部 package 编译
- 未使用 import / 未实现接口残留

---

## 5.2 接口一致性
必须保证关键接口签名不漂移。

### 重点检查
- `CreateTasks`
- `ReadPage`
- `NormalizeView`
- `ComparePage`
- `ApplyInMiniBatches`
- `GetOrInit`
- `Advance`
- `RunOneTask`

### 原则
- 除非任务卡明确允许，否则不得自改接口签名

---

## 6. 规则验收

## 6.1 字段映射门禁

必须符合：

- `03_FIELD_MAPPING_CONTRACT.md`

### 必须满足
- 视图级主键：`mdl_id = f_view_id`
- 字段匹配主键：`technical_name -> original_name`
- 强同步字段必须覆盖
- 本地保留字段不得被源强覆盖

---

## 6.2 标准化门禁

必须符合：

- `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`

### 必须满足
- `FieldKey` 规则正确
- `Type` 归一化正确
- `Nullable` 归一化正确
- `change_time` 正确
- hash 规则正确

---

## 6.3 diff 规则门禁

必须符合：

- `05_DIFF_RULES_SPEC.md`

### 必须满足
- `NEW_VIEW / UPDATE_VIEW / DELETE_VIEW / UNCHANGED_VIEW`
- `FIELD_CREATE / UPDATE / DELETE / SUSPECTED_RENAME`
- `Severity` 规则正确
- `last_safe_cursor` 规则正确

---

## 6.4 apply 规则门禁

必须符合：

- `06_APPLY_RULES_SPEC.md`

### 必须满足
- 不物理删除
- 单视图变更在同一事务内
- mini-batch 事务
- 条件驱动字段策略正确
- 本地业务保留字段不被覆盖

---

## 6.5 状态机门禁

必须符合：

- `SYNC_TASK_STATE_MACHINE.md`

### 必须满足
- 合法状态只有 6 个
- retry/backoff 规则正确
- datasource 互斥规则正确
- 租约与心跳规则正确
- watermark 只进不退

---

## 7. 测试验收

## 7.1 单元测试门禁

每个 Package 必须有对应单测。

### 最低要求
- normalizer 单测
- diff_engine 单测
- apply_service 单测
- state_machine 单测

---

## 7.2 集成测试门禁

以下至少需要集成测试：

- repository
- source reader
- runner 最小链路

### 不允许
- 完全没有真实数据库测试
- 全部用 mock 替代

---

## 7.3 fixture 门禁

必须符合：

- `TEST_PLAN_AND_FIXTURES.md`

### 必须满足
- fixture 目录结构正确
- source/local/expected/edge_cases 分类明确
- expected 输出可回归比对

---

## 8. 行为一致性验收

这是多模型一致性的核心。

## 8.1 同 fixture 同结果

不同模型实现必须保证：

对于同一组 fixtures：

- `DiffResult` 一致
- `ApplyResult` 一致
- `LastSafeCursor` 一致
- `sync_task` 更新结果一致
- `watermark` 更新结果一致

### 说明
代码风格可以不同，但行为输出必须一致。

---

## 8.2 Golden Output 机制

建议对关键场景保存 golden output：

- `diff_case_01.json`
- `apply_case_01_views.json`
- `apply_case_01_fields.json`
- `task_case_01.json`
- `watermark_case_01.json`

### 验收方法
- 当前输出与 golden output 对比
- 不一致即失败，除非有明确 spec 变更

---

## 9. 质量门禁检查清单

## 9.1 通用清单

### 编译
- [ ] `go build ./...` 通过

### 文件范围
- [ ] 未修改 blocked files
- [ ] 未越界修改其他 Package

### 分层
- [ ] SQL 只在 repository
- [ ] command 不写业务逻辑
- [ ] domain 不直接写 SQL
- [ ] repository 不做业务判断

### 规则
- [ ] 字段映射符合 03
- [ ] 标准化符合 04
- [ ] diff 符合 05
- [ ] apply 符合 06
- [ ] 状态机符合 SYNC_TASK_STATE_MACHINE

### 测试
- [ ] 单测通过
- [ ] 集成测试通过
- [ ] fixtures 通过

---

## 9.2 Package 专项门禁

### P2 Repository
- [ ] SQL 正确
- [ ] 唯一键 / 租约 / 幂等逻辑正确
- [ ] 不含业务判断

### P3 Source/Normalizer
- [ ] 非法 `f_fields` 返回错误
- [ ] `FieldKey` / `PayloadHash` 正确

### P4 Diff
- [ ] 分类正确
- [ ] rename 不自动合并
- [ ] `Severity` 正确
- [ ] 顺序恢复正确

### P5 Apply
- [ ] 强同步字段覆盖
- [ ] 本地保留字段不覆盖
- [ ] mini-batch 事务正确
- [ ] `last_safe_cursor` 只基于成功提交推进

### P6 State Machine
- [ ] 状态流转合法
- [ ] retry/backoff 正确
- [ ] datasource 冲突短退避

### P8 Worker Runtime
- [ ] poll -> acquire -> run -> finish 主链路正确
- [ ] 心跳续租正确
- [ ] datasource 互斥正确

---

## 10. 自动化建议

CI 中建议至少执行以下步骤：

```bash
go build ./...
go test ./tests/unit/...
go test ./tests/integration/...
```

### 可选增强
- golden output 对比脚本
- staticcheck / golangci-lint
- coverage 最低门槛

---

## 11. Review 门禁

合并前 reviewer 必须对照以下文档 review：

- `03_FIELD_MAPPING_CONTRACT.md`
- `04_FIELDS_SCHEMA_AND_NORMALIZATION.md`
- `05_DIFF_RULES_SPEC.md`
- `06_APPLY_RULES_SPEC.md`
- `SYNC_TASK_STATE_MACHINE.md`
- `PROJECT_STRUCTURE_AND_LAYERING.md`
- `TEST_PLAN_AND_FIXTURES.md`

### Reviewer 输出至少包含
1. Blocking issues
2. Non-blocking issues
3. Suggestions
4. 是否通过门禁

---

## 12. 不通过门禁的典型情形

以下情形一票否决：

1. 改了 blocked files
2. 改了 spec 合同含义
3. 新增未批准依赖
4. 状态机枚举不一致
5. `last_safe_cursor` 算法不一致
6. 本地保留字段被覆盖
7. 只写 happy path，没有异常测试
8. Golden output 不一致
9. SQL 出现在 domain/command
10. runner 绕过状态机直接写成功/失败

---

## 13. 合格交付的 Definition of Done

一个 Package 只有同时满足以下条件，才视为完成：

1. 代码结构符合分层规范
2. 编译通过
3. 对应规则文档检查通过
4. 对应测试通过
5. Golden output 一致
6. Reviewer 通过
7. 没有 blocking issues

---

## 14. 多模型协同时的一致性判断标准

### 允许不同
- 命名细微差异（在不违反接口合同前提下）
- 局部实现风格
- 少量内部 helper 组织方式

### 不允许不同
- 目录结构
- 核心接口签名
- 状态机
- watermark 行为
- diff 结果
- apply 结果
- fixtures 输出
- 本地保留字段策略

---

## 15. 最终建议

多模型协作时，不要追求“代码逐行一样”，而要追求：

> **结构一致、接口一致、规则一致、测试一致、输出一致**

本门禁文档就是为了把“一致性”从主观感受，变成可检查、可自动化、可回归的事实标准。
