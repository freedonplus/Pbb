# 八股文速查库（方案与审核的原理底稿）

> 对标 javaguide.cn 知识体系整理。用途：写方案、做审核时引用原理性结论的**统一口径**，避免凭印象写错。
> 数据截止：2025 年中；涉及具体版本号时以 WebSearch 实时核对为准。
> 只收「生产级方案/审核会用到」的结论，不是面试题全集。

---

## 1. Java 基础

### 浮点精度（金额红线）
- `float`/`double` 是二进制浮点，多数十进制小数（0.1、0.2）无法精确表示：`2.0f - 1.9f` = 0.100000024，`1.8f - 1.7f` = 0.099999905，二者 `==` 为 false。
- 只有能写成 2 的负幂之和的小数（0.5、0.25）能被精确表示。
- **金额、税率、比例一律 `BigDecimal`**：
  - 构造：`new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)`；**禁止 `new BigDecimal(0.1)`**（把二进制误差原样带入）。
  - 比较：`compareTo(...) == 0`；`equals` 会比较 scale（1.0 ≠ 1.00）。
  - 除法必须指定 scale 与 `RoundingMode`，否则除不尽抛 `ArithmeticException`。
  - 序列化给前端用字符串（防 JS 精度丢失）；落库用 `DECIMAL(18,2)`。
- 超过 `long` 的整数用 `BigInteger`（int[] 存储，运算慢，非必要不用）。

### 包装类与缓存
- `Byte/Short/Integer/Long` 缓存 [-128,127]，`Character` 缓存 [0,127]，Integer 上限可用 `-XX:AutoBoxCacheMax` 调（不建议调大）。
- `Float/Double` 包装类**无缓存**，`==` 永远 false。
- `Integer i = 40` 是装箱（`Integer.valueOf`，命中缓存）；`new Integer(40)` 是新对象——二者 `==` 为 false。
- 军规：**所有包装类比较一律 `equals`**；POJO 字段用包装类（可区分 null 与默认值）。

### 字符串与拼接
- `String` 不可变（final byte[] + 私有），任何"修改"都是新对象；不可变性带来线程安全与常量池复用。
- 编译期可确定的 `+` 拼接会被常量折叠（无开销）；单行 `+` 会被编译器优化为 `StringBuilder`；**循环内 `+` 拼接每轮新建 StringBuilder 与 char[]，必须手写 StringBuilder**。
- `StringBuilder`：非线程安全，快；`StringBuffer`：方法级 synchronized，慢，仅共享变量场景用。
- 日志高频路径用 SLF4J `{}` 占位符（先判级别再求值），禁止 `"log:" + obj` 无条件拼接。
- `String.join`/`Collectors.joining` 用于集合拼分隔符，别手写循环。

### == 与 equals / hashCode
- `==` 比引用；`equals` 比内容（需重写）。重写 `equals` 必须同时重写 `hashCode`，否则 HashMap/HashSet 行为错乱。
- Lombok `@Data` 在 JPA 实体上会把懒加载字段、集合算进 equals/hashCode，引发 N+1 与栈溢出——实体上只用 `@Getter/@Setter`。

### 深拷贝 / 浅拷贝
- 浅拷贝共享引用字段；Java 原生 `clone()` 是浅拷贝。深拷贝用序列化、JSON 转换、或 MapStruct/手工赋值。

### 集合
- `HashMap`（JDK8：数组+链表+红黑树，阈值 8 树化/6 退化，负载因子 0.75，扩容 2 倍）非线程安全；并发用 `ConcurrentHashMap`（JDK8：CAS + synchronized 锁桶头，替代 JDK7 分段锁）。
- `Hashtable`、`Collections.synchronizedMap` 是全表锁，禁用。
- `ArrayList` 扩容 1.5 倍，预估容量传入构造器避免多次拷贝；`LinkedList` 随机访问 O(n)，不是默认选型。
- `Arrays.asList` 返回定长视图，增删抛异常；`List.of` 不可变；遍历中删除用 `Iterator.remove()`。

---

## 2. Java 并发

- **synchronized vs ReentrantLock**：前者 JVM 内置、可重入、自动释放、JDK6 后优化良好（偏向锁→轻量级→重量级升级）；后者可中断、可超时（`tryLock`）、公平锁、多 Condition。无特殊需求用 synchronized。
- **volatile**：保证可见性 + 禁止指令重排，**不保证原子性**（`count++` 照样丢更新）。状态标志用 volatile，计数用 Atomic 类。
- **CAS**：`AtomicInteger` 等基于 CPU `compare-and-swap`；ABA 问题用 `AtomicStampedReference`；高竞争计数用 `LongAdder`（Cell 分段，冲突分散，吞吐高于 AtomicLong）。
- **线程池**：七大参数（core/max/keepAlive/queue/threadFactory/handler）显式给；队列有界；拒绝策略明确（CallerRunsPolicy 提供背压）。**禁止 `Executors.newFixedThreadPool/newCachedThreadPool`**（无界队列 OOM / 线程数无上限）。IO 密集线程数 ≈ CPU 核数 × (1 + 平均等待/平均计算)；CPU 密集 ≈ 核数 +1。
- **ThreadLocal**：线程隔离副本；线程池场景必须 `try { } finally { remove(); }`，否则串数据 + 内存泄漏（Entry 是弱引用 key 强引用 value）。跨线程传上下文用 TransmittableThreadLocal。
- **AQS**：JUC 锁与同步器的基座（state + CLH 队列）；CountDownLatch（等 N 个完成）、Semaphore（限并发数）、CyclicBarrier（凑齐放行）按场景选。
- **死锁四要素**：互斥、占有且等待、不可剥夺、循环等待；破坏任意一个即可（统一加锁顺序、tryLock 超时）。

---

## 3. JVM

### 内存区域
- 堆 = 新生代（Eden + S0/S1，比例 8:1:1）+ 老年代；对象优先 Eden 分配；大对象（G1 中 ≥ Region 一半的 Humongous）直接进老年代；长期存活对象按年龄（MaxTenuringThreshold，Parallel 默认 15 / CMS 6，且受动态年龄判定影响）晋升老年代。
- JDK 8 起永久代被 **Metaspace（本地内存）** 取代，生产必须设 `-XX:MaxMetaspaceSize`。
- 虚拟机栈（栈帧：局部变量表/操作数栈）、本地方法栈、程序计数器为线程私有；堆与方法区共享。

### 垃圾判定与引用
- **可达性分析**（Java 采用）：GC Roots 包括栈中引用、静态属性、常量、JNI、同步锁持有对象。引用计数无法解决循环引用，Java 不用。
- 四种引用：强（不回收）/ 软（OOM 前回收，可做内存敏感缓存）/ 弱（GC 即回收，如 WeakHashMap）/ 虚（只用于跟踪回收，如堆外内存管理）。
- `finalize()` 已废弃（JDK 9 弃用、JDK 18 标记移除），新代码禁用。

### GC 算法与收集器
- 算法：标记-清除（碎片）→ 复制（新生代，Eden+Survivor）→ 标记-整理（老年代）→ 分代收集。
- 收集器：
  - **Serial**：单线程 STW，小堆/客户端。
  - **Parallel Scavenge + Parallel Old**：吞吐优先，JDK 8 默认；批处理适用。
  - **CMS**：并发低停顿，JDK 9 弃用、JDK 14 移除；浮动垃圾 + 碎片是其死穴。
  - **G1**：JDK 9+ 默认；Region 分区、可设 `-XX:MaxGCPauseMillis`（默认 200ms）；Mixed GC 回收部分老年代。
  - **ZGC / Shenandoah**：染色指针/读屏障，停顿亚毫秒级、与堆大小无关；低延迟（P99 < 10ms）场景首选，JDK 17+ 生产可用，ZGC 分代化在 JDK 21+。
- Minor GC（新生代）/ Mixed GC（G1：新生代+部分老年代）/ Full GC（整堆，重点避免）。

### 调优
- 流程：定目标（吞吐 or 停顿）→ 采 GC 日志（`-Xlog:gc*:file=gc.log:time,level,tags`）→ 分析（GCeasy）→ 调参 → 复测。**禁止无日志调参**。
- 容器参数：`-XX:MaxRAMPercentage=75`；`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path`。
- OOM 类型定位：Java heap space（泄漏/堆小）→ heap dump + MAT 支配树；Metaspace（类加载泄漏，常见于动态代理/热部署）；GC overhead limit（98% 时间 GC 回收 <2% 堆，堆太小）；Direct buffer（堆外内存，`MaxDirectMemorySize`）。
- 内存泄漏常见源：静态集合只增不减、ThreadLocal 未 remove、连接/流未关、监听器未注销、缓存无上限。

---

## 4. MySQL 与索引

### 索引数据结构
- InnoDB/MyISAM 均用 **B+ 树**：只有叶子节点存数据、叶子间双向链表（范围查询友好）、内节点只存 key（矮胖树，IO 次数少）。
- Hash 索引 O(1) 但不支持范围与排序，InnoDB 只有自适应哈希索引（AHI），不可控。
- **聚簇索引**（InnoDB 主键）：叶子节点即整行数据；辅助索引叶子存主键值，查非索引列需**回表**。
- 主键应短、单调递增（避免页分裂）；UUID 做主键是反模式（长 + 无序）。

### 关键优化概念
- **覆盖索引**：查询列全在索引中，`Extra: Using index`，免回表——高频查询首选优化。
- **最左前缀**：联合索引 `(a,b,c)` 等价于 `(a)`、`(a,b)`、`(a,b,c)`；遇到范围列（>、<）停止向右匹配（>=、<=、BETWEEN、`LIKE 'x%'` 不停）；优化器会对等值条件重排序，所以 `b=1 AND a=1 AND c=1` 能用上。
- **索引下推（ICP，5.6+）**：把 WHERE 中可在索引上判断的条件下推到引擎层，减少回表。
- MySQL 8 新特性：隐藏索引（invisible，下线灰度）、降序索引（真支持）、函数索引、索引跳跃扫描（ISS，弱鸡勿依赖）。

### 索引失效场景（写 SQL 红线）
- 索引列上做函数/运算（`WHERE DATE(t)=...`）→ 改范围写法或函数索引。
- 隐式类型转换（字符串列传数字）→ 参数类型对齐列类型。
- `LIKE '%x'` 前导模糊 → 全文索引 / ES / 反转列。
- `OR` 一侧无索引 → 全表扫；两侧都有可考虑 union。
- `SELECT *` + 命中率 >20%~30% → 优化器主动放弃索引（不是 bug）。
- 字符集/排序规则不一致的 JOIN → 索引失效。
- `IN` 列表过长（>200 默认阈值）→ 估算失真，分批。

### 事务 / MVCC / 锁
- ACID；隔离级别：RU < RC < **RR（InnoDB 默认，靠 next-key lock 基本解决幻读）** < Serializable。
- MVCC：每行隐藏列（trx_id、roll_pointer）+ undo log 版本链 + ReadView；RC 每句生成 ReadView，RR 事务首个快照读生成。
- 锁：记录锁 / 间隙锁（RR 防幻读）/ next-key lock（左开右闭）；`SELECT ... FOR UPDATE` 当前读，普通 SELECT 快照读。
- 死锁：`SHOW ENGINE INNODB STATUS` 查 LATEST DETECTED DEADLOCK；预防 = 固定加锁顺序 + 事务短小 + 索引精确（范围锁大易死锁）。

### 慢查询治理
- slow log 开启（阈值 1s）；`EXPLAIN` 看 type（const > eq_ref > ref > range > index > ALL，核心查询禁止 ALL）、rows、Extra（Using index 好；Using filesort / Using temporary 警惕）。
- 深分页 `LIMIT 100000,20` 改游标（`WHERE id > ? LIMIT 20`）或延迟关联。

---

## 5. Redis

### 为什么快
- 纯内存操作（纳秒 vs 磁盘毫秒）；单线程事件循环 + I/O 多路复用（无锁竞争与上下文切换，6.0 起网络读写多线程，命令执行仍单线程）；高效内部编码（ziplist/quicklist/skiplist/hashtable 动态选择）；RESP 协议解析轻。
- 不能替代 DB：内存成本高、持久化仍有丢失窗口。

### 数据结构选型
- String：缓存单值、计数器、分布式锁、Session。底层 SDS（预分配、二进制安全）。
- Hash：对象存储（field 级更新，购物车、用户资料）。
- Set：去重集合、交并差（共同好友、点赞）；抽奖 SPOP/SRANDMEMBER。
- ZSet：排行榜（ZREVRANGE）、延迟队列（score=执行时间戳）；底层跳表（实现简单、范围查询优于红黑树、内存场景比 B+ 树省）。
- HyperLogLog：UV 基数统计（0.81% 误差，12KB 计亿级）。
- Bitmap：日活签到（SETBIT 日期为 key、userId 为 offset，BITOP/BITCOUNT 统计）。
- Stream（5.0+）：轻量 MQ（消费组、ACK、持久化）；业务简单量小可用，金融级仍选真 MQ。

### 持久化与内存
- RDB：快照，恢复快，丢窗口大；AOF：追加命令，`appendfsync everysec` 是常态（每秒刷盘，最多丢 1s）；混合持久化（4.0+，RDB 头 + AOF 增量）兼顾。
- 过期策略：惰性删除（访问时查）+ 定期删除（hz 采样），不做定时全扫（成本太高）。
- 内存淘汰：`maxmemory` 必配；策略选 allkeys-lru（缓存场景）/ volatile-lru；**禁止 noeviction**（写失败）。

### 缓存三大问题（方案必写）
- 穿透（查不存在的数据）：空值缓存（TTL 30-60s）/ 布隆过滤器 / 参数校验。
- 击穿（热点 key 过期瞬间）：互斥锁重建（setnx，只放一个回源）/ 逻辑过期（后台异步刷新）。
- 雪崩（大量 key 同时过期）：TTL ±10% 抖动 / 热点不过期 + 主动失效 / 多级缓存（Caffeine + Redis）。

### 分布式锁
- Redisson：`lock()` 看门狗自动续期（默认 30s，每 10s 续）；`tryLock(wait, lease)`；可重入；释放用 Lua 保证「判持有者+删」原子。
- 锁 key 带业务主键；锁在事务外层；锁持有 <1s；锁内禁止远程调用；唯一索引/条件更新做最后兜底。
- Redis 锁不解决主从切换丢锁（RedLock 争议大，金融级用 DB 唯一约束/ZK/fencing token）。

### 高可用
- 主从 + 哨兵（自动故障转移）；Cluster（16384 槽分片，去中心化）；客户端连接池（Jedis/Lettuce，Lettuce 基于 Netty 线程安全推荐）；大 key（>10KB 慎用，`--bigkeys` 巡检）、热 key（本地缓存/拆 key/读写分离）。
- 危险命令：`keys *`、`flushdb/flushall` 生产禁用（rename-command 封禁），用 SCAN 游标遍历。

---

## 6. 消息队列（RabbitMQ / RocketMQ / Kafka）

### 为什么用 / 什么时候不用
- 用：异步提速（主链路快速返回）、解耦、削峰填谷、最终一致。
- 不用：强实时链路（支付结果判定、库存校验）、链路短且 QPS 低的内部调用——引入 MQ = 引入中间件运维 + 异步状态机 + 故障恢复成本。
- 「消息写入 MQ 就返回业务成功」是错的设计：消费失败时用户已被承诺成功，需要状态机 + 补偿衔接。

### 可靠性四段论（回答「消息会不会丢」必须覆盖四段）
1. **生产者**：confirm/ack 机制 + 失败重试 + 本地消息表兜底。
2. **Broker**：消息持久化 + 副本复制（RabbitMQ Quorum Queue / Kafka ISR / RocketMQ 同步复制）。
3. **消费者**：业务处理成功后手动 ack / 提交 offset。
4. **业务侧**：消费幂等（msgId 去重表 / 业务唯一键）+ 死信兜底 + 对账补偿。
- 语义：At Most Once（可能丢）/ At Least Once（可能重复，工程默认）/ Exactly Once（宣传语，实际靠「生产幂等 + 消费幂等 + 事务」拼出来，跨外部 DB 仍做不到）。

### RabbitMQ
- 模型：Exchange（direct/fanout/topic/headers）+ RoutingKey + Binding + Queue；先讲清路由再讲别的。
- Publisher Confirms 只确认 Broker 收到；**不可路由**消息靠 `mandatory` + Return 回调或备份交换器（alternate-exchange）发现。
- 持久化三件套：durable exchange + durable queue + persistent message，缺一重启即丢。
- 消费：手动 ack；失败 nack（requeue 防死循环）；多次失败进 DLX。
- 延迟消息：TTL + DLX 有**队头阻塞**（队首大 TTL 卡住后面小 TTL），生产用 rabbitmq-delayed-message-exchange 插件。
- 队列类型：Classic（无副本）、Quorum（Raft 副本，3.8+ 推荐）、Stream（日志型高吞吐）。
- 集群多节点 ≠ 队列有副本（Classic 队列只在声明它的节点上），高可用要用 Quorum。
- 监控：ready / unacked / 消息年龄 / 确认速率 / 副本状态。

### RocketMQ
- 组件：NameServer（无状态路由，节点间不通信）、Broker、Producer、Consumer。
- 存储：CommitLog（顺序写全量）+ ConsumeQueue（消费索引）+ IndexFile。
- **事务消息**：半消息 → 本地事务 → commit/rollback → Broker 定时回查；解决「本地事务与发消息」一致性，不解决消费端幂等。
- 延迟消息：18 个固定级别（5.x 支持任意时间）；超时关单经典方案。
- 顺序消息：按业务键（sharding key）投同一队列，消费端单线程；全局有序只有一个队列。
- 刷盘：同步（不丢、慢）/ 异步；复制：同步双写 / 异步；按丢失容忍度组合。

### Kafka
- 模型：Topic → Partition（并行度与顺序的基本单位）→ Replica（Leader + Follower，ISR 同步副本集）；HW/LEO 界定可见消息。
- **只保证单分区内有序**；需要顺序的业务键选为分区 key。
- 不丢配置：`acks=all` + `min.insync.replicas ≥ 2` + 副本数 ≥ 3 + 消费成功后手动提交 offset。
- 高吞吐原因：顺序写盘 + Page Cache + 零拷贝（sendfile）+ 批处理 + 压缩。
- 幂等生产者（PID + Sequence）防单分区重试重复；事务跨分区原子写；均不能覆盖外部 DB。
- Rebalance 触发消费停顿：合理设 session timeout、减少频繁上下线、静态成员（group.instance.id）。
- 4.0 起只支持 KRaft（去 ZooKeeper）。

### 选型速查
| 诉求 | 选型 |
| --- | --- |
| 常规业务异步（通知/邮件），中小流量 | RabbitMQ（路由灵活、运维简单） |
| 订单链路（事务消息、延迟消息、顺序消息） | RocketMQ |
| 日志流、埋点、大数据管道，吞吐 >10 万/s | Kafka |
| 只要延迟任务 | Redis ZSet / RabbitMQ 延迟插件，不为单点需求引入集群 |

### 积压治理
- 先定位类型：生产突增（扩容 Broker/分区）、消费变慢（扩容消费者、批量消费）、下游故障（熔断降级、暂停消费）。
- 注意：消费者数量 > 分区/队列数不会提速。

---

## 7. 分布式常识

- **CAP**：分区容忍必选，一致性（C）与可用性（A）二选一；**BASE**（基本可用、软状态、最终一致）是工程常态。
- **分布式 ID**：雪花算法（时间戳+机器+序列，时钟回拨是坑）、号段模式（DB 批量取号）、Leaf/TinyID；UUID 无序不适合做主键。
- **分布式事务**：2PC/XA（强一致、性能差）、TCC（业务侵入大、需 Try/Confirm/Cancel 三接口与幂等防悬挂）、本地消息表（最终一致、最常用）、RocketMQ 事务消息、Seata AT（自动补偿、有性能损耗）。中小项目优先本地消息表。
- **一致性协议**：Raft（Leader + 多数派，Etcd/Nacos/Quorum Queue 用）、Paxos（理论源头）、Gossip（Redis Cluster 元信息）。
- **限流算法**：计数器（临界突刺）、滑动窗口、漏桶（匀速）、令牌桶（允许突发，最常用）；分布式限流用 Redis + Lua 或 Redisson RRateLimiter。
- **负载均衡**：轮询/权重/最少连接/一致性哈希（缩扩容少迁移，缓存分片常用）。
- **服务雪崩治理链**：超时 → 有限重试（退避+抖动，防流量放大）→ 熔断（失败率阈值，半开试探）→ 降级（兜底返回值）→ 隔离（线程池/信号量舱壁）。
