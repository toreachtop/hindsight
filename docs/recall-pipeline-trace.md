# Hindsight Recall（TEMPR）检索时序流程追踪

> 基于最新代码库 `hindsight-api-slim` 梳理，更新日期：2026-08-03

## 总览

Recall 实现了 **TEMPR（Token-budgeted Multi-Path Retrieval）** 检索架构，核心创新是 **Agent-Optimized Retrieval Interface**：调用方指定 token budget `k`，系统自动选择在 budget 内最相关的记忆组合。

```
查询 Q + Token Budget k
     │
     ▼
┌═══════════════════════════════════════════════════════┐
║  Step 1: 查询嵌入生成                                   ║
║  Step 2: 四路并行检索（semantic/BM25/graph/temporal）     ║
║  Step 3: Reciprocal Rank Fusion (RRF)                  ║
║  Step 4: Neural Cross-Encoder Reranking + Combined Scoring ║
║  Step 5: Token Budget 过滤                              ║
╚═══════════════════════════════════════════════════════╝
```

---

## 入口层：从 API 到检索核心

### 调用链

```
MemoryEngine.recall_async()                    # memory_engine.py:4324
  └─ _search_with_retries()                    # memory_engine.py:4633
       ├─ [Step 1] query embedding
       ├─ [Step 2] retrieve_all_fact_types_parallel()   # retrieval.py:834
       │    ├─ 2a. extract_temporal_constraint()         # temporal_extraction.py:34
       │    ├─ 2b. retrieve_semantic_bm25_combined()     # retrieval.py:122
       │    ├─ 2c. retrieve_temporal_combined()          # retrieval.py:476
       │    └─ 2d. LinkExpansionRetriever.retrieve()     # link_expansion_retrieval.py:124
       ├─ [Step 3] reciprocal_rank_fusion()             # fusion.py:29
       ├─ [Step 4] cross-encoder rerank + apply_combined_scoring()  # reranking.py:58
       └─ [Step 5] _filter_by_token_budget()            # memory_engine.py:5888
```

### 关键类/方法说明

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| API 入口 | `memory_engine.py` | `MemoryEngine.recall_async()` | L4324 | 鉴权、参数校验、sanitize、预算解析 |
| 核心检索 | `memory_engine.py` | `_search_with_retries()` | L4633 | 6 步管道主控，重试与 tracing |
| 并行检索 | `search/retrieval.py` | `retrieve_all_fact_types_parallel()` | L834 | N 种 fact type × 4 路检索的协调器 |
| 结果模型 | `search/types.py` | `ParallelRetrievalResult` | L42 | 单 fact type 的四路结果容器 |
| 多类型结果 | `search/types.py` | `MultiFactTypeRetrievalResult` | L58 | 多 fact type 的聚合结果容器 |

---

## Step 1：查询嵌入生成

### 执行时序

```
_search_with_retries()                                # memory_engine.py:4633
  └─ embedding_utils.generate_embeddings_batch()      # embeddings.py
       └─ input_type="query"
       └─ 使用与 retain 阶段 2 相同的嵌入模型
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 嵌入生成 | `engine/embeddings.py` | `generate_embeddings_batch()` | - | 单条查询生成 384 维向量 |
| 默认模型 | 配置 | `BAAI/bge-small-en-v1.5` | - | 384 维向量 |

---

## Step 2：四路并行检索

### 架构总览

```
retrieve_all_fact_types_parallel()                    # retrieval.py:834
  │
  ├─ [2a] 时序约束提取 (CPU, 先于 DB)
  │   └─ extract_temporal_constraint()               # temporal_extraction.py:34
  │        └─ DateparserQueryAnalyzer.analyze()       # query_analyzer.py:180
  │
  ├─ [2b] 语义 + BM25 + 时序 (共享 1 个 DB 连接)
  │   │
  │   ├─ retrieve_semantic_bm25_combined()           # retrieval.py:122
  │   │   └─ retrieve_semantic_bm25_combined_sql()   # retrieval.py:163
  │   │        ├─ Semantic: HNSW 向量索引 (embedding <=> $1)
  │   │        │   - 每个 fact_type 独立 UNION ALL 臂
  │   │        │   - 5x 过取 (min 100) 补偿 HNSW 近似性
  │   │        │   - ef_search=200
  │   │        └─ BM25: GIN tsvector 全文搜索
  │   │            - pg_bigm / pg_trgm 扩展
  │   │            - 每 fact_type 独立臂
  │   │
  │   └─ retrieve_temporal_combined() (if constraint)  # retrieval.py:476
  │        └─ retrieve_temporal_combined_sql()         # retrieval.py:515
  │             ├─ Entry-point 选择: 时间窗口 + 语义排序
  │             ├─ 覆盖度采样: _select_with_temporal_coverage()
  │             └─ Spreading: 从 entry points 沿时间边扩散
  │
  └─ [2d] 图检索 (每个 fact_type 并行)
       └─ LinkExpansionRetriever.retrieve()           # link_expansion_retrieval.py:124
            └─ _expand_combined() / _expand_observations()
                 └─ 单一 CTE 查询: entity + semantic + causal
```

### 2a. 时序约束提取（通道 4 前置）

```
extract_temporal_constraint()                          # temporal_extraction.py:34
  └─ DateparserQueryAnalyzer.analyze()                # query_analyzer.py:180
       └─ 返回 (start_date, end_date) | None
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 时序提取 | `search/temporal_extraction.py` | `extract_temporal_constraint()` | L34 | 从查询中提取时间范围 |
| 查询分析器 | `query_analyzer.py` | `DateparserQueryAnalyzer` | L180 | 基于 dateparser 的轻量解析器 |
| 抽象基类 | `query_analyzer.py` | `QueryAnalyzer` | L147 | 可插拔的查询分析器接口 |

### 2b. 语义检索（通道 1）

```
s_sem(Q, f) = (v_Q · v_f) / (‖v_Q‖ · ‖v_f‖)
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 组合入口 | `search/retrieval.py` | `retrieve_semantic_bm25_combined()` | L122 | 委托给 memories store |
| SQL 实现 | `search/retrieval.py` | `retrieve_semantic_bm25_combined_sql()` | L163 | UNION ALL 多 fact_type |
| HNSW 索引 | PostgreSQL | `embedding <=> $1::vector` | - | 向量余弦距离排序 |
| 过取策略 | `search/retrieval.py` | `hnsw_fetch = max(limit*5, 100)` | L225 | 补偿 HNSW 近似性 |
| 语义阈值 | 配置 | `semantic_min_similarity` | - | 低于阈值的候选被 SQL 过滤 |

### 2b. 关键词检索（通道 2）

```
BM25 全文搜索 (GIN tsvector)
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 查询分词 | `search/retrieval.py` | `tokenize_query()` | L33 | 小写、去标点、分词 |
| BM25 构建 | SQL dialect | `build_bm25_arm()` | - | 构造 `ts_rank` / `tsquery` 查询 |
| 文本扩展 | 配置 | `text_search_extension` | - | `pg_bigm` 或 `pg_trgm` |
| BM25 阈值 | 配置 | `bm25_min_score` | - | 低于阈值的候选被 SQL 过滤 |
| 最大查询词 | 配置 | `bm25_max_query_terms` | - | 限制查询词数防止性能退化 |

### 2d. 图检索（通道 3）— Spreading Activation

```
A(f_j, t+1) = max[A(f_i, t) · w · δ · μ(ℓ)]
```

**LinkExpansionRetriever** 实现三路并行扩展：

| 扩展类型 | 遍历方式 | 评分公式 | 权重 |
|----------|----------|----------|------|
| **Entity** | `unit_entities` 自连接 (shared entity) | `tanh(count × 0.5)` | ∈ [0, 1] |
| **Semantic** | `memory_links` (precomputed kNN) | 直接使用 link weight | ∈ [0, 1] |
| **Causal** | `memory_links` (单向因果) | 直接使用 link weight | ∈ [0, 1] |

**最终激活分数**（三路加和，∈ [0, 3]）：
```python
# link_expansion_retrieval.py:243-246
score = entity_scores.get(fid, 0.0) + semantic_scores.get(fid, 0.0) + causal_scores.get(fid, 0.0)
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 图检索入口 | `search/link_expansion_retrieval.py` | `LinkExpansionRetriever.retrieve()` | L124 | 种子选择 + 三路扩展 + 合并 |
| 语义种子 | `search/link_expansion_retrieval.py` | `_find_semantic_seeds()` | L46 | HNSW 搜索获取 entry points |
| 三路 CTE | `search/link_expansion_retrieval.py` | `_expand_combined()` | L279 | 单查询：entity + semantic + causal |
| Observation 扩展 | `search/link_expansion_retrieval.py` | `_expand_observations()` | L348 | 通过 source_memory_ids 间接遍历 |
| 抽象接口 | `search/graph_retrieval.py` | `GraphRetriever` | L19 | 可插拔图检索器接口 |
| 种子限制 | `search/link_expansion_retrieval.py` | `GRAPH_SEED_LIMIT = 20` | L43 | 最多 20 个 entry points |
| 实体上限 | 配置 | `link_expansion_per_entity_limit` | - | LATERAL 每实体上限防爆炸 |
| 超时保护 | 配置 | `link_expansion_timeout` | - | 实体扩展超时退化为语义+因果 |

### 2d. 时序检索（通道 4）

```
匹配: [τ_s^f, τ_e^f] ∩ [τ_start, τ_end] ≠ ∅
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 时序入口 | `search/retrieval.py` | `retrieve_temporal_combined()` | L476 | 委托给 memories store |
| SQL 实现 | `search/retrieval.py` | `retrieve_temporal_combined_sql()` | L515 | 时间窗口 + 语义排序 |
| Entry 池大小 | `search/retrieval.py` | `_TEMPORAL_POOL_SIZE` | - | ANN 排序池大小 |
| 覆盖度采样 | `search/retrieval.py` | `_select_with_temporal_coverage()` | - | 防止 entry points 聚集 |
| 时间重叠 | SQL | `occurred_start <= end AND occurred_end >= start` | - | 区间重叠检测 |

---

## Step 3：Reciprocal Rank Fusion（RRF）

### 公式

```
RRF(f) = Σ 1/(k + rank_R(f)),  k=60
         R∈{R_sem, R_bm25, R_graph, R_temp}
```

### 执行时序

```
_search_with_retries()                                # memory_engine.py:4633
  ├─ cap_per_source() 每源截断                         # fusion.py:8
  └─ reciprocal_rank_fusion()                         # fusion.py:29
       └─ 或 interleave_fusion()                      # fusion.py:112 (dedup 模式)
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| RRF 融合 | `search/fusion.py` | `reciprocal_rank_fusion()` | L29 | k=60, 加权汇总 |
| 交错融合 | `search/fusion.py` | `interleave_fusion()` | L112 | 轮询取各臂第 N 名，保证覆盖率 |
| 每源截断 | `search/fusion.py` | `cap_per_source()` | L8 | 防止单臂过度膨胀 |
| 截断上限 | 配置 | `recall_max_candidates_per_source` | - | 每源最大候选数 |
| 输出类型 | `search/types.py` | `MergedCandidate` | - | RRF 分数 + 源排名 + arm_scores |

### RRF vs Interleave

| 模式 | 触发条件 | 特点 |
|------|----------|------|
| RRF（默认） | `reranking != "interleave"` | 基于排名求和，多臂一致高位者自然上浮 |
| Interleave | `reranking == "interleave"` | 轮询取各臂第 N 名，保证每个臂的 top 有机会进入 |

---

## Step 4：Neural Cross-Encoder Reranking

### 执行时序

```
_search_with_retries()                                # memory_engine.py:4633
  │
  ├─ [4.0] Pre-filter: boosted_rrf_score() 排序截断    # recall_boost.py
  │         └─ 保留 top reranker_max_candidates
  │
  ├─ [4.1] Cross-encoder rerank                      # cross_encoder.rerank()
  │         └─ 默认: cross-encoder/ms-marco-MiniLM-L-6-v2
  │         └─ 或 "rrf" / "interleave" passthrough 模式
  │
  ├─ [4.5] Combined Scoring: apply_combined_scoring() # reranking.py:58
  │         └─ combined = CE_norm × recency_boost × temporal_boost × proof_count_boost
  │
  ├─ [4.7] Strategy Boosts: additive_strategy_boost() # recall_boost.py
  │
  ├─ [4.8] prefer_observations dedup
  │
  └─ [4.9] min_scores 过滤
```

### 4.1 Cross-Encoder 重排序

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 重排序调用 | `memory_engine.py` | `reranker_instance.rerank()` | L5084 | 调用 cross-encoder 模型 |
| 默认模型 | 配置 | `cross-encoder/ms-marco-MiniLM-L-6-v2` | - | 384 维 MiniLM |
| Pre-filter | `memory_engine.py` | RRf 排序截断 | L5061 | 保留 top `reranker_max_candidates` |
| 策略加权 | `search/recall_boost.py` | `boosted_rrf_score()` | - | pre-filter 时考虑策略 boost |
| 支持后端 | 配置 | local/tei/cohere/google/flashrank/litellm/... | - | 11 种后端 |

### 4.5 多因子联合评分

```
combined = CE_norm × recency_boost × temporal_boost × proof_count_boost

其中:
  recency_boost     = 1 + α_recency × (recency - 0.5)     # α=0.2, ±10%
  temporal_boost    = 1 + α_temporal × (temporal - 0.5)    # α=0.2, ±10%
  proof_count_boost = 1 + α_proof × (proof_norm - 0.5)     # α=0.1, ±5%
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 联合评分 | `search/reranking.py` | `apply_combined_scoring()` | L58 | 乘法融合，确保信号与基础分成正比 |
| 新鲜度衰减 | `search/reranking.py` | `compute_recency_decay()` | L36 | linear(365d)/exponential(90d half-life)/none |
| Recency alpha | `search/reranking.py` | `_RECENCY_ALPHA = 0.2` | L15 | 最大 ±10% 调整 |
| Temporal alpha | `search/reranking.py` | `_TEMPORAL_ALPHA = 0.2` | L16 | 最大 ±10% 调整 |
| Proof alpha | `search/reranking.py` | `_PROOF_COUNT_ALPHA = 0.1` | L17 | 最大 ±5% 调整 |
| 策略 boost | `search/recall_boost.py` | `additive_strategy_boost()` | - | 可配置的各臂加分 |
| 输出类型 | `search/types.py` | `ScoredResult` | - | CE 分数 + 归一化分数 + weight |

### 4.8 Prefer Observations Dedup

当同时请求 `observation` 和 raw fact types（`world`/`experience`）时，已被 observation 覆盖的 raw facts 会被去重，释放的 slot 回填。

---

## Step 5：Token Budget 过滤

### 执行时序

```
_search_with_retries()                                # memory_engine.py:4633
  └─ _filter_by_token_budget(results, max_tokens)     # memory_engine.py:5888
       └─ tiktoken (cl100k_base) 编码
       └─ 贪心选择: 按 rank 遍历，累计 tokens ≤ max_tokens
       └─ 遇到超预算的 fact 立即停止（不跳过）
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| Token 过滤 | `memory_engine.py` | `_filter_by_token_budget()` | L5888 | 贪心截断，仅计算 text 字段 |
| 编码器 | - | `tiktoken` (`cl100k_base`) | - | OpenAI 兼容 token 计数 |
| 行为 | - | 遇到超预算立即停止 | - | 不跳过，保持排序一致性 |

---

## 完整执行时序图

```
MemoryEngine.recall_async()                              # memory_engine.py:4324
│
├─ _authenticate_tenant()
├─ sanitize_text(query)
├─ _resolve_thinking_budget()
│
└─ _search_with_retries()                                # memory_engine.py:4633
   │
   ├─ [Step 1] embedding_utils.generate_embeddings_batch()
   │   └─ query → 384 维向量 (BAAI/bge-small-en-v1.5)
   │
   ├─ [Step 2] retrieve_all_fact_types_parallel()        # retrieval.py:834
   │   │
   │   ├─ 2a. extract_temporal_constraint()              # temporal_extraction.py:34
   │   │   └─ DateparserQueryAnalyzer.analyze()
   │   │
   │   ├─ [共享连接] acquire_with_retry(pool)
   │   │   ├─ 2b. retrieve_semantic_bm25_combined()      # retrieval.py:122
   │   │   │   └─ UNION ALL (semantic + BM25 arms per fact_type)
   │   │   │       ├─ Semantic: embedding <=> $1 (HNSW, 5x overfetch)
   │   │   │       └─ BM25: ts_rank / tsquery (GIN tsvector)
   │   │   │
   │   │   └─ 2c. retrieve_temporal_combined() (if constraint)  # retrieval.py:476
   │   │       └─ 时间窗口 + 语义排序 + coverage 采样 + spreading
   │   │
   │   └─ 2d. [并行] LinkExpansionRetriever.retrieve() per fact_type
   │       └─ _expand_combined() CTE:
   │           ├─ Entity:   unit_entities 自连接 → tanh(count×0.5)
   │           ├─ Semantic: memory_links → link weight
   │           └─ Causal:   memory_links → link weight
   │           └─ 合并: entity + semantic + causal ∈ [0, 3]
   │
   ├─ [合并] 按 fact_type 聚合结果
   │   └─ cap_per_source() 每源截断                    # fusion.py:8
   │
   ├─ [Step 3] reciprocal_rank_fusion()                 # fusion.py:29
   │   └─ RRF(f) = Σ 1/(60 + rank_R(f)), k=60
   │   └─ 或 interleave_fusion() (dedup 模式)
   │
   ├─ [Step 4] Reranking
   │   ├─ Pre-filter: boosted_rrf_score() 排序截断
   │   ├─ Cross-encoder: reranker_instance.rerank()
   │   ├─ apply_combined_scoring()                      # reranking.py:58
   │   │   └─ CE_norm × recency_boost × temporal_boost × proof_count_boost
   │   ├─ additive_strategy_boost() (可选)
   │   ├─ prefer_observations dedup (可选)
   │   └─ min_scores 过滤 (可选)
   │
   ├─ [Step 5.5] Chunk 获取 (可选)
   │   └─ 按 relevance 顺序获取，token budget 截断
   │
   ├─ [Step 5] _filter_by_token_budget()                # memory_engine.py:5888
   │   └─ 贪心: 累计 tokens ≤ max_tokens, 超预算即停
   │
   └─ [序列化] 返回 RecallResultModel
```

---

## 关键数据类型

| 类型 | 文件 | 说明 |
|------|------|------|
| `RetrievalResult` | `search/types.py` | 单条检索结果（id, text, similarity, bm25_score, activation, ...） |
| `ParallelRetrievalResult` | `search/retrieval.py` | 单 fact type 的四路结果容器 |
| `MultiFactTypeRetrievalResult` | `search/retrieval.py` | 多 fact type 的聚合结果 |
| `MergedCandidate` | `search/types.py` | RRF 融合后的候选（rrf_score, source_ranks, arm_scores） |
| `ScoredResult` | `search/types.py` | 重排序后的结果（cross_encoder_score, weight, recency, temporal） |
| `ArmScores` | `search/types.py` | 各臂的原始分数（semantic, keyword） |
| `GraphRetrievalTimings` | `search/types.py` | 图检索各阶段耗时统计 |
| `SemanticBm25Result` | `search/retrieval.py` | 语义+BM25 联合查询结果（含 graph_seeds） |

## 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `thinking_budget` | 100/300/1000 (LOW/MID/HIGH) | 图遍历节点预算 |
| `max_tokens` | 4096 | 返回 token 上限 |
| `semantic_min_similarity` | 配置 | 语义检索阈值 |
| `bm25_min_score` | 配置 | BM25 检索阈值 |
| `rrf_k` | 60 | RRF 公式常数 |
| `reranker_max_candidates` | 配置 | Cross-encoder 前候选截断 |
| `recall_max_candidates_per_source` | 配置 | 每源最大候选数 |
| `recall_strategy_boosts` | 配置 | 各检索臂的加分策略 |
| `recall_connection_budget` | 配置 | 检索连接池预算 |
| `graph_seed_min_similarity` | 配置 | 图入口种子语义阈值 |
| `link_expansion_per_entity_limit` | 配置 | 每实体扩展上限 |
| `link_expansion_timeout` | 配置 | 实体扩展超时 |
| `temporal_semantic_min_similarity` | 0.1 | 时序检索语义阈值 |
| `recency_decay_function` | `linear` | 新鲜度衰减函数 |
| `recency_decay_linear_window_days` | 365 | 线性衰减窗口 |
| `recency_decay_halflife_days` | 90 | 指数衰减半衰期 |
| `_RECENCY_ALPHA` | 0.2 | 新鲜度 boost 幅度 |
| `_TEMPORAL_ALPHA` | 0.2 | 时序 boost 幅度 |
| `_PROOF_COUNT_ALPHA` | 0.1 | 证据量 boost 幅度 |