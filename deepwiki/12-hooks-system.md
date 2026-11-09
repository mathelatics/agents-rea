# Chapter 12 — Hooks System (`pkg/hooks`)

Expert: **Runtime/Loop Expert**. Sources: `pkg/hooks/*.go`, `pkg/hooks/events/contracts.go`, `docs/reference/hooks` (34-event table).

Hooks are user-supplied interceptors at every lifecycle point, defined in YAML (shell commands with exit-code semantics), JS (goja `pkg/js`), model-handler, evaluator-handler, or builtin Go.

## The 34 events (catalog from `events/contracts.go` drives validation/dispatch/aggregation)

**Tool pipeline (ordered):**
| Event | Semantics |
|-------|-----------|
| `tool_input_transform` | Runs even on auto-approved calls; rewrites tool input *before* guards |
| `pre_tool_use` | Sequential; can block; rewrites tool input |
| `tool_guard` | Mandatory checks; evaluator-backed → OpenAI-style decision |
| `permission_request` | Observes/approves the ask-UI round-trip |
| `on_tool_approval_decision` | Records user's verdicts |
| `post_tool_use` | After execution |
| `tool_response_transform` | Scrub/rewrite output before emission & persistence |

**Conversation/session:** `session_start`, `session_end`, `user_prompt_submit`, `user_steering_messages_submit`, `user_followup_submit`, `turn_start`, `turn_end`, `on_user_input`, `stop`, `notification`, `on_error`, `on_max_iterations`.

**LLM:** `before_llm_call` (sequential; can block; rewrites messages), `after_llm_call`.

**Compaction:** `pre_compact`, `before_compaction` (veto/custom summary), `after_compaction`.

**Agents/team:** `subagent_stop`, `on_agent_switch`, `on_session_resume`, `before_agent_run` / `after_agent_complete` (both can **block and route** to another agent), `worktree_create`.

**Content vetting:** `skill_content_guard` (inspect skill content before delivery), `prompt_file_guard` (AGENTS.md files before injection).

## Execution engine (`executor.go`)

- **Lanes**: under preempt-yolo mode `pre_tool_use` splits into a sequential approval lane vs parallel metadata lane — ordering guarantees where they matter, throughput where they don't.
- **Aggregation**: multiple hooks produce one verdict (any block wins for guards; transforms compose left-to-right).
- **Matchers**: hook registrations scope by tool-name patterns (e.g., only `shell`).
- Global user-level hooks + `hooks.d/` drop-in directory merge with per-agent/per-config hooks.

## Exit-code contract (shell hooks)

Non-zero-with-block-payload ⇒ block; documented defaults per contract distinguish warn-vs-block severity. JS hooks return structured objects instead.

## Builtins shipped as hooks

`add_context` (template rendering), `add_date`, `add_environment_info`, `add_git_status/diff/recent_commits`, `limit_large_tool_results`, `keep_awake`, and the three `redact_secrets` hooks (Ch. 11) — dogfooding the public API internally.

## Why it matters

Hooks are the **extension grammar** of the platform: safety customization, secret scrubbing, context injection, routing overrides, and content vetting are all *the same mechanism*, which keeps the core loop honest (Ch. 5) while giving users total interception coverage.

Next: [Chapter 13 — Skills, Harnesses & Code Mode](13-skills-harness-codemode.md)
