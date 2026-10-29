# TASK-009 实现 Embedding 与向量索引写入

## 关联需求

R-003：完成向量化与索引。

## 目标

将文本切片转换为向量，并写入 PostgreSQL + pgvector 索引。

## 输入 / 输出

- 输入：文本切片、Embedding 服务契约、租户和文档元数据。
- 输出：带向量、来源元数据和租户字段的切片记录。

## 允许修改

- src/integrations/embedding_client.py（新建）
- src/ingestion/vector_store.py（新建）
- tests/integrations/test_embedding_client.py（新建）
- tests/ingestion/test_vector_store.py（新建）
- docs/DECISIONS-架构决策.md

## 禁止

- 不得改变已确认的 PostgreSQL + pgvector 选型。
- 不得省略 tenant_id、document_id 或来源元数据。
- 不得新增依赖。

## 验收标准

- [ ] 成功切片可写入向量和元数据。
- [ ] Embedding 超时或错误不会产生半完成索引。
- [ ] 不同租户的向量记录可被隔离查询。

## 测试命令

pytest tests/integrations/test_embedding_client.py tests/ingestion/test_vector_store.py -q

## 预计

0.5–1 天。

## 状态

待 Embedding 服务契约和数据库迁移决策确认。
