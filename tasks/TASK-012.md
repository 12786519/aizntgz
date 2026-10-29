# TASK-012 实现带引用的授权检索

## 关联需求

R-005：返回答案和资料引用。

R-012：检索 P95 不高于 5 秒。

## 目标

在授权范围内执行向量和全文召回，生成可追溯的引用结果。

## 输入 / 输出

- 输入：查询文本、服务端授权范围、向量切片索引。
- 输出：排序结果、引用来源、trace_id 与耗时。

## 允许修改

- src/retrieval/search_service.py（新建）
- src/retrieval/citation_builder.py（新建）
- tests/retrieval/test_search_service.py（新建）
- tests/retrieval/test_citation_builder.py（新建）
- tests/retrieval/test_retrieval_authorization.py（新建）

## 禁止

- 不得在空授权范围下执行检索。
- 不得返回无来源的受限资料内容。
- 不得新增依赖。

## 验收标准

- [ ] 结果仅来自授权课程、分类和租户。
- [ ] 每条结果包含文档与页码或时间戳引用。
- [ ] 检索耗时被记录，并可用于 P95 评估。

## 测试命令

pytest tests/retrieval/test_search_service.py tests/retrieval/test_citation_builder.py tests/retrieval/test_retrieval_authorization.py -q

## 预计

0.5–1 天。

## 状态

待 TASK-009、TASK-011 完成。
