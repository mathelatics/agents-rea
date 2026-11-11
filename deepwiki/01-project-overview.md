# Chapter 1 — Project Overview & Big Picture

## What is docker-agent?

**Docker Agent** (`github.com/docker/docker-agent`) is a Go-based framework from Docker Inc. for **building, running, and sharing AI agents declaratively** — a YAML config plus tools is an agent; no code required. It ships both as a standalone binary and as a **`docker` CLI plugin** (`docker agent …`).

The minimal viable agent (from README):

```yaml
agents:
  root:
    model: openai/gpt-5-mini
    description: A helpful AI assistant
    instruction: |
      You are a knowledgeable assistant that helps users with various tasks.
    toolsets:
      - type: mcp
        ref: docker:duckduckgo
```

```sh
docker agent run agent.yaml
```

## Key features (source-verified)

- **Multi-agent architecture** — teams of specialized agents delegating automatically (sub-agents, handoffs, background parallel agents).
- **Rich tool ecosystem** — ~30 builtin toolsets under `pkg/tools/builtin/` plus any [MCP](https://modelcontextprotocol.io/) server: local stdio processes, remote HTTP servers, or Docker-image based servers via catalog refs (`ref: docker:<name>`).
- **Provider agnostic** — OpenAI, Anthropic, Gemini, Bedrock, Vertex AI, Mistral, xAI, Groq, DeepSeek, Docker Model Runner (local models), OpenRouter, GitHub Copilot, ChatGPT subscription auth, and ~35 more (see `docs/reference/providers/`).
- **Declarative YAML/HCL config** — versioned schema (`version: 16` in `agent-schema.json`, a 164 KB JSON Schema), validated on load, with ordered migration upgraders.
- **Advanced reasoning tools** — builtin `think`, `todo`/plan, `memory`, structured output.
- **RAG** — pluggable retrieval pipelines: BM25, embeddings, hybrid fusion, reranking, tree-sitter-aware chunking (`pkg/rag`).
- **Package & share over any OCI registry** — `docker agent share push` / `share pull`; run an agent like an image: `docker agent run myorg/my-agent:v1`.
- **Serverizable** — expose agents via REST API (`serve api`), as an MCP server (`serve mcp`), Google A2A protocol (`serve a2a`), or ACP (`serve acp`).
- **Dogfooding** — Docker uses it to develop itself: `docker agent run ./golang_developer.yaml` at repo root; project skills live in `.agents/skills/`.

## Entry point anatomy

`main.go` is intentionally tiny:

1. Build a signal-handled `context.Context` (`signal.NotifyContext` on SIGINT/SIGTERM).
2. Call `root.Execute(ctx)` from `cmd/root` — the command tree is constructed using the **Docker CLI library** (`github.com/docker/cli/cli`), so the exact same binary behaves identically as a standalone program or as a `docker` plugin.

## Who reads this book?

- **Operators** → Chapters 3, 11, 16 (config, safety, CLI).
- **Contributors** → Chapters 2, 4–10, 12–14 (internals).
- **Integrators/embedders** → Chapters 9, 15 (providers, servers, SDKs).

Next: [Chapter 2 — Architecture & Repository Map](02-architecture.md)
