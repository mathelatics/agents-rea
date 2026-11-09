# Chapter 4 — The Agent Model (`pkg/agent`)

Expert: **Runtime/Loop Expert**. Sources: `pkg/agent/agent.go`, `opts.go`, `docs/reference/agents/`.

An `Agent` is the static, fully-wired description of one persona. The struct fields (verbatim from source) reveal the entire capability surface:

## Identity & prompt
- `name`, `description`, `instruction` — description is what *other agents* see when deciding delegation; instruction is the system prompt.
- `addDate`, `addEnvironmentInfo` — auto-injected context blocks.
- `addPromptFiles` (+ depth) — AGENTS.md-style files discovered up the directory tree and prepended to context.
- `redactSecrets` — tri-state (`auto|true|false`); when active it injects three hooks scrubbing secrets from tool args, outgoing messages, and tool outputs before persistence/UI/next LLM call.

## Models
- `models []provider.Provider` — primary list (supports `first_available` semantics).
- `fallbackModels`, `fallbackRetries`, `fallbackCooldown` — failover chain consumed by runtime's `fallback.go` + `cooldown_manager.go`.
- `titleModel` — cheap dedicated model for session titles (`pkg/sessiontitle`).
- `compactionModel` + `compactionThreshold` — dedicated summarizer model and fraction-of-window trigger.
- `modelOverrides` (atomic pointer) — runtime-swappable overrides ("alloy" live model switching without rebuilding the agent).

## Tools & capabilities
- `toolsets []*tools.StartableToolSet` — owned vs shared instances (shared ones are supervised once, referenced by many agents).
- `tools []tools.Tool` — flattened collection computed at build time, with name-collision detection (thread-safe collision collector warns instead of silently shadowing).
- `commands` — per-agent slash commands.
- `harness *latest.HarnessConfig` (+ effort override) — if set, the agent delegates its whole loop to an external coding CLI (Ch. 13).
- `code_mode_tools` — collapse tools into a single JS orchestration tool (Ch. 13).
- `structuredOutput` — lazy-compiled output-enforcement tool that forces schema-valid final answers.
- `cache` — provider-side prompt caching toggle.

## Multi-agent wiring
- `subAgents` — delegation children (clean sub-session context).
- `handoffs` + `forceHandoff` — peer graph routing; force = deterministic handoff on natural stop.
- `Routing` — rule/LLM-based model or agent routing config.

## Loop limits
- `maxIterations` — turn cap.
- `maxConsecutiveToolCalls` — runaway-tool guard.
- `maxOldToolCallTokens`, `maxToolResultTokens` — middle-out truncation knobs (see Ch. 10).
- `numHistoryItems` — sliding history window.

## Observability & safety
- `hooks` — per-agent hook registrations merged with global ones.
- `evaluators` — named evaluators attached to this agent.
- `safety latest.SafetyMode` — per-agent autonomy level (strict/balanced/restricted/autonomous).
- Working-directory factory — per-session/per-worktree cwd resolution (`pkg/tools/workingdir`).

## Construction: functional options

```go
a, err := agent.New("root", prompt,
    agent.WithModels(m1, m2),
    agent.WithToolSets(ts...),
    agent.WithSubAgents(children...),
    agent.WithHooks(hk),
    agent.WithSafety(latest.SafetyBalanced),
)
```

`opts.go` defines dozens of `With…` options; agents form trees (parent→children) which `teamloader` later flattens into a `*team.Team` (Ch. 8). Warnings (collisions, degraded configs) accumulate on the agent thread-safely and surface in startup info events rather than crashing runs.

## Design insight

The agent object is deliberately **static after construction** — everything dynamic (current model choice, budget state, session history) lives in the *session/runtime*, so one agent instance can safely serve many concurrent sessions. This separation is what makes `serve api` fan-out possible.

Next: [Chapter 5 — The Runtime & Agentic Loop](05-runtime-loop.md)
