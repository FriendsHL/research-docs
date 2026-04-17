Fetching document 2752372015......
# Memory 模块源码解析


> 研究日期：2026-03-23 | 源码路径：`/app/src/memory/`
>
>


---

## 一、定位

Memory 不是简单的文件读取，而是一个集成了**向量数据库 + 全文搜索 + 实时同步**的独立子系统，让 Agent 具备"长期记忆"能力。


---

## 二、整体架构

```Plain
// 代码块
原始数据来源
  ├── MEMORY.md
  ├── memory/*.md（日记文件）
  └── 对话历史 JSONL
         ↓
   文件监听 / 定时同步
         ↓
   SQLite DB（四张表）
  ├── files（文件变更追踪）
  ├── chunks（文本分块 + 向量嵌入）
  ├── embedding_cache（嵌入向量缓存）
  └── chunks_fts（FTS5 全文索引）
         ↓
   对外接口
  ├── search("query") → 混合检索
  └── readFile(path)  → 直接读文件

```


---

## 三、SQLite 四张表（源码：`sqlite.ts`、`memory-schema.ts`）

```SQL
// 代码块
-- 文件变更追踪
CREATE TABLE files (path, source, hash, mtime, size);

-- 文本分块 + 向量嵌入
CREATE TABLE chunks (id, path, start_line, end_line, text, embedding TEXT, model);

-- 嵌入向量缓存（按内容 hash 缓存，避免重复调 API）
CREATE TABLE embedding_cache (provider, model, hash, embedding);

-- FTS5 全文索引（虚拟表）
CREATE VIRTUAL TABLE chunks_fts USING fts5(id UNINDEXED, text, content='chunks');

```

**向量搜索依赖 sqlite-vec 原生扩展（C++ 模块）：**

```SQL
// 代码块
CREATE VIRTUAL TABLE chunks_vec USING vec0(id TEXT PRIMARY KEY, embedding FLOAT);

-- 查询
SELECT c.*, vec_distance_cosine(v.embedding, ?) AS dist
FROM chunks_vec v JOIN chunks c ON c.id = v.id
ORDER BY dist ASC LIMIT ?

```

score = 1 - dist（余弦距离转相似度）

**与 PostgreSQL 对比：**


- sqlite-vec 的 vec0 ≈ pgvector 的 vector 类型
- `vec_distance_cosine()` ≈ pgvector 的 `<=>` 余弦距离算子
- FTS5 ≈ PostgreSQL 的 tsvector + ts_rank

sqlite-vec 加载失败时**自动降级**为纯关键词搜索，不会崩溃。


---

## 四、Embedding 五种提供商（源码：`embeddings.ts`）


| 提供商 | 默认模型 | 维度 | 方式 |
| --- | --- | --- | --- |
| `openai` | text-embedding-3-small | 1536 | 远程 API |
| `gemini` | gemini-embedding-001 | 768+ | 远程 API |
| `voyage` | voyage-4-large | 1024 | 远程 API |
| `mistral` | mistral-embed | 1024 | 远程 API |
| `local` | embeddinggemma-300m（GGUF） | 256 | 本地 node-llama-cpp |
| `auto` | 有 API key 用远程，否则本地 | — | — |

**本地模式：** 用 Hugging Face GGUF 模型通过 node-llama-cpp（C++ 绑定）直接推理，**完全离线**，适合隐私敏感场景。

**Embedding Cache：** 写入前先按内容 hash 查缓存，命中则复用，只有内容变了才重新调 API。节省大量 token 费用。


---

## 五、搜索流水线（源码：`search-manager.ts`、`hybrid.ts`）

```Plain
// 代码块
search("query")
  1. 同步检查（onSearch=true 时先增量重索引变更文件）
  2. Query Expansion（提取关键词，见下节）
  3. 并行搜索（各自捞 maxResults×4 = 24 条候选）
     ├── 向量检索（sqlite-vec ANN）  → vectorScore
     └── FTS5 关键词检索（BM25）    → textScore
  4. Hybrid 合并（hybrid.ts mergeHybridResults()）
     score = 0.7 × vectorScore + 0.3 × textScore
  5. 时间衰减（可选，默认关闭）：halfLifeDays=30 → 每30天分数减半
  6. MMR 重排序（可选，默认关闭）：λ=0.7 → 70% 相关性 + 30% 多样性
  7. minScore=0.35 过滤 → 截断 maxResults=6 → 返回

```

**BM25 得分归一化（源码 hybrid.ts bm25RankToScore()）：**

```TypeScript
// 代码块
// SQLite FTS5 的 bm25() 返回负值（越负越相关），转换为  正分
function bm25RankToScore(rank: number): number {
  return 1 / (1 + Math.max(0, rank));
}

```


---

## 六、关键词提取（Query Expansion，源码：`query-expansion.ts`）

**完全没有用任何分词模型或 NLP 库**，是纯规则引擎，零依赖。

`tokenize()` 分词策略：


| 语言       | 策略                           |
| -------- | ---------------------------- |
| 英语/西/葡/阿 | 按空格和标点切分，保留 3 字符以上单词         |
| 中文       | 字符 n-gram：单字 + 相邻双字。"讨论方案" → |
| 日语       | 按字体切分，汉字部分再做双字 n-gram        |
| 韩语       | 保留词干，去除助词后缀                  |

内置 7 种语言停用词表（EN/ES/PT/AR/ZH/KO/JA），全部硬编码。

**中文的局限：** 没有真正语义分词，只是字符 n-gram。但向量检索那条腿本来是语义级别的，两者互补后实际效果尚可。


---

## 七、MMR 重排算法详解（源码：`mmr.ts`）


> 来自 Carbonell & Goldstein 1998 年论文《The Use of MMR, Diversity-Based Reranking》
>
>

### 公式

```Plain
// 代码块
MMR得分 = λ × 相关性 - (1-λ) × max(与已选集合中每条的相似度)

```

关键：是"与已选集合里**最相似那条**"的相似度（不是平均），随着已选集合增大，约束越来越强。

### 为什么 λ=0.7 是默认值？


- λ=1.0：纯相关性排序，可能返回"5条都在说同一件事的不同表述"
- λ=0.0：纯多样性，不管相关性，结果很随机
- λ=0.7：保证最相关结果排前面，同时剔除明显重复内容

### 完整计算量（选6条，候选池24条）

```Plain
// 代码块
轮次   候选条数   已选条数   Jaccard计算次数
1      24        0          0（直接选最高分）
2      23        1          23
3      22        2          44
4      21        3          63
5      20        4          80
6      19        5          95
                    合计     305 次

```

相似度用 **Jaccard**（词集合交集/并集），不用向量——MMR 是重排阶段，目的是去重，词级重叠已足够判断"这两条在说同一件事"，且速度极快。

**tokenCache 优化：** 24条候选在 MMR 开始前全部预分词一次存入缓存，后续 305 次比较复用，避免重复 tokenize。

**分数归一化：** 先把所有分数归一到 ，再做 MMR 计算，避免不同来源分数量纲不同。

### 设计意图

MMR 解决向量检索容易返回"一堆说同一件事的不同表述"的问题。候选池中若有 3 条都说"时区是 Asia/Shanghai"，不开 MMR 会全部召回；开启后只保留最相关的一条，名额让给其他信息。


---

## 八、四种同步触发时机（源码：`manager-sync-ops.ts`）


| 触发方式 | 说明 |
| --- | --- |
| `onSessionStart` | Agent 会话启动时（默认开启） |
| `onSearch` | 每次搜索前增量重索引（默认开启） |
| `watch` | chokidar 文件监听，变更后 1.5 秒去抖动（debounce） |
| `interval` | 定时同步（默认关闭，`intervalMinutes=0`） |


---

## 九、记忆自动写入机制（源码：`session-files.ts`）

### Memory Flush（Compaction 前自动保存）

上下文窗口快满时，系统给 Agent 一个额外对话轮次，提示 Agent 把重要内容写入 `memory/YYYY-MM-DD.md`，在上下文被压缩前抢救信息。

### Session Memory Hook（/new、/reset 时自动保存）


1. 从上一个会话 JSONL 取最后 15 条消息
2. 用 LLM 生成描述性 slug（如 `vendor-pitch`）
3. 存储为 `memory/2026-03-23-vendor-pitch.md`

### (http://MEMORY.md "MEMORY.md") 写入方式总结

没有主动的结构化个人信息提取，完全依赖：


| 方式 | 触发者 |
| --- | --- |
| 用户明确要求 AI 写入 | 用户 |
| 用户自己编辑文件 | 用户 |
| Memory Flush（Compaction 前） | 系统自动 |
| Session Memory Hook（/new 时） | 系统自动 |


---

## 十、两种 Backend（源码：`backend-config.ts`）


| Backend | 说明 |
| --- | --- |
| `builtin`（默认） | 内置 SQLite + sqlite-vec，零配置 |
| `qmd` | 外部进程模式，启动独立 `qmd` 二进制，通过 stdin/stdout JSON 通信，适合企业部署或远程记忆服务 |


---

## 十一、设计亮点总结


| 亮点 | 说明 |
| --- | --- |
| sqlite-vec 降级 | C++ 扩展加载失败自动降级，不崩溃 |
| Embedding Cache | 按内容 hash 缓存，节省重复嵌入费用 |
| Hybrid 搜索 | 向量 0.7 + BM25 0.3，语义与关键词两条腿互补 |
| MMR 多样性重排 | 防止"5条都说同一件事"的问题，来自学术论文设计 |
| 纯规则分词 | 零依赖，7 种语言支持 |
| 本地 Embedding 离线模式 | 完全不依赖外部 API，隐私友好 |

Done

