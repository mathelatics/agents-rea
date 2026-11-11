# Chapter 8 — Multi-Agent Orchestration (`pkg/team`, `teamloader`)

Expert: **Multi-Agent Expert**. Sources: `pkg/team/team.go`, `pkg/teamloader/`, `pkg/runtime/agent_router.go`, `pkg/tools/builtin/transfertask|handoff|agent`, `docs/concepts/multi-agent`.

## Two orthogonal patterns

| | **Delegation** (`sub_agents`) | **Handoffs** (`handoffs`) |
|---|---|---|
| Topology | Hierarchical parent → child → parent | Peer graph, cycles allowed |
| Session | Child runs in a **sub-session** (isolated history) | Same session; the *active agent* switches |
| Context passed | Clean task description only (`transfer_task(agent, task, expected_output)`) | Full conversation history |
| Control | Parent stays in control, receives result | Target agent takes over the conversation |
| Tool approval | `transfer_task` always auto-approved | `handoff` tool chosen by the model |
| Best for | Specialists + synthesis ("research then write") | Pipelines, conversational routing ("sales → support") |

```yaml
agents:
  root:
    sub_agents: [researcher, writer]   # delegation
    handoffs:   [escalation]           # peer takeover
```

- **`force_handoff`** — deterministic routing: when the agent stops naturally, the runtime *intercepts and hands the conversation to the target without asking the LLM*. Reliable for strict pipelines where model discretion drifts.

## Background agents (`builtin/agent`)

- Fire off **parallel sub-sessions** from tools: results stream back as `BackgroundAgentStatusEvent`s; wait/recall tools collect them.
- Optional **isolated git worktrees per background task** (`background_worktree.go`, `pkg/worktree`) — parallel code edits don't collide on one checkout.
- Budgets are **shared pots**: every sub-session draws from the root run's wallet, so fan-out can't multiply cost (Ch. 5).

## Team building — `teamloader`

Resolves `latest.Config` into a wired `*team.Team`:

```go
type Team struct {
    agents       map[string]*agent.Agent
    evaluators   []*evaluator.Evaluator
    permissions  *permissions.Checker
    linkedAgents map[string]*remote.RemoteAgent   // imported remote agents
    runtimeSafety latest.SafetyMode
    agentConfigs  map[string]latest.AgentConfig
}
```

Pipeline stages observed in files: `expander.go` (inline definitions), `merge.go` (use_* references), `filter.go` (readonly enforcement / selection), `registry.go` (+cache/diff), `defaults.go` (defaults dir), `validate.go`, `readonly_enforcement.go`. Remote agents imported via OCI refs become first-class `linkedAgents` callable like local ones.

## Agent-level routing

- `runtime/agent_router.go` + config `routing:` — rule-based or LLM-judged selection of which agent/model handles input; decisions persisted to the session for replay/debug.
- Hooks `before_agent_run` / `after_agent_complete` can **block or route** to another agent — hook-driven control flow layered on top of declarative wiring.

## Design insights

1. Sub-sessions vs same-session is the fundamental split — it decides context economics (delegation sends a *task*, handoffs carry *history*).
2. Delegation depth is bounded by budgets & maxIterations rather than rigid tree rules — teams can be graphs with cycles safely.
3. `expected_output` in transfer_task contracts the sub-result — synthesis quality depends on prompt discipline here; docs recommend specifying format explicitly.

Next: [Chapter 9 — Model Providers Layer](09-providers.md)
