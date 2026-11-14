# Go / Python / Node 体系要点

> 数据截止：2025 年中。**每次使用前用 WebSearch 核对各语言当前稳定版与 LTS 支持状态**（Go releases / Node.js LTS schedule / Python EOL 表），文中版本号只作结构参考。

Java 之外的三种常见后端，八股骨架与 Java 同构（架构 → 分层 → 数据 → 中间件 → 并发 → 安全 → 可观测 → 契约），此处只列**差异点**与必答项。Go/Python/Node 以及 PHP/C#/Rust 的详细八股与生产陷阱另见 `../bagu-polyglot.md`；AI 辅助编码必须同时加载 `../security-assurance.md`。

---

## Go

### 版本与框架
- Go 1.22+（推荐）；`go.mod` 锁定依赖，`GOFLAGS=-mod=readonly`。
- Web：Gin（单体/中小）、Go-Zero（微服务，自带代码生成与治理）、Kratos（B 站系，protobuf 契约优先）。
- 结构：`cmd/`（入口）、`internal/`（业务，禁止外部导入）、`pkg/`（可复用）、`api/`（proto/契约）、`configs/`。

### 必答工程点
1. **金额精度**：`float64` 不可用于金额 → `int64`（分）或 `shopspring/decimal`。
2. **error 处理**：不使用 `panic` 做业务控制流；统一 `errors.Is/As` + 自定义错误码；入口层统一转 HTTP 响应。
3. **goroutine 泄漏**：所有 goroutine 必须可退出（`context.WithTimeout` / `done` channel）；`errgroup` 管理并发；禁止裸 `go func()` 无回收路径。
4. **context 传递**：DB/Redis/HTTP 调用全部透传 `ctx`，超时逐层收敛。
5. **并发安全**：`map` 并发读写会 panic → `sync.Map` 或加锁；`-race` 必须进 CI。
6. **连接池**：`sql.DB` 的 `SetMaxOpenConns/SetMaxIdleConns/SetConnMaxLifetime` 三项显式设置，默认无上限。
7. **配置**：viper/envconfig + 启动时校验；敏感项无默认值。
8. **API 文档**：`swaggo/swag` 由注释生成 `docs/`，或**契约优先**用 `openapi.yaml` + `oapi-codegen` 生成 server stub。推荐后者（契约与代码强一致）。

### 依赖基线（按需）
`gin-gonic/gin` / `go-redis/redis/v9` / `gorm.io/gorm` + `gorm.io/driver/mysql` / `redis/go-redis` 分布式锁用 `bsm/redislock` / `golang-jwt/jwt/v5` / `casbin/casbin/v2`（权限）/ `prometheus/client_golang` / `uber-go/zap` / `stretchr/testify`。

---

## Python

### 版本与框架
- Python 3.12+（推荐）；`uv` 或 Poetry 管理依赖，锁定 `uv.lock` / `poetry.lock`。
- Web：FastAPI（推荐，异步 + 自动 OpenAPI）、Django + DRF（管理后台/ORM 完备）。
- 结构：`app/api`（路由）、`app/services`（业务）、`app/models`（ORM）、`app/schemas`（Pydantic）、`app/core`（配置/安全）、`alembic/`（迁移）。

### 必答工程点
1. **金额精度**：`float` 不可用 → `decimal.Decimal` + `quantize(Decimal("0.01"), ROUND_HALF_UP)`；DB 用 `NUMERIC(18,2)`。
2. **同步 vs 异步**：FastAPI 里 `async def` 中禁止调用阻塞库（`requests`、同步 DB 驱动）——会堵死事件循环；要么全异步（`httpx` + `asyncmy`/`asyncpg`），要么用 `def` 让线程池接管。这一条必须在方案里写明。
3. **GIL 与部署**：CPU 密集用多进程 worker（`gunicorn -k uvicorn.workers.UvicornWorker -w 4`），不用多线程。
4. **SQLAlchemy**：session 生命周期与请求绑定（依赖注入），禁止全局 session；N+1 用 `selectinload`。
5. **Pydantic v2**：`model_config = ConfigDict(from_attributes=True)`，schema 与 ORM 分离。
6. **配置**：`pydantic-settings` 从环境变量读取，缺失即启动失败。
7. **API 文档**：FastAPI 内置 `/docs`、`/redoc`、`/openapi.json`；生产关闭或加鉴权。契约即代码，无需额外工具。

### 依赖基线
`fastapi` / `uvicorn[standard]` / `pydantic` + `pydantic-settings` / `sqlalchemy` + `alembic` / `asyncmy` 或 `psycopg[binary]` / `redis` / `python-jose[cryptography]` 或 `pyjwt` / `passlib[bcrypt]` / `httpx` / `structlog` 或 `loguru` / `prometheus-fastapi-instrumentator`。

---

## Node.js

### 版本与框架
- Node 20 LTS（推荐）/ 22 LTS；包管理 pnpm（推荐）；`engines` 字段锁定版本。
- 框架：NestJS（推荐，模块化/DI/装饰器，最接近 Java 分层思想）、Express/Koa（仅小项目或 BFF）。
- 结构：`src/modules/<domain>`（module/controller/service/entity/dto）、`src/common`（过滤器/拦截器/守卫）、`src/config`。

### 必答工程点
1. **数值精度**：JS `Number` 双精度 → 超过 `2^53` 的 ID 失真；金额用整数分，decimal 用 `decimal.js`；**后端返回 ID 一律字符串**。
2. **事件循环**：CPU 密集任务（加密、大文件、复杂计算）用 `worker_threads` 或独立服务；禁止在请求链路里做同步大计算。
3. **单线程崩溃**：未捕获的 Promise rejection 会终止进程 → 全局 `unhandledRejection`/`uncaughtException` 处理器 + PM2/K8s 重启；进程必须无状态。
4. **内存泄漏**：全局数组/Map 缓存必须设上限与 TTL；`EventEmitter` 监听器要移除。
5. **DB**：Prisma（推荐，类型安全 + 迁移）或 TypeORM；连接池配置显式；事务用 `$transaction`。
6. **依赖安全**：`npm audit` 进 CI；lockfile 必须提交；避免 `postinstall` 脚本型恶意包。
7. **API 文档**：`@nestjs/swagger` 装饰器（`@ApiProperty`、`@ApiOperation`）生成 OpenAPI；生产关闭或加鉴权。

### 依赖基线
`@nestjs/core` + `@nestjs/common` + `@nestjs/platform-express` / `@nestjs/config` / `@nestjs/swagger` / `prisma` + `@prisma/client` / `ioredis` / `@nestjs/jwt` + `passport-jwt` / `bcrypt` / `class-validator` + `class-transformer` / `pino` / `prom-client`。

---

## 跨语言通用军规（写方案时逐条对照）

1. 时间统一时区、统一格式；DB 存 UTC 或东八区，二者择一。
2. ID 对外一律字符串（Java Long / JS Number 都会被前端精度坑）。
3. 金额用整数分或 decimal 类型，禁用浮点。
4. 所有外部调用必须设超时与有限重试。
5. 日志结构化 + traceId；错误日志带上下文，不打全量入参（防敏感信息泄露）。
6. 接口文档由代码/契约生成，禁止手写 Word 与代码脱节。
7. 前后端边界：后端只交付接口与契约文档，不写前端。
