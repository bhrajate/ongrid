# 本地开发：启动依赖服务 / 后端 / 前端

本文档介绍在**宿主机原生运行 ongrid 主程序（后端）与前端 SPA**、依赖服务放
Docker 的本地开发方式。适用于日常改代码、调试、跑单测的场景。

> 完整的一体化 Docker 栈（含从源码构建的 ongrid + nginx）见 `deploy/docker-compose.yml`；
> 那套是「production-shape」的自托管形态，不适合边改代码边热重载。

## 拓扑总览

```
┌─────────────── 宿主机（WSL / Linux）───────────────┐
│                                                     │
│  前端 vite dev (:5173) ──/api──▶ 后端 ongrid (:8090)│
│                                        │            │
│                                        ▼            │
│                       ┌── Docker: ongrid-deps 栈 ──┐│
│                       │ mysql:3307   qdrant:6333   ││
│                       │ frontier:40011/40012       ││
│                       │ prometheus:9090 loki:3100  ││
│                       │ tempo:3200/4318            ││
│                       │ searxng:8080 grafana:3000  ││
│                       └────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

- 依赖服务的端口全部只绑 `127.0.0.1`，仅本机可访问。
- 后端直连这些 `localhost:<port>` 依赖。
- 前端 vite dev server 把 `/api/*` 代理到后端。

## 前置要求

| 工具 | 版本 | 说明 |
|---|---|---|
| Go | 1.25.11（见 `.tool-versions`） | 跑后端 `go run` |
| Node.js + npm | Node 18+ | 跑前端 vite |
| Docker + Docker Compose | 近期版本 | 跑依赖服务 |

---

## 第 1 步：启动依赖服务（Docker）

依赖栈定义在 `deploy/docker-compose.deps.yml`，只含 8 个依赖服务（不含 ongrid、nginx），
容器名带 `-dev` 后缀、独立命名卷，与生产栈完全隔离。

```bash
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps up -d
```

常用运维命令：

```bash
# 查看状态
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps ps

# 查看某个服务日志
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps logs -f mysql

# 停止（保留数据卷）
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps down

# 停止并清空数据卷（彻底重来）
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps down -v
```

各服务暴露到宿主机的端口：

| 服务 | 宿主机地址 | 用途 |
|---|---|---|
| mysql | `127.0.0.1:3307` | 主数据库（用 3307 避开宿主机原生 MySQL 的 3306） |
| qdrant | `127.0.0.1:6333` | 向量库（RAG） |
| frontier | `127.0.0.1:40011` / `:40012` | 隧道 broker（manager / edge） |
| prometheus | `127.0.0.1:9090` | 指标（remote_write + 查询） |
| loki | `127.0.0.1:3100` | 日志 |
| tempo | `127.0.0.1:3200` / `:4318` | 链路查询 / OTLP 推送 |
| searxng | `127.0.0.1:8080` | web_search 后端 |
| grafana | `127.0.0.1:3000` | 面板 |

> **等 mysql 就绪**：ongrid 启动时会连 mysql，首次 `up` 后等约 20s（healthcheck
> `start_period`）再启动后端，或用 `... ps` 确认 mysql 状态为 `healthy`。

---

## 第 2 步：启动后端（ongrid）

### 2.1 准备环境变量

后端通过环境变量读取配置，`go run` **不会**自动加载任何 `.env` 文件，
需要你先手动 `source` 进当前 shell。

仓库提供两个文件：

- `deploy/ongrid.local.env` — **随仓库提交的示例模板**，敏感项（LLM/embedding 的
  key、base_url）留空，其余为指向上面依赖栈的 localhost 默认值。
- `deploy/.env` — 你的**真实配置**（已被 `.gitignore` 忽略，可安全填入 key）。

首次使用，复制示例并填入自己的 key：

```bash
cp deploy/ongrid.local.env deploy/.env   # 若还没有 .env
# 编辑 deploy/.env，至少填：
#   ONGRID_OPENAI_API_KEY / ONGRID_OPENAI_BASE_URL / ONGRID_OPENAI_MODEL   (LLM，必填)
#   ONGRID_EMBEDDING_API_KEY / ONGRID_EMBEDDING_BASE_URL                   (RAG，可选)
```

> **embedding base_url 的坑**：ongrid 会对 base 自动补路径——base 以 `/v<数字>`
> 结尾则补 `/embeddings`，否则补 `/v1/embeddings`。所以 base **不要**写到
> `.../embeddings` 结尾，否则会被拼成 `.../v1/embeddings/v1/embeddings` → 404。
> 正确示例：`ONGRID_EMBEDDING_BASE_URL=https://<host>/v1`。

### 2.2 端口说明：后端监听 :8090 与前端对齐

前端 vite dev server 把 `/api` 代理到 **`http://localhost:8090`**（见
`web/vite.config.ts`）。示例 env 里 `ONGRID_HTTP_ADDR` 已默认设为 `:8090`，
开箱即与前端对齐，无需额外改动。

```bash
ONGRID_HTTP_ADDR=:8090   # 已在 deploy/ongrid.local.env 中默认设置
```

> 注：ongrid 二进制的内置默认值是 `:8080`（`config.go`）；这里靠 env 覆盖到
> `:8090`。若只调后端、不跑前端，把它改回 `:8080` 也可以。

### 2.3 启动

```bash
set -a; source deploy/.env; set +a
go run ./cmd/ongrid
```

后端起来后：

- HTTP API：`http://localhost:8090`（按上面设置）
- Metrics：`http://localhost:9100`

> **SearXNG 需手动配一次**：SearXNG 地址没有 env 开关，ongrid 首启会把
> `http://searxng:8080` 写进数据库。原生跑时请登录后在
> **设置 → Web 搜索** 里把 SearXNG URL 改成 `http://localhost:8080`，
> 否则 web_search 连不上。

---

## 第 3 步：启动前端（vite dev）

```bash
cd web
npm ci          # 首次或依赖变动时；平时可用 npm install
npm run dev
```

- 前端地址：`http://localhost:5173`
- vite 自动把 `/api/*` 代理到 `http://localhost:8090`（即第 2 步的后端）
- 改动 `.tsx` 会热重载

用第 2 步 `deploy/.env` 里的 `ONGRID_ADMIN_EMAIL` / `ONGRID_ADMIN_PASSWORD`
登录（示例默认 `admin@ongrid.local` / `change-me-on-first-login`）。

---

## 到底要启动哪些？（edge / 依赖服务）

日常改后端、前端代码，**只需上面 3 步**（依赖栈 + 后端 + 前端），不用起 edge，
也没有别的服务要额外拉起。

### edge 节点：默认不需要，调试端到端功能时才起

`ongrid-edge` 是跑在**被监控主机**上的 agent——它主动拨出连接后端的 frontier
(`:40012`)，负责上报主机指标/日志、执行远程命令。它是后端的**客户端**，不是后端的
依赖。

- 只做**控制面（后端/前端）开发调试** → **不用**起 edge。
- 要验证**边端接入、远程执行、主机指标采集、Browser SSH** 这类端到端功能 →
  按下面「第 4 步」起一个本地 edge。

---

## 第 4 步（可选）：启动 edge 进行调试

前提：第 1~2 步已就绪（deps 栈的 frontier 在 `127.0.0.1:40012`，后端已在跑）。

### 4.1 在 UI 创建一个 edge，拿一次性 access/secret

edge 靠一对 access/secret key 向后端认证。key 在 UI 里创建 edge 时**一次性**给出：

1. 浏览器打开前端 `http://localhost:5173`（或直接后端 `http://localhost:8090`）并登录。
2. 进入「设备 / Edges」页，右上角新建一台设备。
3. 弹窗会给出一次性的 **access key** 与 **secret key**——secret 只显示这一次，
   立刻复制下来。

### 4.2 启动 edge

edge 与后端**共用** `internal/pkg/config`，全部通过 `ONGRID_EDGE_*` 环境变量配置。
⚠️ **不要** `source deploy/.env`——那会把后端的 DB/JWT/metrics 等 env 一起带进来
（尤其 `ONGRID_METRICS_ADDR`），只需显式给 edge 这几个变量即可：

```bash
ONGRID_EDGE_CLOUD_ADDR=127.0.0.1:40012 \
ONGRID_EDGE_ACCESS_KEY=<第 4.1 步的 access key> \
ONGRID_EDGE_SECRET_KEY=<第 4.1 步的 secret key> \
ONGRID_EDGE_COLLECTOR_MODE=embedded \
go run ./cmd/ongrid-edge
```

关键 env 说明：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ONGRID_EDGE_CLOUD_ADDR` | `127.0.0.1:40012` | 后端 frontier 的 edgebound 地址；本地 deps 栈正好是它，可省略 |
| `ONGRID_EDGE_ACCESS_KEY` | 空 | **必填**，UI 创建 edge 时得到 |
| `ONGRID_EDGE_SECRET_KEY` | 空 | **必填**，同上（只显示一次） |
| `ONGRID_EDGE_COLLECTOR_MODE` | `off` | 设 `embedded` 才会周期性采集并上报本机指标；`off` 只保留按需 RPC |
| `ONGRID_EDGE_COLLECTOR_INTERVAL` | `10s` | 采集周期 |

> edge 的本地 debug metrics 端口是 `:9101`（与后端的 `:9100` 错开），所以
> 后端和 edge 可以在同一台机器上同时跑。

### 4.3 验证

- edge 启动日志出现 `configuration loaded`，且 `cloud_addr` 为 `127.0.0.1:40012`。
- UI「设备」页里刚创建的那台 edge 变为**在线**。
- 若设了 `COLLECTOR_MODE=embedded`，稍等一个采集周期后能在该 edge 的指标里看到主机数据。

### 依赖服务：硬依赖 vs 软依赖

后端启动时对 deps 栈里 8 个服务的依赖程度不同：

| 服务 | 类型 | 缺失后果 |
|---|---|---|
| **mysql** | **硬依赖** | 连不上 → 后端直接退出（`os.Exit`） |
| **frontier** | **硬依赖** | 连不上 → 后端直接退出（除非设 `ONGRID_FRONTIER_DISABLED=true`） |
| qdrant | 软依赖 | 知识库/RAG 写入不可用（读仍可），不阻止启动 |
| prometheus | 软依赖 | 指标查询/remote_write 功能不可用 |
| loki | 软依赖 | 日志查询功能不可用 |
| tempo | 软依赖 | 链路查询 / OTel 上报不可用 |
| searxng | 软依赖 | web_search 不可用 |
| grafana | 软依赖 | 面板嵌入 / SA token 自举不可用 |

> 换句话说：只要 **mysql + frontier** 起着、且 `ONGRID_JWT_SECRET` 不是内置默认值，
> 后端就能启动；其余服务缺了只是对应功能降级。用 deps 栈一把全起最省心。

---

## 一次性拉起（速查）

```bash
# 1) 依赖
docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps up -d

# 2) 后端（新开一个终端）
set -a; source deploy/.env; set +a   # 确保其中 ONGRID_HTTP_ADDR=:8090
go run ./cmd/ongrid

# 3) 前端（再开一个终端）
cd web && npm run dev
# 打开 http://localhost:5173

# 4) 可选：edge（再开一个终端；先在 UI 创建 edge 拿 access/secret）
ONGRID_EDGE_CLOUD_ADDR=127.0.0.1:40012 \
ONGRID_EDGE_ACCESS_KEY=<UI 创建得到> \
ONGRID_EDGE_SECRET_KEY=<UI 创建得到> \
ONGRID_EDGE_COLLECTOR_MODE=embedded \
go run ./cmd/ongrid-edge
```

---

## 常见问题

**前端登录页能开，但接口全部 404 / 502？**
后端没跑在 `:8090`（前端 proxy 的目标）。确认 `deploy/.env` 里
`ONGRID_HTTP_ADDR=:8090`（示例默认已是），且已 `source` 后重启后端。

**后端启动即报连不上数据库？**
mysql 还没 ready，或 `deploy/.env` 里 `ONGRID_DB_DSN` 不是指向 `localhost:3307`。
用 `docker compose -f deploy/docker-compose.deps.yml -p ongrid-deps ps` 看 mysql 是否 `healthy`。

**RAG / 知识库报 embedding 错误？**
检查 `ONGRID_EMBEDDING_API_KEY` 是否已填、`ONGRID_EMBEDDING_BASE_URL` 是否误写到
`.../embeddings` 结尾（见 2.1 的坑），以及 `ONGRID_EMBEDDING_DIM` 是否与模型输出维度一致
（`text-embedding-3-small` = 1536）。

**web_search 无结果 / 连接失败？**
在「设置 → Web 搜索」把 SearXNG URL 改成 `http://localhost:8080`（见第 2 步提示）。

**WSL2 环境注意**：以上 `127.0.0.1` / `localhost` 均指 WSL 发行版内部。若在
Windows 侧跑后端或前端再连 WSL 里的 Docker，需另配网络转发，不在本文范围。

