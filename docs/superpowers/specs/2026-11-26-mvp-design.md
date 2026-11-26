# 科研选题 Agent MVP 设计

> 日期：2026-11-26
> 上游文档：`DESIGN.md`（生产级完整设计）、`REQUIREMENTS.md`（评估原文）、`CLAUDE.md`（已确认决策）
> 本文档是 MVP 的 spec：从 `DESIGN.md` 收敛出 2 天内可跑通、可演示、可评估的最小系统。

---

## 1. 意图

ASCI 项目 AI 高级开发岗位评估要求设计一个"科研选题 Agent"，并明确不要求完整实现，但要"看清设计逻辑和验收思路"。`DESIGN.md` 已经是生产级设计（4 个存储组件、6 个 Agent、Next.js 前端、完整评估体系）。

MVP 的目的不是实现 `DESIGN.md`，而是**用一个真实跑通的端到端流程证明设计的核心主张成立**：

1. 范围收敛——6 个固定阶段，不做大而全
2. 证据链——每个结论都能追溯到检索结果中的真实论文，编造的引用会被程序拦下
3. 工程化——LangGraph 状态机 + 人工中断 + checkpoint 持久化 + 可量化的评估

## 2. 目标、非目标、反目标

### 目标

- 6 个 Stage 在真实 Semantic Scholar 数据 + 真实 GLM 调用下端到端跑通
- HIL-1（意图确认）和 HIL-3（选方向）在 CLI 中真正阻塞等待用户输入，用户可修改
- 每条 claim 的 `[paper_id]` 经程序验证，验证结果写入数据库并在报告中标注
- 输出 Markdown 报告到 `reports/<session_id>.md`
- 评估脚本在 3 个 golden case 上跑出自动指标表
- 所有状态存在一个 PostgreSQL 库：业务表 + pgvector 向量列 + LangGraph checkpoint 表
- `uv run topic-agent --resume <session_id>` 能从最后一个 checkpoint 恢复会话，证明 checkpoint 持久化真实可用
- `README.md` 让评估者从 clone 到跑通；`docs/sample/` 提交一份真实运行的报告样例和评估指标表作为可见证据
- 每个会话有结构化 JSONL 日志：节点进出、state 变化、每次 LLM / embedding / S2 调用、引用验证结果，全部可追溯

### 非目标（MVP 明确不做）

| 不做 | 原因 | 对应 DESIGN.md |
|---|---|---|
| Neo4j | 引用图用 NetworkX 内存计算，逻辑等价 | 7.4 |
| BGE-M3 本地模型 | 换 GLM embedding API，省掉 PyTorch 和 2GB 下载 | 4.2 / 4.3 |
| FastAPI / Next.js | CLI 直接调 graph，REST 接口已在 DESIGN.md 10.1 定义 | 3.3 / 10.1 |
| Celery / Redis | 单进程同步执行足够 | 4.5 |
| arXiv / OpenAlex / Crossref | S2 单源已含元数据 + 引用关系 | 4.4 / 8.1 |
| 全文 PDF 解析 | 仅用摘要，`quote` 从摘要取 | 7.3（与 7.7 的矛盾按 7.7 处理） |
| L1 静态知识库（venue_ranking / ontology） | source_tier 用简化规则：有 DOI 且非 arXiv → B，arXiv → D，其余 C；概念标签由 LLM 自由生成 | 7.2 |
| L3 跨会话累积 | 论文表按 paper_id 全局去重存储，但不做周级刷新 | 7.4 |
| LLM-as-Judge 校准 | 评估只用程序化指标 | 9.6.2 |
| 多用户 / 鉴权 | 单机单用户 | — |

### 反目标（技术上跑通也算失败）

- 报告里出现一个不在检索结果集中的 `paper_id` 且未被标注为 `unverified`
- S2 返回空结果或超时时流程崩溃，而不是降级并告知用户
- HIL 点不是真阻塞——用户没有机会修改意图或选择方向
- 候选路径少于 3 条或多于 5 条
- 为了"更像设计"而加回 Web 层、Neo4j 或多源检索

## 3. 全局约束

- **语言与工具链**：Python 3.12（uv 管理，`.python-version` 固定；系统 3.15 对科学计算栈过新）；所有依赖通过 `uv` 安装，`pyproject.toml` 锁定
- **编排**：LangGraph `StateGraph`，人工中断用 `interrupt()`，持久化用 `PostgresSaver`
- **数据库**：PostgreSQL 16 或 17 + pgvector 扩展。本机当前**没有** PostgreSQL 和 Docker，前置条件是通过 Homebrew 安装（见第 8 节），需用户确认
- **LLM**：多 provider，通过 `.env` 切换。首期支持 GLM（`https://open.bigmodel.cn/api/paas/v4/`）和 DeepSeek（`https://api.deepseek.com`），都走 OpenAI 兼容协议，代码里不出现任何 provider 专有调用。`LLM_PROVIDER` 指定 chat 用哪家。API key 由用户在最后手动配置到 `.env`，多组 key 可并存，未激活的 provider 不要求填 key
- **Embedding**：可插拔，`EMBEDDING_PROVIDER=glm | siliconflow | local`。**DeepSeek 官方 API 没有 embedding 接口**（2026-11 确认），chat 用 DeepSeek 时 embedding 必须另选。三选项：`glm`：GLM `embedding-3`，需 GLM key；`siliconflow`：硅基流动托管的 `BAAI/bge-m3`，免费、OpenAI 兼容、同一平台也托管 DeepSeek chat，适合只有 DeepSeek 的场景；`local`：本地 sentence-transformers 模型（默认 `BAAI/bge-m3`，可换 `Qwen/Qwen3-Embedding-0.6B`），离线免费，需 `uv sync --extra local-embedding`。所有选项输出 1024 维，共用同一 pgvector 列，切换只改 env
- **测试分层**（评估者会在本机运行代码，`pytest` 必须零配置可跑）：
  - 单元测试：不依赖 PG、网络、API key。LLM、embedding、S2、仓储全部通过可注入的接口，测试用 fake / 内存实现
  - 集成测试（`@pytest.mark.integration`）：需要 PG，连不上时 **skip 而非 fail**，覆盖 `init_db()`、pgvector 写入查询、checkpoint 存取
  - 评估（`uv run eval`）：需要真实 key，独立于 pytest
- **Semantic Scholar**：无 key 时 1 req/s；支持可选 `S2_API_KEY` 环境变量；所有请求带重试与退避
- **代码与文档**：标识符英文，文档、注释、CLI 输出中文；遵循 `CLAUDE.md` 约定
- **分支**：在 feature 分支开发，不直接提交 main
- **开发顺序**：所有不需要真实 key 的任务（代码、测试、评估脚本的 fake 验证）先完成；真实运行与评估放在计划末尾，届时请用户配置 `GLM_API_KEY`

## 4. 系统形态

### 4.1 用户体验

```
$ uv run topic-agent
研究想法: 想做大模型的幻觉检测，但不想做需要标注数据的方法
领域 [NLP]: NLP
时间约束 [可选]: 1 semester
资源约束 [可选，逗号分隔]: 4xA100

── 意图卡 ──────────────────────────────
核心问题: ...
关键概念: hallucination, detection, LLM, unsupervised
检索式: "LLM hallucination detection unsupervised" | ...
范围: 2021-2026
歧义标记: "幻觉" 可能指事实性幻觉或忠实度幻觉
────────────────────────────────────────
[a] 确认  [e] 修改  [r] 重新输入: a

检索中... 143 篇 → 去重后 128 篇 → 聚成 6 个主题
全景: 主题列表 + 10 篇里程碑文献 + 时间线

生成候选路径... 4 条
── 对比矩阵 ────────────────────────────
         新颖度  可行性  影响  风险  加权
路径 A     4.2    3.8    4.0   2.1   4.0
路径 B     ...
────────────────────────────────────────
选择 1-2 条深入 [如 1,3]: 1

生成报告... → reports/abc123.md
```

### 4.2 包结构

```
src/topic_agent/
  config.py            # pydantic-settings，读 .env；Settings 含各 provider 的 key / base_url / model；
                       #   validate_for_run() 只校验激活的 provider
  db.py                # engine / sessionmaker / init_db()：建 extension、建表
  logging_setup.py     # configure_logging(session_id)、JsonlFormatter、log_event()、@logged_node
  models.py            # SQLAlchemy ORM：Session, Paper, SessionPaper, Evidence
  schemas.py           # Pydantic：UserInput, IntentCard, Paper, TopicCluster, LandscapeView,
                       #          CandidatePath, ComparisonMatrix, TopicReport, EvidenceItem
  llm.py               # get_chat_model(settings, tier) 按 provider 构造 ChatOpenAI（GLM / DeepSeek 同一构造方式）；
                       #   structured(model, prompt) -> 带重试的结构化输出，provider 无关
  embedding.py         # EmbeddingClient 协议：embed(texts) -> list[list[float]]；
                       #   OpenAICompatibleEmbedding（glm / siliconflow，只差 base_url + model）
                       #   LocalEmbedding（sentence-transformers）；get_embedding_client() 按 EMBEDDING_PROVIDER 选择
  verify.py            # 引用提取与验证
  report.py            # Jinja2 模板渲染 Markdown
  cli.py               # 入口；驱动 graph.stream，处理 __interrupt__；--resume 从 checkpoint 续跑；--list 列出会话
  sources/
    s2.py              # search(query, limit) / references(paper_ids) -> 带 S2 id 的 Paper
  analytics/
    cluster.py         # KMeans 聚类 + 调用 llm 为每簇命名
    pagerank.py        # NetworkX PageRank，输入边列表，输出 {paper_id: score}
  graph/
    state.py           # TopicAgentState
    nodes/
      intent.py  retrieval.py  landscape.py  path_gen.py  compare.py  report.py
      hil.py           # hil_intent / hil_path 两个中断节点
    build.py           # build_graph(checkpointer) -> CompiledGraph
eval/
  golden/case_001.json ~ case_003.json
  metrics.py           # 程序化指标
  run_eval.py          # 跑 golden cases（HIL 自动通过），输出指标表
  results/             # 评估输出，gitignore
reports/                # 生成的报告，gitignore
logs/                   # 每会话一个 JSONL 日志，gitignore
tests/                  # 镜像 src 结构
README.md               # 评估者从零跑通：前置条件、安装、配置 .env、跑 demo / 测试 / 评估
.env.example
docs/sample/            # 提交进仓库的样例产物：report_example.md、eval_results.md（真实运行后拷入）
```

### 4.3 数据库

一个库 `topic_agent`，`init_db()` 执行 `CREATE EXTENSION IF NOT EXISTS vector` 后建表。**向量就存在 `papers` 表的 `embedding` 列里**——pgvector 给 PostgreSQL 加了一种 `vector` 类型，向量和标题、摘要在同一行，没有独立的向量数据库。

#### 各表作用

**`sessions` — 一次完整运行。** 一行 = 用户从输入到报告的一次会话，是所有其他表的锚点。

| 列 | 含义 |
|---|---|
| `id` UUID PK | 同时作为 LangGraph 的 `thread_id` |
| `user_input` JSONB | 原始输入（研究想法、领域、约束） |
| `status` TEXT | `running` → `awaiting_intent` → `running` → `awaiting_path` → `running` → `completed` / `failed` |
| `report_path` TEXT | 报告文件路径，Stage 6 结束时写入 |
| `created_at` | — |

写入：CLI 创建，各节点更新 `status`。读取：`--list`、`--resume`。LangGraph checkpoint 表里也有 state，但没有"列出所有会话"的视图，这张表就是会话索引。

**`papers` — 一篇论文的元数据 + 向量，全局共享。** 一行 = 一个 S2 paperId，跨会话去重：十个会话都检索到同一篇只有一行，只 embed 一次。

| 列 | 含义 |
|---|---|
| `paper_id` TEXT PK | S2 paperId（40 位十六进制），也是 `[paper_id]` 引用的对象 |
| `title` / `abstract` | 来自 S2；embedding 的输入是 `title + "\n" + abstract` |
| `authors` JSONB | 作者列表 |
| `venue` / `year` / `doi` / `arxiv_id` / `citations` / `tldr` | S2 元数据 |
| `source_tier` CHAR(1) | 程序计算：有 DOI 无 arXiv → `B`，有 arXiv → `D`，其余 `C` |
| `embedding` vector(1024) | **向量在这里**。建 HNSW 索引（`vector_cosine_ops`） |
| `embedding_model` TEXT | 产生该向量的 provider 名，用于切换 provider 时判断是否重算 |
| `created_at` | — |

写入：`retrieval` 节点 upsert。读取：`landscape`（取全部向量做 KMeans）、`path_gen`（最近邻查新颖度）。所有向量操作都是对这一列的 SQL：

```sql
-- 新颖度：找和候选路径描述最接近的 5 篇
SELECT paper_id, title, 1 - (embedding <=> :path_vec) AS cosine_sim
FROM papers
WHERE paper_id IN (SELECT paper_id FROM session_papers WHERE session_id = :sid)
ORDER BY embedding <=> :path_vec
LIMIT 5;
```

**`session_papers` — 这篇论文出现在这次会话里。** 一行 = (会话, 论文)，存**只对这次会话成立**的事实。

| 列 | 含义 |
|---|---|
| `session_id` / `paper_id` | PK 两列 |
| `rank` INT | 过滤后的排序位置 |
| `cluster_id` INT | 在**这次**全景里属于哪个主题簇——同一篇论文在不同会话可能在不同簇，所以不能放 `papers` |
| `is_landmark` BOOL | 在**这次**引用子图里 PageRank 是否 Top-10 |

写入：`retrieval`（rank）、`landscape`（cluster_id、is_landmark）。读取：引用验证的合法集合 = `SELECT paper_id FROM session_papers WHERE session_id = :sid`。

**`citations` — 引用边。** 一行 = 论文 A 引用了论文 B，只保留两端都在检索集内的边。写入：`retrieval`。读取：`landscape` 取本次会话论文之间的边喂 NetworkX 跑 PageRank。

**`evidence` — 证据链，每条 (claim, paper_id) 一行。** 这是"证据可追溯"的落地表，LLM 在 Stage 4/5/6 输出的每个 `[paper_id]` 都是一行。

| 列 | 含义 |
|---|---|
| `stage` TEXT | `path_gen` / `compare` / `report` |
| `claim` TEXT | 包含该引用的那段 LLM 文本 |
| `paper_id` TEXT | 被引用的论文 |
| `quote` TEXT | 支撑该 claim 的摘要片段 |
| `source_tier` CHAR(1) | 从 `papers` 复制 |
| `confidence` REAL | 置信度 |
| `verification_status` TEXT | `verified`（在 `session_papers` 中）/ `unverified` |

写入：`verify.py`。读取：报告的证据表 = `SELECT * FROM evidence WHERE session_id = :sid ORDER BY stage`；评估的 `citation_verify_rate` / `hallucination_rate` 直接从这算。

**LangGraph checkpoint 表（`checkpoints` / `checkpoint_writes` / `checkpoint_blobs`）。** 由 `PostgresSaver.setup()` 创建——**建表是显式的一次性调用**，放在 `init_db()` 里和业务表一起执行，幂等；之后的读写全部由 LangGraph 自动完成，我们不写任何 checkpoint 代码。每个节点跑完和每次 `interrupt()` 时，LangGraph 把整个 `TopicAgentState` 序列化写一行进 `checkpoints`（只追加，带 `parent_checkpoint_id` 链），节点返回的部分更新写进 `checkpoint_writes`，非 JSON 值写进 `checkpoint_blobs`。三张表所有会话共用，按 `thread_id` 区分。`interrupt()` 和 `--resume` 都靠它。

**`init_db()` 在运行时调用**：每个入口（`topic-agent`、`--resume`、`--list`、`eval`、集成测试的 `conftest`）连上 PG 后、跑图之前调一次，不在 `uv sync` 或 import 时触发。首次运行建全部表，之后每次启动是幂等检查。评估者 clone 后无需单独的迁移步骤；MVP 不引入 Alembic。前提是 PG 服务在跑且 `topic_agent` / `topic_agent_test` 两个空库已 `createdb`（第 8 节）。

**state 里只放 ID 和结构化产出，不放论文正文。** `TopicAgentState.retrieved_paper_ids: list[str]` 而不是 150 个完整对象；节点要正文时通过 Repository 从 `papers` 表取。checkpoint 因此保持小体积，向量永远不进 state。

#### 一次会话走过哪些表

| 步骤 | 写入 | 读取 |
|---|---|---|
| CLI 启动 | `sessions`(running) | — |
| hil_intent 中断 | `sessions`(awaiting_intent) | — |
| retrieval | `papers`(upsert + embedding)、`session_papers`(rank)、`citations` | — |
| landscape | `session_papers`(cluster_id, is_landmark) | `papers`(向量)、`citations` |
| path_gen | `evidence`(path_gen) | `papers`(最近邻)、`session_papers` |
| compare | `evidence`(compare) | — |
| hil_path 中断 | `sessions`(awaiting_path) | — |
| report | `evidence`(report)、`sessions`(completed, report_path) | `evidence` |

#### 去重与向量一致性

`papers` 按 `paper_id` 全局去重（`INSERT ... ON CONFLICT DO UPDATE` 非 embedding 列）。**同一 `embedding` 列的所有向量必须来自同一模型**：upsert 时若已存在行的 `embedding_model` ≠ 当前 `EMBEDDING_PROVIDER`，对该行重新 embed 并更新，避免切换 provider 后混合向量空间导致聚类与最近邻静默失效。

#### Repository

节点通过 `Repository` 协议访问数据库（`save_papers`、`save_evidence`、`get_session_papers`、`get_embeddings`、`nearest_papers`、`get_citations` 等），生产用 `PostgresRepository`，单元测试用 `InMemoryRepository`。集成测试用独立库 `topic_agent_test`，`conftest.py` 在会话开始时 `init_db()`，每个用例 `TRUNCATE`。

### 4.4 图拓扑

```
START → intent → hil_intent → retrieval → landscape → path_gen → compare → hil_path → report → END
```

- `hil_intent` 节点调用 `interrupt({kind: "intent", card: IntentCard})`；resume 值为 `{decision: "approve" | "edit", card?: IntentCard}`。`edit` 时用用户修改后的 IntentCard 覆盖 state 并继续；用户在 CLI 输入 `r`（重新输入）时由 CLI 直接结束会话，不进图
- `hil_path` 节点调用 `interrupt({kind: "path", matrix: ComparisonMatrix, paths: [CandidatePath]})`；resume 值为 `{selected_ids: [str]}`，1-2 个
- HIL-2、HIL-4 不设中断
- `recursion_limit` 设 50
- LangGraph 的 `thread_id` = `sessions.id`；`--resume <session_id>` 即用同一 `thread_id` 重新 `stream(None, config)`，图从中断点继续

### 4.5 各 Stage 的 MVP 逻辑

**Stage 1 意图理解** → `IntentCard`。LLM 结构化输出。`source_tier` 不在此处。`search_queries` 只生成一个 S2 查询串（S2 单源）。

**Stage 2 检索** → `List[Paper]`。
1. `s2.search(query, limit=150)`，fields：`paperId,title,abstract,authors,year,venue,citationCount,externalIds,tldr`
2. 过滤：`abstract` 非空；年份在 IntentCard 的 `time_window` 内
3. `source_tier` 规则：有 DOI 且 `externalIds` 无 ArXiv → `B`；有 ArXiv → `D`；其余 `C`
4. `embedding.embed(title + "\n" + abstract)` → 写入 `papers.embedding`
5. 对 citationCount 前 80 篇调用 `s2.references()`（batch 接口一次拉取），只保留两端都在本次检索集内的边 → 写 `citations`
6. 空结果处理：若过滤后 < 10 篇，放宽年份限制重试一次；仍不足则 state 记录 `warning`，流程继续但报告标注"文献覆盖不足"

**Stage 3 全景梳理** → `LandscapeView`。
1. KMeans：`k` 从 5 到 12 中按轮廓系数选最优（样本 < 50 时 `k = max(3, n // 8)`）
2. 每簇取离质心最近的 5 篇标题 + 摘要，LLM 生成 `name` + `description`
3. PageRank（NetworkX）在 `citations` 子图上跑，Top-10 标 `is_landmark`
4. `timeline`：按年计数；`saturation_level`：近 2 年占比 > 50% 且总数 > 100 → `high`，< 25% → `low`，其余 `medium`
5. `cluster_id` 回写 `session_papers`

**Stage 4 路径生成** → `List[CandidatePath]`，3-5 条。
1. LLM 输入：IntentCard + LandscapeView + 每簇 5 篇代表文献（含 paper_id）
2. 要求每条路径的 `research_question`、`novelty.reasoning`、`feasibility.reasoning` 中至少一处出现 `[paper_id]`
3. `verify.py` 提取所有 `[paper_id]`，不在检索集内的标 `unverified`，按 4.6 写入 `evidence`（stage = "path_gen"）
4. `novelty.score`：路径描述 embedding 与检索集最近邻的余弦距离，线性映射到 1-5
5. `feasibility.score`：LLM 依据用户约束打 1-5
6. 路径间 embedding 余弦距离 < `PATH_MIN_DISTANCE`（配置项，初值 0.3）的，丢弃后出现的那条；不足 3 条则 LLM 再生成一批补齐，仍不足则带 `warning` 继续流程（`path_count` 指标记为不达标）；超过 5 条按 `novelty.score` 取前 5

**Stage 5 比较收敛** → `ComparisonMatrix`。
1. 四维权重：novelty 0.25、feasibility 0.30、impact 0.25、risk 0.20；加权分 = 0.25·novelty + 0.30·feasibility + 0.25·impact + 0.20·(6 − risk)
2. LLM 对每条路径打 impact 和 risk（1-5），novelty / feasibility 沿用 Stage 4
3. 冲突检测：LLM 扫描所有路径的 reasoning，输出互相矛盾的 `path_id` 对及说明；说明中的引用同样验证并写入 `evidence`（stage = "compare"）
4. 每个维度的 `confidence`：LLM 自评 0-1；有冲突的路径该维度乘 0.7
5. 加权排序，但 CLI 和报告展示完整矩阵

**Stage 6 报告** → `TopicReport` + Markdown 文件。
1. 对选中路径 LLM 深度分析：`recommended_approach`、`timeline_estimate`、`potential_risks`，每个 claim 带 `[paper_id]`
2. `verify.py` 验证；每条 claim 生成一条 `evidence` 记录写入数据库
3. Jinja2 渲染 `reports/<session_id>.md`，含：意图卡、全景摘要、对比矩阵、选中路径分析、证据表（标注 verified / unverified）、局限性声明

### 4.6 引用验证规则（`verify.py`）

- 提取正则：`\[([A-Za-z0-9]+)\]`（S2 paperId 为 40 位十六进制）
- 有效当且仅当 `paper_id` 在本次会话的 `session_papers` 中
- 无效引用：claim 保留，`verification_status = "unverified"`，计入 `hallucination_rate`
- **claim 的定义**：LLM 输出中一个自由文本字段（如一条路径的 `novelty.reasoning`、一处冲突说明、报告中一段分析）即为一条 claim
- **验证范围**：Stage 4 全部候选路径（含未选中）、Stage 5 冲突说明、Stage 6 报告分析的所有 claim 都经验证；每个 (claim, paper_id) 对写一行 `evidence`，`stage` 列标记来源
- **指标分母**：`citation_verify_rate` = verified 的 (claim, paper_id) 对 / 全部对；`hallucination_rate` = 含 ≥ 1 个无效引用的 claim / 全部 claim；`claim_support_rate` = 含 ≥ 1 个 `[paper_id]` 的 claim / 全部 claim
- `verify` 的返回：`VerificationResult(valid: list[str], invalid: list[str], rate: float)`

### 4.7 LLM 结构化输出（`llm.py`）

- provider 注册表：`PROVIDERS = {"glm": ..., "deepseek": ..., "siliconflow": ...}`，每个 `ProviderSpec` 含 `base_url_default` 与 `supports_embedding`（glm / siliconflow 为 True，deepseek 为 False）；新增一家 = 加一个 spec + 三组 env 字段
- `get_chat_model(settings, tier="default" | "reasoning") -> ChatOpenAI`：`default` 档读 `LLM_PROVIDER` 对应的 key / base_url / model；`reasoning` 档读 `LLM_REASONING_PROVIDER`（留空则同 `LLM_PROVIDER`）和 `LLM_MODEL_REASONING`（留空则同该 provider 的默认 model）。Stage 4、5 用 `reasoning` 档，其余用 `default` 档（对应 DESIGN.md 4.2 的分级调用）
- 构造方式统一：`ChatOpenAI(model=..., api_key=..., base_url=..., temperature=0.2)`，GLM 与 DeepSeek 无差别
- `structured(model_cls, system, user) -> model_cls`：调用 `with_structured_output(model_cls)`（默认 function calling；若所选 GLM 型号不支持，退化为 `method="json_mode"`，由 builder 在首次联调时确定），`ValidationError` 时把错误信息拼回提示重试，最多 3 次；3 次仍失败抛 `LLMSchemaError`
- 每次调用记录 `{node, model, attempts, latency_ms}` 到 state 的 `llm_calls` 列表，供评估统计
- `embedding.py`：`EmbeddingClient` 协议 `embed(texts: list[str]) -> list[list[float]]`，批量 ≤ 64 条。`OpenAICompatibleEmbedding`：走 OpenAI embeddings 接口，`glm` 用 `embedding-3` 并传 `dimensions=1024`，`siliconflow` 用 `BAAI/bge-m3` 不传 `dimensions`（该平台非 Qwen 系模型不支持此参数，原生 1024）。`LocalEmbedding`：sentence-transformers 加载 `LOCAL_EMBEDDING_MODEL`，首次运行下载；依赖放在 pyproject 的 `local-embedding` optional group，默认不装。首次调用后校验输出维度 = 1024，否则报错

### 4.8 评估

**Golden case**（`eval/golden/case_00N.json`）：

```json
{
  "case_id": "case_001",
  "query": {"research_idea": "...", "field": "NLP", "constraints": {"timeframe": "1 semester"}},
  "expected": {
    "key_concepts": ["hallucination", "detection"],
    "must_find_title_keywords": ["SelfCheckGrep", "Chain-of-Verification"],
    "min_papers": 30
  },
  "difficulty": "medium"
}
```

3 个 case：幻觉检测（NLP，medium）、参数高效微调（LLM，easy）、多模态对齐中的幻觉（CV/NLP 交叉，hard）。

**指标**（`eval/metrics.py`，全部程序化）：

| 指标 | 定义 | 目标 |
|---|---|---|
| `task_completion` | 6 个 Stage 全部完成 | 3/3 |
| `json_schema_validity` | 一次通过 schema 的 LLM 调用 / 总调用 | ≥ 0.90 |
| `citation_verify_rate` | verified paper_id / 全部 paper_id | ≥ 0.95 |
| `claim_support_rate` | 含 ≥ 1 个 `[paper_id]` 的 claim / 总 claim | ≥ 0.90 |
| `hallucination_rate` | unverified / 全部 paper_id | ≤ 0.05 |
| `path_count` | 候选路径数 | 3-5 |
| `path_distinctness` | 路径间最小 embedding 余弦距离 | ≥ 0.3 |
| `concept_coverage` | expected.key_concepts 被 IntentCard 覆盖的比例 | ≥ 0.75 |
| `must_find_recall` | must_find_title_keywords 在检索集中命中的比例 | ≥ 0.5 |
| `latency_total_s` / `llm_call_count` | 记录 | — |
| `provider` / `model` | 本次评估用的 chat provider 与模型名（含 reasoning 档），写进结果表表头 | — |

`run_eval.py`：对每个 case 构造 `UserInput`，HIL 自动 `approve` / 选加权分最高的 1 条，跑完收集 state 与数据库，计算指标，打印表格并写 `eval/results/<timestamp>.md`。单个 case 失败不中断，记为 `task_completion=0` 并继续。门槛不达标时如实写入结果表，不改门槛；允许一轮 prompt 调整后重跑，两次结果都保留。

### 4.9 配置（`.env.example`）

```
# 激活哪个 provider（glm | deepseek | siliconflow）
LLM_PROVIDER=deepseek
EMBEDDING_PROVIDER=siliconflow  # glm | siliconflow | local（本地，需 uv sync --extra local-embedding）
                                # DeepSeek 无 embedding 接口，不在此列

# DeepSeek（仅 chat，无 embedding）
DEEPSEEK_API_KEY=
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=                 # 如 deepseek-chat

# 硅基流动（托管 BAAI/bge-m3 免费；也托管 DeepSeek chat，可只用这一把 key）
SILICONFLOW_API_KEY=
SILICONFLOW_BASE_URL=https://api.siliconflow.cn/v1
SILICONFLOW_MODEL=              # 可选，chat 走硅基流动托管模型时填
SILICONFLOW_EMBEDDING_MODEL=BAAI/bge-m3

# 智谱 GLM
GLM_API_KEY=
GLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4/
GLM_MODEL=
GLM_EMBEDDING_MODEL=embedding-3
GLM_EMBEDDING_DIMENSIONS=1024

# 本地 embedding（EMBEDDING_PROVIDER=local 时生效）
LOCAL_EMBEDDING_MODEL=BAAI/bge-m3   # 或 Qwen/Qwen3-Embedding-0.6B

# 可选：Stage 4/5 深度推理档。provider 留空则同 LLM_PROVIDER，model 留空则同该 provider 默认 model
LLM_REASONING_PROVIDER=
LLM_MODEL_REASONING=

DATABASE_URL=postgresql://localhost:5432/topic_agent
TEST_DATABASE_URL=postgresql://localhost:5432/topic_agent_test
S2_API_KEY=                     # 可选
LOG_LEVEL=INFO                  # DEBUG 额外记录完整 prompt 与响应（截断）
```

`config.py` 在 import 时不报错，在 `Settings.validate_for_run()` 时校验**激活的** provider 的必填项（key、model），以便 pytest 在无 key 环境下能 import 模块。config 对 glm / deepseek / siliconflow 三组解析、reasoning 档回退规则、embedding provider 选择都有单元测试。

### 4.10 日志

每个会话一个日志文件 `logs/<session_id>.jsonl`，一行一个 JSON 事件；控制台同时输出人类可读的摘要。评估者能通过日志完整复现每一步：进了哪个节点、state 变了什么、LLM 调了几次、每次多久、引用验证结果如何。

**实现**：`logging_setup.py` 的 `configure_logging(session_id)` 配置根 logger，`JsonlFormatter` 写文件，`log_event(event, **fields)` 写一条事件。`@logged_node` 装饰器包裹所有节点，自动记录进入/离开和 state 变化，节点内部不写样板。LLM / embedding / S2 的调用日志写在 `llm.py` / `embedding.py` / `s2.py` 的封装函数里，节点无需关心。

**事件类型**（`event` 字段）：

| event | 触发点 | 关键字段 |
|---|---|---|
| `session_start` | CLI 创建会话 | `session_id`, `user_input`, `llm_provider`, `llm_model`, `reasoning_model`, `embedding_provider`, `embedding_model` |
| `session_end` | 流程结束 | `session_id`, `status`, `duration_s`, `report_path` |
| `node_start` | 节点进入 | `node` |
| `node_end` | 节点离开 | `node`, `duration_ms`, `state_changes: {key: {from, to}}`（列表只记长度，如 `retrieved_paper_ids: 0 → 143`） |
| `hil_interrupt` | `interrupt()` 触发 | `point`（intent / path）, `payload_summary` |
| `hil_resume` | 用户恢复 | `point`, `decision`, `waited_s` |
| `llm_call_start` | `structured()` 发请求前 | `node`, `tier`, `provider`, `model`, `attempt`, `prompt_chars` |
| `llm_call_end` | 收到响应并解析后 | `node`, `attempt`, `latency_ms`, `usage: {prompt_tokens, completion_tokens}`, `schema_valid`, `error`（如有） |
| `embedding_call` | `embed()` | `provider`, `model`, `n_texts`, `dim`, `latency_ms` |
| `s2_call` | S2 请求 | `endpoint`, `query`, `n_results`, `latency_ms`, `rate_limit_wait_s` |
| `verify` | `verify.py` | `stage`, `n_citations`, `n_valid`, `n_invalid`, `invalid_ids` |
| `eval_case_start` / `eval_case_end` | 评估脚本 | `case_id`, `metrics` |

**通用字段**：`ts`（ISO 8601）、`level`、`session_id`、`event`。

**规则**：
- `LOG_LEVEL=INFO` 默认；`DEBUG` 额外记录完整 prompt 与响应（各截断至 4000 字符）
- 永远不记录 API key，只记 provider 名和模型名
- `state_changes` 记录每个被节点修改的 key 的前后摘要，不记全文
- `llm_calls` 列表（4.7）仍保留在 state 里供评估脚本统计；日志是更详细的记录，两者不冲突
- `logs/` 加入 `.gitignore`；评估结果表引用对应日志文件名
- 单元测试断言关键事件被写入（用 `caplog` 或内存 handler），不依赖文件系统

## 5. 留给 builder 决定

- 各节点 prompt 的具体措辞
- CLI 的具体排版与颜色
- 重试与退避的具体参数（S2 请求、LLM 请求）
- KMeans 的随机种子与轮廓系数计算细节
- Jinja2 模板的具体格式
- 控制台日志的具体排版

## 6. 与 DESIGN.md 的偏差记录

| 项 | DESIGN.md | MVP | 理由 |
|---|---|---|---|
| 聚类 | UMAP + HDBSCAN | KMeans | 依赖重，Python 3.12 下 numba 兼容性风险 |
| Embedding | BGE-M3 本地 | 托管 BGE-M3（硅基流动，免费）或本地 BGE-M3 / Qwen3-Embedding，可切换 | 评估者跑代码零模型下载；保留原选型可切换 |
| 引用图 | Neo4j + GDS | NetworkX 内存 | 200-500 节点无需图数据库 |
| 交互层 | FastAPI + Next.js | CLI | 评估要求"可选 demo"，CLI 足够展示流程 |
| 异步 | Celery + Redis | 同步单进程 | MVP 无长任务并发需求 |
| 来源分级 | 四级 + venue_ranking | 三级简化规则 | L1 静态库需人工维护，MVP 不建 |
| 概念标签 | L1 ontology 约束 | LLM 自由生成 | 同上 |
| 全文 | 7.3 提到全文 chunk | 仅摘要 | 7.7 已禁止存全文，按 7.7 执行 |
| LLM | Claude 多档（Sonnet / Opus / Haiku） | GLM / DeepSeek 多 provider 可切换 + default / reasoning 两档 | 用户只有国内模型 key；架构保持模型无关 |

## 7. 验收（MVP 的 Definition of Done）

1. `uv run topic-agent` 在本机 PG 上跑通一个真实查询，产出 `reports/<id>.md`（需激活 provider 的真实 API key，在用户配置 key 后执行）
2. HIL-1 选 `e` 修改意图后流程按修改后的检索式执行；HIL-3 选 2 条路径后报告包含 2 条
3. `uv run pytest` 全绿，且不依赖网络与 API key
4. `uv run eval` 在 3 个 golden case 上输出指标表，`citation_verify_rate ≥ 0.95`、`hallucination_rate ≤ 0.05`（同上，需真实 key）
5. 报告中每个 `unverified` 引用都有可见标注
6. 评估者按 `README.md` 在装有 brew PG 的机器上从 clone 到跑通 demo、`pytest`、`eval` 三步无需额外说明；`pytest` 在无 PG 时集成测试 skip 而非报错
7. `docs/sample/` 含一份真实运行的报告样例和评估指标表
8. 每个会话产生 `logs/<session_id>.jsonl`，包含 4.10 全部事件类型；`node_end` 有 state 变化摘要，`llm_call_end` 有 token 用量与延迟

## 8. 前置条件（已确认，2026-11-26）

本机没有 PostgreSQL 和 pgvector，用户已同意通过 Homebrew 安装。Task 1 执行：

```bash
brew install postgresql@17 pgvector
brew services start postgresql@17
createdb topic_agent && createdb topic_agent_test
```

这会在用户机器上安装并启动一个 PostgreSQL 服务。`README.md` 记录同样的步骤供评估者复现。
