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
