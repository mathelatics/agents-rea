# Chapter 5 — The Runtime & Agentic Loop (`pkg/runtime`)

Expert: **Runtime/Loop Expert**. Sources: `pkg/runtime/loop.go`, `event.go` (1337 lines), `runtime.go`, `tool_dispatch.go`, `toolexec/`.

## LocalRuntime

`LocalRuntime` implements the large `Runtime` interface (`runtime.go`): current-agent tracking per session, toolset lifecycle (statuses, restarts), startup-info emission, pause/resume, steering. It supervises every `StartableToolSet` with exponential backoff (`startable_backoff.go`) — a crashing MCP server is restarted and reported to the UI `/tools` dialog instead of killing the run.

## RunStream — the heart

```go
func (r *LocalRuntime) RunStream(ctx context.Context, sess *session.Session) <-chan Event
```

Per-invocation setup:
1. **Budget install** — per-run cost tracker; sub-sessions (delegations, background agents) share the root wallet, so fan-out cannot multiply spend.
2. **OTel correlation** — session ID seeded into baggage as `gen_ai.conversation.id`; every downstream span (provider calls, MCP, sandbox, hooks) joins the trace.
3. **Non-interactive fast-fail** — background agents, `serve mcp/a2a`, and evals refuse OAuth elicitation immediately instead of hanging forever (issue #3200).

### The turn loop (`for { … }`)

Each iteration is one "turn":

1. **Context assembly** — `turn_start` hooks fire; dynamic context is appended: session-start messages, queued user prompts (steering/followups via `message_queue.go`), structured-output reminders, instruction sources. Crucially, changes **extend** the conversation and never rewrite the **frozen instruction prefix** — preserving provider prompt-cache hits.
2. **`before_llm_call` hooks** — may block the call or rewrite outgoing messages.
3. **Model streaming** — span `runtime.stream` measures model duration only; partial tool-call deltas stream out as `PartialToolCallEvent` and can be retracted (`ToolCallsRetractedEvent`).
4. **Tool dispatch** (`tool_dispatch.go`, `toolexec/`) — caller-bound handlers (transfer_task, handoff, change_model) are bound per call; then the full chain runs: permission/safety evaluation + hook phases (input transform → guard → approval → response transform). Parallelizable calls execute concurrently (`pkg/concurrent`).
5. **Continue conditions** — loop until natural stop, `max_iterations` (`MaxIterationsReachedEvent`), budget exceeded (`BudgetExceededEvent`), hook-driven shutdown, or error.

### Teardown (`finalizeEventChannel`)

- `StreamStopped{reason, finishReason}` is emitted with a **bounded blocking send before slow session-end hooks** (#3074/#4136 ordering fix — frontends learn "stopped" promptly even if teardown hooks hang).
- Closing the channel is the guaranteed terminal signal.
- `on_user_input` fires only for root interactive streams (#4004) so sub-agents don't leak input events.

## Event taxonomy (`event.go`)

All implement `Event{ GetAgentName() string }`; session-scoped ones also implement `SessionScoped` so the PersistenceObserver filters sub-session noise out of parent history. Key events:

| Group | Events |
|-------|--------|
| Lifecycle | `StreamStarted`, `StreamStopped`, `Paused`, `Error(code)` |
| Chat | `UserMessage`, assistant deltas, `ShellOutput`, `Warning` |
| Tools | `PartialToolCall`, `ToolCallsRetracted`, `ToolCall`, `ToolCallConfirmation`, `ToolCallOutput`, `ToolCallResponse`, `ToolsetInfo` |
| Agents | `AgentChoice(+Reasoning/Reset)`, `AgentSwitching`, `AgentInfo`, `TeamInfo`, `BackgroundAgentStatus` |
| Session | `SessionTitle`, `SessionSummary`, `SessionCompaction`, `SessionRecovered`, `PlanChanged` |
| Cost | `TokenUsage`, `AttemptCost`, `BudgetUsage`, `BudgetExceeded` |
| Auth/UX | `Authorization`, `ElicitationRequest`, `ElicitationClosed`, `ToolCallConfirmation` |
| Infra | `ModelFallback`, `MCPInitStarted/Finished`, `RAGIndexingStarted` |

## Other runtime subsystems (file-level map)

- **Routing**: `agent_router.go` — LLM/rule-based routing between agents/models; decisions persisted to session.
- **Failover**: `fallback.go`, `cooldown_manager.go` — sticky cooldowns + exponential backoff across fallback chains, emitting `ModelFallbackEvent`.
- **Compaction**: `compaction_controller.go`, `session_compaction.go`, `native_compaction.go` (provider-side), `background_compaction.go`, plus `compactor/` subpackage (Ch. 10).
- **Interaction**: `streaming.go`, `message_queue.go` (steer mid-stream / followup queue), `pause.go`, `resume.go`, `continuation.go`.
- **Cost**: `attempt_cost.go`, `pricing/` — token→USD accounting feeding budgets.
- **Models**: `model_picker.go`, `model_switcher.go` (change_model tool + alloy overrides).
- **Features**: `rag.go`, `skill_runner.go`, `structured_output.go`, `oauth_elicitation.go`, `elicitation_validation.go`.
- **Subpackages**: `toolexec/` (execution pipeline), `remote/` (remote execution backend), `inspection/` (agent inspector API), `modelcatalog/`, `wire/`, `artifacts/`, `eventmerge/`, `jscommands/` (JS expressions as commands).

## Mental model

> A **run** is a coroutine owning one event channel; a **turn** is one LLM call plus its tool batch; **everything else** (routing, compaction, budgets, hooks, fallbacks) is a policy object consulted inside those two loops.

Next: [Chapter 6 — Tools Framework & Built-in Toolsets](06-tools-framework.md)
