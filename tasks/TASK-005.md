# TASK-005 实现 XLS 与 PPT 文本解析器

## 关联需求

R-003：支持 XLS、PPT 的解析。

## 目标

将 XLS 表格与 PPT 页面转换为保留来源位置的标准文本块。

## 输入 / 输出

- 输入：XLS、PPT 真实样本。
- 输出：标准文本块列表及工作表/页码元数据。

## 允许修改

- src/ingestion/parsers/xls_parser.py（新建）
- src/ingestion/parsers/ppt_parser.py（新建）
- tests/ingestion/test_xls_parser.py（新建）
- tests/ingestion/test_ppt_parser.py（新建）

## 禁止

- 不得负责 PDF、DOCX、图片或 MP4 解析。
- 不得在解析器中写入数据库或调用向量服务。
- 不得新增依赖。

## 验收标准

- [ ] XLS 样本可提取工作表、行列位置和单元格内容。
- [ ] PPT 样本可提取页面顺序与文本内容。
- [ ] 空文件或损坏文件返回可识别的解析错误。

## 测试命令

- pytest tests/ingestion/test_xls_parser.py tests/ingestion/test_ppt_parser.py -q

## 预计

0.5–1 天。

## 状态

待真实 XLS、PPT 样本提供。
