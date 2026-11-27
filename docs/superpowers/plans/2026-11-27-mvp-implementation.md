# 科研选题 Agent MVP 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 `docs/superpowers/specs/2026-11-26-mvp-design.md` 定义的 MVP：LangGraph 6-Stage 科研选题 CLI Agent，单 PostgreSQL（pgvector + checkpoint），证据链程序化验证，3 个 golden case 评估。

**Architecture:** CLI 驱动 LangGraph StateGraph（intent → hil_intent → retrieval → landscape → path_gen → compare → hil_path → report），两个 `interrupt()` 人工确认点，PostgresSaver checkpoint 持久化。所有外部依赖（LLM / embedding / S2 / 数据库）经可注入接口，单元测试全 fake，真实 key 只在 Task 21 使用。

**Tech Stack:** Python 3.12（uv）、LangGraph、langgraph-checkpoint-postgres、langchain-openai（OpenAI 兼容协议调 GLM / DeepSeek / 硅基流动）、SQLAlchemy 2 + psycopg3 + pgvector、scikit-learn（KMeans）、NetworkX（PageRank）、Jinja2、httpx、pytest。

**Spec:** `docs/superpowers/specs/2026-11-26-mvp-design.md`（本计划的每个决策从该文档推导；执行者需同时读 spec 与本计划）

## Global Constraints

（摘自 spec §3，对全部任务生效）

- Python 3.12，`.python-version` 固定；所有依赖经 `uv` 管理，`pyproject.toml` 锁定
- LangGraph `StateGraph` + `interrupt()` + `PostgresSaver`；`recursion_limit = 50`
- PostgreSQL 17 + pgvector（brew），库 `topic_agent` / `topic_agent_test`（spec §8，用户已确认安装）
- LLM 代码零 provider 专有调用：GLM / DeepSeek / 硅基流动统一 `ChatOpenAI(model, api_key, base_url, temperature=0.2)`
- 单元测试不依赖 PG、网络、API key；集成测试（`@pytest.mark.integration`）连不上 PG 时 **skip 而非 fail**；`uv run eval` 独立于 pytest
- S2 无 key 时 1 req/s；所有请求带重试与退避
- 标识符英文；文档、注释、CLI 输出中文
- feature 分支开发（`feat/mvp-implementation`，自 dev 切出），不直接提交 main
- 不需要真实 key 的任务先完成；真实运行与评估只在 Task 21（届时用户配置 `GLM_API_KEY`）
- state 只放 ID 与结构化产出（`retrieved_paper_ids`），论文正文与向量永不进 state
- 反目标：不加 Web 层、Neo4j、多源检索；报告出现未标注的无效 `paper_id` 即失败

## Review Focus

（spec 隐含但容易被任务测试遗漏的输入类别；每条的测试已钉到 owning task）

1. **S2 论文 `year` 为 None / `abstract` 为空**（S2 元数据常见缺失）——过滤逻辑不得崩，`year=None` 视为窗口外被剔除。测试：Task 13 `test_retrieval_filters_null_year_and_empty_abstract`。
2. **降级后论文极少（0~8 篇）**——聚类 k 需钳制到样本数，n=0 时产出空全景 + warning，不抛异常。测试：Task 11 `test_kmeans_labels_tiny_inputs`、Task 14 `test_landscape_with_almost_no_papers`。
3. **`structured()` 三次 schema 失败**——`LLMSchemaError` 上抛后由 CLI 捕获，会话置 `failed` 并写日志，进程非零码退出，不是未捕获 traceback。测试：Task 19 `test_main_marks_session_failed_on_llm_schema_error`。
4. **HIL-3 输入 0 条 / >2 条 / 越界序号**——CLI 拒绝并重新提示，不放行不崩溃。测试：Task 19 `test_select_paths_rejects_invalid_input`。
5. **切换 embedding provider 后存量向量重算**——`embedding_model` 不匹配的行必须被重新 embed 覆盖，避免混合向量空间。测试：Task 6 `test_save_papers_returns_stale_embedding_ids`、Task 13 `test_retrieval_reembeds_stale_rows`。

---

### Task 1: 项目脚手架与 PostgreSQL 前置条件

**Files:**
- Create: `pyproject.toml`、`.python-version`、`.env.example`、`tests/conftest.py`、`tests/test_smoke.py`
- Modify: `.gitignore`

**Interfaces:**
- Produces: 可 import 的空包 `src/topic_agent/__init__.py`；`uv run pytest` 可执行；marker `integration` 已注册；`tests/conftest.py` 提供 `pg_dsn` fixture（尝试连接 `TEST_DATABASE_URL`，失败则 `pytest.skip`）

- [ ] **Step 1: 切出 feature 分支**

```bash
git checkout -b feat/mvp-implementation
```

- [ ] **Step 2: 安装 PostgreSQL（spec §8，用户已确认）**

```bash
brew install postgresql@17 pgvector
brew services start postgresql@17
createdb topic_agent && createdb topic_agent_test
```

预期：`psql -d topic_agent -c "SELECT 1"` 成功。

- [ ] **Step 3: 写 `pyproject.toml` 与项目文件**

`pyproject.toml` 要点：`requires-python = ">=3.12,<3.13"`；deps：`langgraph`、`langgraph-checkpoint-postgres`、`langchain-openai`、`sqlalchemy>=2.0`、`psycopg[binary]>=3.1`、`pgvector`、`pydantic>=2.7`、`pydantic-settings>=2.2`、`scikit-learn`、`networkx`、`jinja2`、`httpx`、`numpy`；dev group：`pytest>=8`；optional group `local-embedding`：`sentence-transformers`；`[project.scripts] topic-agent = "topic_agent.cli:main"`；`[tool.pytest.ini_options] markers = ["integration: 需要 PostgreSQL 的集成测试"]`。`.python-version` 写 `3.12`。`.env.example` 照抄 spec §4.9 全文。`.gitignore` 追加：`.env`、`reports/`、`logs/`、`eval/results/`、`.venv/`、`__pycache__/`。`src/topic_agent/__init__.py` 空文件。

- [ ] **Step 4: 写冒烟测试与 conftest**

`tests/test_smoke.py`：

```python
def test_package_imports():
    import topic_agent
```

`tests/conftest.py`：注册 `pg_dsn` fixture——读 `TEST_DATABASE_URL`（默认 `postgresql://localhost:5432/topic_agent_test`），用 `psycopg.connect(connect_timeout=2)` 探测，失败则 `pytest.skip("PostgreSQL 不可用，跳过集成测试")`。

- [ ] **Step 5: `uv sync` 并运行测试**

Run: `uv sync --dev && uv run pytest -v`
Expected: 1 passed（PG 未装时也 passed，无集成用例）

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml .python-version .env.example .gitignore tests/ src/ uv.lock
git commit -m "chore: 项目脚手架、依赖与 PG 前置条件"
```

---

### Task 2: config.py — Settings 与 provider 校验

**Files:**
- Create: `src/topic_agent/config.py`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces:
  - `KNOWN_CHAT_PROVIDERS = {"glm", "deepseek", "siliconflow"}`、`KNOWN_EMBEDDING_PROVIDERS = {"glm", "siliconflow", "local"}`
  - `class Settings(BaseSettings)`：字段与 `.env.example`（spec §4.9）一一对应——`llm_provider: str = "deepseek"`、`embedding_provider: str = "siliconflow"`、各 provider 的 `*_api_key / *_base_url / *_model / *_embedding_model`、`glm_embedding_dimensions: int = 1024`、`local_embedding_model: str = "BAAI/bge-m3"`、`llm_reasoning_provider: str = ""`、`llm_model_reasoning: str = ""`、`database_url`、`test_database_url`、`s2_api_key: str = ""`、`log_level: str = "INFO"`、`path_min_distance: float = 0.3`；`model_config = SettingsConfigDict(env_file=".env", extra="ignore")`
  - `class SettingsError(Exception)`
  - `Settings.validate_for_run() -> None`：激活的 chat provider 必须有 key；model 为空且该 provider 无默认 model 时报错（siliconflow 的 chat model 必填）；`EMBEDDING_PROVIDER=glm` 需 GLM key、`=siliconflow` 需 SF key、`=local` 无 key 要求；`llm_provider` / `embedding_provider` 不在已知集合时报错
  - `load_settings(env_file: str | None = None) -> Settings`

- [ ] **Step 1: 写失败测试**

```python
def test_load_settings_reads_env_file(tmp_path): ...
def test_validate_for_run_ok_with_deepseek(tmp_path): ...          # 只填 DEEPSEEK_API_KEY + EMBEDDING_PROVIDER=siliconflow + SILICONFLOW_API_KEY，不报错
def test_validate_for_run_missing_key_raises(tmp_path): ...        # LLM_PROVIDER=deepseek 无 key -> SettingsError
def test_validate_for_run_siliconflow_needs_chat_model(tmp_path): ...
def test_validate_for_run_local_embedding_needs_no_key(tmp_path): ...
def test_validate_for_run_unknown_provider_raises(tmp_path): ...
```

- [ ] **Step 2: 运行确认失败**

Run: `uv run pytest tests/test_config.py -v` — Expected: FAIL（`ModuleNotFoundError: topic_agent.config`）

- [ ] **Step 3: 实现 `config.py`**

pydantic-settings；不读 env 的字段给默认值，保证 import 时不报错（spec §4.9）。

- [ ] **Step 4: 运行确认通过**

Run: `uv run pytest tests/test_config.py -v` — Expected: PASS

- [ ] **Step 5: Commit** — `git commit -m "feat: Settings 多 provider 配置与激活校验"`

---

### Task 3: schemas.py — 全部 Pydantic 模型

**Files:**
- Create: `src/topic_agent/schemas.py`
- Test: `tests/test_schemas.py`

**Interfaces:**
- Produces（后续所有任务依赖的类型词汇）:
  - `UserInput(research_idea: str, field: str, timeframe: str | None = None, resource_constraints: list[str] = [])`
  - `IntentCard(core_question: str, key_concepts: list[str], search_query: str, time_window: tuple[int, int], ambiguity_notes: list[str] = [])`
  - `Paper(paper_id: str, title: str, abstract: str, authors: list[str] = [], venue: str | None, year: int | None, doi: str | None, arxiv_id: str | None, citations: int = 0, tldr: str | None, source_tier: str)`（所有可选字段默认 None）
  - `ScoredDimension(score: float = Field(ge=1, le=5), reasoning: str)`
  - `TopicCluster(cluster_id: int, name: str, description: str, paper_ids: list[str], representative_ids: list[str])`
  - `LandscapeView(clusters: list[TopicCluster], landmarks: list[str], timeline: dict[int, int], saturation_level: Literal["low", "medium", "high"], warnings: list[str] = [])`
  - `CandidatePath(path_id: str, title: str, research_question: str, approach: str, novelty: ScoredDimension, feasibility: ScoredDimension)`
  - `Conflict(path_a: str, path_b: str, description: str)`
  - `PathComparison(path_id: str, impact: ScoredDimension, risk: ScoredDimension, weighted_score: float, dimension_confidence: dict[str, float])`
  - `ComparisonMatrix(paths: list[PathComparison], conflicts: list[Conflict] = [], weights: dict[str, float])`
  - `DeepAnalysis(path_id: str, recommended_approach: str, timeline_estimate: str, potential_risks: list[str])`
  - `EvidenceItem(stage: str, claim: str, paper_id: str, quote: str | None, source_tier: str | None, confidence: float | None, verification_status: Literal["verified", "unverified"])`
  - `TopicReport(session_id: str, user_input: UserInput, intent_card: IntentCard, landscape: LandscapeView, matrix: ComparisonMatrix, paths: list[CandidatePath], analyses: list[DeepAnalysis], evidence: list[EvidenceItem], warnings: list[str] = [])`
  - `LlmCallRecord(node: str, tier: str, provider: str, model: str, attempts: int, latency_ms: int, prompt_tokens: int | None = None, completion_tokens: int | None = None, schema_valid_first_try: bool = True)`
  - `SessionInfo(session_id: str, status: str, user_input: UserInput | None, report_path: str | None, created_at: str)`

- [ ] **Step 1: 写失败测试** — 覆盖：`ScoredDimension` 拒绝 score=0 和 score=5.5；`Paper` 最小字段构造；`time_window` 接受 `[2021, 2026]` JSON 输入转 tuple；`EvidenceItem` 拒绝 `verification_status="maybe"`；`LandscapeView.timeline` 接受 `{"2024": 10}` 转 int key

```python
import pytest
from pydantic import ValidationError
from topic_agent.schemas import ScoredDimension, Paper, EvidenceItem

def test_scored_dimension_bounds():
    with pytest.raises(ValidationError): ScoredDimension(score=0, reasoning="x")
    with pytest.raises(ValidationError): ScoredDimension(score=5.5, reasoning="x")
    assert ScoredDimension(score=1, reasoning="x").score == 1

def test_paper_minimal(): ...
def test_evidence_status_literal(): ...
```

- [ ] **Step 2: 运行确认失败** — `uv run pytest tests/test_schemas.py -v` — FAIL
- [ ] **Step 3: 实现 `schemas.py`**（纯声明，无逻辑）
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: 全局 Pydantic schemas"`

---

### Task 4: logging_setup.py — JSONL 结构化日志

**Files:**
- Create: `src/topic_agent/logging_setup.py`
- Test: `tests/test_logging_setup.py`

**Interfaces:**
- Consumes: Task 3 无依赖（独立）
- Produces:
  - `configure_logging(session_id: str, log_dir: Path = Path("logs"), level: str | None = None) -> None`（幂等：重复调用先清理已有 handler；`level=None` 读已配置的 `LOG_LEVEL`）
  - `log_event(event: str, **fields: Any) -> None`（写一条含 `ts` ISO8601、`level`、`session_id`、`event` 的记录到当前会话的 JSONL logger）
  - `class JsonlFormatter(logging.Formatter)`（`json.dumps` 一行，`default=str` 兜底）
  - `logged_node(func: Callable[[dict], dict]) -> Callable[[dict], dict]`（装饰器：`node_start` / `node_end`；`node_end` 带 `duration_ms` 和 `state_changes: {key: {from, to}}`，list 值只记长度如 `retrieved_paper_ids: 0 → 143`，其余记摘要）
  - 永不记录 api key：`log_event` 前过滤字段名含 `key` / `api_key` 的 kwarg

- [ ] **Step 1: 写失败测试**（用内存 handler，不依赖文件系统）

```python
def test_log_event_writes_json_line_with_common_fields(memory_handler): ...
def test_logged_node_records_state_change_summary(memory_handler): ...
def test_logged_node_lists_recorded_as_length(memory_handler): ...
def test_log_event_never_writes_api_keys(memory_handler): ...   # log_event("llm_call", api_key="sk-secret") 输出不含 "sk-secret"
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `logging_setup.py`**（模块级 current session_id 由 `configure_logging` 设置；测试用 `logging.Handler` 挂到同一 logger）
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: JSONL 结构化日志与 logged_node 装饰器"`

---

### Task 5: db.py + models.py — ORM 与 init_db

**Files:**
- Create: `src/topic_agent/db.py`、`src/topic_agent/models.py`
- Test: `tests/integration/test_db.py`

**Interfaces:**
- Consumes: 无
- Produces:
  - `get_engine(database_url: str) -> Engine`（`create_engine`，psycopg3 方言）
  - `init_db(database_url: str) -> None`——执行 `CREATE EXTENSION IF NOT EXISTS vector`、`Base.metadata.create_all`、`CREATE INDEX IF NOT EXISTS idx_papers_embedding_hnsw ON papers USING hnsw (embedding vector_cosine_ops)`、`PostgresSaver.from_conn_string(database_url)` 上下文内 `.setup()`；幂等
  - ORM 表（表名即 spec §4.3）：`sessions(id UUID PK, user_input JSONB, status TEXT default 'running', report_path TEXT NULL, created_at server_default now())`；`papers(paper_id TEXT PK, title, abstract, authors JSONB, venue, year INT NULL, doi, arxiv_id, citations INT, tldr, source_tier CHAR(1), embedding Vector(1024) NULL, embedding_model TEXT NULL, created_at)`；`session_papers(session_id, paper_id, rank INT, cluster_id INT NULL, is_landmark BOOL default false, PK(session_id, paper_id))`；`citations(citing_id, cited_id, PK(citing_id, cited_id))`；`evidence(id BIGSERIAL PK, session_id, stage TEXT, claim TEXT, paper_id TEXT, quote TEXT NULL, source_tier CHAR(1) NULL, confidence REAL NULL, verification_status TEXT)`

- [ ] **Step 1: 写失败集成测试**（`@pytest.mark.integration`，用 `pg_dsn` fixture）

```python
@pytest.mark.integration
def test_init_db_idempotent(pg_dsn): ...          # 调两次不报错；查 information_schema 断言 5 张业务表 + checkpoints 存在
@pytest.mark.integration
def test_papers_vector_column_accepts_1024(pg_dsn): ...  # 插入一行带 1024 维向量并读回
```

- [ ] **Step 2: 运行确认失败** — `uv run pytest tests/integration/test_db.py -v` — FAIL（ModuleNotFoundError）；PG 未启动时 SKIP（验证 skip 路径）
- [ ] **Step 3: 实现 `db.py` / `models.py`**（`Vector` 来自 `pgvector.sqlalchemy`）
- [ ] **Step 4: 运行确认通过** — `uv run pytest tests/integration/test_db.py -v` — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: ORM 模型与幂等 init_db（含 pgvector 与 checkpoint 表）"`

---

### Task 6: repository.py — Repository 协议 + InMemory + Postgres

**Files:**
- Create: `src/topic_agent/repository.py`（Protocol、`InMemoryRepository`、`PostgresRepository` 同文件，单一职责：持久化接口）
- Test: `tests/test_repository_memory.py`、`tests/integration/test_repository_pg.py`

**Interfaces:**
- Consumes: Task 3 `schemas`、Task 5 `models`
- Produces:
  ```python
  class Repository(Protocol):
      def create_session(self, session_id: str, user_input: UserInput) -> None: ...
      def set_session_status(self, session_id: str, status: str, report_path: str | None = None) -> None: ...
      def get_session(self, session_id: str) -> SessionInfo | None: ...
      def list_sessions(self) -> list[SessionInfo]: ...
      def save_papers(self, session_id: str, papers: list[Paper], ranks: dict[str, int], embedding_model: str) -> list[str]: ...
      def save_embeddings(self, embeddings: dict[str, list[float]], embedding_model: str) -> None: ...
      def save_citations(self, edges: list[tuple[str, str]]) -> None: ...
      def get_papers(self, paper_ids: list[str]) -> list[Paper]: ...
      def get_session_paper_ids(self, session_id: str) -> list[str]: ...
      def get_embeddings(self, session_id: str) -> list[tuple[str, list[float]]]: ...
      def nearest_papers(self, session_id: str, query_vec: list[float], k: int = 5) -> list[tuple[str, float, str]]: ...
      def get_citations(self, session_id: str) -> list[tuple[str, str]]: ...
      def set_clusters(self, session_id: str, cluster_ids: dict[str, int], landmarks: list[str]) -> None: ...
      def save_evidence(self, session_id: str, items: list[EvidenceItem]) -> None: ...
      def get_evidence(self, session_id: str) -> list[EvidenceItem]: ...
  ```
  语义要点（两个实现一致，测试用同一套参数化用例）：`save_papers` upsert `papers` 非向量列 + 写 `session_papers(rank)`，**返回仍缺当前 `embedding_model` 向量的 paper_id 列表**（新行或 `embedding_model` 不匹配的旧行，见 Review Focus 5）；`nearest_papers` 返回 `(paper_id, cosine_sim, title)` 按距离升序；`save_citations` 全局去重；`PostgresRepository.__init__(self, engine: Engine)`，每方法一个 session 作用域。

- [ ] **Step 1: 写失败测试（内存实现，参数化用例留给 PG 复用）**

```python
def test_save_papers_and_get_roundtrip(): ...
def test_save_papers_returns_stale_embedding_ids(): ...   # 先以 model="glm" 存，再以 model="siliconflow" save 同一批 -> 全部 id 返回；补 save_embeddings 后再 save -> 返回空
def test_nearest_papers_orders_by_cosine(): ...           # 3 个 2 维近似向量（补齐到 1024 维由 fixture 处理），断言排序与相似度
def test_papers_upsert_dedupes_across_sessions(): ...     # 两个 session 存同一 paper -> papers 只有一行
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `InMemoryRepository` + `PostgresRepository`**（PG 版 `nearest_papers` 用 spec §4.3 的 SQL：`ORDER BY embedding <=> :vec LIMIT :k`）
- [ ] **Step 4: 运行确认通过（单测），再写并运行 PG 集成测试**（`tests/integration/test_repository_pg.py` 复用 Step 1 的核心断言 + `pg_dsn` + 每用例 `TRUNCATE`）— PASS / SKIP
- [ ] **Step 5: Commit** — `git commit -m "feat: Repository 协议与内存 / Postgres 双实现"`

---

### Task 7: llm.py — provider 注册表与结构化输出

**Files:**
- Create: `src/topic_agent/llm.py`
- Test: `tests/test_llm.py`

**Interfaces:**
- Consumes: Task 2 `Settings`、Task 3 `LlmCallRecord`
- Produces:
  - `@dataclass(frozen=True) class ProviderSpec(base_url_default: str, default_model: str, supports_embedding: bool)`
  - `PROVIDERS: dict[str, ProviderSpec]` = `{"glm": ("https://open.bigmodel.cn/api/paas/v4/", "glm-4-plus", True), "deepseek": ("https://api.deepseek.com", "deepseek-chat", False), "siliconflow": ("https://api.siliconflow.cn/v1", "", True)}`（default_model 为空的必须由 env 指定）
  - `get_chat_model(settings: Settings, tier: str = "default") -> ChatOpenAI`——`default` 档读 `llm_provider`；`reasoning` 档读 `llm_reasoning_provider`（空则回落 `llm_provider`）+ `llm_model_reasoning`（空则回落该 provider 默认 model）；统一 `ChatOpenAI(model=..., api_key=..., base_url=..., temperature=0.2)`
  - `class LLMSchemaError(Exception)`
  - `structured(chat, model_cls: type[T], system: str, user: str, node: str = "", tier: str = "default", max_attempts: int = 3) -> tuple[T, LlmCallRecord]`——`chat.with_structured_output(model_cls)`，`ValidationError` 时把错误信息拼回提示重试，最多 `max_attempts` 次后抛 `LLMSchemaError`；返回值与调用记录（attempts / latency / schema_valid_first_try）成对返回，节点把 record 追加进 `state["llm_calls"]`；每次尝试发 `log_event("llm_call_start", node=, tier=, model=, attempt=, prompt_chars=)`，结束发 `log_event("llm_call_end", node=, latency_ms=, usage=, schema_valid=, error=)`（spec §4.10）

- [ ] **Step 1: 写失败测试**（FakeChat 只需实现 `with_structured_output(cls) -> RunnableLike`，`RunnableLike.invoke(messages)` 可配置前 N 次抛 ValidationError）

```python
def test_get_chat_model_default_and_reasoning_tier(): ...   # reasoning provider 留空回落 llm_provider；model 留空回落默认
def test_structured_success_first_try(): ...                # record.schema_valid_first_try is True, attempts == 1
def test_structured_retries_then_succeeds(): ...            # 前 2 次 ValidationError，第 3 次成功 -> attempts == 3
def test_structured_raises_llm_schema_error_after_3(): ...  # pytest.raises(LLMSchemaError)
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `llm.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: provider 注册表、get_chat_model 分档与结构化输出重试"`

---

### Task 8: embedding.py — 可插拔 Embedding

**Files:**
- Create: `src/topic_agent/embedding.py`
- Test: `tests/test_embedding.py`

**Interfaces:**
- Consumes: Task 2 `Settings`
- Produces:
  - `class EmbeddingClient(Protocol): def embed(self, texts: list[str]) -> list[list[float]]: ...`（批量内部按 ≤64 分批）
  - `class OpenAICompatibleEmbedding(base_url: str, api_key: str, model: str, dimensions: int | None = None, client: openai.OpenAI | None = None)`——glm 传 `dimensions=1024`，siliconflow 不传（spec §4.7）；首次调用校验输出维度 == 1024 否则 `ValueError`；每次 `embed()` 发 `log_event("embedding_call", provider=, model=, n_texts=, dim=, latency_ms=)`
  - `class LocalEmbedding(model_name: str)`——函数内 import `sentence_transformers`（optional 依赖，未装时报 ImportError 提示 `uv sync --extra local-embedding`），同样发 `embedding_call` 事件
  - `get_embedding_client(settings: Settings) -> EmbeddingClient`

- [ ] **Step 1: 写失败测试**（fake `openai.OpenAI`：注入带 `.embeddings.create(model, input, **kw)` 的假 client）

```python
def test_openai_compatible_batches_by_64(fake_client): ...      # 100 条文本 -> 2 次 create 调用
def test_dimensions_param_passed_for_glm_only(fake_client): ...
def test_dim_mismatch_raises(fake_client): ...                  # 返回 768 维 -> ValueError
def test_get_embedding_client_dispatch(): ...                   # local / glm / siliconflow 三分支
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `embedding.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: 可插拔 embedding（OpenAI 兼容 / 本地）"`

---

### Task 9: verify.py — 引用提取与验证

**Files:**
- Create: `src/topic_agent/verify.py`
- Test: `tests/test_verify.py`

**Interfaces:**
- Consumes: Task 3 `Paper`、`EvidenceItem`
- Produces:
  - `CITATION_RE = re.compile(r"\[([A-Za-z0-9]+)\]")`（spec §4.6）
  - `extract_citations(text: str) -> list[str]`（按出现序去重）
  - `@dataclass VerificationResult(valid: list[str], invalid: list[str], rate: float)`（rate = len(valid) / len(total)，total 为 0 时 rate = 1.0）
  - `check_citations(ids: list[str], valid_ids: set[str]) -> VerificationResult`
  - `build_evidence(stage: str, claims: list[str], valid_ids: set[str], papers: dict[str, Paper], confidence: float | None = None) -> list[EvidenceItem]`——每个 (claim, 提取到的 paper_id) 一行；`verification_status` 按是否 ∈ valid_ids；`quote` = 该论文 abstract 前 280 字符；`source_tier` 从 papers 复制；不在 papers 中的 id quote/tier 为 None；结束时发 `log_event("verify", stage=, n_citations=, n_valid=, n_invalid=, invalid_ids=)`

- [ ] **Step 1: 写失败测试**

```python
def test_extract_citations_dedup_in_order(): ...        # "a [abc] ... [abc] [def]" -> ["abc", "def"]
def test_extract_ignores_non_bracket_text(): ...
def test_check_citations_split(): ...                   # 3 valid 1 invalid -> rate 0.75
def test_check_citations_empty_is_one(): ...
def test_build_evidence_one_row_per_pair(): ...         # 2 claims 引 2+1 个 id -> 3 行
def test_build_evidence_marks_unverified_and_quote(): ...  # 无效 id -> unverified, quote None；有效 id -> quote 来自 abstract 前 280 字符
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `verify.py`**（纯函数，无 IO）
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: 引用提取、验证与 EvidenceItem 构建"`

---

### Task 10: sources/s2.py — Semantic Scholar 客户端

**Files:**
- Create: `src/topic_agent/sources/__init__.py`、`src/topic_agent/sources/s2.py`
- Test: `tests/test_s2.py`

**Interfaces:**
- Consumes: Task 3 `Paper`
- Produces:
  - `compute_source_tier(doi: str | None, arxiv_id: str | None) -> str`——有 DOI 且无 arXiv → `"B"`；有 arXiv → `"D"`；其余 `"C"`（spec §4.5 Stage 2）
  - `class S2Client(api_key: str | None = None, transport: httpx.BaseTransport | None = None, min_interval: float = 1.0, max_retries: int = 3)`——`transport` 供测试注入 `httpx.MockTransport`
    - `search(query: str, limit: int = 150) -> list[Paper]`——GET `https://api.semanticscholar.org/graph/v1/paper/search`，fields = `paperId,title,abstract,authors,year,venue,citationCount,externalIds,tldr`；缺字段的行跳过（不崩）
    - `references(paper_ids: list[str]) -> dict[str, list[str]]`——POST `/graph/v1/paper/batch`（fields 含 `references.paperId`），500 个一批
    - 通用：请求间强制 `min_interval`（无 key 1 req/s）；429 / 5xx 指数退避重试至 `max_retries` 后抛 `S2Error`；每次请求 `log_event("s2_call", endpoint=..., query=..., n_results=..., latency_ms=..., rate_limit_wait_s=...)`

- [ ] **Step 1: 写失败测试**（`httpx.MockTransport` 返回固定 JSON）

```python
def test_search_parses_fields_and_tier(): ...      # DOI 无 ArXiv -> B；有 ArXiv -> D；无 -> C
def test_search_skips_rows_missing_title(): ...
def test_references_batch_filters_null(): ...      # references 列表里 paperId 为 None 的剔除
def test_rate_limit_spacing(mock_clock): ...       # 两次请求间隔 >= min_interval（注入时钟）
def test_retry_on_429_then_success(): ...          # 第一次 429，第二次 200 -> 成功且发了 2 次
def test_retry_exhausted_raises_s2error(): ...
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `s2.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: S2 客户端（限速、退避、tier 规则）"`

---

### Task 11: analytics — PageRank 与 KMeans 聚类

**Files:**
- Create: `src/topic_agent/analytics/__init__.py`、`src/topic_agent/analytics/pagerank.py`、`src/topic_agent/analytics/cluster.py`
- Test: `tests/test_pagerank.py`、`tests/test_cluster.py`

**Interfaces:**
- Consumes: 无（纯函数，LLM 命名不在此层）
- Produces:
  - `pagerank_scores(edges: list[tuple[str, str]]) -> dict[str, float]`（NetworkX DiGraph，citing→cited）
  - `top_landmarks(scores: dict[str, float], k: int = 10) -> list[str]`（按分降序、并列按 id 稳定排序）
  - `kmeans_labels(vectors: list[list[float]]) -> list[int]`——k 在 5..12 中按轮廓系数选优；`n < 50` 时 `k = max(3, n // 8)`；**始终 `k = min(k, n)`；`n == 0` 返回 `[]`**（Review Focus 2）；`random_state=42`
  - `representative_indices(vectors: list[list[float]], labels: list[int], per_cluster: int = 5) -> dict[int, list[int]]`——每簇离质心最近的 per_cluster 个样本下标

- [ ] **Step 1: 写失败测试**

```python
def test_pagerank_hub_ranks_first(): ...           # 链式 + 汇聚边，被引最多的 hub 分最高
def test_pagerank_empty_edges(): ...               # -> {}
def test_top_landmarks_orders_and_caps(): ...
def test_kmeans_three_blobs_get_three_labels(): ...  # 3 组远离的 1024 维向量 -> 恰好 3 个不同 label 且组内一致
def test_kmeans_labels_tiny_inputs(): ...          # n=0 -> []；n=2 -> k 被钳到 2，不抛异常
def test_representative_indices_picks_nearest(): ...
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `analytics` 两个模块**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: NetworkX PageRank 与轮廓系数选 k 的 KMeans"`

---

### Task 12: graph/state.py + nodes/intent.py + nodes/hil.py

**Files:**
- Create: `src/topic_agent/graph/__init__.py`、`src/topic_agent/graph/state.py`、`src/topic_agent/graph/nodes/__init__.py`、`src/topic_agent/graph/nodes/intent.py`、`src/topic_agent/graph/nodes/hil.py`
- Test: `tests/test_intent_node.py`、`tests/test_hil_nodes.py`

**Interfaces:**
- Consumes: Task 3 schemas、Task 7 `structured`
- Produces:
  - `class TopicAgentState(TypedDict)`：`session_id: str`、`user_input: UserInput`、`intent_card: IntentCard | None`、`retrieved_paper_ids: list[str]`、`landscape: LandscapeView | None`、`paths: list[CandidatePath]`、`matrix: ComparisonMatrix | None`、`selected_path_ids: list[str]`、`analyses: list[DeepAnalysis]`、`report_path: str | None`、`warnings: Annotated[list[str], operator.add]`、`llm_calls: Annotated[list[LlmCallRecord], operator.add]`
  - `build_intent_node(chat) -> NodeFn`——`structured(chat, IntentCard, system, user)`，返回 `{"intent_card": card, "llm_calls": [record]}`
  - `build_hil_intent_node(repo) -> NodeFn`——`interrupt({"kind": "intent", "card": state["intent_card"]})`；resume 值 `{"decision": "approve" | "edit", "card"?: IntentCard}`，`edit` 时返回 `{"intent_card": resume["card"]}`，同时 `repo.set_session_status(session_id, "awaiting_intent")`
  - `build_hil_path_node(repo) -> NodeFn`——`interrupt({"kind": "path", "matrix": state["matrix"], "paths": state["paths"]})`；resume 值 `{"selected_ids": [str]}`，返回 `{"selected_path_ids": [...]}`，status `awaiting_path`

- [ ] **Step 1: 写失败测试**

`tests/test_intent_node.py`：FakeChat 返回固定 IntentCard —— `test_intent_node_returns_card_and_record`。

`tests/test_hil_nodes.py`（**用真 LangGraph**：`StateGraph` 两节点 + `MemorySaver`，验证 interrupt 语义而非 mock）：

```python
def test_hil_intent_interrupts_and_applies_edit(memory_graph): ...
# invoke 后 get_state -> tasks 有 interrupts，payload kind == "intent"
# resume Command(resume={"decision": "edit", "card": new_card}) -> 终态 intent_card == new_card
def test_hil_intent_approve_keeps_card(memory_graph): ...
def test_hil_path_returns_selected_ids(memory_graph): ...
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 state 与两个节点文件**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: TopicAgentState、intent 节点与两个 HIL 中断节点"`

---

### Task 13: nodes/retrieval.py — Stage 2

**Files:**
- Create: `src/topic_agent/graph/nodes/retrieval.py`
- Test: `tests/test_retrieval_node.py`

**Interfaces:**
- Consumes: Task 6 `Repository`、Task 8 `EmbeddingClient`、Task 10 `S2Client`
- Produces: `build_retrieval_node(s2: S2Client, embedder: EmbeddingClient, repo: Repository) -> NodeFn`
  逻辑（spec §4.5 Stage 2，按序）：
  1. `s2.search(intent_card.search_query, limit=150)`
  2. 过滤：`abstract` 非空 且 `year` 在 `time_window` 内（**`year is None` 剔除**，Review Focus 1）
  3. `repo.save_papers(...)`（含 tier，tier 已由 s2 层算好）→ 返回缺向量的 id → `embedder.embed(title + "\n" + abstract)` → `repo.save_embeddings`
  4. citationCount 前 80 篇 `s2.references()`，只保留两端都在 `session_papers` 的边 → `repo.save_citations`
  5. 若过滤后 < 10 篇：**放宽年份重试一次**（重新 `s2.search` 后跳过年份过滤只保留 abstract 非空），仍 < 10 则 `warnings` 追加 `"文献覆盖不足：检索后仅 N 篇"` 并继续
  6. 返回 `{"retrieved_paper_ids": [...], "warnings": [...]}`

- [ ] **Step 1: 写失败测试**（FakeS2 / FakeEmbedder / InMemoryRepository）

```python
def test_retrieval_filters_null_year_and_empty_abstract(fake_deps): ...  # 150 篇中混入空 abstract / 年份外 / year=None -> 正确过滤（Review Focus 1）；embed 只对新增 id 调用
def test_retrieval_reembeds_stale_rows(fake_deps): ...           # Review Focus 5：repo 预置 embedding_model="glm" 的行，当前 embedder 名 "sf" -> 该行被重新 embed
def test_retrieval_citations_only_within_set(fake_deps): ...     # 引用指向集外的边被丢弃
def test_retrieval_relaxes_year_when_too_few(fake_deps): ...     # 5 篇在窗口内 + 20 篇窗口外 -> 重试后 25 篇、无 warning
def test_retrieval_warns_when_still_too_few(fake_deps): ...      # 放宽后仍 3 篇 -> warnings 有 "文献覆盖不足"、流程不抛异常
def test_retrieval_s2_empty_result(fake_deps): ...               # S2 返回 [] -> 0 篇 + warning，不崩（反目标 2）
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `retrieval.py`**（节点用 `@logged_node` 装饰）
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: retrieval 节点（过滤、降级、引用边、向量补齐）"`

---

### Task 14: nodes/landscape.py — Stage 3

**Files:**
- Create: `src/topic_agent/graph/nodes/landscape.py`
- Test: `tests/test_landscape_node.py`

**Interfaces:**
- Consumes: Task 6 `Repository`、Task 7 `structured`、Task 11 `kmeans_labels / representative_indices / pagerank_scores / top_landmarks`
- Produces: `build_landscape_node(repo: Repository, chat) -> NodeFn`
  逻辑（spec §4.5 Stage 3）：
  1. `repo.get_embeddings(session_id)`；空则返回 `LandscapeView(clusters=[], landmarks=[], timeline={}, saturation_level="low", warnings=["无文献，全景为空"])`（Review Focus 2）
  2. `kmeans_labels` → 每簇 `representative_indices` 取 5 篇 → `repo.get_papers` 取正文 → `structured(chat, ClusterNaming, ...)` 生成 `name` + `description`（`ClusterNaming(name: str, description: str)` 定义在本文件或 schemas）
  3. `pagerank_scores(repo.get_citations(session_id))` → `top_landmarks(k=10)`
  4. `timeline`：按年计数（升序 dict）；`saturation_level`：近 2 年（相对当前年，参数 `now_year: int | None = None` 便于测试）占比 > 50% 且总数 > 100 → `"high"`，< 25% → `"low"`，否则 `"medium"`
  5. `repo.set_clusters(session_id, cluster_ids, landmarks)`；返回 `{"landscape": view, "llm_calls": [...]}`

- [ ] **Step 1: 写失败测试**（预置 InMemoryRepository：30 篇、3 组远离向量、链式引用使某 hub 被 5 篇引用）

```python
def test_landscape_clusters_named(fake_deps): ...        # 3 个簇、FakeChat 命名生效、cluster_id 已写回 repo
def test_landscape_landmarks_from_pagerank(fake_deps): ...  # hub paper_id 在 landmarks 且 is_landmark=True
def test_landscape_saturation_levels(fake_deps): ...     # 构造三种年份分布 -> high / medium / low（now_year 注入）
def test_landscape_with_almost_no_papers(fake_deps): ... # 2 篇 -> 不抛异常，warning 或空簇容忍（Review Focus 2）
def test_landscape_empty_embeddings(fake_deps): ...      # 0 篇 -> 空视图 + warning
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `landscape.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: landscape 节点（聚类命名、PageRank 里程碑、时间线与饱和度）"`

---

### Task 15: nodes/path_gen.py — Stage 4

**Files:**
- Create: `src/topic_agent/graph/nodes/path_gen.py`
- Test: `tests/test_path_gen_node.py`

**Interfaces:**
- Consumes: Task 6 `Repository`、Task 7 `structured`、Task 8 `EmbeddingClient`、Task 9 `build_evidence / extract_citations`
- Produces: `build_path_gen_node(repo: Repository, chat, embedder: EmbeddingClient) -> NodeFn`
  逻辑（spec §4.5 Stage 4）：
  1. LLM 输入：IntentCard + LandscapeView + 每簇代表文献（含 paper_id），`structured(chat, PathBatch, ...)` 一次生成 4 条左右路径；`PathBatch(paths: list[CandidatePath])` 定义在本文件（LLM 直接输出 novelty / feasibility 的 score + reasoning，reasoning 中须含 `[paper_id]`）
  2. claim 集合 = 每条路径的 `novelty.reasoning`、`feasibility.reasoning`、`research_question`；`build_evidence(stage="path_gen", ...)` → `repo.save_evidence`
  3. `novelty.score` 覆盖为程序计算值：embed `title + " " + research_question` → `repo.nearest_papers(k=5)` 取最大 cosine_sim → `novelty = min(5.0, 1.0 + 4.0 * (1 - max_sim))`
  4. 路径描述 embedding 两两余弦距离 < `path_min_distance`（Settings，默认 0.3）→ 丢弃后出现的那条
  5. 剩余 < 3 条 → 再调一次 LLM 补一批（与已有路径合并去重规则同上）；仍 < 3 → `warnings` 追加 `"路径数量不足：N 条"` 继续流程；> 5 → 按 `novelty.score` 降序取前 5
  6. 返回 `{"paths": [...], "llm_calls": [...], "warnings": [...]}`

- [ ] **Step 1: 写失败测试**（FakeChat 按 `PathBatch` 返回可配置批次；FakeEmbedder 确定性向量）

```python
def test_path_gen_writes_evidence_for_all_paths(fake_deps): ...     # 4 条路径 × ≥3 个 claim 字段 -> evidence(stage="path_gen") 行数正确
def test_path_gen_marks_unverified_citation(fake_deps): ...         # reasoning 含不在检索集的 id -> 该行 unverified（反目标 1 的单元级验证）
def test_path_gen_novelty_overridden_by_program(fake_deps): ...     # LLM 给的 score 被最近邻公式覆盖；构造紧邻路径断言分低
def test_path_gen_drops_near_duplicate_paths(fake_deps): ...        # 两条描述余弦距离 < 0.3 -> 后者被丢弃
def test_path_gen_top5_by_novelty(fake_deps): ...                   # 6 条 -> 5 条
def test_path_gen_refills_when_under_three(fake_deps): ...          # 首批 2 条 -> 第二批补到 ≥3
def test_path_gen_warns_when_still_under_three(fake_deps): ...      # 两批都 2 条 -> warning + 继续
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `path_gen.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: path_gen 节点（证据写入、程序化新颖度、去重补齐）"`

---

### Task 16: nodes/compare.py — Stage 5

**Files:**
- Create: `src/topic_agent/graph/nodes/compare.py`
- Test: `tests/test_compare_node.py`

**Interfaces:**
- Consumes: Task 6 `Repository`、Task 7 `structured`、Task 9 `build_evidence`
- Produces: `build_compare_node(repo: Repository, chat) -> NodeFn`、模块常量 `WEIGHTS = {"novelty": 0.25, "feasibility": 0.30, "impact": 0.25, "risk": 0.20}`
  逻辑（spec §4.5 Stage 5）：
  1. `structured(chat, CompareBatch, ...)` 为每条路径打 `impact` / `risk`（ScoredDimension），`CompareBatch(scores: list[PathScores])`，`PathScores(path_id, impact, risk)`
  2. 冲突检测：`structured(chat, ConflictBatch, ...)` 扫描全部 reasoning，输出矛盾对 `ConflictBatch(conflicts: list[Conflict])`；冲突描述文本中的 `[paper_id]` 经 `build_evidence(stage="compare")` → `repo.save_evidence`
  3. `weighted_score = 0.25 * novelty + 0.30 * feasibility + 0.25 * impact + 0.20 * (6 - risk)`（novelty / feasibility 沿用 Stage 4）
  4. 每维 `confidence`：LLM 自评 0-1（`PathScores` 附带）；出现在任一 conflict 的路径其全部维度 confidence × 0.7
  5. 按 `weighted_score` 降序得 `ComparisonMatrix`；返回 `{"matrix": m, "llm_calls": [...]}`

- [ ] **Step 1: 写失败测试**

```python
def test_compare_weighted_formula_exact(fake_deps): ...   # novelty=4.2, feasibility=3.8, impact=4.0, risk=2.1 -> weighted == 0.25*4.2+0.30*3.8+0.25*4.0+0.20*(6-2.1)
def test_compare_sorted_desc(fake_deps): ...
def test_compare_conflict_penalty(fake_deps): ...         # 冲突路径 confidence *= 0.7
def test_compare_conflict_citations_verified(fake_deps): ...  # 冲突说明含有效+无效 id -> evidence(stage="compare") 两行、一 verified 一 unverified
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `compare.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: compare 节点（加权矩阵、冲突检测、置信度衰减）"`

---

### Task 17: report.py + nodes/report.py — Stage 6

**Files:**
- Create: `src/topic_agent/report.py`、`src/topic_agent/templates/report.md.j2`、`src/topic_agent/graph/nodes/report.py`
- Test: `tests/test_report_render.py`、`tests/test_report_node.py`

**Interfaces:**
- Consumes: Task 6 `Repository`、Task 7 `structured`、Task 9 `build_evidence`、Task 3 `TopicReport`
- Produces:
  - `render_report(tr: TopicReport) -> str`——Jinja2 渲染（`jinja2.PackageLoader("topic_agent", "templates")`）；章节顺序：意图卡、全景摘要（簇 + 里程碑 + 时间线 + 饱和度）、对比矩阵（全路径）、选中路径深度分析、证据表、局限性声明；**unverified 行必须带可见标记 `⚠️ unverified`**（spec DoD 5）
  - `write_report(tr: TopicReport, out_dir: Path = Path("reports")) -> Path`——写 `reports/<session_id>.md`
  - `build_report_node(repo: Repository, chat) -> NodeFn`——对每条选中路径 `structured(chat, DeepAnalysis, ...)`；claim（recommended_approach 等自由文本）`build_evidence(stage="report")` → `repo.save_evidence`；组装 `TopicReport`（evidence 用 `repo.get_evidence` 全量）；`write_report`；`repo.set_session_status(session_id, "completed", report_path)`；返回 `{"analyses": [...], "report_path": str(p), "llm_calls": [...]}`

- [ ] **Step 1: 写失败测试**

`tests/test_report_render.py`（构造固定 TopicReport）：

```python
def test_render_contains_all_sections(sample_report): ...    # 六个章节标题都在输出中
def test_render_marks_unverified(sample_report): ...         # evidence 含 unverified 行 -> 输出含 "⚠️ unverified"
def test_render_evidence_table_rows(sample_report): ...      # 行数 == len(evidence)，每行含 paper_id 与 stage
def test_write_report_creates_file(tmp_path, sample_report): ...
```

`tests/test_report_node.py`（FakeChat + InMemoryRepository）：

```python
def test_report_node_full_flow(fake_deps, tmp_path): ...     # analyses 生成、evidence 三阶段齐、session status completed、report_path 返回且文件存在
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `report.py`、模板与节点**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: Jinja2 报告渲染与 report 节点"`

---

### Task 18: graph/build.py — 全图拼装与 fake 端到端

**Files:**
- Create: `src/topic_agent/graph/build.py`
- Test: `tests/test_build_graph.py`

**Interfaces:**
- Consumes: Task 12-17 全部节点工厂
- Produces: `build_graph(*, intent, hil_intent, retrieval, landscape, path_gen, compare, hil_path, report, checkpointer) -> CompiledGraph`——拓扑 `START → intent → hil_intent → retrieval → landscape → path_gen → compare → hil_path → report → END`（spec §4.4），节点名与文件对应；`checkpointer` 由调用方注入（测试 `MemorySaver`、CLI `PostgresSaver`）

- [ ] **Step 1: 写失败测试**（全部 fake 依赖 + `MemorySaver`；FakeChat 按请求的 model_cls 分发：IntentCard / ClusterNaming / PathBatch / CompareBatch / ConflictBatch / DeepAnalysis 各返回合理固定值；FakeEmbedder 确定性；FakeS2 返回 ~24 篇 3 组语义）

```python
def test_e2e_happy_path_with_fakes(fake_app): ...
# invoke({"session_id": sid, "user_input": ...}, config={thread_id, recursion_limit: 50})
# -> get_state 有 interrupts（intent）-> Command(resume={"decision": "approve"})
# -> 再次 interrupts（path）-> Command(resume={"selected_ids": [top1]})
# -> 终态：report_path 非空、paths 3-5 条、evidence 覆盖 path_gen/compare/report 三 stage、无 unverified
def test_e2e_intent_edit_changes_search_query(fake_app): ...  # edit 换 search_query 后 FakeS2 收到新 query
def test_e2e_checkpoints_persisted(fake_app): ...             # MemorySaver 每 thread 的 checkpoint 数 >= 节点数；两次 resume 后 parent 链存在
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `build.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: 全量回归** — `uv run pytest -v` 全绿
- [ ] **Step 6: Commit** — `git commit -m "feat: 全图拼装与 fake 端到端测试"`

---

### Task 19: cli.py — 入口、HIL 交互、--resume、--list

**Files:**
- Create: `src/topic_agent/cli.py`
- Test: `tests/test_cli.py`

**Interfaces:**
- Consumes: Task 18 `build_graph`、Task 6 `Repository`、Task 2 `load_settings`、Task 4 `configure_logging / log_event`
- Produces:
  - `prompt_user_input(input_fn=input) -> UserInput`——依次问研究想法（必填）、领域（默认 NLP）、时间约束、资源约束（逗号分隔）
  - `confirm_intent_card(card: IntentCard, input_fn=input) -> dict`——打印意图卡（spec §4.1 格式），循环读 `[a] 确认 [e] 修改 [r] 重新输入`；`e` 逐字段提示（当前值为默认）；`r` 返回 `{"decision": "restart"}` 由上层结束会话；返回 `{"decision": "approve"} | {"decision": "edit", "card": IntentCard}`
  - `select_paths(matrix: ComparisonMatrix, paths: list[CandidatePath], input_fn=input) -> list[str]`——打印对比矩阵，读 `1,3` 形式；**校验：1-2 条、序号在范围内，否则重新提示**（Review Focus 4）
  - `run_graph(graph, session_id: str, repo, input_fn=input) -> str | None`——`graph.stream(initial, config, stream_mode="updates")` 循环；每次流结束后 `graph.get_state(config)` 检查 interrupts：kind == intent → `confirm_intent_card` → `Command(resume=...)`（restart → `set_session_status("failed")` 返回）；kind == path → `select_paths` → `Command(resume=...)`；interrupt 触发时发 `log_event("hil_interrupt", point=, payload_summary=)`、用户恢复后发 `log_event("hil_resume", point=, decision=, waited_s=)`；异常捕获 `LLMSchemaError / S2Error` → `set_session_status("failed")` + `log_event` + 重抛给 main（Review Focus 3）
  - `main(argv: list[str] | None = None) -> int`——argparse：无参数跑新会话（`init_db`、`configure_logging`、建 session 行、发 `log_event("session_start", session_id=, user_input=, llm_provider=, llm_model=, reasoning_model=, embedding_provider=, embedding_model=)`、组装真实依赖、`build_graph(checkpointer=PostgresSaver)`，结束时发 `session_end`）；`--resume <session_id>`（session 不存在 → 报错退出；status == completed → 打印已完成与报告路径，返回 0；否则 `graph.stream(None, config)` 续跑）；`--list`（表格打印会话）；入口处 `settings.validate_for_run()`；退出码 0 / 1
  - 组装真实依赖的工厂 `build_app(settings) -> (graph, repo)` 供 main 与 eval 复用

- [ ] **Step 1: 写失败测试**（monkeypatch `input` 序列；graph 用 Task 18 的 fake 依赖 + `MemorySaver`）

```python
def test_confirm_intent_approve_edit_restart(fake_input): ...
def test_select_paths_rejects_invalid_input(fake_input): ...     # "0" / "1,2,3" / "9" -> 重新提示后 "1" 通过
def test_main_full_run_with_fakes(monkeypatch, fake_deps, tmp_path): ...  # 跑完整会话 -> 返回 0、reports/<sid>.md 存在、session status completed
def test_main_marks_session_failed_on_llm_schema_error(monkeypatch, fake_deps): ...  # FakeChat 恒抛 ValidationError -> LLMSchemaError -> status failed、exit code 1、无未处理 traceback
def test_main_list(monkeypatch, fake_deps, capsys): ...
def test_main_resume_completed_session(monkeypatch, fake_deps, capsys): ...  # 输出"已完成"且不再跑图
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `cli.py`**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: CLI 入口（HIL 交互、resume、list、失败处置）"`

---

### Task 20: eval/ — golden cases、指标、评估脚本

**Files:**
- Create: `src/topic_agent/eval/__init__.py`、`src/topic_agent/eval/golden/case_001.json`、`case_002.json`、`case_003.json`、`src/topic_agent/eval/metrics.py`、`src/topic_agent/eval/run_eval.py`
- Test: `tests/test_metrics.py`、`tests/test_run_eval.py`

**Interfaces:**
- Consumes: Task 18 `build_graph`、Task 6 `Repository`、Task 19 `build_app`
- Produces:
  - `GoldenCase(BaseModel)`：`case_id, query: UserInput, expected: Expected(key_concepts: list[str], must_find_title_keywords: list[str], min_papers: int), difficulty: str`；三个 case 主题照 spec §4.8：幻觉检测（NLP，medium）、参数高效微调（LLM，easy）、多模态对齐中的幻觉（CV/NLP，hard）
  - `collect_claims(state: dict) -> list[tuple[str, str]]`——(stage, claim) 全集：path_gen（每条路径 3 字段）、compare（每条冲突说明）、report（每条 analysis 的 recommended_approach）
  - `compute_metrics(state: dict, evidence: list[EvidenceItem], golden: GoldenCase, papers_titles: dict[str, str]) -> dict[str, float | str]`——实现 spec §4.8 全部指标：`task_completion`（6 stage 产出齐）、`json_schema_validity`（`llm_calls` 中 `schema_valid_first_try` 比例，目标 ≥ 0.90）、`citation_verify_rate`（≥ 0.95）、`claim_support_rate`（≥ 0.90）、`hallucination_rate`（≤ 0.05）、`path_count`（3-5）、`path_distinctness`（≥ 0.3）、`concept_coverage`（≥ 0.75）、`must_find_recall`（≥ 0.5）、`latency_total_s`、`llm_call_count`、`provider` / `model`
  - `run_eval(settings=None, graph_factory=None, out_dir=Path("eval/results")) -> Path`——graph_factory 可注入（测试传 fake，生产默认 `build_app`）；对每个 case：`log_event("eval_case_start", case_id=)` 计时、HIL 自动 approve / 自动选加权分最高 1 条、单 case 异常 → `task_completion=0` 并继续、结束发 `eval_case_end`（spec §4.8、§4.10）；输出表格到 stdout 与 `eval/results/<UTC 时间戳>.md`（表头含 provider / model / reasoning model）
  - `format_table(rows: list[dict]) -> str`——Markdown 表格

- [ ] **Step 1: 写失败测试**

`tests/test_metrics.py`（手工构造 state / evidence / golden，断言**精确值**）：

```python
def test_citation_verify_rate_and_hallucination(): ...   # 8 对 verified + 2 对 unverified 分布在 2/10 claim -> 0.80 与 0.20；claim_support_rate 按构造为 0.9
def test_json_schema_validity_from_llm_calls(): ...
def test_concept_coverage_and_must_find_recall(): ...
def test_path_count_and_distinctness(): ...
def test_task_completion_detects_missing_stage(): ...
```

`tests/test_run_eval.py`：

```python
def test_run_eval_with_fakes_writes_results(fake_deps, tmp_path): ...  # 3 case 全跑完、结果文件存在、表格含全部指标列
def test_run_eval_single_case_failure_continues(fake_deps, tmp_path): ...  # 第 2 case 抛异常 -> 结果中 task_completion=0、case 3 仍完成
```

- [ ] **Step 2: 运行确认失败** — FAIL
- [ ] **Step 3: 实现 `eval` 包**
- [ ] **Step 4: 运行确认通过** — PASS
- [ ] **Step 5: Commit** — `git commit -m "feat: 评估（golden cases、程序化指标、run_eval）"`

---

### Task 21: README、真实运行与验收（需真实 key）

**Files:**
- Create: `README.md`、`docs/sample/report_example.md`、`docs/sample/eval_results.md`
- Test: 无新增（手动验收 + 全量回归）

**Interfaces:**
- Consumes: 全部前序任务
- Produces: 通过 spec §7 DoD 1-8 的仓库

- [ ] **Step 1: 写 `README.md`**——前置条件（brew install postgresql@17 pgvector、createdb 两个库）、`uv sync`、`cp .env.example .env` + 配 key 说明、三步验证：`uv run topic-agent`（含 HIL 操作说明）、`uv run pytest`（无 PG 时集成 skip）、`uv run eval`；中文。

- [ ] **Step 2: 请用户配置 key**

暂停并请用户把 `GLM_API_KEY`（或 DeepSeek + 硅基流动 key）填入 `.env`，确认 `.env` 已被 gitignore。

- [ ] **Step 3: 真实 demo 会话（DoD 1、2）**

Run: `uv run topic-agent`——用一个真实研究想法跑通；HIL-1 先选 `e` 修改检索式确认流程按修改后执行，再走完整流程；HIL-3 选 2 条路径；确认 `reports/<session_id>.md` 生成且包含 2 条路径分析。

- [ ] **Step 4: 验证 `--resume`（DoD 1 的 checkpoint 主张）**

新开一个会话，在 HIL-1 中断处 Ctrl-C 退出进程；`uv run topic-agent --resume <session_id>` 从中断点继续到完成。再验证 `--list` 与对已完成会话 resume 的提示行为。

- [ ] **Step 5: 全量测试（DoD 3、6）**

Run: `uv run pytest -v`
Expected: 全绿（PG 运行中：含集成测试；临时停掉 `brew services stop postgresql@17` 再跑一次：集成测试 skip 而非 fail；随后重启 PG）。

- [ ] **Step 6: 真实评估（DoD 4）**

Run: `uv run eval`
Expected: 3 个 case 输出指标表；检查 `citation_verify_rate ≥ 0.95`、`hallucination_rate ≤ 0.05`；不达标则如实保留结果，允许一轮 prompt 调整后重跑一次（两次结果都保留，spec §4.8）。

- [ ] **Step 7: 样例产物与日志核对（DoD 7、8）**

把真实报告拷为 `docs/sample/report_example.md`、评估表拷为 `docs/sample/eval_results.md`；检查 `logs/<session_id>.jsonl` 含 spec §4.10 全部事件类型、`node_end` 有 state 变化摘要、`llm_call_end` 有 token 用量与延迟、无任何 api key 出现。

- [ ] **Step 8: 逐条核对 DoD 1-8，全量回归后提交**

```bash
uv run pytest -v   # 全绿
git add README.md docs/sample/
git commit -m "docs: README 与真实运行样例（报告 + 评估指标）"
```

- [ ] **Step 9: 汇报**——向用户汇报 DoD 核对结果与指标表；提醒可按 spec §5 自行调整 prompt 措辞 / CLI 排版后重跑。

---

## 执行备注

- 每个任务的实现者只见本任务 + Global Constraints + 自己的 Interfaces 块；类型名与签名以 Interfaces 为准，不得自行改名。
- 测试命令统一 `uv run pytest <file> -v`；PG 相关用例必须验证 skip 路径至少一次。
- Task 21 之前任何任务都不得要求网络或真实 key。
