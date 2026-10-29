# TASK-006 实现图片 OCR 解析适配器

## 关联需求

R-003：支持图片解析。

## 目标

将图片提交给已确认的 OCR 服务，并规范化返回文本和页面来源。

## 输入 / 输出

- 输入：图片真实样本、OCR 服务契约和语言配置。
- 输出：标准文本块或可识别的 OCR 错误。

## 允许修改

- src/integrations/ocr_client.py（新建）
- src/ingestion/parsers/image_parser.py（新建）
- tests/integrations/test_ocr_client.py（新建）
- tests/ingestion/test_image_parser.py（新建）

## 禁止

- 未确认 OCR 服务和语言前，不得调用生产服务。
- 不得将图片原始内容写入日志。
- 不得新增依赖。

## 验收标准

- [ ] 成功响应可映射为标准文本块。
- [ ] 超时、认证失败和服务错误可区分。
- [ ] 日志不包含原始图片内容或敏感识别结果。

## 测试命令

- pytest tests/integrations/test_ocr_client.py tests/ingestion/test_image_parser.py -q

## 预计

0.5–1 天。

## 状态

待 OCR 服务契约和真实图片样本提供。
