# Chapter 6 — Tools Framework & Built-in Toolsets (`pkg/tools`)

Expert: **Tools & MCP Expert**. Sources: `pkg/tools/tools.go`, `pkg/tools/builtin/` (32 dirs), `docs/tools/`.

## Core contracts

```go
type ToolSet interface { Tools(ctx context.Context) ([]Tool, error) }
```

- Toolsets needing lifecycle implement **`StartableToolSet`** — supervised start/stop/restart with backoff by the runtime; status surfaces in the TUI `/tools` dialog.

**`Tool` struct** (verbatim fields):
- `Name`, `Category`, `Description` — what the model sees.
- `Parameters` — JSON Schema for arguments; `OutputSchema` — optional schema for structured results.
- `Annotations` — MCP-style hints (`readOnlyHint`, `destructiveHint`, …) that feed directly into permission/safety classification.
- `Handler` / `RuntimeHandler` — execution funcs (runtime variant gets extra context).
- `AllowRepeatedCalls` — repeat-call guard opt-out.
- `Deferred` + `InCatalog`/`SearchOnly` — deferred loading: tool stays out of the prompt until discovered via native tool-search catalog (token economy, Ch. 13 code-mode synergy).
- `ModelOverride` — per-tool cheaper model (e.g., summarize outputs).

**Call/result types**: `ToolCall` mirrors OpenAI function-call shape (+`ProviderID` to preserve native ids across providers); `ToolCallResult{Output, IsError, Meta, Images/Audios/Documents []MediaContent (base64), StructuredContent}` — multimodal results are first-class.

## Type-safe handlers (a signature feature)

```go
func readTool[Args any](name, desc string, h func(ctx, *Args) (*tools.ToolCallResult, error)) tools.Tool
```

`NewHandler[T]` / `NewRuntimeHandler[T]` unmarshal args with `github.com/docker/aijson`: strict JSON first, then **narrow shape repairs** (stringified arrays, bare scalars where objects expected…). Every repair emits a `tool_input_repaired` telemetry event keyed by (model, tool) — so the team can *track which models break which tools* instead of silently swallowing quirks. This is real-world LLM-robustness engineering.

## Supporting machinery in `pkg/tools`

- `graph.go` — relationships between tools (dependency/affinity edges).
- `catalog.go`, `deferrals.go` — tool-search catalog & deferred materialization.
- `elicitation.go` — a tool can pause and ask the user mid-call (round-trips as `ElicitationRequestEvent`).
- `toon/` — compact serialization format for tool payloads.
- `lifecycle/`, `workingdir/` — startable supervision and per-session cwd resolution.
- `codemode/` — JS orchestration wrapper (Ch. 13).
- `testutil/`, `fakemcp/` — test doubles.

## Builtin toolsets (`pkg/tools/builtin/`)

| Toolset | What it gives agents |
|---------|---------------------|
| `agent` | Spawn **background agents** + wait/recall tools (parallel sub-sessions) |
| `api`, `openapi` | Call REST endpoints; import OpenAPI specs as toolsets |
| `backgroundjobs` | Persistent shell jobs with per-OS command management |
| `calculator` | Safe math evaluation |
| `datetime` | Current time/timezone |
| `deferred` | Catalog search entry point for deferred tools |
| `environment` | Read env vars |
| `fetch`, `openurl` | HTTP GET/web content |
| `file`, `filesystem` | Read/write/edit/search files (diff-based edits) |
| `git` | Status/diff/commit/log operations |
| `handoff` | Peer handoff tool (Ch. 8) |
| `lsp` | Language-server diagnostics for edited files |
| `mcpcatalog` | Discover/browse `docker:` MCP catalog refs |
| `memory` | Persistent key-value memory across sessions (`pkg/memory`) |
| `modelpicker` | Let the model pick among configured models |
| `plan`, `tasks`, `todo` | Planning sidebars & task lists (`PlanChangedEvent`) |
| `rag` | Retrieval over configured RAG corpora (Ch. 14) |
| `random` | Deterministic-ish randomness helper |
| `scheduler` | Cron-like future task scheduling |
| `sessioncontext` | List/read *other* sessions' history |
| `shell` | Run commands — each classified safe/destructive/unknown (feeds Ch. 11) |
| `skills` | Load/invoke skills (Ch. 13) |
| `structuredoutput` | Schema-enforced final answer tool |
| `transfertask` | Delegation to sub-agents (Ch. 8) |
| `think` | Scratchpad reasoning that never enters conversation history |
| `userprompt` | Ask the human a question mid-run |
| `webhook` | Fire outbound webhooks |

Each has a doc page under `docs/tools/NN-name.md` with YAML examples.

## Extension points

Adding a custom toolset in Go = implement `ToolSet` (or `StartableToolSet`) and reference it in embedder wiring; config-only users get new capabilities through MCP servers instead (Ch. 7) — the framework treats builtin and MCP tools identically after collection.

Next: [Chapter 7 — MCP Integration](07-mcp-integration.md)
