# Java / Spring 体系要点

> 数据截止：2025 年中。**每次使用前必须用 WebSearch 核对当前稳定版**（搜「Spring Boot latest」「JDK LTS」「Spring Cloud release train」），本表只作结构参考。

## 版本匹配（必须成对给，错配是启动即失败的高频原因）

| Spring Boot | JDK | Spring Cloud | 备注 |
| --- | --- | --- | --- |
| 4.x 线 | 21 / 25 | 2025.x | 新一代主线；升级前核对其 release notes |
| 3.4.x / 3.5.x | 17 / 21 / 25 | 2024.x / 2025.x | 保守主线；3.2/3.3 已出 OSS 支持期，新项目不选 |
| 2.7.x | 8 / 11 | 2021.0.x | 遗留系统专用；免费支持已结束，新项目禁止 |

- JDK 选型：LTS 线优先（17 → 21 → 25）；21 起虚拟线程（`spring.threads.virtual.enabled=true`）适合 IO 密集型；CPU 密集型收益有限，且需注意 `synchronized` 钉住载体线程（JDK 24+ 已改善）。
- Spring Boot 3 起 `javax.*` → `jakarta.*`；引入旧依赖会编译不过，这是升级最常见坑。
- MySQL 选型：8.0 线已近支持尾声，新部署优先 8.4 LTS；客户端 `mysql-connector-j` 版本随服务端核对。

## 必要依赖基线（按需裁剪，不用的删）

```xml
<!-- Web / 校验 / AOP -->
spring-boot-starter-web
spring-boot-starter-validation
spring-boot-starter-aop
spring-boot-starter-actuator

<!-- 数据 -->
spring-boot-starter-jdbc
mybatis-plus-spring-boot3-starter          <!-- 或 spring-boot-starter-data-jpa -->
mysql-connector-j
flyway-core + flyway-mysql                 <!-- 版本化迁移 -->

<!-- 缓存 / 锁 -->
spring-boot-starter-data-redis
redisson-spring-boot-starter               <!-- 分布式锁 + 布隆过滤器 -->

<!-- 认证 -->
sa-token-spring-boot3-starter + sa-token-redis-jackson   <!-- 轻量方案 -->
<!-- 或 spring-boot-starter-security + jjwt -->

<!-- 文档 -->
springdoc-openapi-starter-webmvc-ui        <!-- OpenAPI 3 -->
knife4j-openapi3-jakarta-spring-boot-starter <!-- 中文 UI，可选 -->

<!-- 工具 -->
lombok                                     <!-- 编译期，scope=provided -->
mapstruct + mapstruct-processor            <!-- DTO 转换，比 BeanUtils 安全 -->
hutool-all                                 <!-- 视团队习惯；能不用则不用，体积大 -->
commons-lang3

<!-- 监控 -->
micrometer-registry-prometheus

<!-- 测试 -->
spring-boot-starter-test + mockito + testcontainers（需要真实中间件时）
```

依赖军规：
- `dependencyManagement` 统一版本，子模块不写 `<version>`；
- Lombok 与 MapStruct 同时用时注意注解处理器顺序（lombok-mapstruct-binding）；
- 禁止为了「方便」引入大而全的脚手架依赖（如整个 ruoyi-common），会带来大量不需要的传递依赖。

## 项目分层与包结构

```
com.<org>.<domain>
├── controller      # 入参校验、DTO→VO、不下业务判断
├── service         # 接口；impl 放实现，@Transactional 只在这一层
├── mapper          # MyBatis-Plus BaseMapper + 自定义 XML
├── entity          # 与表一一对应（不要当 DTO 用）
├── dto / vo / query# 入参 / 出参 / 查询条件，`@Sensitive` 注解按需
├── convert         # MapStruct 转换器
└── config          # 该域的自动配置
```

## 常见八股与工程要点（方案里要点名的）

1. **事务**：`@Transactional` 失效场景——同类自调用（走代理才生效）、非 public、异常被 catch、默认只回滚 `RuntimeException`、事务内做远程调用或 MQ 发送。方案里要求：事务只在 Service 层，事务内禁止外部 IO。
2. **循环依赖**：Spring Boot 2.6+ 默认禁止。方案要求通过接口下沉/事件解耦，禁止开 `allow-circular-references`。
3. **线程池**：禁止 `Executors.newFixedThreadPool`（无界队列 OOM）。用 `ThreadPoolTaskExecutor`，核心/最大/队列/拒绝策略四项显式配置；不同外部依赖隔离线程池。
4. **`ThreadLocal`**：必须 `try/finally` remove，线程池复用会串数据（用户上下文、租户 ID 最典型）。
5. **BigDecimal 军规**：
   - 金额、税率、比例一律 `BigDecimal`，禁止 `double`/`float`；
   - 构造用 `new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)`，**禁止 `new BigDecimal(0.1)`**（会把二进制误差原样带进来）；
   - 比较用 `compareTo(...) == 0`，`equals` 会比较 scale（`1.0` ≠ `1.00`）；
   - 除法必须给 scale 和 `RoundingMode`，否则除不尽抛 `ArithmeticException`；
   - 序列化输出字符串（`@JsonSerialize(using = ToStringSerializer.class)` 或全局配置），避免前端 JS 精度丢失；
   - 存库用 `DECIMAL(18,2)`，不要 `DOUBLE`。
6. **时间**：优先 `LocalDateTime`/`Instant`，禁止 `java.util.Date`；`DateTimeFormatter` 是线程安全的，`SimpleDateFormat` 不是；Jackson 统一 `yyyy-MM-dd HH:mm:ss` + 时区。
7. **BigDecimal/Long 精度**：雪花 ID 超出 JS `Number.MAX_SAFE_INTEGER`，Long 主键必须序列化成字符串（全局 `Jackson` 配置 `Long → String`），否则前端拿到错误 ID。
8. **自动装配**：common 模块用 `@AutoConfiguration` + `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot 3 不再用 `spring.factories`），配合 `@ConditionalOnProperty`、`ObjectProvider` 让可选依赖（短信、对象存储）缺失时不阻断启动。
9. **配置文件**：`application-{profile}.yaml` 全量自包含，敏感项 `${ENV_VAR}` 无默认值 → 缺失即失败，杜绝「生产静默跑在开发默认值上」。
10. **@ConfigurationProperties vs @Value**：结构化配置用 `@ConfigurationProperties` + `@Validated`；注意它**不会**因占位符解析失败而报错（会原样保留 `${X}`，直到使用处才炸），所以关键项要在启动时显式校验。
11. **接口文档**：`springdoc-openapi` + `@Operation`/`@Schema`，`@Schema(description=...)` 写清字段含义与取值；生产 `springdoc.api-docs.enabled=false`（只关自己的开关不够，springdoc 自动配置默认开启）。
12. **MyBatis-Plus**：分页插件必须注册；`useAffectedRows=true` 影响「更新影响行数」判断（乐观锁依赖它）；逻辑删除与唯一索引冲突要提前设计。

## Java 基础八股（方案/评审引用口径，原理详见 ../bagu-essentials.md）

### 浮点与金额精度
- `float`/`double` 是二进制浮点，0.2 这类十进制小数无法精确表示，`2.0f - 1.9f` ≠ `0.1`——**金额、比例、任何小数点计算禁止用浮点**。
- `BigDecimal` 用法军规（构造用字符串、`compareTo` 比较、除法指定 scale+RoundingMode、序列化输出字符串）见上文「常见八股」第 5 条。
- `Float`/`Double` 包装类**没有缓存**，`==` 比较必翻车；所有包装类比较一律 `equals`（Integer 缓存 -128~127，可用 `-XX:AutoBoxCacheMax` 调上限，不建议调大）。

### 字符串与拼接
- `String` 不可变；`+` 拼接在循环里每次产生新对象，**循环内拼接必须 `StringBuilder`**（编译器虽会把单行 `+` 优化成 StringBuilder，但循环体优化不了）。
- `StringBuilder` 非线程安全、`StringBuffer` 线程安全（synchronized）；单线程场景用前者，别为「保险」付出同步开销。
- 大量日志拼接用 `String.join` / `String.format` 时注意开销；高频路径优先 SLF4J 的 `{}` 占位符（惰性求值），禁用 `"日志：" + obj.toString()` 先拼后判级别的写法。
- 常量拼接（编译期可确定的 `+`）会被常量折叠，无性能问题。

### 集合
- `HashMap` 非线程安全，并发场景用 `ConcurrentHashMap`（JDK 8 起 CAS + synchronized 锁桶头，不再用分段锁）；`Hashtable`/`Collections.synchronizedMap` 全表锁，禁用。
- `ArrayList` 扩容 1.5 倍，预估容量用 `new ArrayList<>(n)` 避免多次扩容拷贝；`LinkedList` 随机访问是 O(n)，别当默认选型。
- `Arrays.asList` 返回定长视图，增删抛 `UnsupportedOperationException`；`List.of` 不可变。
- 遍历中删除用 `Iterator.remove()`，for-each 里删会 `ConcurrentModificationException`。

### 并发基础
- `synchronized`（JVM 内置锁，可重入、自动释放）vs `ReentrantLock`（可中断、可超时、公平锁、多条件队列）；无特殊需求优先 `synchronized`（JDK 6 后优化很好）。
- `volatile` 只保证可见性与有序性，**不保证原子性**（`i++` 照样丢更新）；状态标志用 volatile，计数用 `AtomicInteger`（CAS）。
- CAS 的 ABA 问题用 `AtomicStampedReference`；高竞争计数用 `LongAdder`（分段 Cell，比 AtomicLong 吞吐高）。
- `ThreadLocal` 用完必须 `remove()`（线程池复用会串数据）；需要跨线程传递上下文用 `TransmittableThreadLocal`（阿里开源）。
- 线程池七大参数（核心/最大/存活时间/队列/线程工厂/拒绝策略）必须显式给；**禁止 `Executors` 工厂方法**（无界队列 OOM / 无限线程）。

## JVM 调优要点（方案性能章引用）

- **内存模型**：堆 = 新生代（Eden + S0/S1，复制算法）+ 老年代（标记-整理/混合）；JDK 8 起永久代被 Metaspace（本地内存）取代，需设 `-XX:MaxMetaspaceSize` 防元空间吃光内存。
- **GC 判定**：可达性分析（GC Roots：栈局部变量、静态属性、常量、JNI、同步锁持有对象）；引用计数无法解决循环引用，Java 不用它。
- **收集器选型**：JDK 9+ 默认 G1（`-XX:MaxGCPauseMillis=200` 调停顿目标）；低延迟场景（P99 < 10ms）用 ZGC/Shenandoah（亚毫秒停顿，JDK 17+ 生产可用）；小堆（<100MB）/客户端用 Serial；吞吐优先批处理用 Parallel。
- **调优方法论**：先定目标（吞吐 or 停顿）→ 看 GC 日志（`-Xlog:gc*:file=gc.log:time,level,tags`）→ 分析（GCeasy/gceasy.io）→ 再调参数。**禁止不看日志就抄参数**。
- **常用参数**：容器内 `-XX:MaxRAMPercentage=75`；`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=`；G1 区域大小 `-XX:G1HeapRegionSize`（大对象 Humongous 超 Region 一半直接进老年代）。
- **性能优化通用序**：监控指标先行（无基线不优化）→ SQL/索引（收益最大）→ 缓存 → JVM 参数 → 代码级（拼接、装箱、集合）→ 架构级（异步、批处理、分库）。顺序不能反。

## API 文档（Java 必给）

- 依赖：`springdoc-openapi-starter-webmvc-ui`（+ 可选 `knife4j-openapi3-jakarta-spring-boot-starter` 提供中文界面）。
- 分组：`GroupedOpenApi` 按业务模块分组（`/admin/**`、`/app/**`）。
- 契约要素：`@Schema` 标注 DTO 字段（必填、示例、枚举）、`@Operation` 标注接口用途与权限点。
- 导出：`/v3/api-docs` 可导出 JSON，直接导入 Apifox/Postman；方案里给出该地址与生产开关策略。
- 安全：生产关闭或挂在管理端口；文档端点不放公网（会泄露全部接口与字段）。

## 认证选型对比（方案里要给的否决项）

| 维度 | Sa-Token | Spring Security |
| --- | --- | --- |
| 上手成本 | 低，API 直白（`StpUtil.login`） | 高，过滤器链概念多 |
| 权限缓存 | 内置 Redis 集成，权限变更即时生效 | 需自己实现 `UserDetailsService` 缓存 |
| 生态标准度 | 国内流行，非 Spring 官方 | 官方，企业/合规场景更被认可 |
| 复杂场景 | OAuth2/SSO 需额外组件 | OAuth2 Resource Server、SAML 原生 |
| 建议 | 中小团队、自建后台 | 有合规要求、需标准 OAuth2/OIDC |

注意：Sa-Token 的权限/角色有 Redis 缓存与「权限版本号」机制，直接改库不会失效，必须走接口或清缓存——方案中要写明这条运维约束。

## 其他语言的降级规则

stacks 目录只覆盖 Java 与 Go/Python/Node。用户语言为 PHP / C# / Rust 等**无基线文件**的语言时：
1. 显式声明「本技能无该语言 stacks 基线」；
2. 版本与依赖以官方文档/官方 release notes 为准（用 WebSearch 核对），禁止凭记忆现编；
3. 语言陷阱部分引用 `audit-checklist.md` Part 3 对应小节；
4. 方案中该语言的依赖清单标注「待人工复核」。

## 启动与部署要点

- JVM 参数：`-XX:MaxRAMPercentage=75` 让容器内自适应堆；`-XX:+HeapDumpOnOutOfMemoryError` 落盘便于事后分析。
- 打可执行 jar（`spring-boot-maven-plugin`），镜像多阶段构建，非 root 用户运行。
- `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase` ≥ 在途请求最长耗时。
- Actuator 独立端口 + 只暴露 `health,prometheus`；主端口不暴露 actuator。
