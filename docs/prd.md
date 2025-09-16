# codex-relay Product Requirements Document (PRD)

## 1. Goals and Background Context

### 1.1 Goals
- 为受地域限制用户提供稳定、合规、可控的 Codex 请求中转能力，显著提升请求成功率（目标 >99%）。
- 在两周内上线 MVP，支持首批 10-20 个活跃租户接入并保持活跃。
- 在确保隐私与合规的前提下，提供端到端低时延体验（示例目标 P95 < 1.2×直连时延）。
- 提供简洁易集成的接入方式（SDK/示例/HTTP 文档），将集成时间控制在 0.5 天以内。
- 建立最小可用的观测能力与统一错误模型，支撑持续迭代与可靠性提升。
- 通过控制面与出口层解耦，将复杂度集中在可演进的控制面，支撑账号池健康度与固定出口 IP 策略。

参考来源：`docs/brief.md` 的 “Goals & Success Metrics”“MVP Scope”“Technical Considerations”。

### 1.2 Background Context
codex-relay 面向无法直接或稳定访问 Codex 的用户群体（个人开发者、小团队、教育与研究机构等），通过“中转站”在前端/客户端与 Codex 之间提供一层代理与控制。当前目标地区用户普遍面临直连失败率高、IP 不稳定、账号共用、限流失衡以及隐私合规不确定性等问题，阻碍了研发效率与业务验证。

本项目拟采用 Cloudflare Workers 为边缘入口，结合 KV/D1/Queues 构建轻量的控制面与数据面分层能力：入站统一鉴权（`X-API-Key`）、请求规范化与安全头策略、统一错误模型与重试、SSE/分块流式透传；对外通过固定出口 IP 代理集群与地域绑定，实现“账号:固定IP=1:N（小 N 冗余）”与就近接入，保障可靠性与可控性。方案优先以低成本与快速落地为原则，先达成 MVP 成功标准，再逐步扩展（多地域自愈与扩缩容、策略引擎、计费与报表等）。

参考来源：`docs/brief.md` 的 “Executive Summary”“Problem Statement”“Proposed Solution”。

### 1.3 Change Log
| Date       | Version | Description                 | Author    |
|------------|---------|-----------------------------|-----------|
| 2025-09-15 | v0.1    | 初始化 PRD：目标与背景草稿 | John (PM) |

## 2. Requirements（需求）

### 2.1 Functional（功能需求，FR）
- FR1: 入站鉴权与租户隔离  
  - 系统必须基于 `X-API-Key` 实现请求鉴权，支撑租户级速率限制、并发控制与配额管理。
- FR2: 请求规范化与安全头策略  
  - 系统必须对上游请求进行规范化与头部策略处理（剥离/改写敏感 headers），输出统一的安全请求形态。
- FR3: 同步 JSON 转发  
  - 系统必须支持将客户端的 JSON 请求可靠转发至 Codex，并将 JSON 响应透传给客户端。
- FR4: 流式响应（SSE/分块）透传  
  - 系统必须支持将 Codex 的流式响应（SSE/分块）稳定、低延迟地透传至客户端。
- FR5: 账号池健康与熔断（基础版）  
  - 系统必须对上游账号进行健康检测与熔断处理：失败快速摘除、指数退避、时间窗禁用与恢复。
- FR6: 出口固定 IP 策略与地域绑定  
  - 系统必须对每个账号维持“账号:固定IP=1:N（小 N 冗余）”的绑定策略，并可按地域就近接入。
- FR7: 统一错误模型与重试  
  - 系统必须提供清晰可判别的统一错误模型，并在可恢复场景下执行可控的重试策略。
- FR8: 观测与审计元数据（不存正文）  
  - 系统必须仅记录必要元数据（时间、区域、状态、延迟、账号ID、用量），默认保留 30 天；不记录请求正文。
- FR9: 最小化接入文档/示例  
  - 系统必须提供最小可用的 SDK/示例或 HTTP 文档，确保典型接入时间 ≤ 0.5 天。
- FR10: 控制面与数据面分层  
  - 系统必须以 Cloudflare Workers（入口）+ KV + D1 + Queues 构建轻量分层，便于演进与扩展。

参考来源：`docs/brief.md` 的 “MVP Scope”“Proposed Solution”“Technical Considerations”。

### 2.2 Non Functional（非功能需求，NFR）
- NFR1: 成功转发率  
  - P0：总体成功转发率 ≥ 99%（MVP 阶段）。
- NFR2: 端到端时延  
  - 目标：P95 延迟 ≤ 1.2×直连基线；流式首字节时间可观测且稳定。
- NFR3: 可用性与稳定性  
  - 边缘入口与控制面具备基础容错与降级能力；账号池故障不影响整体可用性。
- NFR4: 安全与隐私  
  - 严格密钥与路由策略管理；默认不存正文；遵循最小可用观测，支持数据最小化与合规边界。
- NFR5: 可观测性  
  - 暴露基础可观测指标：成功率、SSE 透传成功率、P95/P99 延迟、账号健康度、熔断恢复时间、租户活跃度。
- NFR6: 可维护性与可演进性  
  - 控制面可热更新配置；组件解耦便于替换（例如出口代理或路由策略）。
- NFR7: 成本约束  
  - 初期以最小成本上线；固定 IP 资源优先新加坡地域，逐步扩展；避免不必要的高成本存储与带宽消耗。
- NFR8: 兼容性与可移植性  
  - 与 Codex API 的协议兼容；设计上考虑未来扩展至其他上游 AI 提供商的适配空间。

详尽理由说明（为什么这样拟定）
- 取舍与选择
  - 优先满足 MVP 成功标准与关键差异化：SSE 透传、固定出口 IP 策略、账号健康熔断与统一错误模型，而将复杂策略引擎、自动扩缩容与计费等放入 Post-MVP。
  - 将复杂度集中在控制面，保持数据面与出口层轻量并可替换，从而在两周内实现可运行版本。
- 关键假设
  - 目标用户接受“仅元数据观测，不存正文”的默认策略。
  - 新加坡等优先地域具备可行的固定 IP 与成本空间。
- 需要你关注
  - FR/NFR 是否覆盖你当前 MVP 的“必要”和“充分”范围。
  - 指标阈值（成功率、时延、保留期）是否需要团队标准或历史基线微调。
  - 是否需要补充租户级别的配额/计费约束或 SLA 级别承诺语言（MVP 阶段通常避免硬 SLA）。

## 3. Technical Assumptions（技术假设）

- Repository Structure: Monorepo  
  理由：Workers 入口、控制面配置、D1/KV/Queues 脚本与工具链集中管理，便于快速落地与版本对齐；后续如需拆分，可在同一仓内分目录演进。

- Service Architecture: Serverless（Edge + 控制面轻量分层）  
  理由：入口层采用 Cloudflare Workers（边缘）；控制面与数据面通过 KV/D1/Queues 解耦；出口层为固定 IP 代理集群（可单独运维）。MVP 避免过早的微服务拆分，保持部署与回滚简洁。

- Testing Requirements: Unit + Integration  
  理由：MVP 阶段以稳态可靠为主，覆盖核心逻辑（鉴权、限流、错误模型、SSE 透传关键路径）的单元与集成测试；暂不引入完整金字塔与重型 e2e，但保留手动验证清单。

- Additional Technical Assumptions and Requests
  - 编程语言：TypeScript（Cloudflare Workers 生态成熟、类型安全、工具链完备）。
  - 边缘入口：Cloudflare Workers（支持 TransformStream、SSE 透传能力）。
  - 数据与配置：KV（租户配置/限流计数/动态开关）、D1（账号健康/用量/路由/审计元数据）、Queues（异步任务，如重试与健康检查队列）。
  - 出口策略：固定 IP 代理集群优先部署于新加坡，其后法兰克福/达拉斯；账号:固定IP=1:N（小 N 冗余），不跨账号共享。
  - 观测与日志：仅记录元数据（时间、区域、状态、延迟、账号ID、用量），默认保留 30 天；不记录请求正文。
  - 错误与重试：统一错误模型；可恢复错误启用指数退避重试；对不可恢复错误快速失败并记录。
  - 安全：`X-API-Key` 入站鉴权；严格密钥与路由策略管理；必要头部剥离/改写；最小权限访问。
  - 部署与回滚：采用 Wrangler/CI 流水线；优先“逐步放量/按租户灰度”能力（若可实现）；保留快速回滚路径。
  - 兼容性：优先对接 Codex API；设计上为未来多上游提供商适配预留抽象层。
  - 成本：以最小成本上线先行；避免引入高成本方案（例如不必要的持久化或数据冗余）；按用量扩展出口集群。

## 4. Epic List（高阶里程碑）

1) Epic 1: Foundation & Minimal Relay  
目标：完成仓库与 Workers 基础骨架、`X-API-Key` 鉴权、KV 限流、健康检查/示例路由，并实现最小可用的 JSON 请求中转与统一错误骨架，形成首个可验证的端到端路径。

2) Epic 2: Streaming & Unified Errors  
目标：打通并稳固 SSE/分块流式透传路径，完善 TransformStream 管道与回压处理，落地统一错误模型与可恢复重试策略，显著提升端到端可靠性。

3) Epic 3: Account Health & Circuit Breaker  
目标：设计并实现 D1 表结构、账号健康检测、熔断与指数退避恢复机制，配合 Queues 完成异步健康任务，使账号池在故障时快速自愈、不影响总体可用性。

4) Epic 4: Regional Egress & Fixed IP Strategy  
目标：集成固定出口 IP 代理集群，优先新加坡地域，其后法兰克福/达拉斯；实现“账号:固定IP=1:N（小 N 冗余）”绑定与地域就近接入，确保上游策略合规与稳定出口。

5) Epic 5: Tenant Controls & Onboarding  
目标：完善租户级并发/配额与基础用量指标，提供最小可用文档/示例或 SDK，优化接入体验（典型接入 ≤ 0.5 天）并为小规模试点租户上线做准备。

## 5. Epic Details — Epic 1: Foundation & Minimal Relay

Epic 目标
- 完成仓库与 Workers 基础骨架、`X-API-Key` 鉴权、KV 限流、健康检查/示例路由；
- 实现最小可用的 JSON 请求中转与统一错误骨架，形成首个可验证的端到端路径。

Story 1.1 初始化仓库与 Workers 项目骨架
As a developer,
I want a ready-to-run Cloudflare Workers project skeleton,
so that I can quickly iterate features and deploy an MVP relay.

验收标准
1: 提供包含 `wrangler` 配置的可运行项目骨架（本地与 Cloudflare 环境均可部署）。
2: 提供基础目录结构：`/src`、`/routes`、`/lib`、`/config`、`/tests`。
3: 提供一个健康检查路由（如 `/health`），返回固定 JSON 并含版本/时间戳。
4: 提供基础 CI 配置（如 lint/format/test 占位任务），可在 PR 上运行。
5: 在 `README` 或 `docs/` 中说明本地运行、部署与回滚基本步骤。

Story 1.2 入站鉴权（X-API-Key）与租户识别
As a tenant admin,
I want inbound requests authenticated via X-API-Key and mapped to my tenant,
so that each tenant is isolated and protected.

验收标准
1: 所有非公开路由要求 `X-API-Key`；无/无效密钥返回统一错误（含错误码/追踪 ID）。
2: 鉴权后可解析出租户 ID（来自 KV 或配置映射），并在请求上下文可用。
3: 记录必要元数据但不存请求正文；遵守数据最小化原则。
4: 提供最小示例文档说明如何获取与配置密钥。

Story 1.3 KV 限流与并发控制（基础版）
As a platform operator,
I want per-tenant basic rate limiting and concurrency control,
so that system stability is maintained under load.

验收标准
1: 提供可配置的租户级速率限制与并发门限（KV 计数，含过期窗口）。
2: 超限时返回统一错误（429 或等价），错误模型一致且可观测。
3: 在健康与负载场景下通过集成测试验证（含正常/超限/恢复）。

Story 1.4 最小 JSON Relay 与统一错误骨架
As an end-user client,
I want to post JSON to the relay and receive a proxied JSON response,
so that I can integrate quickly without custom logic.

验收标准
1: 提供最小 JSON 请求转发到 Codex 的路径（目标上游可用占位/模拟），将 JSON 响应透传回客户端。
2: 对上游错误进行统一归类与映射，输出统一错误模型（含可恢复/不可恢复标识）。
3: 对可恢复错误执行有限次指数退避重试；对不可恢复错误快速失败。
4: 覆盖成功/失败/重试的集成测试用例；保留可观测指标（成功率、延迟）。

Story 1.5 安全头策略与请求规范化（基础版）
As a security-conscious operator,
I want sensitive headers stripped or normalized,
so that upstream calls are consistent and secure.

验收标准
1: 定义白/黑名单与必要改写规则；剥离敏感头部，保留必要上下文。
2: 通过配置控制策略开关与例外名单。
3: 在集成测试中覆盖正常/异常头部场景，验证规范化后行为一致。

Story 1.6 最小文档/示例与可运行演示
As a tenant developer,
I want a minimal doc and example to call the relay,
so that I can integrate within 0.5 day.

验收标准
1: 提供最小 Postman 集合或 curl 示例，演示鉴权、JSON Relay 调用与错误响应。
2: 提供“常见错误与排查”小节（鉴权失败、超限、上游异常）。
3: 演示示例在本地与云端环境均可运行通过。

## 6. Epic Details — Epic 2: Streaming & Unified Errors

Epic 目标
- 实现 Codex 流式响应（SSE/分块）的稳定、低延迟透传；
- 完善基于 TransformStream 的数据管道与回压处理；
- 统一错误模型在流式场景下的细化与可恢复重试策略；
- 增强观测指标，度量流式特性（首字节时间、断流率、重连/重试结果等）。

Story 2.1 SSE/分块透传基础实现
As an end-user client,
I want stable SSE/chunked streaming from upstream via the relay,
so that I can consume responses incrementally with low latency.

验收标准
1: 提供流式上游的稳定透传，保持事件边界与数据完整性（不拆分 UTF-8 字符，按 event 分帧）。
2: 透传必须兼容常见浏览器与 Node 客户端（Fetch/EventSource/ReadableStream）。
3: 在上游正常/降速/间歇性场景下，端到端可用，首字节时间可度量。

Story 2.2 TransformStream 管道与回压处理
As a performance-minded engineer,
I want proper TransformStream and backpressure handling,
so that the pipeline remains responsive and memory-safe.

验收标准
1: 对 Readable → Transform → Writable 管道正确连接，尊重 backpressure，避免内存膨胀。
2: 在慢消费端下不丢事件、不阻塞上游（或按策略限速）；提供可配置的缓冲阈值与丢弃策略占位（默认关闭）。
3: 通过性能测试验证在不同速率与网络条件下的稳定性。

Story 2.3 统一错误模型（流式场景）
As an integrator,
I want consistent error signaling for streaming,
so that clients can handle errors predictably.

验收标准
1: 定义流式上下文中的错误分类（可恢复/不可恢复/超时/断流/上游 4xx/5xx）。
2: 在发生错误时提供一致的标识与追踪 ID；在头/首包或最后数据包中以约定方式传达（文档化）。
3: 对可恢复错误支持有限次重试（指数退避），并记录重试次数与最终结果。

Story 2.4 断开、超时与重连策略
As a resilient consumer,
I want clear behavior on disconnects, timeouts, and retries,
so that user experience remains robust.

验收标准
1: 客户端断开时清理资源；上游断流时快速感知并按策略处理（重试或结束）。
2: 可配置的请求超时与空闲超时；对长时间静默提供心跳或 keep-alive 选项（文档化权衡）。
3: 文档示例展示断开/重连/超时的推荐客户端处理范式（含 EventSource/Fetch+ReadableStream）。

Story 2.5 观测增强（流式特性）
As an operator,
I want streaming-focused observability,
so that we can monitor and improve reliability.

验收标准
1: 指标覆盖：SSE 成功率、断流率、首字节时间（TTFB-Streaming）、平均事件间隔、重试次数分布。
2: 记录必要元数据，不存正文；保留期遵守 30 天默认策略。
3: 提供最小化仪表盘定义或查询示例（便于后续可视化）。

Story 2.6 集成与回归测试（流式）
As a QA engineer,
I want automated streaming tests,
so that regressions are caught early.

验收标准
1: 提供端到端流式测试：正常、慢上游、间歇上游、断流、重试、不可恢复错误。
2: 覆盖首字节时间、事件完整性、回压稳定性等关键指标的断言或阈值检查。
3: 在 CI 中运行可控的子集（长时间场景可放入夜间/增量测试）。

Story 2.7 文档与示例（流式）
As a tenant developer,
I want clear docs and samples for streaming,
so that I can implement clients correctly.

验收标准
1: 提供浏览器（EventSource）与 Node（Fetch+ReadableStream）的最小示例。
2: 文档包含错误模型映射、重连/超时/心跳策略的推荐实践与权衡。
3: 提供问题排查清单（常见断流原因、上游/网络/客户端差异）。

## 7. Epic Details — Epic 3: Account Health & Circuit Breaker

Epic 目标
- 设计并落地 D1 表结构与核心读写路径（账号、健康历史、失败计数、禁用窗口、恢复时间等）。
- 实现账号健康检测、熔断与指数退避恢复策略。
- 引入 Queues 处理异步健康检查与重试，避免请求热路径阻塞。
- 提供可观测指标，量化健康度与恢复速度，支撑容量与成本决策。

Story 3.1 D1 表结构与读写 API
As a platform operator,
I want relational tables for account states and health history,
so that I can persist, query, and audit health data reliably.

验收标准
1: 设计并创建核心表：`accounts`、`health_events`、`routing_bindings`（账号-地域/出口IP 绑定）、`throttles`（可选）。
2: 提供读写 API：记录失败/成功事件、更新失败计数、设置禁用窗口、记录恢复时间。
3: 提供索引与查询样例：按账号/地域/时间窗口查询健康趋势；审计最近 N 次事件。

Story 3.2 健康检测与熔断策略（同步路径）
As a reliability engineer,
I want on-path health updates and circuit breaking,
so that failing accounts are quickly isolated.

验收标准
1: 在请求失败时按错误类型更新健康计数；达到阈值→触发熔断，进入禁用窗口。
2: 对不可恢复错误（如认证失败）立即熔断并延长禁用窗口；对可恢复错误采用较短窗口。
3: 同步路径尽量轻量：仅打标与入队任务，不进行重型计算。

Story 3.3 指数退避与恢复（异步路径）
As a system,
I want backoff-based recovery via queues,
so that accounts return to pool safely.

验收标准
1: 使用 Queues 定期探测被熔断账号；指数退避（如 1m → 2m → 4m → … 上限）后执行轻量健康探测。
2: 探测成功→清零失败计数、退出熔断；失败→继续退避并延长窗口（设上限以防抖动）。
3: 可配置最大退避上限与总探测次数；提供默认合理值。

Story 3.4 路由与账号选择策略（与固定 IP 绑定协同）
As a request router,
I want to avoid unhealthy accounts and respect bindings,
so that routing remains compliant and efficient.

验收标准
1: 选择账号时跳过“熔断/禁用中”的账号；优先选择健康度更高、绑定地域就近的账号。
2: 遵守“账号:固定IP=1:N（小 N 冗余）”策略，不跨账号共享 IP。
3: 记录路由决策元数据用于审计（不含正文）。

Story 3.5 观测指标与告警
As an operator,
I want KPIs and alerts for account health,
so that I can detect issues early.

验收标准
1: 指标覆盖：失败率、熔断触发次数、平均禁用时长、恢复成功率、恢复平均时间（MTTR）。
2: 阈值示例与告警规则占位（文档化）：如熔断频率异常升高时提醒。
3: 仅记录必要元数据，遵守 30 天保留与隐私约束。

Story 3.6 集成测试与回归场景
As a QA engineer,
I want automated tests for health and recovery,
so that regressions are minimized.

验收标准
1: 覆盖：连续失败触发熔断、禁用窗口内拒绝路由、退避后恢复、上限退避、不可恢复错误快速熔断。
2: 覆盖路由选择跳过不健康账号、选择健康账号的逻辑。
3: 在 CI 中运行需时可控的子集，长场景可夜间运行。

Story 3.7 运维与运营手册
As an oncall engineer,
I want operational playbooks,
so that incidents are resolved quickly.

验收标准
1: 提供常见故障场景与处置流程（如大量账号熔断、单地域异常、绑定策略冲突）。
2: 提供调试/手动解禁/强制禁用的安全流程（带审计记录）。
3: 提供 D1 表结构变更与数据维护注意事项清单。

## 8. Epic Details — Epic 4: Regional Egress & Fixed IP Strategy

Epic 目标
- 集成固定出口 IP 代理集群，优先新加坡地域，其后法兰克福/达拉斯。
- 落地“账号:固定IP=1:N（小 N 冗余）”绑定，不跨账号共享 IP。
- 按地域就近接入，结合健康度与配额进行路由选择。
- 建立可观测指标与运维流程，确保透明度与快速处置能力。

Story 4.1 出口代理集群对接与基础连通性
As a network integrator,
I want the relay to route via fixed-IP egress clusters,
so that upstream sees stable, controlled IPs.

验收标准
1: 与至少 1 个地域（新加坡）固定 IP 集群打通出站线路，具备健康检测（连通、时延、丢包占位）。
2: 提供可配置的出站代理参数（地址、端口、认证、地域标签、权重）。
3: 在非生产环境验证最小 JSON Relay 通过固定 IP 正常转发。

Story 4.2 账号-固定 IP 绑定策略
As a compliance-focused operator,
I want one-to-N bindings between account and fixed IPs,
so that accounts never share IPs across boundaries.

验收标准
1: 设计并持久化“账号:固定IP=1:N（小 N 冗余）”绑定（D1 表或配置来源），不跨账号共享。
2: 绑定变更需审计记录与最小变更日志；支持安全的回滚处理。
3: 单账号的 N 个 IP 具备优先/降级策略（健康/可用性优先）。

Story 4.3 地域就近接入与路由决策
As a request router,
I want region-aware routing,
so that latency and success rate improve.

验收标准
1: 根据请求地域提示（如 `RELAY_REGION_HINT`）优先选择对应地域集群；无提示则按全局最优策略选择。
2: 结合账号健康度、配额与绑定策略进行综合路由；不可用时执行降级路径（同地域备选 → 相邻地域）。
3: 记录路由决策元数据（不含正文）便于审计与回溯。

Story 4.4 故障与降级策略（地域/出口层）
As an SRE,
I want safe degradation paths,
so that incidents don’t cascade to users.

验收标准
1: 出口集群部分不可用时自动降级到同账号的备用 IP 或相邻地域；提供停用/启用开关。
2: 与账号健康熔断联动，避免将请求发往“熔断/禁用”的账号或其绑定 IP。
3: 文档化常见故障与降级流程（代理证书失效、线路异常、带宽饱和）。

Story 4.5 观测与合规审计
As an operator,
I want visibility and audit trails,
so that decisions are explainable and compliant.

验收标准
1: 指标覆盖：地域分布成功率、出口 IP 成功率、切换/降级次数、平均路由决策时间。
2: 审计：记录账号-固定 IP 的变更日志，路由决策摘要（不含正文），可按时间范围查询。
3: 仅记录必要元数据，遵守 30 天保留与隐私约束。

Story 4.6 性能与稳定性验证
As a QA/perf engineer,
I want performance and stability tests,
so that region-based routing is reliable.

验收标准
1: 不同地域的延迟/稳定性对比测试；验证就近接入的收益（如 P95 延迟改善）。
2: 断链/降速/丢包等网络条件下，验证降级策略与可用性目标。
3: 在 CI/夜间任务运行部分性能回归，确保关键路径不回退。

Story 4.7 运维与容量规划
As an oncall/capacity planner,
I want runbooks and capacity models,
so that scaling decisions are data-driven.

验收标准
1: 产出地域与出口集群的容量模型与扩容判据（示例阈值）。
2: 提供变更 SOP：新增 IP、替换 IP、调节地域权重、临时封禁。
3: 明确安全依赖（证书/认证）轮转流程与检查清单。

## 9. Epic Details — Epic 5: Tenant Controls & Onboarding

Epic 目标
- 提供租户级并发/配额与用量观测的基础能力。
- 建立 API Key 生命周期管理与最小化的租户自助接口。
- 输出最小可用的 Onboarding 文档/示例或 SDK，确保集成 ≤ 0.5 天。
- 形成基本运营/支持闭环（常见问题、排查清单、用量可视）。

Story 5.1 租户模型与 API Key 生命周期
As a tenant admin,
I want to manage tenant and API keys,
so that onboarding and key rotation are safe and simple.

验收标准
1: 定义租户模型（tenant_id、状态、创建时间、备注）与 API Key 模型（key_id、哈希/前缀、状态、创建/失效时间、备注）。
2: 提供最小生命周期操作：创建、禁用、旋转、列出；关键操作有审计记录（不存明文）。
3: 文档化密钥管理最佳实践（定期轮换、最小权限、最小分发）。

Story 5.2 配额与并发控制（租户级）
As a platform operator,
I want tenant-level quotas and concurrency limits,
so that usage remains controlled and predictable.

验收标准
1: 可配置租户级请求配额、并发上限、速率限制；超限返回统一错误（含追踪 ID）。
2: 提供重置/调整入口（仅运维权限），并记录审计变更。
3: 在集成测试中覆盖正常/超限/恢复场景，并与现有 KV 限流机制整合。

Story 5.3 用量计量与基础可观测
As a tenant admin,
I want usage metrics visibility,
so that I can understand consumption and troubleshoot.

验收标准
1: 记录并可查询租户级用量元数据（调用数、成功率、P95 延迟、流式成功率等），遵守“仅记录元数据、不存正文”与 30 天保留。
2: 提供最小查询接口或导出能力（例如租户自助查询近 7/30 天摘要）。
3: 提供示例查询/仪表盘定义占位，便于后续可视化。

Story 5.4 Onboarding 文档与最小 SDK/示例
As a tenant developer,
I want simple onboarding docs or SDK,
so that I can integrate within 0.5 day.

验收标准
1: 提供最小 Postman 集合或 SDK 片段（语言可选 TypeScript/JS），涵盖鉴权、JSON 调用、流式示例、常见错误处理。
2: 提供“接入 30 分钟路径”：环境变量、鉴权、最小请求、错误模型、流式读取的示例。
3: 提供“常见问题与排查”清单，覆盖密钥失败、超限、上游断流、区域路由提示等。

Story 5.5 租户自助与运维接口（最小集）
As an operator,
I want minimal self-serve and admin endpoints,
so that tenants and ops can act without tickets for basics.

验收标准
1: 租户自助：查看用量摘要、查看/禁用/旋转 API Key、获取接入示例链接。
2: 运维接口：调整租户配额/并发、临时封禁/解禁租户、导出近 N 天用量摘要。
3: 所有敏感操作均需鉴权并有审计记录，遵循最小权限。

Story 5.6 合规与隐私（租户层）
As a compliance-minded stakeholder,
I want guardrails for privacy and compliance,
so that tenant operations remain safe.

验收标准
1: 再申明“仅记录元数据，不存正文”的约束；明确可配置的保留周期（默认 30 天）。
2: 在对外文档中明确与上游条款/地域策略的关系与边界指引。
3: 提供导出与删除租户元数据的流程（运维侧），留存审计记录。

Story 5.7 试点租户上线与回访
As a product manager,
I want pilot tenants activated and reviewed,
so that we capture feedback and validate MVP.

验收标准
1: 选择 2-3 个目标试点租户，完成 Onboarding，达成首周使用目标（如日均调用数、成功率、流式占比）。
2: 回访收集集成体验、问题与改进建议，更新“常见问题与排查”与文档。
3: 根据反馈评估是否提前引入 Epic 2/3/4 的部分能力或调整优先级。

## 10. Next Steps（下一步）

### UX Expert Prompt（占位）
请基于本 PRD 提炼最小化的用户流程与信息架构建议，不做详细 UI 规范，重点：
- 关键交互点（鉴权失败、超限、流式加载反馈）的用户提示策略；
- 文档/示例的可读性与“30 分钟接入路径”的信息组织；
- 如未来提供控制台，自助入口的信息架构草图与权限边界建议。

### Architect Prompt（占位）
请基于本 PRD 制定技术落地方案与分层设计，要求：
- 给出 Workers/KV/D1/Queues 的目录与模块划分、核心接口定义草案；
- 定义错误模型与流式透传的关键数据结构与状态机；
- 制定健康/熔断/退避与地域路由的协同流程图；
- 产出 M1（Epic 1）与 M2（Epic 2）的交付件清单与上线检查表。
