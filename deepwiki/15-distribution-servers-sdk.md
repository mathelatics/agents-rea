# Chapter 15 — Distribution (OCI), Servers & SDKs

Expert: **DevOps/Ecosystem Expert**. Sources: `pkg/oci/`, `pkg/server/`, `pkg/api/`, `pkg/a2a/`, `pkg/acp/`, `sdk/typescript/`, `cmd/wasm/`, `docs/concepts/distribution`, `docs/guides/go-sdk`.

## OCI distribution (`pkg/oci`)

Agents as **OCI artifacts** — Docker's signature move: run/share agents like images.

```sh
docker agent share push ./agent.yaml myorg/support-agent:v1 --sign --encrypt
docker agent run myorg/support-agent:v1
```

- Bundle = config YAML + referenced files (instructions, skills, prompts) honoring `.agentignore`; metadata labels from `metadata:` section.
- Works with **any OCI registry** (GHCR, Docker Hub, private); `hubauth` handles Docker Hub auth flows.
- Signing (notary-style) and encryption options — supply-chain posture for shared agents.
- Pulled agents can be *run directly* or **imported as sub-agents** (`linkedAgents` remote imports in teams, Ch. 8) — composition across registries.

## Server modes (`serve …`)

| Mode | What it exposes |
|------|----------------|
| `serve api` | REST + SSE: agents list, remote agent sources, sessions CRUD + fork, execution streams, budgets, capabilities, health, OAuth endpoints. SSE supports **gapless reconnection** (eventlog replay in `pkg/server`), idempotent follow-ups, tool-call approval over HTTP. `--listen` lets a CLI attach to a running TUI. Unix sockets + Windows named pipes supported; `httpsec`/`protect` hardening. |
| `serve mcp` | Agents as MCP tools (Ch. 7B) |
| `serve a2a` | Google Agent-to-Agent protocol interop (evolving; documented limits on tool/artifact mapping) |
| `serve acp` | Agent Client Protocol — editor/client integrations |
| `serve chat` | Chat server for UI frontends; `board` = multi-session dashboard |

All modes drive the same `RunStream` — servers are thin protocol translators (architecture payoff, Ch. 2).

## Embedding SDKs

**Go SDK** (`docs/guides/go-sdk`): pick lean provider/tool registries at construction, load config→team→runtime, embed TUI components or use `pkg/embeddedchat` headless, wire custom tools/hooks, JS command expressions (`pkg/js` goja), remote execution backend, capability inspection. AGENTS.md rules apply (clients built once per team load; vary-per-conversation in middleware).

**TypeScript SDK** (`sdk/typescript`): generated from served API types via `task generate-ts`; CI enforces freshness (`check-ts`). Talks to `serve api`.

**WASM** (`cmd/wasm`: web/node/browser targets): compile agent configs into browser-runnable bundles — local models via DMR or API providers make fully client-side agents possible.

**Telemetry everywhere**: OTel traces + `gen_ai.*` semantic conventions (`pkg/telemetry/genai`); anonymous usage documented; cost/token events feed dashboards.

Next: [Chapter 16 — CLI, TUI & Developer Workflow](16-cli-tui-dev-workflow.md)
