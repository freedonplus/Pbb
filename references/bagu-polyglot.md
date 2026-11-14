# 多语言八股与生产陷阱速查

> 用途：方案撰写、代码审核、人工评审时统一 Go / Python / Node.js / PHP / C# / Rust 的技术口径。Java/JVM/MySQL/Redis/MQ 的通用底稿见 `bagu-essentials.md`。
> 本文件关注“面试常问且生产真会踩”的知识，不追求罗列语法。具体版本、框架支持与依赖漏洞必须在使用时查官方文档和漏洞库。
> 原则：语言特性不能替代业务正确性；类型系统、测试和静态扫描也都不能证明系统没有漏洞。

---

## 1. Go

### 并发与 goroutine
- goroutine 是 Go 运行时调度的轻量执行单元，不等于“免费且无限”。每个 goroutine 都必须有明确退出条件；禁止没有 `context`、退出 channel 或回收路径的裸 `go func()`。
- `context.Context` 用于截止时间与取消传播，必须透传到 DB、Redis、HTTP 等下游；不要把 Context 当业务参数容器，更不要把它长期塞进结构体。
- channel 关闭后再发送会 panic；向 nil channel 收发永久阻塞；`for range` 只在 channel 关闭后结束。通常由发送方关闭 channel，多个发送方时用 `WaitGroup`/单独协调者，不重复关闭。
- 无缓冲 channel 提供同步点，有缓冲 channel 提供有限队列；缓冲不是无限削峰，必须有消费者、背压和退出策略。
- `select` 多个 case 同时就绪时伪随机选择；`select + ctx.Done()` 可防永久阻塞，但不能代替下游真正支持取消。

### 内存、数据与接口
- map 并发读写可能触发 fatal error，普通 mutex 不能靠“概率”省掉；用 `sync.RWMutex` 或合适的并发结构，CI 必须跑 `go test -race`。
- slice 是“指针 + len + cap”的描述符，切片共享底层数组；扩容后可能换数组。函数返回子切片可能意外保活大数组，必要时复制。
- `defer` 在函数返回时执行，参数在 defer 注册时求值；循环内大量 `defer Close()` 会把资源拖到函数结束，批处理应拆函数或显式关闭。
- `error` 是接口；业务错误用 `errors.Is/As` 判断，不比较字符串。业务失败禁止 `panic`，panic 只留给不可恢复的程序错误，并在服务边界 recover。
- 接口值由动态类型和动态值组成：接口里装“有类型的 nil 指针”时，接口本身不等于 nil，这是常见判空坑。
- `sync.Mutex` 不可复制；包含锁的结构体应传指针，复制后会得到两把互不相干的锁。

### 精度、时间与工程化
- `float32/float64` 都是二进制浮点；金额用整数分或十进制 decimal 库，不用浮点做资金比较。
- `time.Time` 可能带单调时钟读数；序列化、跨进程传输后单调部分消失。超时优先 `context`/`time.Timer`，不要靠人工计算 wall clock 猜竞态。
- 常用检查：`go test -race`、`go vet`、`golangci-lint`、`govulncheck`。`-race` 只能在实际执行到的路径上发现数据竞争，不能证明无竞争。
- 性能分析用 `pprof` + `trace`，先看 CPU、堆、阻塞和 goroutine profile，再改代码；禁止凭感觉“优化 GC”。

### 生产审核追问
1. 所有 goroutine 的退出路径在哪里？压测后 goroutine 数是否回落？
2. 共享 map/slice 是否有锁？是否有 `-race` 证据？
3. HTTP/DB/Redis 是否都尊重 Context 截止？
4. `sql.DB` 最大连接、空闲连接、连接寿命是否显式配置？
5. 错误是否保留 wrapping 链？是否把内部错误细节泄露给客户端？

---

## 2. Python

### 类型、对象与精度
- `==` 比值，`is` 比对象身份；不要用 `is` 比较字符串或数字。`None` 判断用 `is None`。
- 默认参数在函数定义时求值：`def f(items=[])` 会跨调用共享列表；默认可变参数改用 `None`，函数体内新建。
- 浅拷贝共享嵌套对象，深拷贝用 `copy.deepcopy`，但深拷贝不是万能方案，可能复制锁、连接等不可复制资源。
- `float` 是二进制双精度；金额用 `decimal.Decimal`，从字符串构造，按业务规则 `quantize` 并指定 rounding。Decimal 也不能自动修复错误的舍入口径。
- Python 类型提示默认不在运行时强制校验；Pydantic/Attrs 等入口校验与静态 mypy/pyright 是两层不同保障。

### GIL、线程、进程与异步
- CPython 的 GIL 使纯 Python CPU 密集代码难以靠多线程利用多核；IO 密集任务可受益于线程/异步，CPU 密集优先多进程或原生扩展。不要把 GIL 简化成“Python 永远不能并行”。
- `asyncio` 是协作式调度：协程里一次阻塞调用就可能拖住整个事件循环。`async def` 中禁止同步 DB、同步 HTTP、重 CPU；阻塞 IO 用线程卸载，CPU 用进程卸载，并限制并发。
- 创建 task 后要持有引用并处理异常；“fire-and-forget”任务会被取消、异常丢失或进程退出时截断。服务关闭时必须取消并等待任务收尾。
- 异步库与同步库不能混着碰同一连接/session；连接池、事务上下文必须按任务/请求隔离。

### ORM、序列化与安全
- SQLAlchemy Session 不是全局单例资源；请求结束必须关闭，异步 Session 不能跨并发任务共享。N+1 用 `selectinload/joinedload` 等明确策略治理。
- `pickle` 反序列化不可信数据可执行任意代码；跨信任边界用 JSON/MessagePack 等安全格式并校验 schema。PyYAML 用 `safe_load`，不用通用 `load`。
- ORM 的参数绑定不覆盖动态表名、列名和排序字段；这些标识符必须白名单，不能直接把用户输入拼进 SQL。
- 密码使用专用密码哈希（Argon2/bcrypt 等）；`hashlib.sha256(password)` 不是密码存储方案。

### 生产审核追问
1. FastAPI 路由里是否有阻塞库？事件循环 lag 是否监控？
2. 后台 task 的异常、取消、重试与关闭流程在哪里？
3. 可变默认参数、全局缓存、全局 Session 是否清理？
4. worker 数是否按 CPU/内存推导，而非抄固定值？
5. lockfile、`pip-audit`/`safety`、Bandit 是否进入 CI？

---

## 3. Node.js / TypeScript

### 事件循环与异步
- Node 的 JS 主线程执行用户代码；异步 IO 由运行时和线程池协作完成。CPU 密集、同步大 JSON、同步压缩/加密都可能阻塞所有请求。
- `setTimeout`、`setImmediate`、Promise microtask 的执行阶段不同；不要依赖未明确文档化的时序实现业务正确性。
- Promise 必须 `await` 或显式处理 rejection；`Promise.all` 任一失败即拒绝，适合“全成功才有意义”；需要收集全部结果用 `allSettled`，但要限制并发，不能一次创建百万 Promise。
- `forEach(async ...)` 不会按预期等待串行完成；串行用 `for...of + await`，并行用受控 worker pool。
- `uncaughtException` 后进程状态可能已损坏，记录并优雅退出，由进程管理器重启；不能捕获后继续当没事发生。

### 数值、对象与类型
- JS `Number` 是 IEEE 754 双精度，安全整数范围到 `2^53-1`；大整数用 `BigInt`，但 BigInt 不能与 Number 直接混合运算，JSON 也不能原生序列化 BigInt。
- 金额用整数分或十进制定点库；大 ID 对外用字符串。TypeScript 的 `number` 不会改变运行时精度。
- 宽松相等 `==` 会隐式转换，业务代码用 `===`；`NaN !== NaN`，判断用 `Number.isNaN`。
- TypeScript 类型在编译后消失；API 入参标注为 DTO 类型不等于运行时安全，必须用 class-validator/Zod 等做 schema 校验，并剥离未知字段。
- 对用户对象做递归 merge、路径赋值时防 `__proto__` / `constructor` / `prototype` 原型污染；优先白名单复制，不信任通用深合并。

### 流、依赖与进程
- 文件上传下载、CSV 导入导出用 Stream 并处理 backpressure；一次性读入内存会把大文件变成 OOM。
- EventEmitter 监听器、全局 Map、未清理 timer 都可能泄漏；连接、订阅、定时器必须有对称释放。
- `npm ci` 按 lockfile 可复现安装；依赖脚本（install/postinstall）是供应链执行入口，必要时禁用并逐项例外，不能盲目安装 AI 推荐的新包。
- `npm audit` 不是完整安全证明，还需审查维护者、包名拼写、下载异常、Lockfile  diff 和恶意版本。

### 生产审核追问
1. 是否有 event-loop delay、heap、句柄数指标？
2. 异步任务是否限并发并处理 rejection？
3. DTO 是否运行时校验并禁止批量赋值敏感字段？
4. 大文件是否有流式背压？
5. 进程收到 SIGTERM 后是否停止接流量并清理任务？

---

## 4. C# / .NET

### 数值、对象与资源
- 金额用 `decimal`；`double/float` 是二进制浮点。`decimal` 也必须在除法、税率、分摊处明确舍入规则。
- 引用类型默认按引用共享；`record` 通常提供值语义相等，`class` 默认引用相等。不要靠类型名字猜相等语义。
- Nullable Reference Types 是编译期分析辅助，不是运行时非空保证；外部 JSON/DB 数据仍需验证。
- `IDisposable/IAsyncDisposable` 管理的连接、流、句柄要用 `using/await using`；依赖注入容器负责其创建对象的释放，不要重复释放共享服务。

### async / await
- 除事件处理器等少数场景，禁止 `async void`：调用方无法等待，异常也难以正常捕获。
- `.Result/.Wait()/GetAwaiter().GetResult()` 可能造成死锁或线程池饥饿；异步链路优先一路 async，不在中间同步阻塞。
- ASP.NET Core 通常没有传统 SynchronizationContext，`ConfigureAwait(false)` 不是每个库的机械必写项；库代码按项目约定使用，真正红线是同步等待异步和无边际并发。
- `Task.Run` 不是让所有代码变快的按钮；ASP.NET 请求里它常用于卸载真正 CPU/阻塞工作，必须有并发限制，IO 应优先原生异步 API。
- `CancellationToken` 要从请求传到底层，超时、断开连接和服务停止都能取消；只创建 token 不传递等于没做。

### EF Core 与 Web
- `DbContext` 不是线程安全对象，通常按请求 Scoped；禁止并发共享，也不要把实体上下文缓存到单例。
- LINQ 常是延迟执行，多次枚举可能重复查询；`IQueryable` 在 Provider 上翻译，随意在表达式里调用本地方法可能客户端求值或报错。
- N+1 用投影、Include/分批加载治理；跟踪查询只读场景用 `AsNoTracking`，但不能因此忽略实体归属校验。
- Model Binding 防止 over-posting：使用专用 DTO，不把包含 `IsAdmin/Balance` 的实体直接绑定请求。
- 性能先看指标和 profiler；`Span<T>`、ArrayPool 是优化工具，不应把业务代码普遍复杂化。

---

## 5. PHP

### 类型与数值
- 项目启用 `declare(strict_types=1)`，但仍要理解输入、JSON、DB 驱动处的类型转换；严格类型不是全链路免疫。
- `==` 会类型 juggling，安全比较用 `===`；hash 比较用 `hash_equals`，不要用普通等号比较签名/令牌。
- 浮点同样不适合金额；用整数分或经过审查的十进制库，DB 使用 DECIMAL 并统一舍入。
- 数组既是有序 map 也是常用列表结构，键转换和字符串数字键可能带来意外；对外响应前先确认 JSON 会编码成数组还是对象。

### Web 与代码执行红线
- `include/require/eval/assert` 等执行能力绝不能接受用户可控路径或代码；远程 URL include 必须禁用，不把“过滤 ..”当完整防线。
- `unserialize()` 不可信数据可能触发对象注入链；跨信任边界用 JSON 并做 schema 校验。
- `exec/shell_exec/system/passthru/proc_open` 不拼用户输入；确实需要系统调用时用固定命令、参数数组/严格转义、最小权限和隔离环境。
- PDO 预处理语句防 SQL 注入；动态表名、列名、ORDER BY 仍需白名单。
- 密码使用 `password_hash`（优先 Argon2id，按环境能力选）与 `password_verify`，不要自研加密或快速哈希。

### FPM、OPcache 与依赖
- PHP-FPM `pm.max_children` 必须按单进程实际内存与机器内存推导；设置过大是 OOM，不是性能优化。
- OPcache 发布后要确保新代码与 worker 一致，部署流程包含 reload/重启策略；长驻 worker 的连接与静态状态要防跨请求污染。
- 文件上传不能依赖浏览器 MIME；检查大小、扩展名、内容与存储隔离，上传目录不可执行 PHP。
- Composer 使用 lockfile，审查包维护者与脚本；`composer audit`、Roave Security Advisories 等进入 CI，不能只检查语法。

---

## 6. Rust

### 所有权、借用与错误
- Ownership 决定资源释放，Borrowing 允许在不转移所有权时访问；可变借用独占、不可变借用可共享，规则在编译期阻止大量数据竞争与悬垂引用。
- Lifetime 描述引用有效关系，不是“延长对象寿命”的语法。不要为消除编译错误随意加 `'static`，真正要修的是所有权边界。
- `Option`/`Result` 把缺失和失败显式化；`unwrap/expect` 会在值不符合预期时 panic。服务入口和不可恢复初始化可谨慎 expect，普通请求路径禁止随手 unwrap。
- 错误库（thiserror/anyhow 等）只是组织方式；对外仍要映射稳定错误码，内部错误链不直接泄露。
- Rust 默认 release 的整数溢出行为与 debug 不同；金额、库存、计数等使用 checked/saturating 运算并明确业务语义，不能假设一定 panic。

### 并发与异步
- `Send` 表示可跨线程转移，`Sync` 表示可跨线程共享引用；编译器能防许多数据竞争，但防不了死锁、业务竞态、重复扣款和逻辑错误。
- `Arc<Mutex<T>>` 是共享状态工具，不是默认答案；锁粒度、持锁跨 `.await`、锁顺序必须审查。异步上下文按运行时选择合适锁，避免持标准库锁跨 await。
- Tokio 异步任务里禁止阻塞运行时线程；阻塞 IO/CPU 用 `spawn_blocking` 或专用池并限流。`block_on` 在异步运行时内误用会直接造成问题。
- 任务要处理取消与关闭；`tokio::spawn` 后不管 JoinHandle，异常和结果可能被忽略，关闭时也可能直接丢任务。
- channel 有界才具备背压；无界 channel 会把下游慢转成内存持续增长。

### unsafe、FFI 与供应链
- `unsafe` 不关闭全部借用规则，而是把部分正确性责任交给开发者；每个 unsafe 块必须有局部理由、边界封装、测试和人工评审。
- FFI 的字符串编码、指针所有权、释放方、线程模型必须写清；跨语言内存不是“编译通过就安全”。
- panic 策略（unwind/abort）影响服务恢复与资源清理；进程边界、supervisor 和 readiness 要按策略设计。
- Cargo.lock 对应用入库，CI 跑 `cargo clippy`、测试、`cargo audit`/`cargo deny`；关注 build.rs 和 proc macro 的供应链执行能力。

---

## 7. 跨语言统一问题清单

| 主题 | 必须回答的问题 |
| --- | --- |
| 金额与 ID | 是否定点/整数分？大整数序列化是否失真？舍入规则是否唯一？ |
| 时间与超时 | 时区、时钟来源、deadline 传递、重试上限是否明确？ |
| 并发 | 共享状态在哪里？谁负责取消？队列是否有界？关闭时如何排空？ |
| 空值与错误 | 空值、异常、部分失败是否显式建模？错误是否被吞？ |
| 序列化 | 不可信反序列化是否安全？未知字段是否拒绝/剥离？ |
| 数据访问 | 参数化是否覆盖所有输入？动态标识符是否白名单？N+1 是否测试？ |
| 资源释放 | 连接、流、task、timer、订阅是否有对称释放？ |
| 供应链 | lockfile、漏洞扫描、构建脚本审查、制品溯源是否齐全？ |
| 性能 | 是否有 profile/指标依据？是否先修 SQL/架构再谈语言微优化？ |
| 业务正确性 | 即使语言层面“内存安全”，业务不变量是否有独立测试与人工复核？ |

## 8. 使用规则

1. 根据实际语言加载对应章节，不把 Java 结论机械翻译到其他语言。
2. 原理底稿用于提出核查问题，不替代官方文档、实际编译和运行测试。
3. 审核结论必须附代码位置、测试输出或运行时证据；只写“该语言很安全”一律无效。
4. 涉及高风险业务时，同时加载 `security-assurance.md`，语言级安全不能替代人工业务放行。
