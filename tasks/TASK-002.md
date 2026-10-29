# TASK-002 实现手动上传文件校验

## 关联需求

R-002：支持非课程资料手动上传。

## 目标

实现手动上传入口的文件类型、大小、租户和元数据校验。

## 输入 / 输出

- 输入：上传文件、分类 ID、租户上下文和文件限制配置。
- 输出：待入库文档记录或结构化校验错误。

## 允许修改

- src/ingestion/upload_validator.py（新建）
- src/ingestion/upload_service.py（新建）
- tests/ingestion/test_upload_validator.py（新建）

## 禁止

- 不得解析文件内容或调用 OCR/ASR。
- 不得跳过租户、分类或文件类型校验。
- 不得新增依赖。

## 验收标准

- [ ] 六类允许格式通过白名单校验。
- [ ] 不允许的格式、超限文件和缺失分类返回明确错误。
- [ ] 创建的待入库记录带 tenant_id、category_id 和文件哈希。

## 测试命令

- pytest tests/ingestion/test_upload_validator.py -q

## 预计

0.5–1 天。

## 状态

待文件大小限制确认。
