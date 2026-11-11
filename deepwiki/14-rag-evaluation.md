# Chapter 14 — RAG & Evaluation

Expert: **DevOps/Ecosystem Expert**. Sources: `pkg/rag/`, `pkg/evaluation/`, `cmd/root/eval.go`, `docs/features/retrieval-augmented-generation`, `docs/reference/eval`.

## RAG pipeline (`pkg/rag`)

Named pipelines in config (`rag:` section) consumed by the `rag` toolset:

```
ingest → chunk → embed → store → query → fuse → rerank → prompt
```

| Stage | Implementation |
|-------|----------------|
| Ingestion | `documents.go` — files/dirs with ignore rules, hashing for incremental reindex |
| Chunking | `chunk/` generic splitters + `treesitter/` AST-aware code splitting (functions/blocks as boundaries) |
| Embedding | via `EmbeddingProvider`/`BatchEmbeddingProvider` contracts (Ch. 9) — any configured vendor |
| Storage | `database/` embedded stores; index progress via `RAGIndexingStartedEvent` |
| Retrieval | `strategy/` — BM25 lexical, vector similarity, or both |
| Fusion | `fusion/` — reciprocal-rank-style hybrid merge of lexical+vector results |
| Reranking | `rerank/` via `RerankingProvider` contract (cross-encoder vendors) |
| Prompting | `prompts/` templates wrapping retrieved context for the model |
| Lifecycle | `manager.go` owns build/update/query across sessions; `builder.go` composes the above from config |

Agents call it through the `rag` builtin toolset (search tools per named pipeline), keeping retrieval *outside* the main conversation until needed.

## Evaluation (`docker agent eval`)

Reproducible quality measurement for agents:

- **Inputs**: recorded sessions or question sets replayed against an agent/config non-interactively (same runtime; elicitation fast-fail makes this safe).
- **Assertions**: tool-call accuracy (did it pick right tools?), relevance judgments, output size/format checks.
- **Isolation**: each eval runs **inside containers** (`Dockerfile.template`s per language, `agent_image` build) so filesystem side-effects don't leak and runs are reproducible.
- **LLM-as-judge**: `judge_llm.go` / `judge_container.go` / `judge_factory.go` score outputs using configured `evaluators` (the same evaluator objects power tool_guard hooks and routing agents — one mechanism, three uses).
- **Reporting**: scoring with repeats (variance visible), baseline comparison (`eval_baseline_test`), termination reasons including budget-stop, saved artifacts for diffing runs.
- Telemetry/cost tracked per attempt (`attempt_cost`) so you evaluate *quality per dollar*, not just quality.

## Why these two belong together

Retrieval supplies grounding; evaluation measures whether grounding + tools + prompts actually improved answers. The framework closes that loop with the same config-driven, container-first philosophy as everything else.

Next: [Chapter 15 — Distribution, Servers & SDKs](15-distribution-servers-sdk.md)
