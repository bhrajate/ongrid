# Ongrid 系统架构文档

> 本文档描述 Ongrid 的整体系统架构、核心组件、关键数据流与设计约束，面向开发者、运维与集成方。
>
> 最后更新：2026-06-09 · 对应版本：v0.8.x

## 目录

- [1. 系统概述](#1-系统概述)
- [2. 架构总览](#2-架构总览)
- [3. 部署拓扑](#3-部署拓扑)
- [4. Manager（云端控制面）](#4-managercloud-端控制面)
- [5. Edge（边缘 Agent）](#5-edge边缘-agent)
- [6. Tunnel 通信模型](#6-tunnel-通信模型)
- [7. AIOps Agent 内核](#7-aiops-agent-内核)
- [8. 可观测性数据面](#8-可观测性数据面)
- [9. 身份、权限与多租户](#9-身份权限与多租户)
- [10. 数据存储](#10-数据存储)
- [11. 前端](#11-前端)
- [12. 关键设计约束](#12-关键设计约束)

---

## 1. 系统概述

Ongrid 是一个**运维 AI Agent 平台**：它理解你的基础设施、定位故障根因，并能直接从 Slack / Telegram 等 IM 渠道执行修复。核心能力包括：

- **协调者 + 专家 Agent 模型**：协调者（coordinator）将任务派发给 SRE / 网络 / 数据库等专家子 Agent。
- **告警驱动的自动调查**：告警触发后自动派生 RCA（根因分析）worker，将分析结论写回会话。
- **根因关联分析**：遍历拓扑、关联 metrics/logs/traces（m/l/t），将"为什么"定位到源代码行。
- **零入站端口**：Edge 主动外拨，主机上不开放 22 / 80 / 443 任何入站端口。
- **浏览器 SSH**：通过反向隧道进入任意主机的 Shell，无需密钥、无需跳板机，全程审计。
- **一条命令自托管**：`docker compose up` 拉起完整技术栈。
- **自带可观测性**：内置 Prometheus + Loki + Tempo + Grafana，Agent 自己写查询。
- **多模型自带**：Anthropic / OpenAI / GLM / DeepSeek / Gemini / Kimi，支持热路由。
- **双向 IM 渠道**：Slack / Telegram / 飞书 / 钉钉 / 企业微信，按渠道设置语言。
- **只读主机工具**：bash 沙箱 + 26+ 巡检工具，每次调用都审计。

技术栈：**Go**（后端，go 1.25）· **TypeScript + React**（前端）· **eino**（LLM 编排框架）· **frontier/geminio**（隧道）。

---

## 2. 架构总览

系统由两个 Go 二进制 + 一组开箱即用的基础设施容器构成：

| 二进制 | 角色 | 入口 |
|--------|------|------|
| `ongrid` | 云端 Manager（控制面） | `cmd/ongrid/main.go` |
| `ongrid-edge` | 边缘 Agent（部署在被纳管主机） | `cmd/ongrid-edge/main.go` |

```
                                  ┌────────────────────────────────────────────────────┐
                                  │                  云端 (单台 VPS / 自托管)               │
                                  │                                                      │
   浏览器 SPA ──HTTPS──► nginx ───┼──► ongrid (Manager)  ◄──► MySQL                       │
   IM 平台   ──webhook─►  :443    │       │  │  │  │                                       │
   (Slack/TG/飞书/钉钉/企微)       │       │  │  │  └──► LLM Provider (OpenAI/Claude/GLM…)  │
                                  │       │  │  └─────► Prometheus / Loki / Tempo         │
                                  │       │  └────────► Qdrant (向量库) / SearXNG (搜索)   │
                                  │       └───────────► Grafana                           │
                                  │                                                      │
                                  │             frontier broker (geminio)                │
                                  └──────────────────────┬───────────────────────────────┘
                                                         │  反向隧道 (Edge 主动外拨 :40012)
                                          ┌──────────────┴───────────────┐
                                          ▼                              ▼
                                   ongrid-edge (主机A)            ongrid-edge (主机B)
                                   node_exporter / 插件            bash 沙箱 / host_files
```

后端遵循 [gospec](https://github.com/singchia/gospec) 规范，采用**严格分层 + 按领域（domain）切分**的单体（monorepo）架构。

### 调用分层（强制约束）

```
cmd  →  server  →  service  →  biz  →  data  →  model
              （HTTP）   （编排）  （用例/领域）（仓储）（实体）
```

- 禁止跨层调用、禁止反向依赖。
- `internal/<domain>` 之间禁止直接 import，必须通过 API / 事件 / `internal/pkg`（共享）解耦。
- 接口在消费方定义，依赖通过构造函数注入，不使用全局可变变量。

---

## 3. 部署拓扑

生产部署由 `deploy/install/docker-compose.yml` 描述，`install.sh` 一键拉起。所有服务运行在同一 docker 网络内，仅 nginx（:443/:80）与 frontier（:40012）对外暴露端口。

| 容器 | 作用 | 对外端口 | 说明 |
|------|------|---------|------|
| `nginx` | TLS 终止 + SPA 托管 + `/api` 反代 | 443 / 80 | 镜像内置 `web/dist`；ADR-008 后 Manager 8080 不再直接对外 |
| `ongrid` | Manager 主进程 | 9100（仅 metrics） | HTTP API 仅 nginx 可达 |
| `frontier` | 上游隧道 broker（singchia/frontier） | 40012（edge 侧） | 40011（service 侧）仅 docker 网络内 |
| `mysql` | 业务主存储 | 无（内网） | utf8mb4 |
| `prometheus` | 云端指标库（核心服务，ADR-009） | 无 | 接收 Manager remote_write，retention 90d/20GB |
| `loki` | 日志后端（ADR-012） | 无 | 经 nginx `/loki/api/v1/push` + auth_request 写入 |
| `tempo` | 链路追踪后端（ADR-013） | 无 | OTLP 经 nginx `/v1/traces` 写入 |
| `qdrant` | 知识库 / RAG 向量库 | 无 | collections 持久化 |
| `searxng` | web_search 技能默认后端 | 无 | 自托管元搜索，零 API key |
| `grafana` | 可观测性可视化 | 经 nginx `/grafana` | 匿名 viewer + SA token bootstrap |

**关键设计：数据面 vs 控制面分离**
- **控制面**：浏览器/IM → nginx → Manager HTTP API（JWT 鉴权）。
- **数据面**：Edge 遥测推送（logs/traces）→ nginx → auth_request 校验 Edge basic-auth → 直达 Loki/Tempo，**不经过 Manager Go 进程的字节流**。Edge 凭据复用隧道握手的 access key。

所有有状态服务通过 bind-mount 落到 `${ONGRID_DATA_DIR:-/var/lib/ongrid}/<service>`，便于备份与外置存储。

---

## 4. Manager（云端控制面）

`ongrid` 二进制组合了 `iam` 与 `manager` 两个限界上下文（bounded context），对外暴露公共 HTTP API、Prometheus `/metrics` 端点，以及面向 frontier 的 service-end SDK。

### 4.1 限界上下文与领域

`internal/manager` 按领域切分，每个领域内部都遵循 `biz / data / model / server / service` 分层：

| 领域 | 职责 |
|------|------|
| `aiops` | AI Agent 核心：chat 运行时、graph 内核、tools、investigator、mentions |
| `alert` | 告警生命周期、incident、静默（silence）、通知投递、RCA investigator |
| `edge` | Edge 注册/心跳、access key 鉴权、插件配置下发、升级包分发 |
| `device` | 主机设备清单，与 topology 镜像 |
| `topology` | 拓扑节点/关系/关系类型，支撑 RCA 的爆炸半径分析 |
| `metric` / `promwrite` | 指标读（PromQL）与写（remote_write 摄取） |
| `monitor` | 用户自管 Monitor 面板，单向镜像到 Grafana dashboard |
| `knowledge` | 知识库 + 代码仓库 RAG（embedding + qdrant） |
| `imbridge` | IM 多轮对话桥接（飞书/钉钉/Telegram/Slack/企微） |
| `marketplace` | 技能/Agent 市场，安装后热重载注册表 |
| `skill` | 技能元数据 + 经 frontierbound 派发执行 |
| `webshell` | 浏览器 SSH（Manager 跑 SSH 客户端，Edge 仅转发字节） |
| `grafana` / `setting` | Grafana 集成 + 管理员可编辑的运行时配置 |
| `audit` / `report` | HLD-010 审计日志 + HLD-014 定时运营报告 |

### 4.2 运行时配置（system_settings）

`setting` 领域提供**管理员可在线编辑**的运行时配置（当前主要是 LLM 凭据、Prom/Loki/Tempo URL、Grafana token、web search provider）。

- LLM 客户端在每次 `Chat()` 时通过 Resolver 读取，内置 60s TTL 缓存，**管理员改动无需重启**即在约 60s 内生效。
- 环境变量值仅在首次启动时 seed（`SetIfAbsent`），后续管理员编辑跨重启保留。

### 4.3 启动编排

`main.go` 是一个长依赖注入装配链，关键顺序：

1. 加载配置 → 初始化 OTel tracing → 打开 DB 并运行各领域 `Migrate`。
2. 装配 iam（用户/组织/成员/casbin）→ bootstrap admin → seed 默认组织。
3. 装配 settings（seed LLM/Prom/Grafana 等默认值）。
4. 构建多 provider LLM 路由器（`llm.NewMultiClient`），空 key 的 provider 自动从目录剔除。
5. 装配各领域 biz/server，构建 frontierbound SDK 并安装隧道回调与反向调用 handler。
6. 构建 AIOps 运行时（legacy 或 graph 内核，见 §7），装配 tools registry。
7. 挂载 chi 路由：`/healthz`、`/readyz`、`/internal/auth/*`（数据面校验，不带 JWT）、`/api/*`（业务，JWT 保护）、`/r/{token}`（公开报告分享）。
8. 启动 API 监听、metrics 监听、各类后台 ticker（告警评估、DB 池采样、IM stream supervisor 等）。

---

## 5. Edge（边缘 Agent）

`ongrid-edge` 部署在每台被纳管主机上，由 systemd 托管。核心特征是**零入站端口**——它主动外拨到云端 frontier。

### 5.1 职责

- **建立并维持隧道**：`internal/pkg/tunnel.NewClient` 拨号到云端 broker，多路复用 RPC 通道。
- **注册反向调用 handler**：响应 Manager 发来的工具 RPC（见 §6）。
- **采集器（collector）**：根据 `CollectorMode` 决定推送策略（见下）。
- **插件运行时**：以 goroutine（进程内）或受监督子进程方式运行 logs / traces / metrics / hostmetrics / procmetrics 插件，由 supervisor 按 Manager 下发的配置 reconcile。
- **能力插件**：`host_files`（只读文件巡检）、`bash`（受 cmdpolicy 沙箱约束的只读 shell）、`restart_service`（首个变更类技能）、`webshell`（字节转发）。

### 5.2 采集器模式（CollectorMode）

| 模式 | 行为 |
|------|------|
| `off` / `""`（默认） | 不周期推送；按需 RPC（host_info / get_host_load / get_host_processes）仍走内嵌 gopsutil。配合 hostmetrics/procmetrics 插件由云端 Prom 直接抓取 |
| `auto` | 内嵌 gopsutil 推送 + scraper |
| `embedded` | 仅内嵌推送 |
| `scrape` | 仅 scraper |

### 5.3 升级

Manager 通过隧道下发 `agent_upgrade` / `fetch_package` / `apply_package`。Edge 拉取新 bundle 暂存到 `.upgrade` 目录，收到 apply 信号后由 systemd 完成二进制热替换；`agent.Run` 返回时取消 rootCtx，所有 goroutine 优雅退出，systemd 完成 swap。

---

## 6. Tunnel 通信模型

隧道基于上游 `singchia/frontier` broker（独立容器，终止 geminio 协议）。**Manager 与 Edge 都不直接监听对方**——双方都连到 frontier：

- **Edge 侧**：`internal/pkg/tunnel` 维护 client，拨向 frontier 的 edge-bound 端口（:40012）。
- **Manager 侧**：`internal/manager/service/frontierbound` 打开长连接到 frontier 的 service-bound 端口（:40011），注册生命周期回调（`GetEdgeID` / `EdgeOnline` / `EdgeOffline`）与反向调用 handler。

RPC 不声明 gRPC service，而是**按方法名注册 handler**，消息体为 JSON（`tunnel/messages.go` 定义共享 shape）。

### 6.1 RPC 方向与方法

| 方向 | 方法 | 用途 |
|------|------|------|
| Edge → Manager | `register_edge` | Edge 握手注册 |
| Edge → Manager | `heartbeat` | 心跳（携带插件健康快照） |
| Edge → Manager | `push_host_metrics` | 旧版主机指标（现由 no-op ingester 接收，向后兼容） |
| Edge → Manager | `push_prom_samples` | 新版指标直推云端 Prom（注入 device_id label） |
| Edge → Manager | `get_plugin_configs` | 拉取本 Edge 插件配置（启动时 + 60s 兜底轮询） |
| Manager → Edge | `plugin_configs_changed` | 配置变更实时推送，触发 supervisor reload |
| Manager → Edge | `get_host_load` / `get_process_list` / `get_netstat` | 主机巡检（AIOps tools） |
| Manager → Edge | `host_files.{find_large_files,du_summary,stat_file}` | 文件系统巡检 |
| Manager → Edge | `bash.exec` | 受沙箱约束的命令执行 |
| Manager → Edge | `execute_skill` | 技能派发（统一 dispatcher RPC） |
| Manager → Edge | `restart_service.restart` | 服务重启（变更类技能） |
| 双向 | `shell_*`（open/input/resize/close/output/exit） | WebSSH 字节流 |
| Manager → Edge | `agent_upgrade` / `fetch_package` / `apply_package` | Edge 自升级 |

### 6.2 WebSSH 模型

浏览器 SSH 的 SSH 客户端 + PTY **完全运行在 Manager**；Edge 仅作哑字节转发器（`io.Copy` 到本机 `127.0.0.1:22`）。Manager 用 `fbClient.OpenStream` 在 frontier 上叠加一条裸字节流，承载 ssh + pty。Edge 不含 SSH 库、不含 PTY、无会话表。全程审计落 MySQL `webshell` 表。

---

## 7. AIOps Agent 内核

AIOps 是产品核心，支持两套内核，通过 `ONGRID_AGENT_KERNEL` 切换（默认 `graph`，2026-05-08 起）：

| 内核 | 说明 |
|------|------|
| `legacy` | 历史 for-loop ReAct runner，所有工具始终全 schema 暴露（`biz/aiops/agent`） |
| `graph` | 基于 eino graph 的内核 + 技能激活关键词过滤 + ToolBag 延迟（deferral）管线（`biz/aiops/chatruntime` + `graph`） |

构建失败时 graph 内核会带告警回退到 legacy，保证开箱可用。

### 7.1 chatruntime（graph 内核）

`chatruntime.Runtime` 是进程内编排入口，单次 `Handle` 的职责链：

1. **所有权校验**（caller user_id == session user_id，admin 上游已放行）。
2. **技能解析**：`SkillRegistry.Resolve` 按用户 query 激活技能；`activation:keyword` 的技能仅在 query 命中关键词时才进入 prompt。
3. **系统提示组装**：`ComposeSystemPrompt` = 基础 prompt + 激活技能 prompt + 可选 Agent persona。
4. **@-mention 内联**：与 legacy 字节一致，保证跨内核会话回放一致。
5. **用户消息持久化**（`chat_messages` role=user，先落盘后调 LLM）。
6. **Graph invoke**：默认回调链（`callbacks.NewDefaultHandlers`）负责持久化、SSE 流式、审计、metrics、预算门控。
7. **回复翻译**：`chatruntime.Reply` → `agent.Reply`，保证 HTTP 响应 shape 与 SPA 兼容。

### 7.2 协调者 + 专家 Agent

graph 内核额外装配协调者专属的工具三件套：

- `AgentTool`：同步派发专家 worker（阻塞至 worker 完成完整 ReAct 循环，超时 180s）。
- `SendMessage` / `TaskStop`：控制面微操作（超时 15s）。

worker 通过 `filterToolsForAgent` 剥离这些协调者专属工具，确保只有协调者能派发。

### 7.3 工具（tools）

工具位于 `biz/aiops/tools`，每个工具有 base 实现 + `basetool` 适配 + `decorators` 装饰链（超时、限流、metrics）。工具按依赖条件注册：

- `query_promql` / `query_logql` / `query_traceql`：分别在 Prom / Loki / Tempo 配置就绪时注册。
- 拓扑类：`get_topology` / `expand_topology` / `find_topology_node`。
- 关联类：`correlate_incident`、`find_outlier_edges`、`get_incident_detail`、`get_edge_summary`、`query_change_events`（RCA "T 附近发生了什么变更"）。
- 主机类：`get_host_load`、`host_files.*`、`bash`（经隧道 RPC 回 Edge）。
- `query_knowledge`：RAG 检索。
- ToolBag 延迟：工具数超过 `ONGRID_TOOLBAG_DEFERRAL_THRESHOLD`（默认 30）时，专家级工具暴露 redacted schema，LLM 需调 `ToolSearch` 展开。

安全技能自动注册为 LLM function-calling 工具（走与 HTTP 层相同的审计 + 权限路径）；变更/危险类技能需 SOP 签名，不自动注册。

### 7.4 多 Provider LLM 路由

`llm.NewMultiClient` 统一封装 OpenAI / Anthropic / Zhipu(GLM) / Gemini / DeepSeek / Kimi。OpenAI 子客户端走 resolver-aware 路径，其余 provider 从 env seed 后经 `LLMSettingsResolver` 读取实时值。每请求可带 Provider override 实现热路由。

### 7.5 告警驱动的 RCA investigator

两级主动调查（`ONGRID_INVESTIGATOR_ENABLED`，默认 true）：

1. **legacy 初诊**：`ai_initial_diagnosis`，单次 LLM 调用（`correlate_incident`），在 incident 时间线写约 3 段轻量结论。
2. **结构化 RCA**：派生 `incident-investigator` chatruntime worker，完整 transcript 持久化为 `kind=investigation` 会话，写 `investigation_reports` 供 IncidentDetail 页展示。

两者共享 `alert.Investigator` 接口，经 `investigatorChain` 在每次新 fire 时 fan-out。启动时对孤儿/未启动 incident 做补偿（backfill），避免进程崩溃导致报告永久卡转圈。

---

## 8. 可观测性数据面

Ongrid 内置完整的 metrics / logs / traces / 可视化栈，Agent 自己生成查询：

| 信号 | 后端 | 写入路径 | 读取路径 |
|------|------|---------|---------|
| Metrics | Prometheus | Edge `push_prom_samples` → Manager remote_write；或 Prom 直抓 node/process exporter 子进程 | `query_promql` 工具 + Monitor 页 |
| Logs | Loki | Edge logs 插件 → nginx `/loki/api/v1/push`（auth_request 校验） | `query_logql` 工具 + Logs 页代理 |
| Traces | Tempo | Edge traces 插件（otelcol-contrib）→ nginx `/v1/traces` | `query_traceql` 工具 + Traces 页代理 |
| 可视化 | Grafana | Manager 单向镜像 Monitor 面板 | nginx `/grafana` 子路径 |

Manager 自身也通过 OTel 上报到 Tempo（`ongrid-manager` service），其 `otelhttp` 中间件给每个请求按 chi 路由命名 span，Tempo 的 spanmetrics generator 据此派生 `traces_spanmetrics_*`，供 trace_latency / trace_error_rate 告警评估器查询。

**Logs/Traces 页代理**：为避免通过 nginx 暴露 Loki/Tempo 读路径，Manager 提供 `logsHandler` / `tracesHandler` 代理 LogQL/TraceQL 查询；数据面 push 路由保持独立的 auth_request 门控（仅供摄取）。

---

## 9. 身份、权限与多租户

`internal/iam` 是独立限界上下文，领域包括 user / org / membership / authz。

- **认证**：JWT（access TTL 15m / refresh TTL 720h），`auth.Signer` 签发，`auth.Middleware` 校验。密码用 bcrypt/argon2id（`internal/pkg/passwd`）。
- **授权**：Casbin（`gorm-adapter`）。`authzmw` 中间件注入各 mutating 路由；superuser 在中间件内短路，避免坏策略锁死管理员。
- **组织模型**：`默认组织` 是唯一顶级 org，其余 org 必须挂其下；启动时 seed 默认组织并回填所有现有用户成员关系，casbin g 规则从 memberships hydrate。
- **多租户约束**：多租户接口强制 `tenant_id` 过滤（gospec 红线）。

数据面 Edge 鉴权独立：nginx `auth_request` 调 Manager `/internal/auth/*`，复用与隧道握手相同的 Edge access key 校验（`edgeAuthn`）。

---

## 10. 数据存储

| 存储 | 用途 | 约束 |
|------|------|------|
| MySQL（默认）/ SQLite（opt-in） | 业务主存储 | 各领域 `data` 包暴露 `Migrate(db)`，启动时按序 AutoMigrate；生产 schema 变更走 expand-contract 滚动兼容 |
| Prometheus | 时序指标 | 90d/20GB retention；高基数字段禁止作 label |
| Loki | 日志 | — |
| Tempo | 链路 | — |
| Qdrant | RAG 向量 | collection 维度随 embedding 模型同步（`ONGRID_EMBEDDING_DIM`） |

ORM 为 GORM，仓储（repo）在各领域 `data/<domain>/store` 实现。所有 SQL 参数化，禁止字符串拼接。

---

## 11. 前端

`web/` 是 React 18 + TypeScript + Vite SPA：

- **路由**：react-router-dom v6
- **状态**：zustand
- **可视化**：recharts（图表）、@xyflow/react + dagre（拓扑图）、xterm（WebSSH 终端）
- **Markdown**：react-markdown + remark-gfm（渲染 Agent 回复）
- **i18n**：内置多语言（与产品默认 locale `ONGRID_DEFAULT_LOCALE` 配合）

构建产物 `web/dist` 被打进 nginx 镜像，`/api` 由 nginx 反代到 Manager。Agent 回复通过 SSE 流式推送到 SPA。

---

## 12. 关键设计约束

以下为任何变更都必须遵守的红线（详见 `AGENTS.md` 与 gospec）：

**架构**
- 单服务严格分层 `cmd → server → service → biz → data → model`，禁止跨层。
- monorepo 下 `internal/<domain>` 之间禁止直接 import，必须经 API / 事件 / `internal/pkg`。
- 接口在消费方定义，禁止循环依赖；依赖构造函数注入，无全局可变变量。
- `init()` 仅允许做注册（pprof / metrics collector / driver），禁止 IO 或可能 panic。

**API**
- API 变更先改 `.proto`，禁止改生成代码（`api/` 下按领域组织 proto）。
- Handler 必须有 Swagger 注释；响应统一 `{code, message, data}`；破坏性变更走新版本。

**可观测性**
- 所有对外服务暴露 `/healthz`、`/readyz`、`/metrics`。
- 日志结构化（slog）+ `trace_id`；高基数字段禁止作 Prometheus label；敏感字段禁止明文入日志。

**安全**
- 密码 bcrypt/argon2id；SQL 全参数化；密钥禁入仓库/镜像/日志。
- 容器以非 root 运行；多租户接口强制 `tenant_id` 过滤；CI 含 `govulncheck` + 漏洞扫描。

**测试**
- 新功能必须有单测；CI 强制 `-race`；E2E 必须清理数据。

---

## 附录：源码导航

| 路径 | 内容 |
|------|------|
| `cmd/ongrid/` | Manager 入口与依赖装配 |
| `cmd/ongrid-edge/` | Edge 入口与采集器/插件装配 |
| `internal/manager/` | Manager 各领域（biz/data/model/server/service） |
| `internal/edgeagent/` | Edge 采集器、插件、能力（bash/host_files/webshell） |
| `internal/iam/` | 身份与权限限界上下文 |
| `internal/pkg/` | 跨领域共享库（tunnel/llm/prom/embedding/auth…） |
| `internal/skill/` | 技能注册表与内置技能 |
| `api/` | proto 定义（iam / manager / tunnel） |
| `web/` | React SPA |
| `deploy/` | docker-compose、Dockerfile、安装脚本、各组件配置 |
| `docs/` | 文档（本文件、安装、E2E 目录） |
| `AGENTS.md` / `ROADMAP.md` | 开发规范 / 路线图 |
