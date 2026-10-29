# TASK-003 实现课程资料拉取适配器

## 关联需求

R-002：从旧系统拉取课程资料。

## 目标

实现旧系统课程资料列表和下载接口的适配层。

## 输入 / 输出

- 输入：课程 ID、旧系统资料列表与下载接口契约。
- 输出：标准化课程资料元数据和可读取的文件流。

## 允许修改

- src/integrations/course_client.py（新建）
- src/ingestion/course_material_adapter.py（新建）
- tests/integrations/test_course_client.py（新建）
- tests/ingestion/test_course_material_adapter.py（新建）

## 禁止

- 不得直接连接旧系统数据库。
- 不得假设未确认的生产接口字段、鉴权或错误码。
- 不得执行解析、切片和向量化。

## 验收标准

- [ ] 正确映射课程资料 ID、文件名、类型、哈希和更新时间。
- [ ] 下载超时、4xx、5xx 能返回可区分的适配层错误。
- [ ] Mock 接口可覆盖成功和失败下载。

## 测试命令

- pytest tests/integrations/test_course_client.py tests/ingestion/test_course_material_adapter.py -q

## 预计

0.5–1 天。

## 状态

待 G-002 确认。
