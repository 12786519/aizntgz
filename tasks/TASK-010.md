# TASK-010 实现入库任务状态与重试

## 关联需求

R-004：入库任务可查询状态、失败重试和版本更新。

## 目标

实现解析、切片、向量化、索引四阶段的任务状态与受控重试。

## 输入 / 输出

- 输入：文档 ID、任务阶段、trace_id 和执行结果。
- 输出：可查询的任务状态、失败原因和重试计数。

## 允许修改

- src/ingestion/task_state.py（新建）
- src/workers/ingestion_worker.py（新建）
- tests/ingestion/test_task_state.py（新建）
- tests/workers/test_ingestion_worker.py（新建）
- docs/DECISIONS-架构决策.md

## 禁止

- 不得吞掉异常或无限重试。
- 不得直接修改已完成任务为运行中。
- 不得新增依赖。

## 验收标准

- [ ] 任务能在 pending、running、success、failed 之间合法流转。
- [ ] 失败任务达到最大重试次数后保留失败原因。
- [ ] 重复消息不会重复执行已成功阶段。

## 测试命令

pytest tests/ingestion/test_task_state.py tests/workers/test_ingestion_worker.py -q

## 预计

0.5–1 天。

## 状态

待开始。
