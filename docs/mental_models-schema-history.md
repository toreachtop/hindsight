# mental_models 表字段变更历史

> 基于 `hindsight-api-slim/hindsight_api/alembic/versions` 迁移记录梳理
> 整理日期：2026-08-05

## 概述

`mental_models` 表有**两条独立的演化线**，最终在 `t5o6p7q8r9s0` 重命名后合并为当前形态：

- **演化线 A（v4 原始表，已废弃）**：`h3c4d5e6f7g8` 创建 → `p1k2l3m4n5o6` DROP
- **演化线 B（当前表，源于 pinned_reflections）**：`n9i0j1k2l3m4` 创建为 `pinned_reflections` → `p1k2l3m4n5o6` 重命名为 `reflections` → `r3m4n5o6p7q8` 加列 → `t5o6p7q8r9s0` 重命名为 `mental_models` → 后续增删字段

当前线 B 表已脱离 v4 原始表的 schema，成为反射响应的存储表。

---

## 演化线 A：v4 原始 mental_models（已废弃）

### A.1 创建：`h3c4d5e6f7g8_mental_models_v4`（2026-01-08）

**创建初始表**，字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | VARCHAR(64) NOT NULL | 主键（联合） |
| `bank_id` | VARCHAR(64) NOT NULL | 主键（联合），FK→banks |
| `subtype` | VARCHAR(32) NOT NULL | CHECK: structural/emergent/pinned/learned |
| `name` | VARCHAR(256) NOT NULL | 名称 |
| `description` | TEXT NOT NULL | 描述 |
| `entity_id` | UUID | FK→entities |
| `observations` | JSONB DEFAULT `{"observations": []}` | 观察列表 |
| `links` | VARCHAR[] | 链接 |
| `tags` | VARCHAR[] DEFAULT `{}` | 标签 |
| `last_updated` | TIMESTAMPTZ | 最后更新时间 |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | 创建时间 |

- **主键**：`(id, bank_id)`
- **CHECK**：`ck_mental_models_subtype` (structural/emergent/pinned/learned)
- **索引**：idx_mental_models_bank_id, idx_mental_models_subtype, idx_mental_models_entity_id, idx_mental_models_tags (GIN)

### A.2 `m8h9i0j1k2l3_mental_model_id_to_text`（2026-01-19）

**修改字段**：
- `id`: VARCHAR(64) → TEXT（支持长 ID）

### A.3 `j5e6f7g8h9i0_mental_model_versions`（2026-01-16）

**新增字段**：
- `version` INT NOT NULL DEFAULT 0

**附带**：创建 `mental_model_versions` 表（不是 mental_models 本身字段）

### A.4 `k6f7g8h9i0j1_add_directive_subtype`（2026-01-16）

**修改约束**：
- `ck_mental_models_subtype` CHECK 增加 `'directive'` 值

### A.5 `o0j1k2l3m4n5_migrate_mental_models_data`（2026-01-21）

**数据迁移**（不改字段）：
- pinned → pinned_reflections 表
- learned → learnings 表
- 删除非 directive 数据
- DROP `mental_model_versions` 表
- **修改约束**：`ck_mental_models_subtype` 改为只允许 `('directive', 'pinned')`

### A.6 `p1k2l3m4n5o6_new_knowledge_architecture`（2026-01-21）

🔴 **DROP TABLE mental_models CASCADE** — v4 原始表彻底删除。

> 此后演化线 A 终止。当前 `mental_models` 表来自演化线 B。

---

## 演化线 B：pinned_reflections → reflections → mental_models（当前表）

### B.1 创建：`n9i0j1k2l3m4_learnings_and_pinned_reflections`（2026-01-21）

**创建 `pinned_reflections` 表**，字段如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID PRIMARY KEY DEFAULT gen_random_uuid() | 主键 |
| `bank_id` | VARCHAR(64) NOT NULL | FK→banks |
| `name` | VARCHAR(256) NOT NULL | 名称 |
| `source_query` | TEXT NOT NULL | 源查询 |
| `content` | TEXT NOT NULL | 内容 |
| `embedding` | vector(384) | 嵌入向量 |
| `tags` | VARCHAR[] DEFAULT ARRAY[]::VARCHAR[] | 标签 |
| `last_refreshed_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | 最后刷新时间 |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | 创建时间 |
| `search_vector` | tsvector / bm25vector / TEXT | 全文搜索向量（依扩展） |

- **主键**：`id`（单字段，UUID）
- **索引**：idx_pinned_reflections_bank_id, idx_pinned_reflections_embedding, idx_pinned_reflections_tags (GIN), idx_pinned_reflections_text_search

### B.2 `p1k2l3m4n5o6_new_knowledge_architecture`（2026-01-21）

**重命名**：`pinned_reflections` → `reflections`
- 索引、FK 约束一并重命名（idx_reflections_*, fk_reflections_bank_id）

### B.3 `r3m4n5o6p7q8_add_reflect_response_to_reflections`（2026-01-21）

**新增字段**：
- `reflect_response` JSONB — 存储完整 reflect API 响应

### B.4 `t5o6p7q8r9s0_rename_mental_models_to_observations`（2026-01-26）

**重命名**：`reflections` → `mental_models`（当前名称）
- 索引、FK 重命名为 idx_mental_models_*, fk_mental_models_bank_id

### B.5 `u6p7q8r9s0t1_mental_models_text_id`（2026-01-27）

**修改字段**：
- `id`: UUID → TEXT（`USING id::TEXT`）— 支持自定义文本 ID

### B.6 `v7q8r9s0t1u2_add_max_tokens_to_mental_models`（2026-01-27）

**新增字段**：
- `max_tokens` INT NOT NULL DEFAULT 2048 — 内容生成 token 上限
- `trigger` JSONB NOT NULL DEFAULT `{"refresh_after_consolidation": false}` — 触发器配置

### B.7 `w8r9s0t1u2v3_fix_mental_models_pk_isolation`（2026-02-05）

🔴 **关键修复**：主键从 `(id)` 改为 `(bank_id, id)` — 修复 bank 隔离 bug
- 旧主键：`pinned_reflections_pkey` 或 `mental_models_pkey`（仅 id）
- 新主键：`mental_models_pkey`（bank_id, id）

### B.8 `c3d4e5f6g7h8_add_history_to_mental_models`（2026-03-06）

**新增字段**：
- `history` JSONB DEFAULT `[]` — 变更历史数组

### B.9 `a2v3w4x5y6z7_add_last_refreshed_source_query`（2026-04-15）

**新增字段**：
- `last_refreshed_source_query` TEXT — 最近刷新用的源查询（用于 delta 模式检测查询变化）

### B.10 `b3w4x5y6z7a8_add_structured_content_to_mental_models`（2026-04-16）

**新增字段**：
- `structured_content` JSONB（Nullable）— 结构化文档表示（sections/blocks），delta 模式刷新的真实数据源

### B.11 `a7b8c9d0e1f2_split_history_into_own_tables`（2026-06-05）

🔴 **删除字段**：
- `history` JSONB — 移到独立的 `mental_model_history` 表

**原因**：单 JSONB 列无界增长，会撞 Postgres 256MB jsonb 上限（SQLSTATE 54000），且整列+TOAST 每次刷新都重写，破坏 HOT 更新。

**替代方案**：新建 `mental_model_history` 表（id, mental_model_id, bank_id, content JSONB, changed_at），写时按行数上限 DELETE 最旧行。

### B.12 `a1d3f5b7c9e2_widen_remaining_bank_id_to_text`（2026-06-13）

**修改字段**：
- `bank_id`: VARCHAR(64) → TEXT — 修复 78 字符 bank_id 触发 StringDataRightTruncation 500 错误（issue #2106）

### B.13 `d5y6z7a8b9c0_backfill_mental_models_subtype`（2026-04-18，分叉链路）

> 此迁移用于修复从 reflections 重命名链路来的表缺失 v4 字段的问题。仅在分叉路径上执行。

**新增字段**（IF NOT EXISTS）：
- `subtype` VARCHAR(32) NOT NULL DEFAULT 'structural'
- `description` TEXT NOT NULL DEFAULT ''
- `entity_id` UUID
- `observations` JSONB DEFAULT `{"observations": []}`
- `links` VARCHAR[]
- `last_updated` TIMESTAMPTZ

**修改约束**：
- `ck_mental_models_subtype` CHECK: structural/emergent/pinned/learned

### B.14 `86f7a033d372_repair_mental_models_subtype_at_head`（2026-05-14，修复）

> 与 B.13 相同的修复，针对 m3rg3h3ad5f6 头部的部署。idempotent。

---

## 兼容性修复路径说明

由于 v0.5.3 存在分叉头（`8c6fa6f7230b` 合并），部分数据库走了不同的迁移路径，导致 `mental_models` 表存在两种可能的 schema 状态：

| 路径 | 特征 |
|------|------|
| 主路径（B 线） | 来自 pinned_reflections → reflections → mental_models 重命名，**无** subtype/description/entity_id/observations/links/last_updated 字段 |
| 修复路径（B.13/B.14） | 通过 backfill 迁移补齐 v4 字段 |

**最终状态**：在 `86f7a033d372`（head 修复）之后，所有路径的 `mental_models` 表都包含 v4 字段集，subtype 允许 `structural/emergent/pinned/learned`。

---

## 最终字段清单（当前 head 状态）

### PostgreSQL

| 字段 | 类型 | Nullable | 默认值 | 来源迁移 |
|------|------|----------|--------|----------|
| `id` | TEXT | NOT NULL | - | B.1 创建为 UUID，B.5 改为 TEXT |
| `bank_id` | TEXT | NOT NULL | - | B.1 创建为 VARCHAR(64)，B.12 改为 TEXT |
| `name` | VARCHAR(256) | NOT NULL | - | B.1 |
| `source_query` | TEXT | NOT NULL | - | B.1 |
| `content` | TEXT | NOT NULL | - | B.1 |
| `embedding` | vector(384) | NULL | - | B.1 |
| `tags` | VARCHAR[] | NULL | ARRAY[]::VARCHAR[] | B.1 |
| `last_refreshed_at` | TIMESTAMPTZ | NOT NULL | now() | B.1 |
| `created_at` | TIMESTAMPTZ | NOT NULL | now() | B.1 |
| `search_vector` | tsvector / bm25vector / TEXT | NULL | - | B.1（依文本搜索扩展） |
| `reflect_response` | JSONB | NULL | - | B.3 |
| `max_tokens` | INT | NOT NULL | 2048 | B.6 |
| `trigger` | JSONB | NOT NULL | `{"refresh_after_consolidation": false}` | B.6 |
| `history` | JSONB | NULL | `[]` | B.8 新增，B.11 删除 |
| `last_refreshed_source_query` | TEXT | NULL | - | B.9 |
| `structured_content` | JSONB | NULL | - | B.10 |
| `subtype` | VARCHAR(32) | NOT NULL | 'structural' | B.13/B.14（修复路径） |
| `description` | TEXT | NOT NULL | '' | B.13/B.14（修复路径） |
| `entity_id` | UUID | NULL | - | B.13/B.14（修复路径） |
| `observations` | JSONB | NULL | `{"observations": []}` | B.13/B.14（修复路径） |
| `links` | VARCHAR[] | NULL | - | B.13/B.14（修复路径） |
| `last_updated` | TIMESTAMPTZ | NULL | - | B.13/B.14（修复路径） |

### 主键与约束

| 约束 | 定义 | 来源 |
|------|------|------|
| `mental_models_pkey` | PRIMARY KEY (bank_id, id) | B.7（修复 bank 隔离） |
| `fk_mental_models_bank_id` | FOREIGN KEY (bank_id) REFERENCES banks(bank_id) ON DELETE CASCADE | B.4 |
| `ck_mental_models_subtype` | CHECK (subtype IN ('structural','emergent','pinned','learned')) | B.14（head 修复） |

### 索引

| 索引名 | 字段 | 来源 |
|--------|------|------|
| `idx_mental_models_bank_id` | bank_id | B.4 重命名 |
| `idx_mental_models_embedding` | embedding (HNSW/DiskANN/vchord) | B.4 重命名 |
| `idx_mental_models_tags` | tags (GIN) | B.4 重命名 |
| `idx_mental_models_text_search` | search_vector (GIN/BM25) | B.4 重命名 |
| `idx_mental_models_subtype` | (bank_id, subtype) | B.13/B.14 |

### Oracle 23ai（o1a2b3c4d5e6_oracle_baseline）

Oracle 通过 baseline 一次性创建完整表，字段映射：

| PG 字段 | Oracle 字段 | Oracle 类型 |
|---------|------------|-------------|
| id TEXT | id | VARCHAR2(256) |
| bank_id TEXT | bank_id | VARCHAR2(256) |
| subtype | subtype | VARCHAR2(32) |
| name | name | VARCHAR2(256) |
| description | description | CLOB |
| source_query | source_query | CLOB |
| content | content | CLOB |
| embedding | embedding | VECTOR(384, FLOAT32) |
| entity_id | entity_id | RAW(16) |
| observations | observations | CLOB (IS JSON) |
| links | links | CLOB |
| tags | tags | CLOB (JSON array) |
| max_tokens | max_tokens | NUMBER(10) |
| trigger | "trigger" | CLOB (IS JSON) |
| structured_content | structured_content | CLOB (IS JSON) |
| last_refreshed_source_query | last_refreshed_source_query | CLOB |
| reflect_response | reflect_response | CLOB (IS JSON) |
| history | history | CLOB (IS JSON) — Oracle 仍保留 |
| last_refreshed_at | last_refreshed_at | TIMESTAMP WITH TIME ZONE |
| last_updated | last_updated | TIMESTAMP WITH TIME ZONE |
| created_at | created_at | TIMESTAMP WITH TIME ZONE |

Oracle 约束：
- `pk_mental_models`: PRIMARY KEY (id, bank_id)
- `fk_mm_bank`: FOREIGN KEY (bank_id) → banks
- `fk_mm_entity`: FOREIGN KEY (entity_id) → entities
- `chk_mm_subtype`: CHECK (subtype IN ('directive', 'pinned'))

> ⚠️ Oracle 的 subtype CHECK 与 PG 不同：Oracle 允许 `('directive', 'pinned')`，PG head 修复为 `('structural','emergent','pinned','learned')`。

---

## 字段变更时间线总览

| 日期 | 迁移 | 操作 | 字段 |
|------|------|------|------|
| 2026-01-08 | h3c4d5e6f7g8 | CREATE | v4 原始表（演化线 A） |
| 2026-01-16 | j5e6f7g8h9i0 | ADD | `version` INT |
| 2026-01-16 | k6f7g8h9i0j1 | MODIFY | subtype CHECK 加 'directive' |
| 2026-01-19 | m8h9i0j1k2l3 | MODIFY | `id` VARCHAR(64) → TEXT |
| 2026-01-21 | n9i0j1k2l3m4 | CREATE | pinned_reflections 表（演化线 B 起点） |
| 2026-01-21 | o0j1k2l3m4n5 | MODIFY | subtype CHECK 改为 ('directive','pinned') |
| 2026-01-21 | p1k2l3m4n5o6 | DROP | v4 原始 mental_models 表；RENAME pinned_reflections → reflections |
| 2026-01-21 | r3m4n5o6p7q8 | ADD | `reflect_response` JSONB |
| 2026-01-26 | t5o6p7q8r9s0 | RENAME | reflections → mental_models |
| 2026-01-27 | u6p7q8r9s0t1 | MODIFY | `id` UUID → TEXT |
| 2026-01-27 | v7q8r9s0t1u2 | ADD | `max_tokens` INT, `trigger` JSONB |
| 2026-02-05 | w8r9s0t1u2v3 | MODIFY | 主键 (id) → (bank_id, id) |
| 2026-03-06 | c3d4e5f6g7h8 | ADD | `history` JSONB |
| 2026-04-15 | a2v3w4x5y6z7 | ADD | `last_refreshed_source_query` TEXT |
| 2026-04-16 | b3w4x5y6z7a8 | ADD | `structured_content` JSONB |
| 2026-04-18 | d5y6z7a8b9c0 | ADD | subtype/description/entity_id/observations/links/last_updated（修复路径） |
| 2026-05-14 | 86f7a033d372 | ADD | 同上（head 修复，idempotent） |
| 2026-06-05 | a7b8c9d0e1f2 | DROP | `history` JSONB（移到 mental_model_history 表） |
| 2026-06-13 | a1d3f5b7c9e2 | MODIFY | `bank_id` VARCHAR(64) → TEXT |

---

## 字段变更统计

### 演化线 A（v4 原始表，已废弃）

- **创建字段**：11 个
- **新增字段**：1 个（version）
- **修改字段**：1 个（id → TEXT）
- **修改约束**：2 次（subtype CHECK）
- **最终命运**：DROP TABLE

### 演化线 B（当前表）

- **创建字段**（pinned_reflections 起点）：10 个
- **新增字段**：9 个（reflect_response, max_tokens, trigger, history, last_refreshed_source_query, structured_content, subtype, description, entity_id, observations, links, last_updated）
- **删除字段**：1 个（history，移到独立表）
- **修改字段**：2 个（id UUID→TEXT, bank_id VARCHAR→TEXT）
- **修改主键**：1 次（(id) → (bank_id, id)）

### 当前最终字段数

- PostgreSQL：21 个字段（含修复路径补齐的 6 个 v4 字段）
- Oracle：20 个字段（保留 history，无 search_vector 单独列）

---

## 关键设计决策

1. **术语重命名**（2026-01-26）：`reflections` → `mental_models`，同时 `memory_units.fact_type='mental_model'` → `'observation'`。这导致 `mental_models` 表语义从"v4 知识图谱节点"变为"存储的 reflect 响应文档"。

2. **主键隔离修复**（2026-02-05）：原始 `pinned_reflections` 主键仅 `(id)`，导致不同 bank 无法使用相同自定义 ID。改为 `(bank_id, id)` 复合主键修复隔离。

3. **history 列移除**（2026-06-05）：单 JSONB 列无界增长撞 256MB 上限，改为独立 `mental_model_history` 表按行存储，写时按行数上限 DELETE 最旧行。

4. **bank_id 类型统一**（2026-06-13）：VARCHAR(64) → TEXT，修复 78 字符 bank_id 写入失败（issue #2106）。

5. **分叉头修复**（2026-04-18 / 2026-05-14）：v0.5.3 分叉导致部分数据库缺 v4 字段，通过两次 idempotent backfill 修复。

---

## 相关迁移文件清单

### 演化线 A（v4 原始表）
- `h3c4d5e6f7g8_mental_models_v4.py` — CREATE
- `j5e6f7g8h9i0_mental_model_versions.py` — ADD version
- `k6f7g8h9i0j1_add_directive_subtype.py` — MODIFY CHECK
- `m8h9i0j1k2l3_mental_model_id_to_text.py` — MODIFY id
- `o0j1k2l3m4n5_migrate_mental_models_data.py` — MODIFY CHECK + 数据迁移
- `p1k2l3m4n5o6_new_knowledge_architecture.py` — DROP TABLE

### 演化线 B（当前表）
- `n9i0j1k2l3m4_learnings_and_pinned_reflections.py` — CREATE pinned_reflections
- `p1k2l3m4n5o6_new_knowledge_architecture.py` — RENAME → reflections
- `r3m4n5o6p7q8_add_reflect_response_to_reflections.py` — ADD reflect_response
- `t5o6p7q8r9s0_rename_mental_models_to_observations.py` — RENAME → mental_models
- `u6p7q8r9s0t1_mental_models_text_id.py` — MODIFY id
- `v7q8r9s0t1u2_add_max_tokens_to_mental_models.py` — ADD max_tokens, trigger
- `w8r9s0t1u2v3_fix_mental_models_pk_isolation.py` — MODIFY 主键
- `c3d4e5f6g7h8_add_history_to_mental_models.py` — ADD history
- `a2v3w4x5y6z7_add_last_refreshed_source_query.py` — ADD last_refreshed_source_query
- `b3w4x5y6z7a8_add_structured_content_to_mental_models.py` — ADD structured_content
- `a7b8c9d0e1f2_split_history_into_own_tables.py` — DROP history
- `a1d3f5b7c9e2_widen_remaining_bank_id_to_text.py` — MODIFY bank_id

### 修复路径（分叉头）
- `d5y6z7a8b9c0_backfill_mental_models_subtype.py` — backfill v4 字段
- `86f7a033d372_repair_mental_models_subtype_at_head.py` — head 修复

### Oracle baseline
- `o1a2b3c4d5e6_oracle_baseline.py` — 一次性创建完整表

### 相关表（非 mental_models 本身）
- `a7b8c9d0e1f2_split_history_into_own_tables.py` — 创建 `mental_model_history` 表
- `j5e6f7g8h9i0_mental_model_versions.py` — 创建 `mental_model_versions` 表（后被 DROP）
- `f4d1c2b3a5e6_add_scheduled_mental_model_refresh_routine.py` — 创建 `mental_models_with_cron()` 函数
