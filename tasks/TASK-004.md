# TASK-004 实现 PDF 与 DOCX 文本解析器

## 关联需求

R-003：支持 PDF、DOCX 的解析。

## 目标

将 PDF 与 DOCX 转换为保留页码或段落来源的标准文本块。

## 输入 / 输出

- 输入：PDF、DOCX 真实样本。
- 输出：标准文本块列表及来源元数据。

## 允许修改

- src/ingestion/parsers/pdf_parser.py（新建）
- src/ingestion/parsers/docx_parser.py（新建）
- tests/ingestion/test_pdf_parser.py（新建）
- tests/ingestion/test_docx_parser.py（新建）

## 禁止

- 不得负责 XLS、PPT、图片或 MP4 解析。
- 不得在解析器中写入数据库或调用向量服务。
- 不得新增依赖。

## 验收标准

- [ ] PDF 样本可提取文本和页码来源。
- [ ] DOCX 样本可提取段落及标题层级。
- [ ] 空文件或损坏文件返回可识别的解析错误。

## 测试命令

- pytest tests/ingestion/test_pdf_parser.py tests/ingestion/test_docx_parser.py -q

## 预计

0.5–1 天。

## 状态

待真实 PDF、DOCX 样本提供。
