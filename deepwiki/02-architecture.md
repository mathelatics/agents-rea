# Chapter 2 — Architecture & Repository Map

## Top-level layout (verified against `rough/docker-agent`)

```
docker-agent/
├── main.go                 # entry: signal ctx → cmd/root.Execute via docker/cli
├── go.mod / go.sum         # module github.com/docker/docker-agent
├── agent-schema.json       # JSON Schema for configs (164 KB; version up to 16)
├── Taskfile.yml            # dev tasks: build/test/lint/format/dev/cross/generate-ts
├── Dockerfile              # multi-stage production image (docker/docker-agent)
├── golang_developer.yaml   # "dogfooding" agent that develops docker-agent itself
├── AGENTS.md               # contributor guidance (comment philosophy, provider checklist)
├── CHANGELOG.md            # very active release history (~540 KB)
├── cmd/root/               # ~40 cobra commands (run, new, serve {api,mcp,a2a,acp,chat},
│                           #   share push/pull, eval, sandbox, board, sessions, doctor,
│                           #   debug, models, toolsets, setup, alias, version …)
├── pkg/                    # the core: ~105 packages
├── docs/                   # Hugo static site → https://docker.github.io/docker-agent
├── examples/               # runnable YAML configs (multi-agent, RAG, MCP, budgets…)
├── e2e/                    # end-to-end tests (binary, tui, api, mcp, a2a, proxy)
├── sdk/typescript/         # generated TypeScript SDK (task generate-ts)
├── cmd/wasm/               # WASM targets: web / node / browser
└── .agents/skills/         # project skills (triage-prs, bump-config-version, …)
```

## Layering inside `pkg/` (dependency direction: top → bottom)

| Layer | Packages | Role |
|-------|----------|------|
| **Interface hosts** | `tui`, `leantui`, `server`, `api`, `mcp`(server side), `a2a`, `acp`, `board`, `desktop`, `cli` | Consume runtime event streams; own no agent logic |
| **Orchestration** | `runtime` (≈212 files — the heart), `team`, `teamloader`, `chatserver`, `embeddedchat` | Agentic loop, delegation, budgets, compaction control |
| **Domain** | `agent`, `tools` (+ `tools/builtin`, `tools/mcp`, `tools/codemode`), `skills`, `harness`/`codingharness`, `hooks`, `evaluator`/`evaluation`, `rag`, `permissions`, `sandbox`, `compaction`, `session`, `plan`, `memory` | Capabilities & behavior |
| **Abstractions (ports)** | `model/provider` (contracts + factory registry), `chat` (provider-neutral messages/streams), `config` (latest/v0..v15/legacy/hcl), `paths`, `logging` | Stable interfaces everything else depends on |
| **Adapters (implants)** | `model/provider/{openai,anthropic,gemini,vertexai,bedrock,dmr,oaistream,…}`, `tools/mcp` client, `oci`, `remote`, `gateway`, `hubauth`, `chatgpt` | Concrete SDK/protocol integrations |
| **Utilities** | `cache`, `backoff`, `concurrent`, `lrucache`, `memoize`, `fuzzy`, `httpclient`, `telemetry`, `shellpath`, `sqliteutil`, `fsx`, `js` (goja), `protect`, `safety`, `recording`, `replay`, `worktree`, `selfupdate`, `toolinstall` | Shared helpers |

## Three architectural decisions that explain everything else

1. **Ports & adapters around providers.** The doc comment of `pkg/model/provider/provider.go` states explicitly that the provider package *never imports concrete SDKs*. Embedders build lean registries (`providers.NewDefaultRegistry` or custom `NewRegistry`) so unused vendor SDKs drop out of the binary. This keeps the dependency graph clean and binaries trimmable (also enabling WASM builds).

2. **Everything flows through one event channel.** `LocalRuntime.RunStream(ctx, session)` returns `<-chan Event`. The TUI, REST API (SSE), MCP server mode, A2A, chat server, board dashboard, eval harness, and recording/replay are all *dumb consumers* of the same ~40-event taxonomy (`pkg/runtime/event.go`, 1337 lines). Adding a frontend never touches agent logic.

3. **Config is versioned and migratable.** Each schema version has a frozen parser (`pkg/config/v0 … v15`); ordered upgraders chain any older file into `latest.Config` (`pkg/config/legacy.Register`). User configs therefore never break when the schema evolves — they are upgraded at load time.

## Cross-cutting subsystems worth knowing early

- `pkg/hooks` — lifecycle interception used by safety, redaction, context injection (Ch. 12).
- `pkg/permissions` + `pkg/safety` + `pkg/sandbox` — layered autonomy (Ch. 11).
- `pkg/session` — SQLite persistence, branching/forking (Ch. 10).
- `pkg/oci` + `pkg/protect` — distribution with signing/encryption (Ch. 15).
- `pkg/telemetry` — OpenTelemetry traces following `gen_ai.*` semantic conventions; session ID propagated as OTel baggage in every run.

Next: [Chapter 3 — Configuration System](03-configuration.md)
