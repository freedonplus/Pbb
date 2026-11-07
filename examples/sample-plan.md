# 示例输出（节选）

> 虚构项目：某区域旅游线路电商。以下为 Skill 产出的方案节选，用于展示**结构与密度**，不是完整方案（完整方案约 2500 字，12 章齐全）。
> 输入条件：Java、中小型（日活 3 万）、团队 3 人、有库存扣减、接微信支付、无前端团队（前端外包）。

---

# 青野旅行 生产级后端方案

> 版本 v1.0 · 语言/框架：Java 21 + Spring Boot 3.3 ｜ 架构：多模块单体
> 需求来源：线路浏览下单、优惠券、客服工单，3 人团队半年内上线

## 二、架构选型

**选定**：多模块单体（Maven），模块 `app` / `common` / `catalog` / `trade` / `customer` / `operations`。

| 模块 | 职责 | 依赖 |
| --- | --- | --- |
| app | 启动、配置、Controller 装配 | 全部业务模块 |
| common | Result/异常/加解密/Redis/自动装配 | 无 |
| catalog | 线路、班期、库存 | common |
| trade | 订单、支付、退款、优惠券 | common、catalog（Service 接口） |
| customer | 会员、认证 | common |
| operations | 工单、客服、运营 | common、trade（只读查询） |

**理由**：
1. 3 人团队独立发布，微服务的注册中心/配置中心/分布式事务运维成本无人承担；
2. 模块边界与业务域一致，未来拆服务时 `catalog`、`trade` 天然是服务边界，接口已隔离；
3. 单体下单链路可在一个本地事务内完成，避免分布式事务。

**否决项**：微服务（团队规模不足，Nacos + 链路追踪 + 分布式事务的成本 > 收益）；简单单模块（6 个业务域挤在一个模块，2 个月后必然循环依赖）。

**演进触发条件**：任一模块的发布频率与主应用冲突、或单模块 QPS 超过单机承载的 60% → 先拆 `trade`。

## 三、技术栈

| 层 | 选型 | 版本 | 理由 |
| --- | --- | --- | --- |
| 语言 | JDK | 21 LTS | 虚拟线程可降低 IO 等待成本；3 年 LTS 支持 |
| 框架 | Spring Boot | 3.3.x | 与 JDK 21 匹配；Spring Cloud 2023.0.x 备用 |
| 构建 | Maven | 3.9 | 多模块 + dependencyManagement 统一版本 |
| ORM | MyBatis-Plus | 3.5.7 | CRUD 零 XML；复杂查询仍可写 SQL |
| 数据库 | MySQL | 8.0 | 单库可承载（估算峰值 120 QPS，日增 2 万订单） |
| 缓存 | Redis | 7.2 | 会话、权限、线路详情、库存预扣 |
| 锁 | Redisson | 3.32 | 看门狗续期，避免固定 TTL 提前释放 |
| 认证 | Sa-Token | 1.44 | 3 人团队，Spring Security 过滤器链成本过高 |
| 文档 | springdoc + Knife4j | 2.5 / 4.5 | OpenAPI 3，可直接导入 Apifox |
| 监控 | Actuator + Prometheus | — | 独立端口 9090，不对公网暴露 |

必要依赖：`web` `validation` `aop` `actuator` `jdbc` `mybatis-plus` `mysql-connector-j` `flyway` `data-redis` `redisson` `sa-token-redis-jackson` `springdoc` `lombok`（provided）`mapstruct` `micrometer-registry-prometheus`。
版本锁定：根 `pom.xml` 的 `dependencyManagement`，子模块不写版本号。

## 五、数据设计（节选）

- 主键：`BIGINT AUTO_INCREMENT`，对外一律序列化为 **字符串**（雪花 ID 会超出 JS 安全整数）。
- 金额：`DECIMAL(18,2)`，Java 侧 `BigDecimal`，接口输出字符串。
- 关键索引：
  - `t_order (customer_id, created_at)` —— 我的订单列表
  - `t_order (order_no)` UNIQUE —— 幂等兜底
  - `t_departure (route_id, depart_date)` —— 班期查询
  - `t_coupon_customer (customer_id, status)` —— 可用券统计
- 分库分表：**不分**。订单表预估 3 年 700 万行，未到 2000 万阈值；超过后按 `customer_id` 水平分 4 片（ShardingSphere-JDBC），分片键选查询命中率最高的维度。
- 迁移：Flyway，`V{版本}__{描述}.sql`，禁止手工改生产表结构。

## 七、高并发与一致性（节选）

**锁方案**

| 场景 | 方案 | 参数 |
| --- | --- | --- |
| 班期余位扣减 | Redisson `lock:departure:{id}` + 条件更新 `SET sold = sold + n WHERE stock - sold >= n` | 锁在事务外层；等待 3s、租约 10s（看门狗） |
| 订单状态流转 | 乐观锁 `WHERE id=? AND status='待支付'` | AFFECTED=0 → 抛业务异常 |
| 定时关单 | Redisson `tryLock`，多实例互斥 | 租约 60s，失败即跳过本轮 |
| 优惠券领取 | Redis Lua 预扣 + DB 唯一索引 `(customer_id, coupon_id)` | 防超发双保险 |

**缓存三害**：线路详情 `catalog:route:{id}`，TTL 30min ±10% 抖动；回源加 Redisson 互斥锁（击穿）；不存在的 ID 缓存空值 60s（穿透）；字典类数据启动预热且不过期（雪崩）。
**一致性**：线路编辑「先改库再删缓存」，删失败投递重试；库存不缓存，直接走 DB 条件更新。
**限流**：登录 5 次/分钟/IP、短信 1 次/60s/手机号、导出 3 次/小时/用户，令牌桶实现，超限返回 429 + Retry-After。

**容量估算**：日活 3 万 × 人均 18 次请求 = 54 万/日；峰值系数 8 → 峰值约 50 QPS；下单占 1% → 写 QPS < 1。HikariCP `maximumPoolSize=20`（单请求 40ms，20 连接理论 500 QPS，留 10 倍余量）。Redis 内存估算 800MB，配置 `maxmemory 1.5gb` + `allkeys-lru`。

## 九、安全体系（节选）

| 字段 | 加密 | 盲索引 | 展示 |
| --- | --- | --- | --- |
| 密码 | BCrypt cost 12 | — | 永不回显 |
| 手机号 | AES-256-GCM | HMAC-SHA256 存 `phone_index` | `138****8000` |
| 身份证 | AES-256-GCM | — | 明文需 `customer:idcard:view` 权限 + 审计 |
| 支付回调 | RSA2 验签 | — | 不落明文密钥 |

密钥：`ROUTEGO_CRYPTO_AES_KEY` / `HMAC_KEY` 由环境变量注入，AES 与 HMAC 分离；**上线后不可更换**（历史密文不可解），需在密码管理器中备份。
高危接口：改密、提现、导出、退款 → 二次校验（短信验证码）+ 独立限流 + 全量审计。

## 十、接口契约与文档

- **方案**：springdoc-openapi 注解生成，按 `GroupedOpenApi` 分 `/admin/**` 与 `/app/**` 两组；Knife4j 提供中文界面。生产 `springdoc.api-docs.enabled=false`（springdoc 自身自动配置默认开启，只关业务开关不够）。
- **统一响应体**：`{ "code": 0, "message": "ok", "data": {}, "traceId": "..." }`；分页 `{ list, total, pageNum, pageSize }`。
- **错误码**：`0` 成功、`10xxx` 通用（参数/鉴权/限流）、`20xxx` 会员、`30xxx` 线路、`40xxx` 订单支付、`50xxx` 运营。错误码表随文档维护。
- **字段规范**：时间 `yyyy-MM-dd HH:mm:ss`（东八区）；金额单位「元」的字符串（两位小数）；所有 ID 返回字符串；枚举在 `@Schema(allowableValues=...)` 全列。
- **鉴权**：`Authorization: Bearer <token>`；401 未登录、403 无权限；Token 有效期 7 天，滑动续期。
- **幂等**：下单接口接受 `Idempotency-Key` 头，同键 24h 内返回首次结果。
- **版本**：`/api/v1/**`，破坏性变更升 v2 并保留 v1 过渡 3 个月。
- **导出**：`GET /v3/api-docs` 导出 JSON，交付给前端导入 Apifox；CI 中生成并存档，保证与当次发布版本一致。
- **交付边界**：本方案只包含后端接口与契约，不含任何前端实现；页面交互、状态管理、渲染由前端团队负责。

## 十二、落地优先级（节选）

| 级别 | 项 | 验收标准 |
| --- | --- | --- |
| P0 | 密码 BCrypt + 手机号加密与脱敏 | 库中无明文手机号；接口回显脱敏 |
| P0 | 下单 / 支付回调幂等 | 同键重复提交返回同一订单；回调重放不重复入账 |
| P0 | 余位扣减双保险 | 并发 200 次下单不超卖（压测通过） |
| P0 | 登录/短信/导出限流 | 超限返回 429 |
| P0 | API 契约可导入 | OpenAPI JSON 导入 Apifox 无报错 |
| P1 | 缓存三害治理 | 热点 key 过期时 DB 回源 QPS < 5 |
| P1 | 优雅停机 + readiness | 发布期间无 5xx |
| P1 | 备份与恢复演练 | 从备份恢复至新库并通过冒烟 |
| P2 | 分片（触发阈值 2000 万行） | — |

**不做的事**：前端页面与组件、MQ（当前写 QPS < 1，本地事务足够）、分库分表（未到阈值）、多活容灾（单区域业务）。

**已知风险**：微信支付证书轮换需人工介入（未做自动告警）；导出大文件走同步接口，超过 10 万行需改异步（已列 P1）。
