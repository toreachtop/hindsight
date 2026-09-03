# Hindsight Retain 处理时序流程追踪 Spec

## Why
梳理 Hindsight 的 retain 处理管道中每个阶段的执行类和方法，生成一份从入口到出口的完整执行时序文档，帮助开发者理解代码架构和数据流。

## What Changes
- 创建一份本地文档，记录 retain 6 阶段处理流程中每个阶段对应的类、方法和关键调用链
- 文档覆盖：LLM 叙事事实抽取 → 嵌入向量生成 → 实体解析与归一化 → 图链接构建 → 观点自动强化 → 背景合并
- 文档基于最新代码库（hindsight-api-slim）的实际实现

## Impact
- Affected specs: 无（新增文档）
- Affected code: 无代码变更，仅生成文档

## ADDED Requirements
### Requirement: Retain Pipeline Trace Document
系统 SHALL 生成一份 Markdown 文档，记录 retain 处理管道中每个阶段的执行类和方法，包括关键参数和调用链。

#### Scenario: 文档包含完整的执行时序
- **WHEN** 开发者阅读文档
- **THEN** 可以追踪从 `MemoryEngine.retain_batch_async` 入口到数据库写入的完整调用链
- **AND** 每个阶段标注了对应的源文件、类名、方法名和行号