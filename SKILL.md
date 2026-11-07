---
name: prod-backend-blueprint
description: Plan, design, audit, or review a production-grade backend project. Use when the user wants to build a new backend, check an existing codebase against production standards, or audit a project against its own design docs. Covers architecture, stack selection, concurrency, caching, idempotency, payments, security, deployment, and API contracts. Trigger phrases include 生产级 / 生产环境 / 上线方案 / 架构方案 / 技术选型 / 后端方案 / 审核 / 上线前检查 / 帮我看看这个项目 / 是否符合规范. Three routes — existing project without a baseline gets a gap check with suggestions only, a project built by this skill gets a 340-item audit report, and a greenfield idea gets a 7-round interview plus a 12-chapter blueprint. Backend only, never touches the frontend. 中文摘要：生产级后端方案规划与审核助手，新建项目出方案书、已有项目做差距检查、本技能产出的项目做审核报告，只做后端不碰前端。
agent_created: true
---

# 生产级后端方案规划

## 核心原则

- **先问后写**：未完成提问阶段前，禁止直接输出方案。
- **分轮提问，每轮最多 4 个**：使用 AskUserQuestion 工具；每题提供 2–4 个选项，把推荐项放在第一位并标注「（推荐）」。
- **尊重已有信息**：当前对话、工作区代码、历史记忆中已明确的答案（如语言、框架）不重复提问，直接进入下一轮。
- **用户放弃选择时**：用推荐值继续并在最终方案中注明「采用推荐默认」，不追问、不停滞。
- **方案必须过检**：产出前对照 `references/production-checklist.md` 逐项查漏，凡涉及项未覆盖的必须补齐或显式说明「不适用原因」。
- **输出必须套模板**：最终方案严格遵循 `references/plan-template.md` 的章节结构（八股文要领）。
- **API 文档是交付物**：方案必须包含「接口契约与文档」章节（OpenAPI 方案、统一响应体、错误码表、字段规范、鉴权约定、导出/导入方式），文档与代码同源，生产环境关闭或加鉴权。
- **前后端边界是硬规则**：后端只交付接口 + 契约文档 + 错误码表 + 示例。**禁止**写前端页面/组件、禁止修改前端仓库、禁止替前端决定交互与状态管理。发现诉求本质属于前端时，记录为「前端侧需求」交回，不展开实现。跨域与登录态配置属后端责任，但 Token 如何存储由前端决定。
- **语言特定内容**：先确定语言，再加载 `references/stacks/` 下对应文件（Java 为 `java.md`，Go/Python/Node 合并于 `go-python-node.md`），方案中的依赖清单、版本匹配、语言陷阱（如 Java 浮点精度）必须出自对应文件。
- **审核不代改**：审核模式只出报告与建议，不直接修改用户代码；用户明确要求整改后才动手，且逐项说明影响面。

## 场景判定（任何请求的第一步）

开口回答前先判定场景，**不要默认走提问流程**。判定不清时用 AskUserQuestion 问一个二选一问题，不要猜。

| 场景 | 判定信号 | 动作 |
| --- | --- | --- |
| **A. 已有项目 · 无本技能基线** | 工作区有代码，但找不到本技能产出的方案书 / API 契约 / 错误码表 | 差距检查：按 `references/audit-checklist.md` 抽取关键组比对，输出「差距 + 建议」，**由用户自行决定改哪些**，不代改 |
| **B. 已有项目 · 本技能产出** | 存在方案书（含「接口契约」「生产级保障清单」章节）或本技能生成的契约基线 | 审核模式：走阶段 4，逐项出审核报告 |
| **C. 无项目 · 计划新建** | 无代码或只有想法（"我想做个 XX 系统"） | 走阶段 0–3：分轮提问 → 方案书 → 检阅迭代 |

判定要点：

- A 与 B 的分界是**有无基线**。有基线 → 对照「基线 + 清单」审核；无基线 → 只能对照清单做差距检查，且必须说明「本项目从未按本技能设计，以下为差距而非缺陷」。
- 场景 A 禁止直接改代码，也不要把差距写成「必须整改」，一律表述为「建议项」，并问用户「要按哪几项整改」。
- 场景 B 若发现基线本身与最佳实践冲突，单列为「基线问题」，与实现问题分开统计，不混入开发者缺陷。
- 用户只贴一段代码问「这样行不行」时，也按场景 A 处理：点到为止给建议，不展开全量审核，除非用户要求。

## 工作流（严格按顺序执行）

### 阶段 0 · 语言与生态判定
从用户输入或工作区推断语言；无法推断时作为第一轮的第一个问题。语言决定后续问题集与 stacks 参考文件。

### 阶段 1 · 分轮提问
按 `references/interview.md` 的轮次执行：规模与架构 → 框架与版本 → 数据层 → 中间件与异步 → 业务风险面 → 安全与合规 → 交付与前端协作。
- 根据前面回答动态裁剪后续问题（例如选了「纯内部工具」可跳过支付轮；选了「无状态 API」可跳过分布式锁细节）。
- 每轮开场用一句话说明本轮目的，再调用 AskUserQuestion。
- 全部轮次结束后，用一张紧凑的「决策回执表」（问题 → 用户选择 → 推导结论）向用户复述，确认无误再进入写作。推导规则见 `references/decision-matrix.md`。

### 阶段 2 · 撰写方案
1. 加载语言对应的 stacks 参考：`references/stacks/java.md` 或 `references/stacks/go-python-node.md`。
2. 按 `references/plan-template.md` 的章节骨架撰写；技术决策必须给出「选择 + 理由 + 否决项（为什么不选备选）」三件套。
3. 章节「生产级保障清单」必须与 `references/production-checklist.md` 对齐，按 P0/P1/P2 标注。
4. 涉及并发、缓存、幂等、支付、加密的结论必须与 `references/decision-matrix.md` 的规则一致；偏离时显式说明理由。
5. 「接口契约与文档」章节为必写项，写完自检：契约可导出、可导入、字段规范齐全、生产已收敛。

### 阶段 3 · 检阅与迭代
- 把方案完整呈现给用户（长文落盘为 Markdown 文件并 present_files，不要只贴在对话里）。
- 明确请用户检阅：「以上方案请检阅，指出要调整的章节」。
- 用户提出修改 → 局部修订并重新呈现差异；用户确认后定稿。
- 定稿后询问是否进入实施（脚手架/建表/接口编码），不要默认开始写代码。

### 阶段 4 · 审核（场景 B；场景 A 复用同一清单但只出建议）

1. **建基线**：读方案书、OpenAPI、错误码表，记下各自的版本号与日期；基线缺失的先说明缺哪几份，再继续。
2. **逐组过清单**（`references/audit-checklist.md`）：Part 1 通用二十四组 → Part 2 攻击面六组 → Part 3 取语言对应小节（Java/Go/Python/Node/C#/PHP/Rust/SQL）→ Part 4 取项目实际用到的中间件小节。不适用的组整组跳过并留一句理由。
3. **每条带证据**：`文件:行号` / 接口路径 / 配置项名 / SQL / 命令输出。三态判定：通过 / 不通过 / 不适用（附理由）。无法验证的标「待验证」，**不得臆断为通过**。
4. **分级**：阻断 P0（安全、资金、数据丢失、不可回滚发布）/ 建议 P1 / 记录 P2。
5. **出报告**：按清单 Part 5 模板输出，落盘 Markdown 并 present_files，同时在对话里给结论摘要（通过率、阻断项数、最小可上线集）。
6. **不代改**：报告结束后问「要整改哪些项」，用户勾选后按阻断 → 建议顺序处理，每项改动前说明影响面与回滚方式。

### 实时同步（审核与整改期间强制，不攒到最后）

- **API 文档**：端点新增/字段变更/错误码变更/接口废弃，**当场**更新 OpenAPI 与错误码表；示例值同步改为可真实调通的值。
- **方案书**：实现偏离方案时先登记偏差（章节 / 原计划 / 实际 / 原因 / 影响面），再按用户选择回写章节。
- **变更记录**：每次同步更新方案书版本号 + 日期 + 变更点 + 影响范围，让人能看出「哪天因为什么改了什么」。
- 审核报告末尾固定记一行：`本次同步：API 文档 n 处 / 方案书 n 处 / 未同步 n 处（原因）`，避免出现代码改了文档没改的静默漂移。

## 参考文件索引

| 文件 | 何时加载 |
| --- | --- |
| `references/interview.md` | 阶段 1 全程 |
| `references/decision-matrix.md` | 阶段 1 推导结论、阶段 2 撰写并发/缓存/幂等/支付章节时 |
| `references/production-checklist.md` | 阶段 2 查漏、产出 P0/P1/P2 清单时 |
| `references/audit-checklist.md` | 场景 A 差距检查、阶段 4 审核全程（340 条 / 41 组，含攻击面、9 种语言、6 类中间件、报告模板） |
| `references/plan-template.md` | 阶段 2 全程 |
| `references/stacks/java.md` | 用户语言为 Java / Kotlin / Spring 时 |
| `references/stacks/go-python-node.md` | 用户语言为 Go / Python / Node 时 |
