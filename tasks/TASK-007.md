# TASK-007 实现 MP4 语音转写适配器

## 关联需求

R-003：支持 MP4 解析。

## 目标

将 MP4 交给已确认的 ASR 服务，并输出带时间戳的标准文本块。

## 输入 / 输出

- 输入：MP4 真实样本、ASR 服务契约、语言配置。
- 输出：按时间片段组织的文本块或可识别的转写错误。

## 允许修改

- src/integrations/asr_client.py（新建）
- src/ingestion/parsers/mp4_parser.py（新建）
- tests/integrations/test_asr_client.py（新建）
- tests/ingestion/test_mp4_parser.py（新建）

## 禁止

- 不得实现视频画面理解或 OCR。
- 未确认 ASR 服务、语言和文件大小限制前，不得调用生产服务。
- 不得新增依赖。

## 验收标准

- [ ] 成功转写结果保留时间戳。
- [ ] 超时、认证失败和服务错误可区分。
- [ ] 日志不包含原始音频或完整转写文本。

## 测试命令

pytest tests/integrations/test_asr_client.py tests/ingestion/test_mp4_parser.py -q

## 预计

0.5–1 天。

## 状态

待 ASR 服务契约、语言和真实 MP4 样本确认。
