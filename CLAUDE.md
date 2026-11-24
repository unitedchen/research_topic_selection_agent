# 科研选题 Agent — 项目说明

## 项目背景

ASCI 项目 AI 高级开发岗位评估作业：设计一个"科研选题副驾子系统"，帮助研究者从研究兴趣/问题域出发，经文献检索 → 方向全景调研 → 候选路径比较 → 收敛支持，输出有证据链的选题建议。

- 时限 2 天，**不需要完整实现**，但要展示设计逻辑和验收思路
- 评估重点：范围收敛（不做大而全平台）、结果验证与证据链可信度、工程化与产品化思维
- 原始需求见 `REQUIREMENTS.md`，完整设计见 `DESIGN.md`

## 文件结构

| 文件 | 说明 |
|---|---|
| `REQUIREMENTS.md` | 评估作业原文整理（需求基准） |
| `DESIGN.md` | 设计文档 v1.0：12 章 + 2 附录 |
| `.claude/settings.local.json` | 本机私有权限配置，**不要提交** |

## 设计文档关键章节

- 3.2/3.3 能力分层与架构总览（PostgreSQL / Qdrant / Neo4j / Redis 四个存储组件）
- 4.1 编排框架选型（LangGraph，对比 CrewAI / AutoGen / Dify / 纯 Python）
- 4.2 LLM 选型（**目前写的是 Claude，待改为 GLM / DeepSeek**）
- 6 核心 Workflow：6 个固定 Stage + 4 个 HIL 人工确认点
- 7 知识库设计：L1 领域知识（静态）/ L2 文献知识（动态）/ L3 图谱知识（半动态）
- 8 数据源、来源分级（A-D 四级）、冲突处理
- 9 评估方案：分 Stage 指标 + Golden Dataset + LLM-as-Judge
- 10 接口定义 + LangGraph 状态定义
- 11.2 MoSCoW 优先级

## 已确认的技术决策

### 编排框架：LangGraph

Pi、DeepSeek Harness 属于 agent harness（模型自主循环调工具，面向编码场景），不是工作流编排框架。本设计是"6 个固定阶段 + 4 个强制 HIL 点 + 分节点独立测试"，是编排形态，harness 不匹配。DeepSeek Harness 目前仅 v0.1，不适合押注。

如果 demo 觉得 LangGraph 太重，纯 Python 状态机（每 Stage 一个 async 函数）也能跑通。

### LLM：Claude → GLM / DeepSeek 可行

架构本身模型无关（LangGraph 只是图，LLM 在节点内通过 OpenAI 兼容 API 调用），换模型是配置层改动。映射：

| 原用途 | 替代 |
|---|---|
| 主力推理（Sonnet） | GLM-4.x 或 DeepSeek V3.x |
| 复杂分析（Opus） | DeepSeek R1 或 GLM thinking 模式 |
| 轻量任务（Haiku） | GLM flash / DeepSeek chat |
| LLM-as-Judge | **用另一家**，保证 judge 与生成模型分离 |

需要关注的风险：
1. 结构化输出可靠性——设计要求 JSON Schema 合法率 ≥ 99%，需 Pydantic 校验 + 重试
2. 强制 `[paper_id]` 引用的指令遵循——设计 6.2 已有程序化兜底（paper_id 必须在检索结果中存在）
3. LangSmith 国内访问——改用 Langfuse（可自部署）
4. 成本策略——"batch API 降 50%"是 Anthropic 机制，改为 DeepSeek 错峰折扣 / GLM 免费 flash

### 数据源：Semantic Scholar 单源优先

S2 一个 API 同时提供元数据、摘要、TLDR、作者、venue 和引用关系（references / citations），**单源即可跑通全流程**。arXiv（预印本、PDF）和 OpenAlex（机构信息）是冗余补充，MVP 不接。

### 存储：一个真源 + 三个派生

PostgreSQL 是唯一真源，Qdrant / Neo4j / Redis 都可从 PG 重建。

- **PostgreSQL**：会话状态、Paper 元数据、各 Stage 产出（JSONB）、EvidenceItem 证据链、L1 静态知识
- **Qdrant**：向量。title + abstract → BGE-M3 本地推理（dense + sparse）→ 带 payload（year / venue_tier）的 Point
- **Neo4j**：图谱。CITES 边来自 S2 references/citations；ABOUT 边由 LLM 从 L1 CS Ontology 中选概念打标（不是自由生成）；PARENT_OF 来自 Ontology
- **Redis**：LLM 缓存、Celery 队列、API 限流

**MVP 用单个 PostgreSQL 即可**（pgvector + JSONB）。单次会话 200-500 篇文献，远低于 Qdrant / Neo4j 的必要规模（向量 10 万、图节点 1 万）。Beta 引入向量库，Production 引入图库。

### 数据如何进入系统

Qdrant 和 Neo4j 没有预置数据，每次会话 Stage 2-3 从 API 拉取后派生：

```
S2 / arXiv / OpenAlex ──▶ 合并去重 ──▶ PostgreSQL
                                        ├─▶ BGE-M3 ──▶ Qdrant
                                        ├─▶ S2 references ──▶ Neo4j CITES
                                        └─▶ LLM 抽取 concept ──▶ Neo4j ABOUT
初始化（人工）：venue_ranking.yaml + CS Ontology JSON ──▶ PG + Neo4j
```

跨源 ID 对齐是真正难点：PG 的 Paper 表需存 `s2_id` / `openalex_id` / `arxiv_id`，去重链 DOI → arXiv ID → 标题模糊 → embedding。

## 待处理

- [ ] `DESIGN.md` 4.2 模型表改为 GLM / DeepSeek
- [ ] `DESIGN.md` 3.1 边界图 "LLM API (Claude / GPT)" 改为中性表述
- [ ] `DESIGN.md` 11.2 MoSCoW 补充"MVP 单库、Beta 向量库、Production 图库"
- [ ] `DESIGN.md` 7.3 与 7.7 矛盾：7.3 说"摘要 + 全文"分块，7.7 说"仅存摘要 + 元数据"。建议统一为 MVP 仅摘要，EvidenceItem.quote 从摘要取
- [ ] `DESIGN.md` 4.4 / 8.1 标注 S2 为主源，arXiv / OpenAlex 为 Should 级冗余

## 约定

- 文档使用中文
- 修改 `DESIGN.md` 时保持章节编号稳定，`CLAUDE.md` 按章节号引用
