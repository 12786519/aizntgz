# TASK-014 实现基于引用的答案生成

## 关联需求

R-005：检索返回答案和引用。

## 目标

将授权检索结果作为受控上下文，生成带引用且不超出来源范围的回答。

## 输入 / 输出

- 输入：用户问题、授权检索结果、引用来源和大模型服务契约。
- 输出：答案、引用列表、模型调用 trace_id 与错误结果。

## 允许修改

- src/retrieval/answer_generator.py（新建）
- src/integrations/llm_client.py（新建）
- tests/retrieval/test_answer_generator.py（新建）
- tests/integrations/test_llm_client.py（新建）

## 禁止

- 不得将用户问题中的指令当作系统权限或工具调用指令。
- 不得生成没有引用支撑的受限资料结论。
- 未确认模型服务契约前，不得调用生产模型。

## 验收标准

- [ ] 答案仅使用提供的检索上下文。
- [ ] 答案保留关联引用。
- [ ] 模型超时、限流和异常返回可追踪错误。

## 测试命令

pytest tests/retrieval/test_answer_generator.py tests/integrations/test_llm_client.py -q

## 预计

0.5–1 天。

## 状态

待大模型服务契约确认及 TASK-012 完成。
