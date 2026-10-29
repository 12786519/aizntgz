# 身份与租户授权映射草案

> 日期：2026-10-28 ｜ 整理者：Codex ｜ 状态：平台学员到机构课程映射已确认（D-012）；新服务接入及实际验证待完成。

## 1. 主体和资源字段

| 概念 | 权威来源 | 用途与禁止事项 |
|---|---|---|
| user_id | 验签后的 RequestContext | 当前学员/管理员/服务操作人；不能从问题、模型工具参数或页面提示读取 |
| requester_tenant_id | ctx.tenant_id | 账号请求租户，保留原值用于审计与会话 owner，不用资源租户覆盖 |
| resource_tenant_id | 旧 M3 实时有效权益 grant | 课程资源所属租户；需与 resource_id 成对使用，不能只以课程 UUID 授权 |
| institution_id | 旧 M1 机构范围或客服策略 | 规范机构范围；机构 ID、账号 tenant_id、资源 tenant_id 不保证相同 |
| data_tenant_id | 已授权知识库的 owner / 文档所属租户 | RAG 查询必须按此租户分组过滤，再叠加该组 kb_ids 和课程范围；值只能来自服务端授权证据 |
| course_ids | 旧 M3 当前有效且满足协议要求的权益 | 先限制请求可用课程，再查询，不能事后仅从答案删除越权结果 |
| kb_ids | 服务端知识库状态和有效授权记录 | 同一请求内不得与其他数据租户的课程/知识库集合任意交叉组合 |
| permission_version / scope 指纹 | ctx 与服务端解析结果 | 用于追踪、分区及变更检测，不代表本身完成实时撤权或已验证当前权限 |
| request_id / trace_id | 可信入口及观测链路 | 分别关联请求与调用链；不把 ID 当授权凭证 |

来源：[RequestContext](D:/zljy/zljy/packages/auth/src/auth/context.py:10)、[权益 grant](D:/zljy/zljy/packages/contracts/src/contracts/m3/package.py:82)、[机构范围 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m1/access_context.py:9)。

## 2. 旧系统已存在的映射事实

- 平台学员：客服入口要求平台租户且有 student 角色；按本人有效权益生成服务机构范围。无权益时生产默认关闭，源码中的主归属回退属于显式测试开关，不是本项目的可用授权来源。
- 课程权益：旧 M3 从可信 user_id 查询跨机构资源 grant；旧 M2 根据 grant.resource_tenant_id 读取对应课程，并验证正式发布。协议未完成、到期、撤权必须从有效集合排除。
- 机构知识库：旧 M1 校验活跃用户与知识读取 scope，从角色/班级机构关系产生规范机构 ID；不是让 M6 猜测机构。
- 知识授权：旧 RAG 读取自有 KB、租户授权、机构授权，按有效期和状态筛选，再映射为明确 data_tenant_id → kb_ids 分组。机构服务失败时对应授权分支关闭；显式请求含未授权 KB 时不返回部分混合结果冒充成功。
- 学员课程范围：旧学员检索将当前客服绑定、有效权益及课程 ID 进一步组合；平台知识开关只限制可选授权集合，不意味着免费获得平台全部知识。
- 内容状态：旧授权内容仓储还限定活动文档版本与文档状态，授权存在不代表未发布、删除或历史版本可读。

证据：[客服主体校验](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/customer_service/context.py:15)、[课程准入](D:/zljy/zljy/services/m2_learning/app/modules/course/access_service.py:56)、[实时权益及协议](D:/zljy/zljy/services/m3_commerce/app/modules/package/repository.py:468)、[机构解析](D:/zljy/zljy/services/m1_governance/app/modules/user/access_context_service.py:42)、[授权快照](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/rag_retrieval/access/repository.py:75)、[授权分组](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/rag_retrieval/access/service.py:175)、[学员检索链](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/rag_retrieval/student_access.py:60)、[活动版本过滤](D:/zljy/zljy/services/m6_ai_multimodal/app/modules/rag_retrieval/access/content_repository.py:25)。

## 3. 已确认的课程授权映射流程

用户于 2026-10-28 明确确认：平台学员可凭服务端核实的有效课程权益访问对应机构资料，同时按资料所属租户严格过滤。账号租户不被改写，资源范围仅由服务端有效授权确定；课程、协议、知识库权限和活动文档状态均须通过。未经授权的跨租户访问仍禁止。本次不扩展为非课程企业资料或平台知识库的普遍共享。

```text
验签后的主体（账号租户保持不变）
  → 权威实时课程权益 / 协议条件
  → M1 规范机构及客服策略（涉及该授权分支时）
  → 服务端知识库有效状态及显式授权
  → 按数据租户形成独立的课程 + KB 范围
  → 仅在已批准映射范围内召回活动内容
  → 缓存命中 / 输出 / 引用访问重新核验
```

已确认课程映射的约束与实施规则：

1. 客户端可提供课程或 KB 定位条件，但服务端只允许其缩小既有授权；任意请求 tenant_id/user_id 不能改写可信上下文。
2. 课程资料必须同时满足课程权益、协议、KB 授权、文档与版本状态。课程授权与 KB 授权是交集，不是任一通过即放行。
3. 对每个已批准数据租户分别形成 `(data_tenant_id, course_ids, kb_ids)`。查询不得把 A 租户课程集合与 B 租户知识库集合混用；同租户仍要过滤到具体资源，不能只带 tenant_id。
4. 不把请求主体“改成资源所有者”获取权限。旧系统内按授权切换 RLS 的实现只是参考，新项目只能访问自身数据存储，跨服务业务数据继续通过 API。
5. 非课程企业资料的学员访问政策未单独确定前，不以 course_id 为空推导公开；管理员也只能按明确权限与 KB 范围读取。平台管理员不是无限跨租户主体。
6. 无权益、协议未满足、权限服务失败、资源租户空值、过期、撤销、未批准租户映射均不得返回受限片段或引用；M1 分支故障不能放行依赖它的机构授权。
7. 查询授权与副作用权限分离：读取课程源文件的服务权限不等于替学员检索，审计写入权限不等于发送邮件，报表统计工具权限须另经 G-005 确认。
8. 按 D-009 在缓存命中、流式发送和引用访问时重新确认授权；缓存至少区分主体、请求租户、已批准数据租户、课程/KB 范围、文档活动版本。旧 scope 指纹不含全部课程/权益变化，不能作为撤权机制的替代。

## 4. 可执行的后续验收矩阵

| 编号 | 给定条件 / 触发 | 预期 |
|---|---|---|
| AUTH-01 | 未登录、签名无效、aud/kid 不匹配或缺 scope | 鉴权失败，召回与引用读取均不执行 |
| AUTH-02 | 平台学员有资源租户 A 的有效课程 C 与 KB K 授权，且该映射已获批准 | 只查询 A/C/K；审计仍保留原平台账号租户与 user_id |
| AUTH-03 | 有课程权益但无 KB 授权，或有 KB 授权但无课程权益/未签协议 | 两种均拒绝课程资料内容、切片与引用 |
| AUTH-04 | 伪造 tenant_id、机构、course_id 或 KB；拼接另一租户的课程与 KB | 不扩大范围；显式未授权资源请求按约定拒绝且不泄露存在性 |
| AUTH-05 | 到期、撤权或退款后再次访问缓存/引用/继续流式发送 | 按 D-009 拒绝或停止后续受限输出，测量 60/30 秒目标 |
| AUTH-06 | 资源 tenant 为空、账号停用、机构服务失败、主归属但无权益 | 不降级为默认租户或测试回退；依赖失败分支关闭 |
| AUTH-07 | 知识授权到期/撤回、文档删除/停用、版本不活动 | 不召回受限内容，不从旧答案缓存放行 |
| AUTH-08 | 管理角色无专用工具权限，或模型指定收件人/用户身份 | 不调用报表/导出/邮件，不依角色名笼统放行 |
| AUTH-09 | 未注册新服务调用旧 BFF 工具，或请求注入额外身份参数 | 不冒充 m6；白名单或参数校验失败，旧业务接口不执行 |
| AUTH-10 | 相同 request_id 下重放有效委托，或失败回写响应丢失 | 不把现有 jti 或 event_id 当作已证明严格单次语义；须增加针对性契约测试或保留阻断 |

这是未来任务测试设计，不声称以上新项目用例均已实现；旧源码测试只覆盖部分对应行为。

## 5. 本次验证证据

2026-10-28，以旧项目已有环境、Python 3.13.16 独立进程运行以下选定测试；未安装依赖、加载 .env、连接业务服务或修改旧代码。禁用字节码、pytest 缓存和插件自动发现；仅显式加载 pytest-asyncio，保留 Windows 事件循环 socketpair，其余普通 socket 连接被隔离器拒绝。

| 命令中的测试路径（旧仓库根目录下） | 结果 |
|---|---|
| services/m1_governance/tests/unit/test_user_access_context.py | 15 passed |
| services/gateway/api_bff/tests/test_customer_service_tool_proxy.py | 9 passed |
| services/m6_ai_multimodal/tests/unit/customer_service/test_student_access.py | 11 passed |

本次合计 **35 passed**，与前次 77 项分开记录。学员测试包含显式测试回退分支，用于核实现有行为；通过它不代表允许新项目开启回退。BFF 测试使用运行时内存测试密钥与替身上游，未读取、打印或保存实际密钥。

复现命令如下，在当前项目目录使用 PowerShell 执行；三组各启动独立 Python 进程。与前次核验说明见 [技术核验报告](VERIFICATION-旧系统技术核验.md)。

```powershell
$verificationCode = @'
import os, socket, sys
sys.dont_write_bytecode = True
for name in tuple(os.environ):
    if name.startswith(('M1_', 'M2_', 'M3_', 'M4_', 'M5_', 'M6_', 'RAG_', 'BFF_', 'APP_ENV', 'PYTEST_', 'INTERNAL_', 'AUTH_INTERNAL_')):
        os.environ.pop(name, None)
os.environ['PYTEST_DISABLE_PLUGIN_AUTOLOAD'] = '1'
os.environ['M6_DEPLOYMENT_ENVIRONMENT'] = 'test'
sys.path.insert(0, sys.argv[1])
original_connect, original_pair = socket.socket.connect, socket.socketpair
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
    raise RuntimeError('Verification forbids service connections')
socket.socketpair = local_pair
socket.socket.connect = guarded_connect
socket.socket.connect_ex = lambda *a, **kw: (_ for _ in ()).throw(RuntimeError('Network forbidden'))
import pytest
print('Runtime:', sys.version.split()[0])
raise SystemExit(pytest.main(['-q', '-p', 'no:cacheprovider', '-p', 'pytest_asyncio.plugin', '--tb=short', *sys.argv[2:]]))
'@
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/m1_governance' 'D:/zljy/zljy/services/m1_governance/tests/unit/test_user_access_context.py'
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/gateway/api_bff' 'D:/zljy/zljy/services/gateway/api_bff/tests/test_customer_service_tool_proxy.py'
$verificationCode | uv run --offline --no-sync --no-env-file --project 'D:\zljy\zljy' python -B - 'D:/zljy/zljy/services/m6_ai_multimodal' 'D:/zljy/zljy/services/m6_ai_multimodal/tests/unit/customer_service/test_student_access.py'
```

未执行真实接口/生产数据验证、撤权时间测量、并发幂等验证、全仓测试、ruff 或 mypy；此次仅修改 Markdown 文档。

文档检查：六份本次文档共 53 处本地链接及行号范围检查通过，Markdown 代码块配对与 `git diff --check` 通过。

## 6. 评审前剩余明确结论

1. 平台账号到本人已授权机构课程的可信资源租户映射已由 D-012 确认，不再重复提问；非课程企业资料与平台知识库的普遍共享不在本次确认范围，仍按明确权限和既有边界处理。
2. 技术方确认新服务在旧网关的身份、白名单、受众和认证路径，不能直接套用 m6。详见 [对接契约](CONTRACT-旧系统API对接草案.md)。
3. 核对旧部分退款实现是否满足 D-009 的订单项范围；不符合时进入 B0–B2 改造分析，不在此直接修改。
4. 补齐测试环境、真实样本、邮件 API 以及 SLA 等外部证据后，再关闭相关门禁。
