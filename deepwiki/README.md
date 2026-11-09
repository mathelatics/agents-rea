# 📘 Docker Agent — DeepWiki Book

A complete, source-verified deep study of **docker/docker-agent**, written by a team of experts after cloning the repository to `rough/docker-agent`.

## Table of Contents

| # | Chapter | File | Focus |
|---|---------|------|-------|
| 1 | Project Overview & Big Picture | `01-project-overview.md` | What docker-agent is, why it exists, key features |
| 2 | Architecture & Repository Map | `02-architecture.md` | Go package layout, layering, dependency direction |
| 3 | Configuration System | `03-configuration.md` | YAML/HCL, versions v0–v16, migrations, schema |
| 4 | The Agent Model | `04-agent-model.md` | `pkg/agent`, properties, options pattern |
| 5 | The Runtime & Agentic Loop | `05-runtime-loop.md` | RunStream, events, turns, tool dispatch |
| 6 | Tools Framework & Built-in Toolsets | `06-tools-framework.md` | `pkg/tools`, 30+ builtins |
| 7 | MCP Integration | `07-mcp-integration.md` | Client + server modes, OAuth |
| 8 | Multi-Agent Orchestration | `08-multi-agent.md` | Delegation, handoffs, background agents |
| 9 | Model Providers Layer | `09-providers.md` | 35+ providers, fallbacks, routing |
| 10 | Sessions, Compaction & Memory | `10-sessions-compaction.md` | SQLite store, context management |
| 11 | Permissions, Safety & Sandbox | `11-permissions-safety.md` | Safety modes, hooks, isolation |
| 12 | Hooks System | `12-hooks-system.md` | 30+ lifecycle events |
| 13 | Skills, Harnesses & Code Mode | `13-skills-harness-codemode.md` | On-demand instructions, external CLIs |
| 14 | RAG & Evaluation | `14-rag-evaluation.md` | Retrieval pipelines, `docker agent eval` |
| 15 | Distribution, Servers & SDKs | `15-distribution-servers-sdk.md` | OCI push/pull, serve api/a2a/acp/mcp, WASM, TS SDK |
| 16 | CLI, TUI & Developer Workflow | `16-cli-tui-dev-workflow.md` | Commands, Taskfile, testing |

## Expert Team

- **Architect Lead** — overall structure, layering, entry points
- **Runtime/Loop Expert** — `pkg/runtime`, event streams
- **Tools & MCP Expert** — `pkg/tools`, builtin toolsets, MCP client/server
- **Config/Schema Expert** — `pkg/config`, versioned parsers, JSON schema
- **Providers Expert** — `pkg/model/provider` and adapters
- **Multi-Agent Expert** — `pkg/team`, delegation/handoff semantics
- **Security/Sandbox Expert** — permissions, safety modes, sandbox, protect
- **DevOps/Ecosystem Expert** — OCI distribution, servers, SDKs, CI/dev workflow

All statements are traceable to files in `rough/docker-agent` (commit on `main`, Nov 2026). Start with Chapter 1; each chapter stands alone but builds on prior ones.
