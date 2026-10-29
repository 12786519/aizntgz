# 旧系统技术核验

> 日期：2026-10-28 ｜ 核验者：Codex ｜ 状态：源码核对与选定单元测试完成；非接口联调或业务验收。

## 1. 结论

可确认：旧系统存在受信课程资料读取、资料事件校验、实时课程权益、知识库授权与解析实现；本次选定的 **77 项单元测试全部通过**。

同时确认两处接入差异：已核对的通用消息投递流程只支持站内信；课程资料事件接入的格式集合与手动上传不同，事件接入集合不包含 PPTX。这些事实不能被“旧系统已有相关模块”替代。

G-002、G-003、G-004 仍未全部关闭。已有实现、单元测试通过、实际接口可供新服务使用、业务验收通过是不同证据层级；本文分别记录。

## 2. 源码确认的接口事实

以下是旧系统内部路径和源码字段摘录，不是本项目新增或已批准的接口契约。网关入口、部署状态与开放给新项目的身份尚未联调。

| 能力 | 源码可确认的契约 | 证据 |
|---|---|---|
| 指定资料版本 | `GET /api/course/internal/materials/{material_id}/versions/{version_no}`；要求 service 角色与 `course:material:source:read`；只返回当前正式发布版本 | [路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:79)、[发布校验](D:/zljy/zljy/services/m2_learning/app/modules/course/material_source_service.py:25)、[服务权限](D:/zljy/zljy/services/m2_learning/app/modules/course/material_source_service.py:130) |
| 资料元数据字段 | `material_id/source_type/source_id/source_revision/course_id/chapter_id/version_id/version_no/material_name/material_type/original_filename/content_type/file_size/content_hash/content_path`；其中部分字段可空，以原 DTO 为准 | [CourseMaterialSourceVersionOut](D:/zljy/zljy/packages/contracts/src/contracts/m2/course.py:363) |
| 资料文件读取 | `GET /api/course/internal/materials/{material_id}/versions/{version_no}/content`；复用正式版本校验，仅允许受管文件；M6 客户端接收 max_bytes 参数进行限量读取 | [文件路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:96)、[流式客户端](D:/zljy/zljy/services/m6_ai_multimodal/app/clients/m2_course_material.py:40) |
| 资料发现 / 领取 | 有 `GET /api/course/internal/rag/content-manifest`，支持 updated_since；有 `POST /api/course/internal/material-outbox/claim`，limit 为 1–100，默认 20；清单路由返回列表，未声明分页参数 | [领取路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:27)、[清单路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:48) |
| 同步结果回写 | `POST /api/course/internal/rag/content-outbox/result`；入参 event_id、success、可选 error，error 上限 2000 字符 | [结果路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:37)、[结果 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m2/course.py:355) |
| 实时权益 | `GET /api/package/entitlement/course-access`；资源查询使用 `/api/package/entitlement/resource-access`，resource_type 只接受 course 或 practice_paper；grants 含 resource_tenant_id 和 resource_id | [权益路由](D:/zljy/zljy/services/m3_commerce/app/modules/package/router.py:153)、[权益 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m3/package.py:82) |
| 超时与错误处理 | 资料客户端配置 90 秒超时；通用内部请求分离 HTTP 错误和业务包络错误，当前 request 方法未包含自动重试循环；不能据此断言整个 Outbox 管线没有重试 | [资料客户端](D:/zljy/zljy/services/m6_ai_multimodal/app/clients/m2_course_material.py:16)、[通用请求](D:/zljy/zljy/packages/internal_client/src/internal_client/client.py:46) |
| 错误枚举 | 可定位 `M1.AUTH.FORBIDDEN`、`M2.COURSE.MATERIAL_NOT_FOUND`、`M6.RAG.INVALID_FILE`；完整 HTTP 状态与对外错误映射仍需契约核对，不能把枚举附带数值直接当作 HTTP 状态 | [错误枚举](D:/zljy/zljy/packages/contracts/src/contracts/errors.py:6) |

未确认的技术事项：订单面向管理报表的完整查询契约、审计完整事件矩阵、各服务实际限流/SLA、新服务认证注册及密钥配置流程、重试与故障补偿全链路。未启动服务导出 OpenAPI，未读取密钥或环境文件。

## 3. 权限、退款与格式的技术结论

| 核验点 | 可确认的事实 | 限制 / 待确认 |
|---|---|---|
| 课程准入 | M2 测试验证无课程权益时拒绝、未发布资料不可下载、视频读取复用课程权益、资料列表筛除未发布项 | 不代表已验证新项目所有检索、引用与缓存访问入口 |
| 知识库授权 | M6 测试验证自有/租户授权/机构授权按数据租户分组；机构服务失败时机构授权关闭；请求含未授权知识库时整体拒绝 | 不代表确认新项目允许跨租户访问；账号租户、数据租户及可接受授权关系仍需业务确认 |
| 部分退款 | 源码在累计退款等于支付额，或 scope 为 full / order_item 时调用按订单来源撤权方法 | 属于现有代码行为，不能将其简化成“任何部分退款都撤权”或“部分退款均不撤权”；本次单测未完整覆盖这张撤权矩阵 |
| 购买与协议 | 选定套餐/交易测试含协议未完成不能后台开课、重复支付回调不重复产生权益等行为 | 生效时限、续费计算与在途请求失效仍无完整验收证据 |
| 解析 | DOCX 标题/表格、PPTX 页号、文本结构及恶意 XLSX XML 拒绝等选定单测通过 | PDF 的 MinerU 和 DOC 的 LibreOffice 被 Mock；不构成实际 OCR、外部转换器或真实样本质量验证 |
| 课程同步格式 | 事件集合是 pdf、docx、xlsx、txt、md、csv、html、json，含来源 ID、版本、哈希和同租户服务权限校验 | 不包含 pptx、xls、ppt、独立图片、mp4；不能以手动上传支持 PPTX 推导课程同步支持 PPTX |
| 消息与邮件 | 已核对的 MessageDeliveryService 对非 in_app 通道记录失败并返回“当前通道未接入” | 该消息投递流程不能直接承担本期邮件发送；其他独立邮件服务是否存在、可开放，仍需负责人提供入口 |

新增源码证据：

- [部分退款与撤权分支](D:/zljy/zljy/services/m3_commerce/app/modules/order/settlement_service.py:211)。
- [课程资料事件集合及 DTO](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/rag_ingestion/course_material_contract.py:13)。
- [消息投递通道分支](D:/zljy/zljy/services/m5_operations/app/modules/message/delivery.py:36)。

## 4. 测试结果与复现

| 分组 | 选定测试文件（相对旧仓库根目录） | 最终结果 |
|---|---|---|
| M6 | services/m6_ai_multimodal/tests/unit/test_course_material_sync.py；test_rag_parsers_models.py；rag_retrieval/test_access_service.py（后两项与前项同属 tests/unit） | 30 passed |
| M2 | services/m2_learning/tests/unit/test_course_access.py | 5 passed |
| M3 | services/m3_commerce/tests/unit/test_refund_service.py；test_cross_tenant_settlement_service.py；test_package_service.py（后两项与前项同属 tests/unit） | 42 passed |

实际测试进程由 uv 选择 **Python 3.13.16**；不安装/同步依赖，不加载 .env；禁用字节码和 pytest 缓存，禁用自动加载第三方测试插件，仅显式加载 pytest-asyncio。每个服务使用独立进程，避免同名 app 包相互污染。测试中用替身处理数据库和服务调用，不启动旧系统。

首轮隔离器拦截全部 socket.connect，连 Windows 异步循环创建自身 socketpair 也被拦截，产生 25 passed、3 failed、2 errors。已修正**验证隔离器**，只允许原始 socketpair 构造期间的回环连接，继续拒绝普通服务连接；未修改、跳过或弱化旧项目测试。完整重跑得到上表 77 passed。

uv 首次执行另遇缓存落盘拒绝访问，获自动审批后在同样的离线、禁用环境文件条件下执行成功；此限制不是产品测试失败。

复现用 PowerShell：先定义以下验证器，再按三个分组分别调用。命令应在当前项目目录运行，以保持工作目录与此次核验一致。

```powershell
$verificationCode = @'
import os, socket, sys
sys.dont_write_bytecode = True
for name in tuple(os.environ):
    if name.startswith(('M1_', 'M2_', 'M3_', 'M4_', 'M5_', 'M6_', 'RAG_', 'APP_ENV', 'PYTEST_')):
        os.environ.pop(name, None)
os.environ['PYTEST_DISABLE_PLUGIN_AUTOLOAD'] = '1'
os.environ['M6_DEPLOYMENT_ENVIRONMENT'] = 'test'
original_connect = socket.socket.connect
original_pair = socket.socketpair
creating_pair = False
def local_pair(*args, **kwargs):
    global creating_pair
    creating_pair = True
    try:
        return original_pair(*args, **kwargs)
    finally:
        creating_pair = False
def guarded_connect(sock, address):
    if creating_pair and address[0] in ('127.0.0.1', '::1'):
        return original_connect(sock, address)
    raise RuntimeError('Verification forbids service/network connections')
socket.socketpair = local_pair
socket.socket.connect = guarded_connect
socket.socket.connect_ex = lambda *args, **kwargs: (_ for _ in ()).throw(RuntimeError('Network forbidden'))
import pytest
print('Runtime:', sys.version.split()[0])
raise SystemExit(pytest.main(['-q', '-p', 'no:cacheprovider', '-p', 'pytest_asyncio.plugin', '--tb=short', *sys.argv[1:]]))
'@
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/m6_ai_multimodal/tests/unit/test_course_material_sync.py' 'D:/zljy/zljy/services/m6_ai_multimodal/tests/unit/test_rag_parsers_models.py' 'D:/zljy/zljy/services/m6_ai_multimodal/tests/unit/rag_retrieval/test_access_service.py'
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/m2_learning/tests/unit/test_course_access.py'
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/m3_commerce/tests/unit/test_refund_service.py' 'D:/zljy/zljy/services/m3_commerce/tests/unit/test_cross_tenant_settlement_service.py' 'D:/zljy/zljy/services/m3_commerce/tests/unit/test_package_service.py'
```

未执行全仓测试、真实 HTTP 联调、OCR/ASR、生产性能或 SLA 验证；文档变更不涉及业务代码，未执行全仓 ruff/mypy。后续实现与合并仍执行项目门禁。

文档验证：本次四份文档共 39 处本地链接的目标存在性及行号范围检查通过，`git diff --check` 通过；仅更新当前项目文档，未修改旧项目源码或测试。

## 5. 供负责人确认的最小问题集

下表保留原始选项；2026-10-28 用户先选择 A、B、A（D-008），随后回复“好的，按你说的来”，接受第 6 节建议（D-009 至 D-011）。规则与目标已确认；接口开放、服务承诺、租户映射、样本与实测证据仍需补齐。

| 问题 | 明确选项 / 需填写内容 | 确认方 |
|---|---|---|
| XLS / PPT 的含义 | A：必须支持 .xls/.ppt，同时支持 .xlsx/.pptx；B：业务名称泛指表格/幻灯片，实际只需 .xlsx/.pptx。B 涉及对 D-006 范围的澄清，必须书面确认后再更新决策 | 业务 / 内容负责人 |
| MP4 检索内容 | A：语音转写并按时间戳引用；B：语音转写 + 画面文字/内容检索，并按时间戳引用；两者均保持 MP4 首期交付 | 业务 / 内容负责人 |
| 续费与部分退款 | 续费选择 A：原到期时间叠加，或 B：按新购有效期重算；另填写过期后起算规则、部分退款对应撤销哪些权益，不以现有分支代替业务确认 | 业务 / 订单负责人 |
| 生效与失效 | 分别填写支付/签署/退款/到期后允许的最长生效或失效延迟，以及在途回答、已有引用的处理规则 | 业务 / 权限负责人 |
| 文件验收 | 提供六类可用于测试的脱敏真实样本位置，逐类大小/时长上限、OCR/ASR 语言与质量标准 | 业务 / 内容负责人 |
| 服务对接 | 指定邮件服务入口或负责人；确认新服务身份、接口契约与限流/SLA，不能通过业务人员勾选代替技术方服务承诺 | 旧系统技术负责人 |

平台账号租户与跨机构资源授权的映射仍需单独确认，不接受前端或模型指定可信租户。若需修改旧系统以补齐邮件、格式或其他能力，先执行 B0–B2，不能直接纳入新项目实现。

## 6. 后续事项建议（已接受为项目规则与目标）

以下保留原建议明细。用户于 2026-10-28 回复“好的，按你说的来”，已接受为本项目设计与验收目标，权威记录为 D-009 至 D-011；**不表示旧系统实测能力、第三方服务承诺或准入证据已经齐备**。以下“建议”指建议形成时的表述，当前状态以本段为准。

### 6.1 生效、失效与未决续费规则

| 项目 | 推荐规则与目标 | 原因 / 验收边界 |
|---|---|---|
| 支付 / 签署生效 | 必要支付和协议条件均满足、权威权益服务确认有效后开放；从最后一个必要条件成功到可查询有效权益的目标不超过 60 秒 | 处理中的状态不给受限内容；延迟目标需旧系统负责人确认并在测试环境计时 |
| 退款失效 | 从退款成功到权威撤权完成目标不超过 30 秒；撤权后下一次请求拒绝，权限状态无法确认时拒绝受限请求 | 不以等待 RAG 内容重建作为撤权条件；30 秒是建议上限，非现有承诺 |
| 到期 | 服务端以权威 end_at 为准，当前时间达到 end_at 即拒绝，无缓存宽限期 | 校验边界时刻和时区；建议存储 UTC、界面按业务时区显示，具体契约另行确认 |
| 缓存与在途回答 | 查询、缓存命中、引用打开均重新确认授权；回答发送前再次校验，流式输出每次发送前核对授权状态；失效后停止继续发送受限正文和引用 | 已送达客户端的文字无法追回；本规则约束后续输出和访问。首期不自行引入可放行的过期授权缓存 |
| 已过期后续费 | 以 max(原到期时间, 续费生效时间) 为起点叠加购买时长；原权益无到期时间时不自行叠加，按既有规则人工确认 | 避免续费时长被过期空窗消耗；D-009 已确认 |
| 部分退款 | 建议按明确的订单项撤销所退课程/资源；金额调整而未退资源时保留权益；无法明确映射时转人工 | 不从退款金额比例猜测课程范围；旧代码 order_item 分支调用按订单来源撤权，需技术方确认是否满足此目标，未经 B0–B2 不改造 |

### 6.2 文件限制与验收

| 类别 | 建议首期单文件上限 | 补充条件 |
|---|---|---|
| PDF、DOCX、XLS/XLSX、PPT/PPTX | 各 200 MiB | 解析展开后的大小、页/行/单元格数与处理时间仍需按样本评估，不能只限制上传字节数 |
| 独立图片 | 20 MiB | 首期建议 JPEG/PNG；超大像素图需单独限制，像素阈值通过压测后确认 |
| MP4 | 1 GiB 且时长不超过 120 分钟 | 分片上传、分段处理；语音与画面两路均纳入验收；性能与费用未实测 |

- 语言：首期建议普通话及简体中文，兼容英文文字；方言、繁体和其他语言列入单独评估，不作未经验证的质量承诺。
- 样本：每类至少 10 份脱敏真实样本；表格至少 5 份 XLS + 5 份 XLSX，幻灯片至少 5 份 PPT + 5 份 PPTX。另加损坏、超限和类型伪装样本，不计入正常样本数量；自动生成夹具只补充边界测试。
- OCR 建议门槛：在人工标注、忽略空白排版差异后的基准文本上，清晰印刷体字符错误率不超过 2%，普通扫描件不超过 5%；模糊/手写单独分层报告。字符错误率按替换、删除、插入错误总数除以标准文本字符数计算。
- ASR 建议门槛：在人工标注的清晰普通话样本上，归一化字符错误率不超过 10%；噪声、专业词和中英混读分别报告，不用总体均值掩盖失效场景。
- 视频检索：准备至少 20 个答案只在语音中、20 个只在画面中的标注问题，两组分别要求前 5 条召回命中率不低于 90%；可核对的引用时间点偏差建议不超过 5 秒。
- 引用：所有返回的引用均能定位到正确文档页/幻灯片/表格区域/图片或视频时间段；抽样引用定位正确率要求 100%。质量不足时明确失败或提示证据不足，不编造答案或引用。
- 上述阈值已由 D-010 确认为验收目标，样本位置和使用授权仍需内容负责人提供；没有真实样本就不能关闭 G-004。

### 6.3 服务对接与可靠性

- 邮件：优先由旧系统负责人指定已有邮件 API，复用模板与发送记录；若没有，先作为独立缺口记录并使用 Mock。不得把站内信接口改称邮件服务，也不在本项目中自行重建邮件主服务。
- 身份：独立服务使用经批准的服务身份，保留终端用户授权上下文；课程、知识库、租户及收件人均由服务端确认。优先由权威权益服务返回资源租户和资源集合，不由模型选择。
- 超时：普通查询建议连接超时 3 秒、响应读取超时 10 秒；资料流式下载读取超时可先沿用旧客户端 90 秒，但需另设可配置总耗时与大小上限。读取超时不是总耗时。
- 重试：建议暂时性网络故障、429、可重试 5xx 最多共 3 次尝试，间隔 1 秒、3 秒并加随机扰动；429 优先尊重 Retry-After 且受总时间预算限制。鉴权拒绝和请求校验错误不重试。写调用只有在确认幂等键和服务端去重后才能重试。
- 限流：建议普通只读查询从每用户每分钟 60 次、每租户每分钟 600 次开始评估；上传、导出、模型请求和内部回调分别定额，不套用同一额度。最终额度依压测和旧系统承载确认。
- 可用性：建议对外知识查询月可用性目标 99.9%；如何计量、故障归属与上游预算须由负责人确认。该数字不表示旧系统已承诺 SLA。
- 验收：成功、超时、429、权限拒绝、服务不可用、重复回调与重试去重均需测试；邮件、报表等操作另验证收件人、租户和审计上下文。

设计原则参考 [OWASP 授权指南](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)的逐请求权限校验，以及 [OWASP 文件上传指南](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)的类型/内容/大小校验。具体秒数、容量、质量、限流和可用性目标是本项目的建议值，不是这些指南规定的数值。
