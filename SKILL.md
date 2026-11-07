---
name: prod-backend-blueprint
description: 生产级后端项目架构规划助手。当用户要新建、重构或评估一个后端项目，咨询后端架构设计（单体/多模块/微服务）、技术选型（语言/框架/数据库/中间件）、高并发设计、支付与安全方案，或提到「生产环境」「生产级」「上线方案」「架构方案」「技术选型」「后端方案」时使用。本技能按企业标准分轮提问收集需求（规模、语言、框架、数据库、中间件、并发量、支付、安全等），再产出按八股文结构组织的完整生产级方案，交给用户检阅后迭代定稿。This skill should be used when the user wants to plan, design, or review a production-grade backend project.
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

## 参考文件索引

| 文件 | 何时加载 |
| --- | --- |
| `references/interview.md` | 阶段 1 全程 |
| `references/decision-matrix.md` | 阶段 1 推导结论、阶段 2 撰写并发/缓存/幂等/支付章节时 |
| `references/production-checklist.md` | 阶段 2 查漏、产出 P0/P1/P2 清单时 |
| `references/plan-template.md` | 阶段 2 全程 |
| `references/stacks/java.md` | 用户语言为 Java / Kotlin / Spring 时 |
| `references/stacks/go-python-node.md` | 用户语言为 Go / Python / Node 时 |
