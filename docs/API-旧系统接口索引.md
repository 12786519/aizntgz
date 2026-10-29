# 旧系统接口源码索引

> 日期：2026-10-28 ｜ 整理者：Codex ｜ 来源：D:/zljy/zljy ｜ 方式：只读静态查找，不启动、不安装、不联调。

## 1. 入口已定位

仓库同时包含 `services/m1_governance` 等拆分服务和 `services/zljy_monolith` 单体实现。本次优先核对**单体入口确实挂载的路由**，不从重复文件名推断生产部署使用哪一套。

- 单体入口：[app/main.py](D:/zljy/zljy/services/zljy_monolith/app/main.py:355)。业务路由直接挂载，路径如 `/api/course/...`。
- 课程子路由组合：[course/router.py](D:/zljy/zljy/services/zljy_monolith/app/modules/course/router.py:37)。资料同步和资料管理继承 `/api/course` 前缀。
- 同一单体还挂载旧 BFF 网关：[main.py](D:/zljy/zljy/services/zljy_monolith/app/main.py:426)，匹配 `/api/{full_path:path}`，按 m1–m6 映射到域内路径；例如 `/api/m2/course/...` → `/api/course/...`。
- 映射依据：[proxy_network.py](D:/zljy/zljy/services/zljy_monolith/app/gateway/proxy_network.py:254)与[默认域映射](D:/zljy/zljy/services/zljy_monolith/app/config.py:57)。这是源码规则，不是已证实的部署地址或已开放入口。

以下表格列**业务路由实际声明并经父路由组合后的路径**。不把域内受信凭证与用户登录凭证混用；是否通过网关开放需逐接口确认。已明确 `/api/m1/audit/event` 在网关的 INTERNAL_ONLY_PATHS 中被拒绝，不能用加 m1 前缀的方式调用审计写接口。

证据：[网关入口限制](D:/zljy/zljy/services/zljy_monolith/app/gateway/proxy.py:167)、[审计内部路径限制](D:/zljy/zljy/services/zljy_monolith/app/gateway/proxy_constants.py:94)。

## 2. 课程资料与 RAG 同步

| 用途 | 方法与完整业务路径 | 输入 / 返回摘要 | 源码位置 |
|---|---|---|---|
| 资料清单及增量发现 | GET /api/course/internal/rag/content-manifest | 可选 updated_since；返回已发布来源元数据列表 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:49) |
| 当前正式资料来源 | GET /api/course/internal/rag/sources/course_material/{material_id} | material_id；返回当前正式来源版本 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:64) |
| 指定版本元数据 | GET /api/course/internal/materials/{material_id}/versions/{version_no} | UUID + 版本号；返回课程、文件、修订、哈希、内容路径等 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:79) |
| 指定版本文件 | GET /api/course/internal/materials/{material_id}/versions/{version_no}/content | UUID + 版本号；返回文件流 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:96) |
| 同步事件领取（有副作用） | POST /api/course/internal/material-outbox/claim | query limit，1–100，默认 20；返回待处理事件 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:26) |
| 同步结果回写（有副作用） | POST /api/course/internal/rag/content-outbox/result | event_id、success、可选 error；返回处理状态 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_sync_router.py:38) |
| 课程资料列表 | GET /api/course/{course_id}/material | 课程 ID；按路由及服务权限返回资料 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_router.py:39) |
| 资料版本历史 | GET /api/course/{course_id}/material/{material_id}/versions | 课程、资料 ID；返回版本列表 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_router.py:109) |
| 资料知识同步状态 | GET /api/course/{course_id}/material/{material_id}/knowledge-status | 课程、资料 ID；返回同步状态 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_router.py:126) |
| 课程资料对象清单 | GET /api/course/{course_id}/knowledge-export | 课程 ID；返回 CourseKnowledgeExportOut，要求 CourseResourceManagerDep | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/course/router.py:112) |

资料源接口存在不等于学员可直接读取内部文件：源读取校验 service 角色与 `course:material:source:read`，并复核正式发布版本。参见[服务端校验](D:/zljy/zljy/services/zljy_monolith/app/modules/course/material_source_service.py:130)。Outbox 失败回写重复计数的边界仍按已有契约处理，不能一律自动重试。

## 3. 权益、身份与订单

| 用途 | 方法与完整业务路径 | 输入 / 返回摘要 | 源码位置 |
|---|---|---|---|
| 当前学员可访问课程 | GET /api/package/entitlement/course-access | 无外部 user_id/tenant_id；返回 course_ids 及资源租户 grants | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/package/router.py:180) |
| 当前学员资源权益 | GET /api/package/entitlement/resource-access | resource_type=course 或 practice_paper；返回 resource_ids 与 grants | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/package/router.py:187) |
| 当前主体机构范围 | GET /api/user/internal/knowledge-institution-scope | 服务端主体；返回 institution_ids、permission_version | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/user/access_context_router.py:28) |
| 学员主归属机构 | GET /api/user/internal/student-primary-organization-scope | 服务端主体；返回绑定状态及可空机构/租户 ID | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/user/access_context_router.py:60) |
| 学员客服归属与知识策略 | GET /api/user/internal/student-customer-service-policy | 服务端主体；返回 service_tenant_id、institution_id、use_platform_knowledge | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/user/access_context_router.py:76) |
| 当前学员订单/权益概览 | GET /api/order/student/overview | 无身份参数；返回 CommerceOverviewOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/order/router.py:141) |

统一 ContextDep 对内部凭证验签；部分接口额外使用带 audience、scope、kid 的依赖。订单本人概览要求 aud=m3 与 order:self:read；不能推断所有路由都使用同一组 scope。参见[依赖定义](D:/zljy/zljy/services/zljy_monolith/app/dependencies.py:24)。课程及 KB 权限仍按 D-012 取交集，不因找到订单接口就放行资料。

用户登录路由也可定位：学员 `POST /api/user/student-login`、机构 `POST /api/user/login`、平台 `POST /api/user/platform-login`；外部网关对应 m1 前缀。这些是用户入口，不是新服务身份注册接口。参见[学员登录](D:/zljy/zljy/services/zljy_monolith/app/modules/user/router.py:144)、[机构登录](D:/zljy/zljy/services/zljy_monolith/app/modules/user/router.py:236)、[平台登录](D:/zljy/zljy/services/zljy_monolith/app/modules/user/router.py:271)。

## 4. 学习进度、统计与导出

| 用途 | 方法与完整业务路径 | 输入 / 返回摘要 | 源码位置 |
|---|---|---|---|
| 当前学员学习记录 | GET /api/edu/student/learning-record | page、size、可选 course_id；返回分页 LearningRecordOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/edu_admin/router.py:192) |
| 后台学习记录查询 | GET /api/edu/learning-record | 分页、机构/班级/学员/课程/状态筛选；服务端校验管理权限 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/edu_admin/router.py:132) |
| 学习汇总 | GET /api/analytics/learning-overview | 可选 class_id；StudentLearningOverviewOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:128) |
| 平台报表 | GET /api/analytics/platform-report | 分页、keyword、tenant_status；PlatformReportOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:141) |
| 学员课程学习列表 | GET /api/analytics/student-learning | 分页、班级/学员/课程/状态；分页 StudentLearningOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:168) |
| 课程学习进度 | GET /api/analytics/course-learning-progress | 分页、班级/学员/课程/状态；分页 CourseLearningProgressOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:206) |
| 创建报表导出（有副作用） | POST /api/analytics/exports | ReportExportCreateIn、可选幂等 header；受理返回 202 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:261) |
| 查询导出任务 | GET /api/analytics/exports | 分页、可选 export_status；分页 ReportExportOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:288) |
| 下载导出文件 | GET /api/analytics/exports/{export_id}/download | export_id；鉴权后返回文件 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/analytics/router.py:306) |

这些路由证明已有统计/导出能力入口，不代表满足本项目所有指标和大客户数据口径；G-005 仍需逐项核对。导出不是无副作用查询，后续智能体工具仍须专用权限与白名单。

## 5. 审计、消息与邮件

| 用途 | 方法与完整业务路径 | 输入 / 返回摘要 | 源码位置 |
|---|---|---|---|
| 追加审计（有副作用） | POST /api/audit/event | AuditEventCreateIn；服务端 ctx 提供租户/操作人 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/audit/router.py:50) |
| 审计查询 | GET /api/audit/log | 分页及事件/目标/结果筛选 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/audit/router.py:24) |
| 创建通用消息任务（有副作用） | POST /api/message/tasks | MessageTaskCreateIn；返回 MessageTaskOut | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/message/router.py:156) |
| 消息发送记录 | GET /api/message/send-records | 分页、task_id、send_status；返回发送记录 | [路由](D:/zljy/zljy/services/zljy_monolith/app/modules/message/router.py:189) |

审计写要求 service 角色和 m1:audit:event:create，参见[权限校验](D:/zljy/zljy/services/zljy_monolith/app/modules/audit/service.py:268)。

**MAIL01 尚未找到可用邮件发送实现。** 已核对的单体和旧 M5 消息投递流程只支持 in_app；非该通道会返回“当前通道未接入”。源码中的 email 用户字段不等于邮件 API。检索 smtp/smtplib/aiosmtplib/send_email/send_mail 等符号，未定位独立发送客户端；这不证明部署外部绝对没有其他邮件系统。

证据：[单体投递实现](D:/zljy/zljy/services/zljy_monolith/app/modules/message/delivery.py:36)、[旧 M5 对应实现](D:/zljy/zljy/services/m5_operations/app/modules/message/delivery.py:36)。

## 6. 智能体工具入口

`POST /internal/agent-tools/{capability}` 已在单体注册，query_my_commerce 固定映射到本人订单概览。当前工具白名单的调用方仍是 m6；找到路由不等于独立新服务已经获准调用。参见[入口与白名单](D:/zljy/zljy/services/zljy_monolith/app/gateway/agent_tools.py:53)及[单体注册点](D:/zljy/zljy/services/zljy_monolith/app/main.py:429)。

## 7. 查找结果与验证边界

- 已定位当前需求相关的核心业务接口及单体注册关系，参数与 DTO 摘要可结合 [已有对接契约](CONTRACT-旧系统API对接草案.md)继续细化。
- 本次只解析/阅读源码与 Markdown；未导入旧应用、运行启动钩子、安装依赖、构建环境、请求任何接口或执行任何写操作。
- 未确定实际部署版本、基址、凭证、对新服务开放范围与真实可用性；邮件缺口如实保留。
- 用户暂缓的本地 Mock 与旧测试环境联调仍保持暂缓；源码查找不是恢复联调，也不是验收通过。
- 验证结果：使用 Python 标准库 AST 静态核对表格中 29 条路由的方法、组合前缀、路径及源码行号，10 个单体主入口注册点核对通过；本索引与契约文档共 77 处本地链接/行号检查通过，`git diff --check` 通过。未运行应用测试或 HTTP 联调。
