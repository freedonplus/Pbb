# Java / Spring 体系要点

## 版本匹配（必须成对给，错配是启动即失败的高频原因）

| Spring Boot | JDK | Spring Cloud | 备注 |
| --- | --- | --- | --- |
| 3.2.x / 3.3.x | 17 / 21 | 2023.0.x（Leyton） | 推荐基线；21 可用虚拟线程 |
| 3.0.x / 3.1.x | 17 | 2022.0.x | 稳定但已过主流支持 |
| 2.7.x | 8 / 11 | 2021.0.x | 仅遗留系统；2023 年底已停止免费支持 |

- JDK 21 + Spring Boot 3.3：虚拟线程（`spring.threads.virtual.enabled=true`）适合 IO 密集型；CPU 密集型收益有限，且需注意 `synchronized` 钉住载体线程。
- Spring Boot 3 起 `javax.*` → `jakarta.*`；引入旧依赖会编译不过，这是升级最常见坑。

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

## 启动与部署要点

- JVM 参数：`-XX:MaxRAMPercentage=75` 让容器内自适应堆；`-XX:+HeapDumpOnOutOfMemoryError` 落盘便于事后分析。
- 打可执行 jar（`spring-boot-maven-plugin`），镜像多阶段构建，非 root 用户运行。
- `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase` ≥ 在途请求最长耗时。
- Actuator 独立端口 + 只暴露 `health,prometheus`；主端口不暴露 actuator。
