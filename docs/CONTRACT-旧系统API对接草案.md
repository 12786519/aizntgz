# 旧系统 API 对接契约草案

> 日期：2026-10-28 ｜ 整理者：Codex ｜ 状态：源码契约已核对；新服务接入待确认与联调。

> 最新范围：用户已确认邮件先不做（D-015）。下文 MAIL01 保留为历史查找与未来恢复依据，邮件入口、Mock、投递及相关验收不再属于本期要求或阻断项。

## 1. 适用边界

本稿补齐课程、订单、权益、机构范围和审计的现有路径、字段、鉴权及失败处理。邮件没有可核实的可用发送接口，明确保留缺口；不编造路径。管理报表指标及查询接口继续受 G-005 约束，不用学员本人订单接口替代管理报表。

2026-10-28 源码入口补充：[旧系统接口索引](API-旧系统接口索引.md)已核对 services/zljy_monolith 的实际注册关系，列出单体课程/权益/订单/进度/统计/审计/消息路由及网关映射。下文保留此前拆分服务证据，不能据此假定部署必然采用拆分版本；具体单体与网关入口以索引及后续部署确认分别记录。本轮只找接口，不恢复已暂缓的环境与联调。

旧系统 M1–M6 是旧系统域编号，与本项目 M1 入库、M2 检索、M3 智能体编号不是同一含义。下文所有 `/api/...` 均为旧域内部源码路径，BFF 工具入口除外；部署基址、对新项目开放的网关映射均待技术方确认，本文不指定生产地址。

```text
新项目服务端（主体身份不由模型指定）
  → 经批准的旧网关/内部调用入口
  → 固定课程、权益、订单、机构范围、审计 API
  → 校验响应与授权证据
  → 本项目 integrations / 授权解析
```

## 2. 通用传输、身份和响应

- 现有内部调用使用 Authorization Bearer 凭证；域依赖、具体路由 scope 和业务角色校验共同决定权限。不能把读取课程源资料的服务身份用于面向学员的内容检索授权。
- 当前 `RequestContext` 有 user_id、tenant_id、roles、scopes、permission_version、可选 display_name 和 request_id；trace_id 由观测中间件的 traceparent 单独维护，不在此 DTO 中。
- 传递 `X-Request-ID` 和 `traceparent`，新项目在可信入口验证/生成追踪上下文；不在日志或契约样例记录真实 Bearer 值、密钥或用户资料。
- JSON 成功响应为 `code: 200, message: "ok", data: <DTO>, request_id: <可空字符串>`；业务错误 code 为命名空间字符串，data 为空。文件内容接口返回字节流，不套 JSON 包络。
- 认证依赖可能返回 HTTPException 的 detail，框架参数校验可能返回 422 detail；不能假设所有非 2xx 都能解包为 Envelope。适配器需分别处理 HTTP、包络与 DTO 解析失败。
- 本项目已确认的连接/读取超时、重试与初始限流值见 D-011；总耗时预算、各上游真实限流、Retry-After 语义和 SLA 尚未获对接承诺，不冒充现有能力。

证据：[上下文 DTO](D:/zljy/zljy/packages/auth/src/auth/context.py:10)、[统一包络](D:/zljy/zljy/packages/contracts/src/contracts/envelope.py:8)、[鉴权依赖](D:/zljy/zljy/packages/auth/src/auth/deps.py:63)、[追踪中间件](D:/zljy/zljy/packages/observability/src/observability/middleware.py:15)。

## 3. 课程资料（旧 M2）

| 编号 | 方法与内部路径（前缀 /api/course） | 输入 | 输出 / 校验 |
|---|---|---|---|
| C01 | GET /internal/rag/content-manifest | 可选 updated_since: datetime | CourseMaterialSourceVersionOut 列表；路由无分页参数，增量水位/并发更新一致性需联调确认 |
| C02 | GET /internal/materials/{material_id}/versions/{version_no} | material_id: UUID；version_no: int | 当前仍正式发布的版本元数据；校验课程、资料和版本的发布状态及 published_version_id |
| C03 | GET /internal/materials/{material_id}/versions/{version_no}/content | 同 C02 | 受管文件字节流；存在 /internal/rag/materials/.../content 别名；不接受模型指定任意 URL |
| C04 | POST /internal/material-outbox/claim | query limit: int，1–100，默认 20 | CourseContentOutboxTaskOut 列表；会领取任务，虽用于同步也属于有副作用调用 |
| C05 | POST /internal/rag/content-outbox/result | event_id: UUID；success: bool；error?: str，最长 2000 | 状态结果；另有 /internal/material-outbox/result 别名 |

身份：C01–C03 要求 service 角色且含 `course:material:source:read`。C04/C05 的旧实现拒绝条件是“既无 service 角色，又无 course:outbox:dispatch scope”，即旧代码接受两者之一；本项目仍需技术方分配最小权限，不能把该条件擅改成已存在的 AND 校验。

元数据 DTO 字段：material_id、source_type（固定 course_material）、source_id、source_revision、course_id、chapter_id（可空）、version_id、version_no、material_name、material_type、original_filename、content_type（可空）、file_size、content_hash（可空）、content_path。source_revision 是内容修订标识，不能与文件 version_no 混用。

Outbox 任务字段：event_id、event_type、aggregate_type、aggregate_id、source_type、source_id、source_revision、event_payload、retry_count、trace_id（可空）。event_payload 仍需按具体事件 DTO 校验，不能作为可信身份或任意 HTTP 地址使用。

C05 首次结果含 event_id、event_status、retry_count、next_retry_at；若已 published，重复回写只返回 event_id、event_status，调用方不能强制要求另外两项始终存在。success=true 后清空失败与重试时间；success=false 会增加 retry_count 并计算下一重试时间，源码为 `min(2 ** min(retry_count, 12), 21600)` 秒，按当前公式实际最多 4096 秒。

**重试边界**：C04 会领取任务，C05 失败回写会重复计数，均不得作为普通 GET 自动重试。C05 成功响应丢失可按确认后的相同 event_id 成功回写；失败响应丢失需先通过既有领取/状态机制对账，缺少可用状态依据时转人工，不补造状态接口。幂等、租约、最大重试与死信的端到端保证仍需联调。

证据：[路由](D:/zljy/zljy/services/m2_learning/app/modules/course/material_sync_router.py:27)、[源资料权限及发布校验](D:/zljy/zljy/services/m2_learning/app/modules/course/material_source_service.py:25)、[DTO](D:/zljy/zljy/packages/contracts/src/contracts/m2/course.py:341)、[领取与回写实现](D:/zljy/zljy/services/m2_learning/app/modules/course/material_outbox_service.py:24)。

## 4. 学员订单与实时权益（旧 M3）

| 编号 | 方法与路径 | 输入及权限 | 输出 |
|---|---|---|---|
| O01 | GET /api/order/student/overview | 无身份参数；路由依赖验证 aud=m3、order:self:read 和配置 kid；业务按可信学员主体查询本人 | CommerceOverviewOut：orders、entitlements、默认空列表 entitlement_groups |
| P01 | GET /api/package/entitlement/course-access | 无身份参数；要求 student 角色，用户来自 ctx | CourseAccessOut：course_ids、grants；每条 grant 为 resource_tenant_id + resource_id |
| P02 | GET /api/package/entitlement/resource-access | resource_type 仅 course 或 practice_paper；其他身份同 P01 | ResourceAccessOut：resource_type、resource_ids、grants |

O01 orders 字段包含 id、order_no、product_type、package_id/bundle_id、package_name、product_tenant_id、product_org_id/name、order_amount、pay_amount、order_status、pay_channel、paid_at、order_expires_at、created_at；字段类型、可空性以 DTO 为准，不假设金额单位已由字段名确定。

entitlements 字段：id、package_id、resource_type、resource_id、resource_name、start_at、end_at、entitlement_status、source_type、resource_tenant_id、product_org_id。资源租户字段可空；空值不能退回使用账号 tenant_id。概览仅能说明本人订单/权益，不可跳过协议判断当作最终内容授权；P01/P02 的实时有效查询包含协议条件，应与知识库权限及文档状态共同校验。

P01/P02 当前 ContextDep 使用通用内部验签，不能照搬 O01 的特定 audience/scope 保证；其对新服务开放时应采用何种受控入口尚需技术方确认。不得自签主体凭证以绕过注册流程。

证据：[O01 路由](D:/zljy/zljy/services/m3_commerce/app/modules/order/router.py:141)、[O01 鉴权](D:/zljy/zljy/services/m3_commerce/app/dependencies.py:19)、[权益路由](D:/zljy/zljy/services/m3_commerce/app/modules/package/router.py:153)、[订单与权益 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m3/order.py:91)、[实时权益实现](D:/zljy/zljy/services/m3_commerce/app/modules/package/service.py:395)、[协议与时间条件](D:/zljy/zljy/services/m3_commerce/app/modules/package/repository.py:468)。

## 5. 机构范围与审计（旧 M1）

| 编号 | 方法与路径 | 输入及权限 | 输出 / 注意事项 |
|---|---|---|---|
| A01 | GET /api/user/internal/knowledge-institution-scope | 无外部机构参数；aud=m1 依赖；业务要求非平台主体、m6:knowledge:read 及活跃用户 | institution_ids、permission_version；机构 ID 由旧 M1 规范化，不按 tenant_id 字符串猜测 |
| A02 | GET /api/user/internal/student-primary-organization-scope | 无外部机构参数；专用身份依赖；仅平台学员 | is_bound、可空 organization_id / organization_tenant_id；机构归属不等于课程权益 |
| A03 | GET /api/user/internal/student-customer-service-policy | 无外部机构参数；专用身份依赖 | service_tenant_id、institution_id、use_platform_knowledge；这是客服策略，不是平台知识库无条件授权 |
| AU01 | POST /api/audit/event | service 角色且含 m1:audit:event:create；租户与 operator_id 取 ctx | Envelope.data={recorded:true}；调用方提供稳定 source_event_id |

AU01 请求：source_event_id: UUID；event_type: 1–64 字符；event_result: 最大 32，默认 success；target_type?: 最大 64；target_id?: UUID；detail?: object；client_ip?: 最大 64；user_agent?: 最大 1024。禁止在 detail 放密钥、原始凭证或未脱敏业务资料。

旧审计实现先按租户和 source_event_id 构造事务级 PostgreSQL advisory lock，再查找去重后追加，已有并发串行化的源码保护；本次未连接数据库验证真实隔离级别、事务提交和竞争场景，不声称实测严格一次写入。重试前要以契约与测试确认端到端语义；本项目本地审计 trace_id、上游 request_id 与 source_event_id 分别保留关联，不互相替代。

证据：[M1 路由](D:/zljy/zljy/services/m1_governance/app/modules/user/access_context_router.py:28)、[主体机构计算](D:/zljy/zljy/services/m1_governance/app/modules/user/access_context_service.py:42)、[身份映射 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m1/access_context.py:9)、[审计路由](D:/zljy/zljy/services/m1_governance/app/modules/audit/router.py:50)、[审计 DTO](D:/zljy/zljy/packages/contracts/src/contracts/m1/governance.py:819)、[审计实现](D:/zljy/zljy/services/m1_governance/app/modules/audit/service.py:182)。

## 6. 受控工具入口与邮件缺口

旧 BFF 已有 `POST /internal/agent-tools/{capability}`，外层与 arguments 均禁止额外参数；当前本人查询用 `{ "arguments": {} }`。query_my_commerce 固定到 O01，query_my_learning_records 固定到旧 M4 `/api/edu/student/learning-record`（page=1,size=20）。目标域 URL、身份和权限不可由模型选择。

当前白名单仅允许 caller_service=m6、student 角色和平台租户；要求请求 X-Request-ID 等于委托 request_id。委托采用签名、audience、kid、exp 和 capability 校验，再换发目标域令牌；源码有 jti，但本入口未展示已消费 jti 存储，不能据此承诺同一凭证严格单次使用。新项目服务不能把自己命名为 m6 或复用旧私钥取得权限，需技术方确认接入拓扑。

MAIL01：**尚无可用契约**。已核对通用消息投递对非 in_app 返回“当前通道未接入”。缺少邮件入口、收件人服务端解析、模板/附件、幂等、状态回执、超时、错误码及 SLA 时，遵照 D-011 使用 Mock；不新增臆造的真实发送接口。

证据：[工具入参](D:/zljy/zljy/packages/contracts/src/contracts/internal_tools.py:8)、[白名单及校验](D:/zljy/zljy/services/gateway/api_bff/app/agent_tools.py:53)、[委托签发边界](D:/zljy/zljy/services/gateway/api_bff/app/delegation.py:20)、[委托验签](D:/zljy/zljy/packages/auth/src/auth/delegation.py:72)、[消息投递缺口](D:/zljy/zljy/services/m5_operations/app/modules/message/delivery.py:36)。

## 7. 错误、兼容与验收矩阵

| 场景 | 已核对的行为 / 本项目约束 |
|---|---|
| 域认证失败 / 权限不足 | 认证依赖通常为 401 / 403；读取资料的 AUTH_FORBIDDEN 映射 403，M2.COURSE.MATERIAL_NOT_FOUND 映射 404 |
| Outbox 找不到事件 | M2.COURSE.MATERIAL_OUTBOX_NOT_FOUND；当前 M2 状态映射未列入 404 集合，落入默认 400，不能根据名字猜 HTTP 状态 |
| 工具委托失败 | 缺失/无效/过期为 401；能力、caller、角色、租户或 request_id 不符为 403；多余 arguments 为 400；上游未配置/不可用为 502 |
| 请求或响应不合法 | 请求错误不重试；响应 DTO/包络解析失败记录受控错误并拒绝授权，不从不完整响应推导权限 |
| 超时 / 429 / 5xx | 依 D-011 区分只读与已证明幂等的写调用；不对所有 POST 统一重试；上游错误摘要不得暴露资料内容 |
| 接口兼容与版本 | 当前无统一版本协商证据；锁定评审后的 DTO、保留别名语义，新增必填字段、字段变义或事件变化须先决策 |

证据：[M2 状态映射](D:/zljy/zljy/services/m2_learning/app/core/exceptions.py:82)、[错误枚举](D:/zljy/zljy/packages/contracts/src/contracts/errors.py:85)、[BFF 错误分支](D:/zljy/zljy/services/gateway/api_bff/app/agent_tools.py:109)。

后续契约测试必须覆盖：正确主体成功、签名/aud/scope 错误、伪造身份参数、跨租户资料、无权益/未签协议/过期、未发布版本、错误码和流式响应、清单增量边界、成功重复与失败重复回写、响应丢失、限流、超时及服务不可用。实际写入和外部调用仅在批准的测试环境进行。

本次仅文档补齐，无接口、数据库、事件或依赖变更，无迁移。未来若变更旧系统，先 B0–B2；回滚需能撤销新身份权限、停止投递/发送并保留审计，不回退为越权访问。

## 8. 评审就绪与仍缺材料

- 已补齐可核实的字段、内部路径、鉴权、错误与重试边界，可用于评审及后续 Mock 契约测试设计。
- 仍需：新服务正式身份/接入入口、完整认证拓扑、MAIL01、旧系统限流/SLA、总请求预算、测试环境联调及端到端幂等证据。
- 管理报表契约和文件处理服务商未在本次确定；G-002/G-003/G-004 不因草案完成而自动关闭。

后续准备进展（2026-10-28）：[接入与联调准备方案](INTEGRATION-接入与联调准备方案.md)已补齐认证拓扑候选、身份注册清单、MAIL01 逻辑要求、总预算建议和分层幂等验证矩阵，并运行 20 项相关既有隔离测试。D-013 仍待评审；正式身份、真实邮件 API、上游限流/SLA 和测试环境端到端证据尚未取得，不以方案完成替代实际开通。
