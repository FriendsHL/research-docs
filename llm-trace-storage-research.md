# LLM Observability Trace Storage 调研

调研对象：Langfuse、Opik、LangSmith、Coze Loop（字节开源版的 Coze 罗盘）
目的：为 SkillForge **OBS-1-2（LLM call 完整 payload 落库）** 提供 schema / 存储 / 截断 / 保留策略参考。
信息源：以 GitHub 上各家开源仓库的 migration / GORM model 为主，官方 docs 为辅。Coze 罗盘 SaaS（火山引擎）非开源部分推断为辅。

---

## TL;DR

1. **Trace + Span（Observation）两表分开是行业共识**：trace 一行（root），每个 LLM/tool/通用 span 一行；input/output 直接挂在 span 行上、可空。Langfuse / Opik / Coze Loop 都是这种结构，LangSmith 概念上一致（trace = a collection of spans）。
2. **存储引擎从 PG 全面切到 ClickHouse**：Langfuse 在 v3（2024-09）从 Postgres 主存改为 ClickHouse，Opik 一开始就用 ClickHouse，Coze Loop 也是 ClickHouse + MySQL（MySQL 只放 metadata、不放 trace 数据）。LangSmith 内部不公开但 bulk export 选 Parquet→S3 与 ClickHouse 常被列为下游入仓选项。
3. **大 payload 不在主表硬塞**：Opik **截断阈值 10001 bytes**（`MATERIALIZED` 列），全量字段保留，附件走单独 `attachments` 表挂 MinIO；Langfuse 引入 `blob_storage_file_log` 表 + S3；Coze Loop GORM model 直接有 `object_storage` 字段做对象存储引用；LangSmith SDK 提供 `Attachment` 类型走二进制上传。**没有任何一家把 1MB 级 payload 直接塞进列**。
4. **OTel GenAI semantic convention 已成事实标准**：Langfuse 提供 `/api/public/otel` OTLP 入口；LangSmith 通过 `LANGSMITH_OTEL_ENABLED` + collector 接入；Opik 也支持 OpenTelemetry ingestion。**不是底层 storage，是 ingest 协议**。
5. **给 SkillForge 的核心建议**：单机 dev、PG embedded、规模小，**短期不要切 ClickHouse**。在现有 Postgres 上分两表（trace + span），input/output 用 `TEXT`（PG 自动 TOAST 压缩）+ **服务端 16-32KB 软截断**，超阈值原文挂文件系统/S3 兼容路径，trace 行存路径引用。OTel `gen_ai.*` 字段名做 metadata key 抄过来即可。

---

## 各家概览

### 1. Langfuse（开源 / 自托管 + cloud）

**存储引擎演进**：v2 之前 Postgres 单库，v3 切到 **ClickHouse 做 trace 主存** + Postgres 做应用元数据 + Redis 做队列 + S3 做 blob。
信息源：

- 官方 blog "Langfuse v3 stable release" — https://langfuse.com/blog/2024-09-langfuse-v3-stable-release
- Migration to ClickHouse 大型 issue — https://github.com/langfuse/langfuse/issues/6198 （observations migration deadlock，规模大到要拆批 + 死锁）

**Schema 关键字段**（来自 `packages/shared/clickhouse/migrations/unclustered/0001_traces.up.sql` 和 `0002_observations.up.sql`，commit hash 见 main）：

`traces` 表（root 一行）：

```
id String
timestamp DateTime64(3)
name, user_id, session_id, project_id, release, version
metadata Map(LowCardinality(String), String)   -- 任意 key-value
tags Array(String)
input  Nullable(String) CODEC(ZSTD(3))         -- 整 trace 的 input
output Nullable(String) CODEC(ZSTD(3))         -- 整 trace 的 output
ENGINE = ReplacingMergeTree(event_ts, is_deleted)
PARTITION BY toYYYYMM(timestamp)
ORDER BY (project_id, toDate(timestamp), id)
```

`observations` 表（每个 span / LLM call / tool call 一行）：

```
id, trace_id, parent_observation_id
project_id, type LowCardinality(String)        -- 'GENERATION' / 'SPAN' / 'EVENT'
start_time, end_time, completion_start_time
name, level, status_message, version
input  Nullable(String) CODEC(ZSTD(3))
output Nullable(String) CODEC(ZSTD(3))
provided_model_name, internal_model_id, model_parameters
provided_usage_details Map(LowCardinality(String), UInt64)   -- input_tokens/output_tokens/...
usage_details          Map(LowCardinality(String), UInt64)
provided_cost_details  Map(LowCardinality(String), Decimal64(12))
cost_details           Map(LowCardinality(String), Decimal64(12))
total_cost Nullable(Decimal64(12))
prompt_id, prompt_name, prompt_version
```

特点：**单一 observations 大表**用 `type` 字段区分 LLM call / 普通 span / event；input/output 用 ClickHouse 的 `String CODEC(ZSTD(3))` 保存；usage 和 cost 用 `Map` 灵活存任意 token 维度（cache_read / cache_creation / reasoning 都能塞）。

**Payload 处理**：ZSTD(3) 列压缩；附件/binary 走单独 `blob_storage_file_log` 表（migration 0011，`bucket_name + bucket_path` 引用 S3）。
信息源：`packages/shared/clickhouse/migrations/unclustered/0011_add_blob_storage_file_log.up.sql`

**OTel 遵从度**：原生 `/api/public/otel` OTLP HTTP 入口（JSON + protobuf），不要求 OpenInference / OpenLLMetry 但兼容。
信息源：https://langfuse.com/integrations/native/opentelemetry

**保留期**：cloud SaaS 看 plan；自托管自定义。

---

### 2. Opik（Comet 出品 / 开源）

**存储引擎**：**ClickHouse + MySQL** 双库。ClickHouse 装 trace/span/feedback，MySQL 装 user/workspace/project 等关系数据。
信息源：`apps/opik-backend/src/main/resources/liquibase/db-app-analytics/migrations/000001_init_script.sql`

**Schema 关键字段**：

`spans`（每条 LLM call / tool call / generic）：

```
id FixedString(36), workspace_id, project_id, trace_id, parent_span_id
name, type Enum8('unknown', 'general', 'tool', 'llm')
start_time DateTime64(9, 'UTC'), end_time
input  String DEFAULT ''
output String DEFAULT ''
metadata String                               -- JSON 字符串
tags   Array(String)
usage  Map(String, Int32)                     -- token 用量
ENGINE = ReplacingMergeTree(last_updated_at)
ORDER BY (workspace_id, project_id, trace_id, parent_span_id, id)
```

`traces` 表结构同 spans（id/start_time/end_time/input/output/metadata/tags/created_at），但**没有** parent / type / usage 字段；trace 是 root，spans 挂下面。
信息源：同上 file。

**截断策略（重点）**：migration `000041_add_truncated_input_output_to_spans_and_traces_tables.sql`

```
ADD COLUMN truncation_threshold UInt64 DEFAULT 10001
ADD COLUMN truncated_input  MATERIALIZED if(length(input) >= 10001, substring(input,1,10001), input)
ADD COLUMN truncated_output MATERIALIZED if(length(output) >= 10001, substring(output,1,10001), output)
```

**全量原文保留在 `input/output`，列表/搜索界面查 `truncated_input`**——用 ClickHouse `MATERIALIZED` 列做"按需截断视图"，而不是真的丢数据。后续 `000045` 加了 `workspace_configurations.truncation_on_tables Bool DEFAULT true` 让 workspace 自定义是否启用。

**附件处理**：单独 `attachments` 表（migration `000015`）：

```
workspace_id, container_id, entity_id, entity_type ENUM('trace','span')
file_name, mime_type, file_size Int64
```

配合 MinIO multipart upload，REST API 暴露 `attachmentList` / `uploadAttachment` / `downloadAttachment`。
信息源：https://www.comet.com/docs/opik/tracing/log_traces 提到 attachment endpoints。

**OTel 遵从度**：Opik docs 暴露 OpenTelemetry ingestion endpoint（reference REST API 中有 `subpackage_openTelemetryIngestion`）。

---

### 3. LangSmith（LangChain 商业产品，闭源）

**存储引擎**：内部不公开。可推断的：

- 概念上 **trace = collection of spans / runs**（doc 原话："you can think of a LangSmith trace as a collection of spans... a run as a span"）。
- bulk export 输出 Parquet → S3，下游入仓选项是 BigQuery / Snowflake / Redshift / **ClickHouse** / DuckDB。说明数据是表结构化的、列式可移植。
  信息源：https://docs.smith.langchain.com/langsmith/data-export

**OTel 遵从度**：通过环境变量 `LANGSMITH_OTEL_ENABLED` 开启 OTel 模式，支持 OTel Collector fanout（一份 trace 同时发到 LangSmith 和其他 backend）。
信息源：https://docs.smith.langchain.com/langsmith/trace-with-opentelemetry

**大 payload 处理**：SDK 提供 `Attachment` 类型，binary 不走 input/output JSON 字段。Python 用 `Attachment(mime_type, data)`，TS 用 `extractAttachments` 选项 + Uint8Array。
信息源：https://docs.smith.langchain.com/observability/how_to_guides/upload_files_with_traces

**保留期**：SaaS 默认 **400 天**，可付费延长到更长 tier。
信息源："LangSmith (SaaS) retains trace data for 400 days from ingestion. For details on retention tiers and pricing..." — https://docs.smith.langchain.com/observability/how_to_guides/data_retention

---

### 4. Coze Loop（字节开源版 / 火山引擎闭源版"罗盘"）

**存储引擎**：**ClickHouse 主存 + MySQL 元数据 + Redis + MQ**。仓库：https://github.com/coze-dev/coze-loop（5.4k stars）。
信息源：`backend/modules/observability/infra/repo/ck/`（ClickHouse DAO）和 `…/mysql/`（MySQL DAO）并存。

**Schema 关键字段**（从 GORM 生成模型 `backend/modules/observability/infra/repo/ck/gorm_gen/model/observability_spans.gen.go` 反推）：

```go
type ObservabilitySpan struct {
    TraceID, SpanID, ParentID, SpaceID  string
    SpanType, SpanName                  string
    StartTime int64 (primaryKey)        // 微秒
    Duration  int64
    StatusCode int32
    Method, Psm, Logid, CallType       *string  // 字节内部 RPC 字段（PSM = Product/Subsystem/Module）
    Input, Output                       string
    ObjectStorage                       *string  // ★ 大 payload 引用对象存储路径
    LogicDeleteDate                     int64    // 软删
    TagsBool   map[string]uint8
    TagsFloat  map[string]float64
    TagsString map[string]string
    TagsLong   map[string]int64
    TagsByte   map[string]string        // base64/binary
    SystemTagsFloat / Long / String     map[string]...
}
```

特点：

- **按数据类型拆 5 个 tags Map + 3 个 systemTags Map**——OTel attribute 风格，但区分用户 tag 和系统 tag。
- **有显式 `ObjectStorage` 字段**：input/output 太大时主表存对象存储引用，拉详情时再去对象存储读。
- 用 `Int64` 微秒戳做 primary key（不是 DateTime64）。
- 没看到 `usage_details`/`cost_details` 列，token 用量大概率塞 `TagsLong`（`gen_ai.usage.input_tokens` 这类 key）。

**OTel 遵从度**：仓库内有 OpenTelemetry collector 适配（`infra/collector` 目录），但具体 OTLP endpoint 细节没看 docs。

**保留期**：开源版未明确，火山引擎 SaaS 罗盘需查 console 配置（公开页面动态渲染抓不到）。

---

## 7 问汇总对比表

| 问题                  | Langfuse                                             | Opik                                                       | LangSmith                                   | Coze Loop                              |
| ------------------- | ---------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------- | -------------------------------------- |
| **Q1 表结构**          | trace + observations（单大表，type 区分）                    | trace + spans（type Enum 区分 llm/tool/general）               | trace = collection of runs（runs 即 spans）    | trace + spans（type 字段）                 |
| **Q1 input/output** | `Nullable(String) CODEC(ZSTD(3))`                    | `String DEFAULT ''` + 10KB 截断列                             | 字段+`Attachment` 走 binary                    | `String` + `ObjectStorage` 引用          |
| **Q1 token usage**  | `Map(LowCardinality(String), UInt64)`                | `Map(String, Int32)`                                       | 不公开                                         | 推断在 TagsLong Map                       |
| **Q2 引擎**           | ClickHouse（v3 后）+ PG + Redis + S3                    | ClickHouse + MySQL + MinIO                                 | 闭源；export 走 Parquet→S3                      | ClickHouse + MySQL + Redis + MQ + 对象存储 |
| **Q2 分层**           | 热（CH）+ blob（S3）                                      | 热（CH）+ 附件（MinIO）                                           | 热 + bulk export S3                          | 热（CH）+ ObjectStorage                   |
| **Q3 OTel**         | OTLP HTTP 原生 `/api/public/otel`                      | OTel ingestion endpoint                                    | `LANGSMITH_OTEL_ENABLED` + collector fanout | collector 适配在仓库内                       |
| **Q3 gen_ai 约定**    | 兼容（不强制）                                              | 兼容                                                         | 兼容（OTel mapping）                            | 推断（Tags Map 风格匹配）                      |
| **Q4 截断**           | 不强制，依赖 ZSTD                                          | **10001 bytes 阈值，MATERIALIZED 列**                          | 走 Attachment 类型（客户端）                        | ObjectStorage 引用                       |
| **Q4 大文件**          | `blob_storage_file_log` → S3                         | `attachments` 表 → MinIO                                    | SDK Attachment                              | `object_storage` 字段                    |
| **Q5 主索引**          | `(project_id, toDate(timestamp), id)`                | `(workspace_id, project_id, trace_id, parent_span_id, id)` | 闭源                                          | `(start_time)` PK + map keys 二级        |
| **Q5 metadata 索引**  | `bloom_filter` on map keys/values                    | 无（metadata 是单 String）                                      | 闭源                                          | 无显式（map）                               |
| **Q6 默认保留**         | 自托管自定 / cloud 看 plan                                 | 自托管自定                                                      | **400 天 SaaS 默认**                           | 不公开                                    |
| **Q6 TTL**          | ClickHouse `Partition by toYYYYMM` 方便 drop partition | 同上                                                         | 自动                                          | 同上                                     |
| **Q7 踩过的坑**         | PG 主存撑不住切 CH（issue #6198 死锁）                         | 早期 input/output 不截断列表慢，加 MATERIALIZED 截断列                  | /                                           | /                                      |

---

## 给 SkillForge OBS-1-2 的建议

**前置约束**：SkillForge 当前是 PG embedded（zonky 15432）+ Flyway + 单机 dev，规模"几百到几千 trace/天"级别。**不要切 ClickHouse**，但 schema 设计要为未来切 CH 留路。

### Schema 提议（PG 上跑，两表）

```sql
-- V{n}__create_llm_traces.sql
CREATE TABLE llm_trace (
  id              VARCHAR(36) PRIMARY KEY,
  session_id      VARCHAR(36) NOT NULL,
  agent_id        VARCHAR(64),
  start_time      TIMESTAMPTZ NOT NULL,
  end_time        TIMESTAMPTZ,
  status          VARCHAR(16) NOT NULL,           -- ok/error/running
  total_cost      NUMERIC(12,6),
  total_input_tokens  INT,
  total_output_tokens INT,
  metadata        JSONB,                          -- gen_ai.* OTel 风格 keys
  tags            TEXT[],
  created_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_llm_trace_session ON llm_trace(session_id, start_time DESC);
CREATE INDEX idx_llm_trace_time    ON llm_trace(start_time DESC);

CREATE TABLE llm_span (
  id              VARCHAR(36) PRIMARY KEY,
  trace_id        VARCHAR(36) NOT NULL REFERENCES llm_trace(id) ON DELETE CASCADE,
  parent_span_id  VARCHAR(36),
  type            VARCHAR(16) NOT NULL,           -- 'llm' | 'tool' | 'span'
  name            VARCHAR(128),
  start_time      TIMESTAMPTZ NOT NULL,
  end_time        TIMESTAMPTZ,
  status          VARCHAR(16),
  level           VARCHAR(16),
  status_message  TEXT,
  -- payload
  input_text      TEXT,                           -- 截断后或全量（小）
  output_text     TEXT,                           -- 同上
  input_blob_ref  VARCHAR(512),                   -- 超阈值时存文件路径/S3 URI
  output_blob_ref VARCHAR(512),
  input_size      BIGINT,                         -- 原始 byte 数（截断前）
  output_size     BIGINT,
  -- LLM-specific
  model           VARCHAR(128),
  provider        VARCHAR(32),
  model_params    JSONB,
  usage_details   JSONB,                          -- {input_tokens, output_tokens, cache_read, ...}
  cost_details    JSONB,
  -- meta
  metadata        JSONB,
  created_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_llm_span_trace ON llm_span(trace_id, start_time);
CREATE INDEX idx_llm_span_model ON llm_span(model) WHERE model IS NOT NULL;
CREATE INDEX idx_llm_span_time  ON llm_span(start_time DESC);
```

**对照各家选择的理由**：

- 两表分（不是 trace 单表）：和 Langfuse / Opik / Coze Loop 一致，root 行单独存方便列表查询不扫所有 span。
- `JSONB` 替代 `Map` 类型：PG 没 ClickHouse Map，JSONB 是最接近的等价物，未来切 CH 直接转 `Map(String, String)`。
- `usage_details` / `cost_details` 用 JSONB 存任意 token 维度：抄 Langfuse，能塞 `cache_read_tokens` / `reasoning_tokens` 等 OTel `gen_ai.usage.*` 标准 key，模型新增维度不用 schema migration。
- `*_blob_ref` 双字段（text + ref）：抄 Coze Loop 的 `ObjectStorage` 思路，不像 Langfuse 单独 blob_log 表那么重。

### 存储引擎选择

**短期（<10k trace/天）**：PG 主存。理由：

1. 已有 Flyway migration 流程，加两张表零运维成本。
2. PG 的 TOAST 自动处理大字段（>2KB 行外存储 + 压缩），相当于免费的 ZSTD。
3. JSONB 加 GIN 索引足够覆盖 metadata 查询。

**未来切 CH 的触发条件**（不是现在）：

- 单日 trace > 50 万行 / 单表 > 1 亿行；
- 时间范围扫描 + 多 metadata 维度过滤变慢（>2 秒）；
- 用户要做 trace 级聚合 dashboard。

切 CH 时 schema 直接对得上：`JSONB → Map(String,String)`、`TIMESTAMPTZ → DateTime64(3)`、index 改 `ORDER BY (session_id, start_time, id)`。

### 截断策略

抄 Opik 思路但调阈值：

- **服务端落库前软截断 32KB**（Opik 是 10KB，对 LLM 长 prompt 太小，SkillForge 默认 context 已经 128K，32K 是单个 message 的合理上限）。
- 超阈值：截断后存 `input_text` / `output_text`，**原文写文件** `~/.skillforge/blobs/{trace_id}/{span_id}-input.json`，`input_blob_ref` 存相对路径，`input_size` 存原始字节数。
- 客户端也做一道 cap（如单 message > 5MB 直接拒收），防止恶意上传。
- **不要在 dashboard 列表查询 `input_text` 的全文**：列表只看 `LEFT(input_text, 200)` + `input_size` 提示。

### 保留期

**默认 30 天**（SkillForge 是 dev 工具不是合规审计系统，400 天太奢侈）。

- 用 PG `pg_partman` 按月分区或简单的 cron `DELETE FROM llm_trace WHERE start_time < now() - interval '30 days'`。
- blob 文件配套 cleanup（trace 删了就把 `~/.skillforge/blobs/{trace_id}/` 整个删）。
- 配置项 `skillforge.observability.retention.days` 让用户改。

### 风险点

1. **TOAST 不是银弹**：PG 行外存储仍占磁盘，30 天保留期下 1k trace/天 × 平均 50KB payload ≈ 1.5GB/月，可控；10x 后要重新评估。
2. **JPA + JSONB**：Hibernate 6 用 `@JdbcTypeCode(SqlTypes.JSON)` 是稳的，但 `usage_details` 字段如果直接映射成 `Map<String, Long>` 要确认序列化注册了 JavaTimeModule 之外的处理（**项目已知 footgun #1**）。
3. **OTel `gen_ai.*` key 规范化**：现在 SkillForge LLM provider 各自塞 token 字段名不一致（claude 用 `cache_creation_input_tokens` / openai 用 `prompt_tokens`），落库前要统一映射到 `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` / `gen_ai.usage.cache_creation.input_tokens`，否则后续 dashboard 没法做"全 provider 总 token 数"。建议先建一个 `LlmUsageNormalizer`。
4. **trace_id / span_id 一致性**：现在 SkillForge 没有 trace_id 概念，要决策"一次 chat session = 一个 trace"还是"一次 LLM call = 一个 trace"。从用户体验看应该是 **session+一轮 user message → 一个 trace**，子 LLM call / tool call / SubAgent 都是 span。
5. **流式响应**：LLM streaming 时 output 增量到达，落库时是写 final assembled 还是写 chunks？所有四家都是写 final（`completion_start_time` 字段独立标 TTFT），跟随这个做法。

---

## 不确定 / 需进一步确认

1. **Coze Loop 火山引擎 SaaS 版**的具体限额（payload 大小、保留期、QPS）：volcengine.com docs 是 React SPA 动态渲染，curl 抓不到具体数值；如果要确切信息需要登录 console 或用 agent-browser 实测。开源版本的 GORM model 已能反推 schema，**可以视作接近真实**。
2. **LangSmith 的服务端存储引擎**（PG/CH/自研）官方从未公开；只能从 bulk export 选项推断它"可移植到 ClickHouse"，但内部不一定是 CH。
3. **Langfuse 的 PG → CH 迁移痛点细节**：issue #6198 提到 deadlock，但没找到完整 post-mortem blog；如有需要可以读 PR 历史。
4. **各家在 OTel `gen_ai.system_instructions` / `gen_ai.input.messages` 这种新增字段（2024-2025）的覆盖度**：OTel spec 列出的属性比任何一家 schema 都丰富，实际落地都是 schema 留 metadata Map 兜底，没把 messages 数组拆 row。
