# TASK-011 实现检索授权范围解析

## 关联需求

R-005：检索前校验登录、购买、有效期和知识库范围。

R-010：实施课程和文档级权限隔离。

## 目标

从服务端身份上下文和已有权限接口生成可检索的课程与分类范围。

## 输入 / 输出

- 输入：服务端 user_id、tenant_id、权限接口响应。
- 输出：授权课程 ID、分类 ID 与拒绝原因。

## 允许修改

- src/auth/retrieval_scope.py（新建）
- src/integrations/permission_client.py（新建）
- tests/auth/test_retrieval_scope.py（新建）
- tests/integrations/test_permission_client.py（新建）

## 禁止

- 不得信任前端或模型传入的 tenant_id、user_id、course_id。
- 未确认旧系统契约前，不得调用生产权限服务。
- 不得新增依赖。

## 验收标准

- [ ] 未登录、未购买、已过期和跨租户用户均获得拒绝或空范围。
- [ ] 合法用户仅得到被授权的课程和分类范围。
- [ ] 外部接口超时返回可追踪的受控错误。

## 测试命令

pytest tests/auth/test_retrieval_scope.py tests/integrations/test_permission_client.py -q

## 预计

0.5–1 天。

## 状态

待 G-002、G-003 确认。
