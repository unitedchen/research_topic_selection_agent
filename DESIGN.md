# 科研选题 Agent 系统设计文档

> 版本: v1.0 | 日期: 2026-11-24
> 项目: ASCII 项目 - AI 高级开发岗位评估

---

## 目录

1. 项目概述
2. 目标用户与核心任务
3. 系统边界与能力结构
4. 技术选型
5. 用户输入与输出
6. 核心 Workflow
7. 知识库设计
8. 数据源、来源分级与冲突处理
9. 评估方案
10. 接口定义
11. 实施计划
12. 风险与缓解
- 附录 A: Golden Dataset 构建指南
- 附录 B: 评估工具链

---

## 1. 项目概述

### 1.1 背景与目标

科研选题是学术研究的起点，也是最依赖经验的环节。研究者（尤其是博士生和早期研究者）常常面临三个痛点：

1. **信息过载**：一个领域动辄数万篇文献，难以判断哪些是必读
2. **方向盲区**：不知道某个方向是否已经被做过、做到什么程度
3. **收敛困难**：有多个候选方向，但缺乏客观的对比依据

本系统构建一个**选题 Agent**，帮助研究者从一个研究兴趣、问题域或初步想法出发，经过**文献检索 → 方向全景调研 → 候选路径比较 → 收敛与判断支持**的完整流程，最终输出有证据支撑的选题建议。

### 1.2 核心价值主张

- **不是文献搜索工具**，而是**选题决策支持系统**
- **每个结论都有证据链**：可追溯到具体文献、具体段落
- **人在回路**：关键节点需要人工确认，而非全自动黑箱
- **范围收敛**：聚焦"选题"这一件事，不做大而全平台

### 1.3 设计原则

| 原则 | 说明 |
|------|------|
| Evidence-First | 所有判断必须有可验证的引用支撑 |
| Human-in-the-Loop | 关键决策点由人确认，Agent 负责信息聚合与建议 |
| Fail-Safe | 不确定时明确说"不确定"，而非编造答案 |
| Composable | 各阶段可独立测试、独立替换 |
| Observable | 全链路可追踪，每步输出可审计 |

---

## 2. 目标用户与核心任务

### 2.1 用户画像

**主要用户：科研选题决策者**

| 维度 | 描述 |
|------|------|
| 身份 | 博士生 (Year 1-2)、博士后、Junior Faculty、企业研究员 |
| 学科 | 首期聚焦计算机科学 (AI/ML/NLP/CV)，验证后扩展 |
| 特征 | 有基础领域知识，但缺乏全景视野；有大致方向，但需要验证 |
| 痛点 | 不确定方向新颖度、不知道竞品进展、难以横向对比候选方向 |
| 技术素养 | 熟悉基础工具 (Google Scholar, arXiv)，但不会写复杂检索式 |

**次要用户**：导师（判断学生选题是否可行）、实验室 PI（为团队规划新方向）

### 2.2 核心任务定义

| 项目 | 内容 |
|------|------|
| 输入 | 一个模糊的研究兴趣 / 问题域 / 初步想法 + 约束条件 |
| 输出 | 3-5 条候选研究方向，每条附完整证据链、对比矩阵、风险提示 |
| 核心任务 | 将模糊想法收敛为可执行的研究选题 |

**任务边界（做什么）**：
- 帮用户"看到"领域全景
- 帮用户"发现"潜在 gap
- 帮用户"对比"候选路径

**任务边界（不做什么）**：
- 不替用户做最终决策
- 不保证选题一定"正确"
- 不覆盖论文写作、实验设计等下游环节

### 2.3 非目标（Out of Scope）

明确不做的事情，避免范围蔓延：

| 不做 | 原因 |
|------|------|
| 文献管理（笔记、标签、PDF 管理） | Zotero/Mendeley 已成熟，不重复造轮 |
| 论文写作辅助 | 属于下游任务，且已有成熟产品 |
| 实验设计与代码生成 | 超出选题阶段 |
| 全学科覆盖 | 首期聚焦 CS/AI，验证后再扩展 |
| 实时文献推送 | 非核心任务，可后续迭代 |
| 同行评议预测 | 伦理风险，不做 |

---

## 3. 系统边界与能力结构

### 3.1 系统边界

```
+-------------------------------------------------------------+
|                     选题 Agent 系统边界                       |
|                                                               |
|  [意图理解] -> [文献检索] -> [全景梳理] -> [路径生成]           |
|                                        |                      |
|  [报告生成] <- [比较收敛] <- [路径生成]                         |
|                                                               |
|  - - - - - - - - - - - 外部依赖 - - - - - - - - - - - - - -  |
|  * LLM API (Claude / GPT)                                     |
|  * 学术 API (Semantic Scholar, arXiv, OpenAlex)               |
|  * 向量数据库 / 图数据库                                        |
+-------------------------------------------------------------+
```

### 3.2 能力分层

| 层级 | 组件 | 职责 |
|------|------|------|
| L1 交互层 | Web UI / CLI | 用户输入、结果展示、人工确认 |
| L2 编排层 | LangGraph Workflow | 阶段调度、状态管理、人工中断 |
| L3 Agent 层 | 6 个专用 Agent | 意图理解、检索、全景、路径、比较、报告 |
| L4 能力层 | RAG / 知识图谱 / 工具调用 | 检索、嵌入、重排、图谱查询 |
| L5 数据层 | PostgreSQL / Qdrant / Neo4j | 元数据、向量、图谱 |

### 3.3 技术架构总览

```
                        +---------------+
                        |   Frontend    |
                        |   (Next.js)   |
                        +-------+-------+
                                | REST / SSE
                        +-------v-------+
                        |    FastAPI    |
                        |    Backend    |
                        +-------+-------+
                                |
              +-----------------+-----------------+
              |                 |                 |
      +-------v------+  +-------v------+  +-------v------+
      |  LangGraph   |  |    Celery    |  |    Redis     |
      |  Workflow    |  |   Workers    |  | Cache/Broker |
      +-------+------+  +--------------+  +--------------+
              |
    +---------+---------+-----------+-----------+
    |         |         |           |           |
+---v---+ +---v-----+ +-v--------+ +-v-------+ +-v----------+
|Agent 1| |Agent 2  | |Agent 3   | |Agent 4  | |Agent 5/6   |
|Intent | |Retrieve | |Landscape | |Path Gen | |Compare/Rpt |
+---+---+ +---+-----+ +-+--------+ +-+-------+ +-+----------+
    |         |         |           |           |
    +---------+---------+-----------+-----------+
                                |
              +-----------------+-----------------+
              |                 |                 |
      +-------v------+  +-------v------+  +-------v------+
      | PostgreSQL   |  |   Qdrant     |  |    Neo4j     |
      | Metadata     |  |   Vectors    |  |    Graph     |
      +--------------+  +--------------+  +--------------+
```

---

## 4. 技术选型

### 4.1 Agent 编排框架

**选择：LangGraph**

| 框架 | 优势 | 劣势 | 适配度 |
|------|------|------|--------|
| **LangGraph** | 状态机模型、原生 human-in-the-loop、checkpoint、可观测性好 | 学习曲线略陡 | 最高 |
| CrewAI | 角色化清晰、上手快 | 工作流控制弱，难以精确中断 | 中 |
| AutoGen | 对话式多 agent 灵活 | 流程控制不够显式，调试困难 | 中 |
| Dify / Coze | 低代码、快速搭建 | 定制性差，难以做复杂评估 | 低 |
| 纯 Python | 完全可控 | 开发成本高，重复造轮 | 低 |

**选择 LangGraph 的核心理由：**

1. **State machine** 天然契合多阶段 workflow
2. **interrupt()** 原生支持人工中断，无需自己实现
3. **Checkpointer** 支持断点恢复，长流程友好
4. **LangSmith** 集成，全链路可追踪
5. 每个节点可独立测试，便于分阶段评估

### 4.2 LLM 选型

| 用途 | 模型 | 理由 |
|------|------|------|
| 主力推理 | Claude Sonnet 4.x | 长上下文、指令遵循好、输出结构化能力强 |
| 复杂分析 | Claude Opus 4.x | 路径生成、比较收敛等需要深度推理的节点 |
| 轻量任务 | Claude Haiku 4.x / GPT-4o-mini | 分类、抽取、简单摘要，降低成本 |
| Embedding | BGE-M3 | 中英双语、长文本 (8192)、学术场景表现好 |
| Reranker | BGE-reranker-v2-m3 | 开源、中英双语、rerank 效果好 |
| LLM-as-Judge | Claude Opus 4.x | 评估时使用，与生成模型分离避免自我偏好 |

**成本控制策略：**

- 分级调用：简单任务用 Haiku，复杂任务用 Sonnet/Opus
- 缓存：相同 query + 模型 + 参数 → Redis 缓存 24h
- 批处理：非实时任务走 batch API（成本降低约 50%）

### 4.3 RAG 与知识库组件

| 组件 | 选型 | 理由 |
|------|------|------|
| 向量数据库 | Qdrant | 开源、性能好、支持过滤、payload 丰富 |
| 图数据库 | Neo4j | 引用网络、作者网络、概念图谱的天然选择 |
| Embedding | BGE-M3 | 多语言、多粒度、支持 sparse + dense |
| Reranker | BGE-reranker-v2-m3 | 中文场景表现好 |
| 文档解析 | GROBID | 学术 PDF 解析的事实标准 |
| 分块策略 | Semantic Chunking | 按章节 + 语义边界切分，保留上下文 |
| 检索框架 | LlamaIndex | 集成度高，便于快速原型 |

### 4.4 数据源

| 数据源 | 用途 | 访问方式 | 限制 |
|--------|------|---------|------|
| Semantic Scholar | 主力：文献元数据、引用网络 | 官方 API（免费） | 有 rate limit |
| arXiv | 预印本全文 | 官方 API | 元数据较少 |
| OpenAlex | 全景补充、机构/作者信息 | 官方 API（免费） | 覆盖广但质量参差 |
| Crossref | DOI 元数据校验 | 官方 API | 仅有元数据 |
| PubMed | 生医方向 | E-utilities | 仅生医 |
| DBLP | CS 会议/期刊 | 官方 API | 仅 CS |
| Venue Ranking | 来源分级 | 自建映射表 | 需维护 |

> **不使用 Google Scholar**：无官方 API，爬虫不稳定且违反 ToS。

### 4.5 基础设施

| 组件 | 选型 | 理由 |
|------|------|------|
| 后端框架 | FastAPI | 异步、OpenAPI 自动生成、生态好 |
| 任务队列 | Celery + Redis | 成熟稳定，支持长任务 |
| 主数据库 | PostgreSQL 16 | 关系数据、JSONB 灵活 |
| 前端 | Next.js 14 + shadcn/ui | 快速开发、组件丰富 |
| 可观测性 | LangSmith + Langfuse | 全链路追踪、成本追踪 |
| 容器化 | Docker + docker-compose | 本地开发与部署一致 |
| CI/CD | GitHub Actions | 自动化测试与部署 |
| 评估框架 | Ragas + DeepEval + promptfoo | RAG 评估 + LLM 输出评估 + prompt 回归 |

---

## 5. 用户输入与输出

### 5.1 输入 Schema

```typescript
interface UserInput {
  // 必填
  research_idea: string;          // 研究想法，自由文本
  field: string;                  // 学科领域，e.g. "NLP", "CV"

  // 可选
  background?: string;            // 用户背景，e.g. "2nd year PhD"
  constraints?: {
    timeframe?: string;           // e.g. "1 semester", "2 years"
    resources?: string[];         // e.g. ["8xA100", "no GPU"]
    methods_preference?: string[]; // 偏好的方法
    methods_avoid?: string[];     // 想避开的方法
  };
  prior_work?: string;            // 已做过的工作
  advisor_preferences?: string;   // 导师偏好
  target_venue?: string[];        // 目标会议/期刊
}
```

**示例：**

```json
{
  "research_idea": "想做大模型的幻觉检测，但不想做需要标注数据的方法",
  "field": "NLP",
  "constraints": {
    "timeframe": "1 semester",
    "resources": ["4xA100"],
    "methods_avoid": ["supervised"]
  },
  "target_venue": ["ACL", "EMNLP"]
}
```

### 5.2 输出 Schema

```typescript
interface TopicReport {
  report_id: string;
  created_at: string;
  query: UserInput;

  intent: IntentCard;
  landscape: LandscapeView;
  candidate_paths: CandidatePath[];
  comparison: ComparisonMatrix;

  evidence: EvidenceItem[];
  conflicts: ConflictItem[];
  confidence: ConfidenceReport;

  usage_notes: string;
  limitations: string[];
}
```

### 5.3 核心数据模型

```typescript
interface IntentCard {
  core_question: string;
  key_concepts: string[];
  related_subfields: string[];
  search_queries: {
    semantic_scholar: string;
    arxiv: string;
    openalex: string;
    embedding_text: string;
  }[];
  scope: {
    included: string[];
    excluded: string[];
    time_window: [number, number];
  };
  ambiguity_flags: string[];
  requires_clarification: boolean;
  clarification_questions?: string[];
}

interface LandscapeView {
  overview: string;
  main_clusters: TopicCluster[];
  timeline: TimelineEvent[];
  landmark_papers: Paper[];
  key_venues: Venue[];
  active_groups: ResearchGroup[];
  open_problems: string[];
  saturation_level: "low" | "medium" | "high";
}

interface CandidatePath {
  path_id: string;
  title: string;
  description: string;
  research_question: string;
  novelty: {
    score: number;
    reasoning: string;
    closest_prior_work: Paper[];
  };
  feasibility: {
    score: number;
    reasoning: string;
    required_resources: string[];
    potential_risks: string[];
  };
  evidence_chain: EvidenceItem[];
  recommended_approach: string;
  timeline_estimate: string;
}

interface EvidenceItem {
  evidence_id: string;
  claim: string;
  paper_id: string;
  quote: string;
  source_tier: "A" | "B" | "C" | "D";
  confidence: number;
  verification_status: "verified" | "partial" | "unverified";
}
```

---

## 6. 核心 Workflow

### 6.1 总体流程

```
[用户输入]
     |
     v
+-------------------------------------------------------------+
| Stage 1: 意图理解 (Intent Understanding)                     |
|  - 解析研究想法 -> 结构化 IntentCard                          |
|  - 生成多源检索式                                            |
|  - 检测歧义，判断是否需要澄清                                 |
|  [HIL-1] 用户确认意图（可修改）                               |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Stage 2: 文献检索 (Retrieval)                                |
|  - 并行多源检索: Semantic Scholar + arXiv + OpenAlex         |
|  - 初筛 (相关性 > 阈值) + 去重 (title/DOI/embedding)         |
|  - Rerank (BGE-reranker)                                    |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Stage 3: 全景梳理 (Landscape Mapping)                        |
|  - 主题聚类 (embedding + HDBSCAN)                            |
|  - 关键文献识别 (PageRank on citation graph)                 |
|  - 时间线抽取 (发表时间 + 引用爆发点)                         |
|  - 活跃作者/机构识别                                          |
|  [HIL-2] 用户调整全景边界                                     |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Stage 4: 候选路径生成 (Path Generation)                      |
|  - Gap 分析: 哪些问题被部分研究但未解决？                      |
|  - 生成 3-5 条候选研究方向                                    |
|  - 每条路径附证据链                                           |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Stage 5: 比较收敛 (Comparison & Convergence)                 |
|  - 多维对比矩阵 (novelty/feasibility/impact/risk)            |
|  - 候选路径两两对比                                           |
|  - 冲突检测 + 置信度评分                                      |
|  [HIL-3] 用户选择优先深入的方向                               |
+-----------------------------+-------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Stage 6: 报告生成 (Report Generation)                        |
|  - 入选路径深度分析                                           |
|  - 证据链整理                                                |
|  - 风险提示                                                  |
|  - 输出结构化 TopicReport                                     |
|  [HIL-4] 用户审阅、导出                                       |
+-----------------------------+-------------------------------+
                              |
                              v
                        [用户输出]
```

### 6.2 各阶段详细设计

#### Stage 1: 意图理解 (Intent Understanding)

**目标**：将模糊想法转为可执行的检索计划

**输入**：UserInput → **输出**：IntentCard

**处理步骤：**

1. **实体抽取**：识别核心概念、方法、任务、限定词
2. **意图归类**：判断是"新方向探索"还是"现有方向深化"
3. **Query 生成**：为每个数据源生成适配的检索式
4. **歧义检测**：识别多义词、范围过宽/过窄
5. **澄清判断**：若歧义严重，生成澄清问题

**Prompt 策略：**

```
System: 你是科研信息学专家，擅长将研究想法转为精确的检索计划。

User:
研究想法: {research_idea}
领域: {field}
约束: {constraints}

请输出 JSON:
{
  "core_question": "...",
  "key_concepts": [...],
  "search_queries": {...},
  "scope": {...},
  "ambiguity_flags": [...]
}
```

**人工确认点 HIL-1：**

用户看到 IntentCard，可以：
- 确认 → 进入 Stage 2
- 修改 → 修改后重新确认
- 拒绝 → 返回重新输入

#### Stage 2: 文献检索 (Retrieval)

**目标**：从多源获取高质量文献集

**输入**：IntentCard → **输出**：List[Paper]（约 200-500 篇）

**处理步骤：**

1. **查询路由**：根据 field 选择数据源
2. **并行检索**：asyncio.gather 并行调用各 API
3. **初步过滤**：
   - 相关性阈值：embedding similarity > 0.6
   - 时间范围：默认近 5 年，里程碑文献不限
   - 去重：DOI → arXiv ID → 标题模糊 → embedding
4. **Rerank**：BGE-reranker-v2-m3
5. **存储**：元数据 → PostgreSQL，向量 → Qdrant

**异常处理：**

- API 超时：重试 3 次，指数退避
- Rate limit：请求队列 + 多 API key 轮换
- 空结果：放宽查询条件，重新检索

#### Stage 3: 全景梳理 (Landscape Mapping)

**目标**：构建领域知识结构

**输入**：List[Paper] + IntentCard → **输出**：LandscapeView

**处理步骤：**

1. **主题聚类**：
   - Embedding (BGE-M3) → 降维 (UMAP) → HDBSCAN
   - 每簇用 LLM 生成主题名称和描述
2. **关键文献识别**：
   - 构建引用图 → PageRank → Top-10
   - 结合专家规则：高被引 + 跨簇被引
3. **时间线抽取**：
   - 按年份统计发表量
   - 识别引用爆发点 (citation burst)
4. **活跃作者/机构**：统计发表量和被引量
5. **饱和度评估**：基于近 2 年发表量趋势 + 开放问题数量

**人工确认点 HIL-2（建议）：**

用户可以调整：
- 添加/删除主题簇
- 调整时间窗口
- 标记不相关的簇

#### Stage 4: 候选路径生成 (Path Generation)

**目标**：识别研究 gap，生成候选方向

**输入**：LandscapeView + IntentCard → **输出**：List[CandidatePath]（3-5 条）

**处理步骤：**

1. **Gap 识别**：
   - 跨簇组合：两个簇的交叉点
   - 时间线空白：新兴但未成熟的主题
   - 方法迁移：其他领域的方法引入
   - 负面证据：被多次提到但未解决的问题
2. **路径生成**：
   - 每条路径 = 核心问题 + 方法思路 + 预期贡献
   - 用 LLM 生成，但必须基于检索到的证据
3. **新颖度评估**：
   - 与已有文献的 embedding 距离
   - 最接近的 prior work 识别
4. **可行性评估**：
   - 是否有相关方法可参考
   - 资源需求是否匹配用户约束
5. **证据链构建**：每个 claim 附 1-3 条 EvidenceItem

**质量保障：**

- 强制引用：LLM 输出中每个 claim 必须有 [paper_id]
- 引用验证：paper_id 必须在检索结果中存在
- 去重：路径间 embedding 距离 > 0.3，避免同质化

#### Stage 5: 比较收敛 (Comparison & Convergence)

**目标**：多维度对比，支持用户决策

**输入**：List[CandidatePath] → **输出**：ComparisonMatrix

**处理步骤：**

1. **构建对比矩阵**：

| 维度 | 权重 | 说明 |
|------|------|------|
| Novelty | 25% | 新颖度 |
| Feasibility | 30% | 可行性 |
| Impact | 25% | 潜在影响 |
| Risk | 20% | 风险（负向） |

2. **两两对比**：用 LLM 做 pairwise comparison
3. **冲突检测**：识别路径间的矛盾
4. **置信度评分**：每个维度的置信度
5. **推荐排序**：加权和，但展示完整矩阵

**人工确认点 HIL-3：**

用户选择 1-2 条优先深入的方向，可调整权重。

#### Stage 6: 报告生成 (Report Generation)

**目标**：生成结构化、可验证的选题报告

**输入**：所有前序阶段输出 → **输出**：TopicReport

**处理步骤：**

1. **深度分析**：对用户选择的方向做深度分析
2. **证据链整理**：按 claim 组织证据
3. **风险提示**：明确列出潜在风险和不确定性
4. **使用说明**：如何解读报告、如何验证
5. **局限性声明**：系统的局限和适用范围

**人工确认点 HIL-4：** 用户审阅报告，可导出 PDF/Markdown。

### 6.3 人工确认点总结

| HIL 点 | 阶段 | 必要性 | 用户操作 |
|--------|------|--------|---------|
| HIL-1 | 意图理解 | 必须 | 确认/修改意图 |
| HIL-2 | 全景梳理 | 建议 | 调整边界 |
| HIL-3 | 比较收敛 | 必须 | 选择方向 |
| HIL-4 | 报告生成 | 可选 | 审阅/导出 |

**设计原则：**

- 关键决策必须人工确认（HIL-1, HIL-3）
- 可选确认点提升用户控制感，但增加交互成本
- 所有 HIL 点支持"跳过"（使用 Agent 默认建议），但会记录跳过率

---

## 7. 知识库设计

**结论：本系统涉及知识库，且是三层结构。** 知识库不是可选组件，而是"证据链可追溯"这一核心价值的基础设施。

### 7.1 知识库分层

| 层级 | 类型 | 内容 | 更新频率 |
|------|------|------|---------|
| L1 领域知识库 | 静态/半静态 | 学科 ontology、venue ranking、方法术语表 | 季度更新 |
| L2 文献知识库 | 动态 | 每次会话检索的文献 + 元数据 | 实时 |
| L3 图谱知识库 | 半动态 | 引用网络、作者网络、概念关联 | 周级更新 |

### 7.2 领域知识库 (L1) — 静态知识

**内容：**

| 知识类型 | 存储形式 | 用途 |
|---------|---------|------|
| Venue Ranking | YAML / DB | 来源分级权重 |
| CS Ontology | JSON | 概念层级 (e.g. "LLM" → "NLP" → "AI") |
| Method Taxonomy | JSON | 方法分类 (supervised / RL / prompting 等) |
| Task Taxonomy | JSON | 任务分类 (classification / generation 等) |

**Venue Ranking 示例：**

```yaml
# data/venue_ranking.yaml
tier_A:
  - name: NeurIPS
    field: ML
    h5_index: 300
    type: conference
  - name: ACL
    field: NLP
    h5_index: 250
    type: conference
  - name: Nature
    field: cross_domain
    h5_index: 400
    type: journal
tier_B:
  - name: TMLR
    field: ML
    h5_index: 80
    type: journal
  - name: EMNLP_Findings
    field: NLP
    h5_index: 70
    type: conference
```

### 7.3 文献知识库 (L2) — 动态知识

**数据模型：**

```typescript
interface Paper {
  paper_id: string;
  title: string;
  abstract: string;
  authors: string[];
  venue: string;
  year: number;
  doi?: string;
  arxiv_id?: string;
  citations: number;
  tldr: string;
  source_tier: "A" | "B" | "C" | "D";
  data_source: "semantic_scholar" | "arxiv" | "openalex";
  embedding_vector_id: string;
  retrieved_at: string;
}
```

**构建流程：**

1. **Ingestion**：检索结果 → 元数据标准化 → PostgreSQL
2. **Chunking**：摘要 + 全文 → 按章节切分 → semantic chunking
3. **Embedding**：BGE-M3 → Qdrant
4. **Graph**：引用关系 → Neo4j

### 7.4 图谱知识库 (L3) — Citation Graph

**Neo4j Schema：**

```cypher
// Nodes
(:Paper {id, title, year, venue, citations})
(:Author {id, name, affiliation})
(:Venue {id, name, tier})
(:Concept {id, name, parent_id})

// Relationships
(:Paper)-[:CITES]->(:Paper)
(:Paper)-[:AUTHORED_BY]->(:Author)
(:Paper)-[:PUBLISHED_IN]->(:Venue)
(:Paper)-[:ABOUT]->(:Concept)
(:Concept)-[:PARENT_OF]->(:Concept)
```

**图谱构建：**

1. **数据来源**：Semantic Scholar 的 references + citations
2. **抽取**：LLM 抽取 concept (paper → concept)
3. **存储**：批量导入 Neo4j
4. **刷新**：每周增量更新

**典型查询：**

```cypher
// 找出某个主题的核心文献
MATCH (p:Paper)-[:ABOUT]->(c:Concept {name: "hallucination detection"})
RETURN p.title, p.citations
ORDER BY p.citations DESC LIMIT 10

// 找出研究空白：有概念但少文献
MATCH (c:Concept)
WHERE NOT (p:Paper)-[:ABOUT]->(c)
RETURN c.name

// 找出跨主题的桥梁文献
MATCH (p:Paper)-[:ABOUT]->(c1:Concept), (p)-[:ABOUT]->(c2:Concept)
WHERE c1 <> c2
RETURN p.title, count(DISTINCT c1) AS breadth
ORDER BY breadth DESC LIMIT 10
```

### 7.5 检索策略 (Hybrid Retrieval)

**三级检索漏斗：**

```
Query -> [BM25 (sparse)]   --+
Query -> [Dense (BGE-M3)]  --+--> [RRF Fusion] -> [BGE-reranker] -> Top-K
Query -> [Graph lookup]    --+
```

1. **First-stage**：BM25 + Dense + Graph，各取 top-100
2. **Fusion**：Reciprocal Rank Fusion (RRF)
3. **Rerank**：BGE-reranker-v2-m3，取 top-20

**元数据过滤：**

```typescript
interface RetrievalFilter {
  year_range?: [number, number];
  venue_tier?: ("A" | "B" | "C" | "D")[];
  min_citations?: number;
  document_type?: ("conference" | "journal" | "preprint");
  open_access_only?: boolean;
}
```

### 7.6 RAG 策略

**为什么需要 RAG：**

选题 Agent 的核心是"基于证据做判断"。LLM 的参数知识有以下问题：
- 知识截止：不知道最新进展
- 幻觉风险：可能编造论文或结论
- 无法溯源：用户难以验证

**RAG Pipeline：**

```
User Query -> Query Rewrite -> Hybrid Retrieval -> Rerank
           -> Context Assembly -> LLM Generation -> Citation Verification
```

**Context Assembly 策略：**

- 每篇文献：title + abstract + key findings (from TLDR)
- 按来源 tier 排序：A > B > C > D
- 控制 context length：不超过 50K tokens
- 强制引用：LLM 输出必须带 [paper_id]

### 7.7 知识库边界与维护

**不纳入知识库的内容：**

| 内容 | 原因 |
|------|------|
| 全文 PDF | 版权风险，仅存摘要 + 元数据 |
| 付费数据库内容 | 无法获取 |
| 社交媒体讨论 | 质量不可控 |

**知识库质量保障：**

1. **去重**：DOI + arXiv ID + 标题相似度 (fuzzy match) + embedding
2. **版本控制**：文献元数据变更记录
3. **过期处理**：超过 2 年未更新的会话数据归档
4. **隐私**：用户输入加密存储，不用于训练

---

## 8. 数据源、来源分级与冲突处理

### 8.1 数据源接入策略

| 数据源 | 覆盖范围 | 优势 | 限制 |
|--------|---------|------|------|
| Semantic Scholar | 全学科 | 引用网络完整、有 TLDR、API 免费 | Rate limit (1 req/s) |
| arXiv | 物理/CS/数学 | 预印本最全 | 元数据较简单 |
| OpenAlex | 全学科 | 覆盖最广、有机构信息 | 数据质量参差 |
| Crossref | 全学科 | DOI 权威 | 仅元数据 |
| DBLP | CS | CS 文献最全 | 仅 CS |

**多源合并策略：**

1. **Query 路由**：根据领域选择数据源组合
2. **并行检索**：asyncio.gather 并行调用
3. **去重**：DOI → arXiv ID → 标题模糊匹配 → embedding 相似度
4. **合并**：union with provenance（保留每条记录来自哪个源）

### 8.2 来源分级规则

**四级分类：**

| Tier | 定义 | 权重 | 例子 |
|------|------|------|------|
| A | 顶会顶刊 | 1.0 | NeurIPS, ACL, Nature |
| B | 知名期刊/会议 | 0.7 | EMNLP Findings, TMLR |
| C | 一般同行评议 | 0.5 | Workshop, 小型会议 |
| D | 预印本/灰色文献 | 0.3 | arXiv, 博客 |

**动态调整因子：**

```
final_weight = base_weight * venue_factor * citation_factor * time_factor

venue_factor:     tier A=1.0, B=0.8, C=0.5, D=0.3
citation_factor:  log(1 + citations) / log(1 + median_citations)
time_factor:      exp(-lambda * (current_year - pub_year)), lambda=0.1
```

### 8.3 冲突检测与处理

**冲突类型：**

| 冲突类型 | 例子 | 处理策略 |
|---------|------|---------|
| 结论冲突 | 同一问题不同论文结论相反 | 展示双方证据 + 置信度评分 |
| 数据冲突 | 同一指标不同论文数值差异大 | 标注数据来源 + 样本差异 |
| 定义冲突 | 同一术语不同含义 | 按上下文区分 + 明确定义 |
| 时间冲突 | 旧结论被新研究推翻 | 时间加权 + 标注"已被推翻" |

**冲突处理规则：**

1. **检测**：基于 embedding 相似度 + 关键词匹配，识别潜在冲突
2. **分级**：
   - High：直接矛盾且都来自高权重来源 → 明确标注
   - Medium：部分矛盾或权重较低 → 提示注意
   - Low：表述差异但结论一致 → 不标注
3. **展示**：冲突两侧证据并列展示，不替用户做判断
4. **置信度衰减**：有冲突的结论，置信度乘以 0.7

**置信度计算公式：**

```
confidence = w_source * w_consistency * w_recency * w_relevance

w_source:      来源权重 (Tier A=1.0 到 D=0.3)
w_consistency: 证据一致性 (有冲突则乘 0.7)
w_recency:     时间衰减 (半衰期 3 年)
w_relevance:   与 query 相关度
```

### 8.4 引用规范

每个 claim 必须携带 EvidenceItem（见 5.3 节）。

**验证流程：**

1. **自动验证**：paper_id 在数据库存在 + DOI/arXiv ID 有效
2. **半自动验证**：quote 与原文 embedding 相似度 > 0.85
3. **人工抽检**：随机抽 10% evidence 人工验证

---

## 9. 评估方案

### 9.1 评估目标

评估要回答 3 个核心问题：

| 层次 | 问题 | 关注点 |
|------|------|--------|
| Functional | 它能否正常工作？ | 流程跑得通、不崩溃、不编造 |
| Quality | 它输出的东西有用吗？ | 建议可采纳、证据链完整 |
| Value | 它真的帮到科研了吗？ | 用户愿意用、节省时间、改变决策 |

### 9.2 评估维度总览

```
                    评估金字塔

                        +-----+
                        | L3  |  Value: 用户价值
                        +-----+   (采纳率、满意度、时间节省)

                    +---------+
                    |   L2    |  Quality: 输出质量
                    +---------+   (证据链、幻觉率、排序一致性)

                +-------------+
                |     L1      |  Functional: 系统健康度
                +-------------+   (完成率、延迟、错误率)
```

### 9.3 分阶段评估指标

每个 Stage 有独立输入输出，可独立测试。这是 LangGraph 架构的红利。

#### Stage 1: 意图理解

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Intent Accuracy | 结构化意图与专家标注一致 | 人工评分 1-5 | >= 4.0 | 人工 |
| Query Recall@20 | 扩展查询覆盖专家查询 | \|Q 且 G\| / \|G\| | >= 0.80 | 自动 + Golden |
| Concept Coverage | 关键概念覆盖度 | \|C 且 G\| / \|G\| | >= 0.75 | 自动 + Golden |
| Ambiguity Detection | 歧义检测准确率 | F1 | >= 0.75 | Golden |
| Clarification Precision | 澄清问题的必要性 | 人工评估 | >= 80% 必要 | 人工 |
| JSON Schema Validity | 输出符合 schema | valid / total | >= 99% | 自动 |

#### Stage 2: 文献检索

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Recall@100 | 覆盖 golden set | \|R 且 G\| / \|G\| | >= 0.75 | Golden |
| nDCG@10 | 排序质量 | 标准 nDCG | >= 0.65 | Golden |
| MRR | 首个相关结果排名 | 1/rank | >= 0.55 | Golden |
| Source Diversity | 来源多样性 | entropy over sources | >= 1.2 | 自动 |
| Dedup Accuracy | 同一文献跨源识别 | precision / recall | >= 0.95 / 0.90 | Golden |
| API Success Rate | 外部 API 调用成功 | success / total | >= 95% | 自动 |
| Latency p95 | 检索延迟 | percentile | <= 30s | 自动 |

#### Stage 3: 全景梳理

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Cluster Coherence | 聚类内聚度 | 人工 1-5 | >= 4.0 | 人工 |
| Landmark Recall@10 | 里程碑文献识别 | \|L 且 G\| / \|G\| | >= 0.80 | Golden |
| Timeline Accuracy | 发展脉络准确性 | 人工 1-5 | >= 4.0 | 人工 |
| Cluster Granularity | 聚类粒度合理性 | 人工 | 5-12 clusters | 人工 |
| Concept Coverage | 关键概念覆盖 | \|C 且 G\| / \|G\| | >= 0.75 | Golden |
| Graph Consistency | 图谱查询结果一致 | 单元测试 | 100% | 自动 |

#### Stage 4: 路径生成

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Path Count | 候选路径数量 | count | 3-5 | 自动 |
| Novelty Score | 新颖度评分合理性 | 人工 1-5 | >= 3.5 | 专家 |
| Feasibility Reasonableness | 可行性判断合理性 | 人工 1-5 | >= 3.5 | 专家 |
| Claim Support Rate | 每个 claim 有引用 | supported / total | >= 95% | 自动 |
| Hallucination Rate | 虚构文献/结论 | unverified / total | <= 3% | 人工抽检 |
| Evidence Diversity | 证据来源多样性 | unique_sources | >= 3 sources | 自动 |
| Path Distinctness | 路径间区分度 | pairwise embedding dist | >= 0.3 | 自动 |
| Gap Validity | gap 是否真实存在 | 人工验证 | >= 80% | 专家 |

#### Stage 5: 比较收敛

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Ranking Consistency | 排序与专家排序一致 | Kendall's tau | >= 0.60 | Golden |
| Dimension Coverage | 对比维度完整性 | covered / required | 100% | 自动 |
| Conflict Detection F1 | 冲突识别准确率 | F1 | >= 0.75 | Golden |
| Confidence Calibration | 置信度校准 | ECE | <= 0.15 | Golden |
| Pairwise Consistency | 两两对比逻辑一致 | transitivity check | >= 95% | 自动 |

#### Stage 6: 报告生成

| 指标 | 定义 | 计算方式 | 目标值 | 评估方法 |
|------|------|---------|--------|---------|
| Citation Correctness | 引用真实且相关 | verified / total | >= 95% | 自动 (DOI) + 人工 |
| Citation Faithfulness | 引用内容与原文一致 | 人工抽检 | >= 95% | Ragas faithfulness |
| Report Completeness | 必填字段完整 | filled / total | 100% | 自动 |
| Readability | 可读性 | 人工 1-5 | >= 4.0 | 人工 |
| Actionability | 建议可执行性 | 人工 1-3 | >= 2.5 | 人工 |

### 9.4 端到端评估指标

#### 9.4.1 离线评估 (Offline)

| 指标 | 定义 | 计算方式 | 目标值 |
|------|------|---------|--------|
| Task Completion Rate | 端到端完成 | completed / total | >= 90% |
| Expert Adoption Rate | 专家认为至少 1 条建议可采纳 | adopted / total | >= 70% |
| Expert Overall Rating | 专家整体评分 | 1-5 均值 | >= 3.8 |
| Top-1 Hit Rate | Top-1 建议被专家认可 | hit / total | >= 55% |
| Evidence Chain Coverage | 结论有证据支撑比例 | evidenced / total | >= 90% |
| Hallucination Rate | 无法验证的 claim | unverified / total | <= 3% |
| Citation Verify Rate | 引用可通过 API 验证 | verified / total | >= 98% |
| Time to Insight | 从输入到可用建议 | median | <= 10min |
| Cost per Session | 每次会话 LLM 成本 | USD | <= $2 |

#### 9.4.2 在线评估 (Online / Beta)

| 指标 | 定义 | 计算方式 | 目标值 |
|------|------|---------|--------|
| User Satisfaction (CSAT) | 用户满意度 | 5-point Likert | >= 4.0 |
| NPS | 净推荐值 | %promoters - %detractors | >= 30 |
| Suggestion Adoption Rate | 用户实际采纳建议 | adopted / total | >= 40% |
| Human Override Rate | 人工推翻 Agent 结论 | overridden / total | <= 30% |
| Task Completion Time | 用户完成选题时间 | median | 比基线快 50% |
| 7-day Retention | 7 日留存 | returning / total | >= 30% |
| Critical Bug Rate | 严重 bug 比例 | critical / total | 0 |

### 9.5 Golden Dataset 构建

这是评估的基础。没有 golden set，多数指标无法计算。

#### 9.5.1 构建流程

```
[案例收集]  ->  [专家标注]  ->  [一致性检验]
 20-30 cases    3 experts       Cohen's kappa
 覆盖难度       独立标注        >= 0.6
                                     |
          +--------------------------+
          |
          v
    [分歧讨论]  ->  [版本管理]
     达成共识        git 版本化
                    季度更新
```

#### 9.5.2 案例设计原则

| 原则 | 说明 |
|------|------|
| 多样性 | 覆盖 AI/NLP/CV/ML theory 等子领域 |
| 难度分层 | easy: 30%, medium: 50%, hard: 20% |
| 真实感 | 来自真实用户提问，非构造 |
| 可验证 | 有明确专家共识的答案 |
| 时效性 | 包含近 2 年的新兴方向 |

#### 9.5.3 标注 Schema

```json
{
  "case_id": "case_001",
  "query": {
    "research_idea": "想做大模型的幻觉检测",
    "field": "NLP",
    "constraints": {}
  },
  "expected": {
    "intent": {
      "core_question": "...",
      "key_concepts": ["hallucination", "detection", "LLM"]
    },
    "golden_papers": [
      {"title": "...", "doi": "...", "reason": "里程碑"}
    ],
    "golden_topics": [
      {"name": "...", "papers": []}
    ],
    "golden_directions": [
      {"title": "...", "should_appear": true, "priority": "high"}
    ]
  },
  "difficulty": "medium",
  "domain": "NLP/LLM"
}
```

#### 9.5.4 一致性检验

- Cohen's kappa >= 0.6 才视为标注有效
- kappa < 0.5 的案例需重新标注或剔除
- 保留分歧记录，用于理解任务主观性边界

### 9.6 评估方法与工具

#### 9.6.1 自动化评估

| 工具 | 用途 | 指标 |
|------|------|------|
| Ragas | RAG 评估 | faithfulness, answer_relevancy, context_precision, context_recall |
| DeepEval | LLM 输出评估 | 自定义 metric (e.g. claim_support_rate) |
| promptfoo | Prompt 回归 | 输出稳定性、schema 合规性 |
| pytest | 单元/集成测试 | 各节点功能正确性 |
| 自研脚本 | 领域指标 | nDCG, Kendall's tau, hallucination rate |

#### 9.6.2 LLM-as-Judge 协议

为降低人工评估成本，部分指标用强模型评估，但需校准。

**校准流程：**

1. 选取 50 个样本，人工 + LLM 同时评分
2. 计算 agreement (Cohen's kappa 或 Pearson)
3. kappa >= 0.7 才可批量使用 LLM-as-Judge
4. kappa < 0.7 → 调整 prompt 或退化为人工评估

**Judge Prompt 示例：**

```
你是一个科研选题评估专家。请评估以下候选研究方向的可行性。

评分标准 (1-5):
- 1: 完全不可行（需要不存在的资源或违背已知约束）
- 2: 可行性低（风险极高，证据不足）
- 3: 中等可行（有一定证据，但存在明显风险）
- 4: 可行（证据充分，风险可控）
- 5: 非常可行（证据链完整，资源匹配）

候选方向: {candidate_path}
相关证据: {evidence}
用户约束: {constraints}

请输出 JSON: {"score": 1-5, "reasoning": "..."}
```

### 9.7 评估流程

#### 9.7.1 CI/CD 集成

```
[PR 提交]  ->  [单元测试]  ->  [集成测试]
                (pytest)       (10 cases)
                                    |
          +-------------------------+
          |
          v
    [Golden 回归]  ->  [人工抽检]
     (全部 30)          (5%)
          |
          v
      [部署]
```

#### 9.7.2 评估节奏

| 频率 | 评估内容 | 负责人 |
|------|---------|--------|
| 每次 commit | 单元测试、schema 校验 | 开发者 |
| 每日 | 集成测试 (10 cases)、smoke test | CI |
| 每周 | Golden 全量回归、LLM-as-Judge | 开发者 |
| 每里程碑 | 专家评估 (全量 30 cases) | 研究团队 |
| Beta 阶段 | 用户研究 (5-10 人) | 产品团队 |

#### 9.7.3 回归失败处理

```
IF golden_score[metric] < baseline - 0.05:
    1. 阻塞合并
    2. 定位下降的 case
    3. 分析根因 (prompt / model / retrieval)
    4. 修复或回滚
    5. 更新 baseline (如果是预期变化)
```

### 9.8 验收标准 (Definition of Done)

分三档验收，适应不同阶段。

#### Level 1: MVP (可演示)

| 指标 | 目标 |
|------|------|
| Task Completion Rate | >= 80% |
| Hallucination Rate | <= 5%（无编造文献） |
| Citation Verify Rate | >= 90% |
| Expert Rating | >= 3.0 |
| Critical Bugs | 0 |

#### Level 2: Beta (可试用)

| 指标 | 目标 |
|------|------|
| Task Completion Rate | >= 90% |
| Expert Adoption Rate | >= 60% |
| Hallucination Rate | <= 3% |
| Citation Verify Rate | >= 98% |
| Expert Rating | >= 3.8 |
| Kendall's tau (ranking) | >= 0.5 |

#### Level 3: Production (可发布)

| 指标 | 目标 |
|------|------|
| Task Completion Rate | >= 95% |
| Expert Adoption Rate | >= 70% |
| User Satisfaction | >= 4.0 |
| Hallucination Rate | <= 1% |
| NPS | >= 30 |
| Critical Bugs | 0 |

### 9.9 消融实验 (Ablation Study)

验证每个组件的贡献，证明设计合理性。

| 实验 | 移除组件 | 预期影响 |
|------|---------|---------|
| A1 | 移除 RAG（纯 LLM） | 幻觉率显著上升 |
| A2 | 移除 Reranker | nDCG 下降 |
| A3 | 移除图谱 | Landmark Recall 下降 |
| A4 | 移除人工确认点 | 采纳率下降、用户信任度下降 |
| A5 | 移除冲突检测 | 用户发现矛盾但系统未标注的比例上升 |

**消融实验执行：** 在 Golden Dataset 上跑 5 组，对比各指标差异，用统计检验（paired t-test）验证显著性。

### 9.10 如何判断"Agent 真的帮到了科研选题"

这是最难评估的部分，需要多层验证。

#### 9.10.1 直接证据

| 证据 | 收集方式 |
|------|---------|
| 专家认可建议 | 专家评分 + 访谈 |
| 用户采纳建议 | 跟踪用户后续动作 |
| 发现用户未想到的方向 | 用户自评 + 访谈 |

#### 9.10.2 对照实验

设计 A/B 测试：

| 组别 | 任务 |
|------|------|
| Control | 用户自行用 Google Scholar 完成选题 |
| Treatment | 用户使用本 Agent 完成选题 |

对比指标：
- 任务完成时间
- 文献覆盖广度（专家评估）
- 最终选题的专家评分
- 用户主观负担（NASA-TLX）

#### 9.10.3 长期追踪

- 追踪使用 Agent 选题的用户，6 个月后课题进展
- 对比未使用 Agent 的对照组
- 这是最强证据，但周期长，作为 Production 阶段评估

#### 9.10.4 反向指标（防止"假成功"）

| 反向指标 | 含义 |
|---------|------|
| 人工修改率过高 | Agent 输出质量差 |
| 用户跳过 HIL 点比例高 | 用户不信任或不愿参与 |
| 建议高度同质化 | Agent 缺乏多样性 |
| 用户事后报告"被误导" | 存在幻觉或误导性结论 |

---

## 10. 接口定义

### 10.1 REST API

**创建会话**

```
POST /api/v1/sessions
Content-Type: application/json

{
  "research_idea": "想做大模型的幻觉检测",
  "field": "NLP",
  "constraints": {}
}

Response:
{
  "session_id": "sess_xxx",
  "status": "awaiting_intent_confirmation",
  "intent_card": { ... },
  "requires_clarification": false
}
```

**人工确认**

```
POST /api/v1/sessions/{id}/confirm
Content-Type: application/json

{
  "stage": "intent" | "landscape" | "path_selection",
  "decision": "approve" | "edit" | "reject",
  "payload": { ... }
}

Response:
{
  "session_id": "sess_xxx",
  "status": "retrieval_started",
  "next_hil_point": "landscape_review"
}
```

**获取状态**

```
GET /api/v1/sessions/{id}
```

**获取报告**

```
GET /api/v1/sessions/{id}/report
```

### 10.2 LangGraph 状态定义

```python
from typing import TypedDict
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver

class TopicAgentState(TypedDict):
    # Input
    user_input: dict

    # Stage outputs
    intent_card: dict
    retrieved_papers: list
    landscape: dict
    candidate_paths: list
    comparison: dict
    report: dict

    # Meta
    errors: list
    current_stage: str
    hil_decisions: dict


def build_graph() -> StateGraph:
    graph = StateGraph(TopicAgentState)

    graph.add_node("intent", intent_understanding_node)
    graph.add_node("retrieval", retrieval_node)
    graph.add_node("landscape", landscape_node)
    graph.add_node("path_gen", path_generation_node)
    graph.add_node("compare", comparison_node)
    graph.add_node("report", report_node)

    graph.set_entry_point("intent")
    graph.add_edge("intent", "retrieval")
    graph.add_edge("retrieval", "landscape")
    graph.add_edge("landscape", "path_gen")
    graph.add_edge("path_gen", "compare")
    graph.add_edge("compare", "report")
    graph.add_edge("report", END)

    # HIL-1: 在 intent 之后中断
    graph.add_node("hil_intent", human_confirm_node)
    graph.add_edge("intent", "hil_intent")
    graph.add_edge("hil_intent", "retrieval")

    return graph.compile(checkpointer=MemorySaver())
```

### 10.3 核心节点伪代码

```python
async def retrieval_node(state: TopicAgentState) -> dict:
    intent = state["intent_card"]

    # 1. 并行多源检索
    tasks = [
        search_semantic_scholar(intent["search_queries"]["semantic_scholar"]),
        search_arxiv(intent["search_queries"]["arxiv"]),
        search_openalex(intent["search_queries"]["openalex"]),
    ]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # 2. 合并 + 去重
    papers = merge_and_dedupe(results)

    # 3. 过滤
    papers = [p for p in papers if p["relevance"] > 0.6]

    # 4. Rerank
    papers = await rerank(papers, state["user_input"]["research_idea"])

    # 5. 存储
    await store_papers(papers)

    return {"retrieved_papers": papers[:500]}
```

---

## 11. 实施计划

### 11.1 2 天开发计划

| 时间 | 任务 | 产出 |
|------|------|------|
| Day 1 AM | 需求分析、用户画像、边界定义 | 文档 1-3 章 |
| Day 1 PM | 技术选型、Workflow 设计、知识库设计 | 文档 4-8 章 |
| Day 2 AM | 评估方案、接口定义、伪代码 | 文档 9-10 章 |
| Day 2 PM | Demo 脚本、测试用例、最终审阅 | Demo + 测试报告 |

### 11.2 开发优先级 (MoSCoW)

| 优先级 | 功能 |
|--------|------|
| Must | 意图理解、检索、全景、路径生成、比较、报告 |
| Must | 人工确认点 (HIL-1, HIL-3) |
| Should | HIL-2、图谱可视化、PDF 导出 |
| Could | 多用户协作、历史记录 |
| Won't | 实时推送、移动端 |

### 11.3 后续迭代路线

| 阶段 | 目标 |
|------|------|
| v0.1 | 端到端跑通，单学科 (NLP) |
| v0.2 | 加入 Golden Dataset 评估，优化检索 |
| v0.3 | 加入图谱，提升全景质量 |
| v0.4 | Beta 用户测试 |
| v1.0 | 多学科支持，Production |

---

## 12. 风险与缓解

### 12.1 技术风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| 数据源 API 不稳定 | 检索失败 | 重试 + 退避 + 多源 fallback |
| LLM 幻觉 | 编造文献/结论 | 强制引用 + DOI 验证 + 人工抽检 |
| 评估主观性 | 专家判断不一致 | 多专家 + Cohen's kappa 校准 |
| 范围蔓延 | 想做太多功能 | 严格 Out-of-Scope 清单 |
| 数据源 rate limit | 检索延迟高 | 请求队列 + 缓存 + 多源冗余 |

### 12.2 长期风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| LLM 模型更新导致行为漂移 | 指标波动 | 模型版本锁定 + 回归测试 |
| 学术数据源政策变化 | 数据中断 | 多源冗余设计 |
| 用户过度依赖 | 伦理问题 | 明确局限性说明 + 人工确认 |

---

## 附录 A: Golden Dataset 构建指南

### A.1 案例来源

- 从 Reddit r/MachineLearning、知乎、导师实际学生提问收集
- 覆盖不同难度：初学者问题、前沿问题、跨领域问题
- 每个案例需有领域专家标注

### A.2 标注流程

1. 案例收集（20-30 个）
2. 专家标注（每个案例 3 名专家独立标注）
3. 一致性检验（Cohen's kappa >= 0.6）
4. 分歧讨论 → 达成共识
5. 版本管理（git 管理 golden set）
6. 季度更新（跟进领域发展）

### A.3 标注内容

| 字段 | 说明 |
|------|------|
| query | 用户原始输入 |
| expected_intent | 期望的结构化意图 |
| golden_papers | 必须覆盖的文献列表 (20-50 篇) |
| golden_topics | 期望识别的主题 |
| golden_directions | 期望生成的候选方向 |
| difficulty | easy / medium / hard |
| domain | 子领域 |

---

## 附录 B: 评估工具链

| 工具 | 用途 |
|------|------|
| Ragas | RAG 指标 (faithfulness, relevance, context precision) |
| DeepEval | LLM 输出评估（自定义 metric） |
| promptfoo | Prompt 回归测试 |
| LangSmith | 全链路追踪 + 数据集评估 |
| pytest | 单元/集成测试 |
| Label Studio | 人工标注平台 |
