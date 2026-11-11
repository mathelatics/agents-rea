# Chapter 9 — Model Providers Layer (`pkg/model/provider`)

Expert: **Providers Expert**. Sources: `pkg/model/provider/*.go`, provider subpackages, `docs/reference/providers/` (~37 pages), `pkg/chat/`.

## The port contracts

```go
type Provider interface {              // contracts.Provider
    Send(ctx, chat.Request) (<-chan chat.StreamItem, error)  // streaming chat
    Name() string
    // capabilities, close, etc.
}
type EmbeddingProvider  interface { … } // vector embeddings (RAG)
type BatchEmbeddingProvider interface { … }
type RerankingProvider  interface { … } // rerank candidates (RAG fusion)
```

`pkg/chat` defines the **provider-neutral** message/stream/`FinishReason` types that all adapters translate to/from. The provider package itself never imports vendor SDKs (documented invariant) — concrete SDKs live in sibling dirs and register through a factory.

## Registry & factory

- `factory.go`: `NewRegistry(...)`, `EmptyRegistry`, `providers.NewDefaultRegistry` — embedders pick which vendors compile in (lean binaries, WASM-friendly).
- Adapter directories: `openai/`, `anthropic/`, `gemini/`, `vertexai/`, `bedrock/`, `dmr/` (+`dmrmodels` catalog for Docker Model Runner local models), `oaistream/`, `rulebased/`, alias vendors.
- Middleware pattern per request: e.g. `chatgptAuthMiddleware` injects ChatGPT subscription auth per session (subscription auth without API keys); `pkg/chatgpt` implements the OAuth-ish browser flow.

## Vendor breadth (docs enumerate ~37)

OpenAI, Anthropic, Google Gemini/Vertex, AWS Bedrock, Mistral, xAI/Grok, Groq, DeepSeek, Cerebras, Fireworks, Together, NVIDIA, HuggingFace, OpenRouter, GitHub Copilot, ChatGPT/Claude subscription auth, Cloudflare Workers AI & AI Gateway, Vercel AI, Baseten, AtlasCloud, MiniMax, Moonshot, OVHcloud, OpenCode Go/Zen, Local/custom OpenAI-compatible endpoints, Docker Model Runner…

## Cross-cutting behaviors found in code/tests

- **Capability overrides** (`capability_override_test.go`) — force-enable/disable vision/tools/thinking per model string.
- **Thinking-effort cloning** (`clone_thinking.go`) — preserve reasoning params across fallback hops.
- **Model catalog & pricing** — `pkg/modelsdev`, `pkg/modelinfo`, `runtime/modelcatalog`, `runtime/pricing`: context windows, capabilities, $/MTok tables feeding budget accounting and the `docker agent models` command.
- **Fallback chains** — retries/backoff/cooldown with sticky penalties; emits `ModelFallbackEvent`; `first_available` picks at load.
- **Routing** — BM25 lexical matcher over example phrases selects a route model (`docs/configuration/routing`) — cheap classifier-free routing.
- **Runtime switching** — `change_model` caller-bound tool, `modelpicker` toolset, alloy `modelOverrides` atomic swap (Ch. 4).
- **Gateway** — `pkg/gateway`/`modelsgateway` routes calls through Docker infrastructure; `bypass_models_gateway.yaml` example shows opt-out.
- **Prompt caching** — cache toggles per agent; frozen instruction prefix design (Ch. 5) maximizes hit rates; Anthropic/OpenAI cache params mapped in adapters.

## Adding a provider (AGENTS.md checklist distilled)

1. Create adapter pkg implementing `contracts.Provider` translating `chat.Request` ⇄ vendor SDK.
2. Register in factory; add capability table entry & pricing.
3. Honor middleware rules: build clients **once per team load**; per-conversation variation belongs in request middleware, not client state.
4. Add docs page under `docs/reference/providers/`, schema enum entry, e2e fake-provider test.

Next: [Chapter 10 — Sessions, Compaction & Memory](10-sessions-compaction.md)
