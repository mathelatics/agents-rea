# Chapter 11 — Permissions, Safety Modes & Sandbox

Expert: **Security/Sandbox Expert**. Sources: `pkg/permissions/`, `pkg/safety/`, `pkg/sandbox/`, `pkg/protect/`, `pkg/httpsec/`, `docs/concepts/safety`, `docs/configuration/permissions`.

Four concentric layers of autonomy control:

## Layer 1 — Permission rules (`pkg/permissions`)

Declarative lists evaluated **Deny → Allow → Ask**, otherwise fall through to safety mode:

```yaml
permissions:
  allow: ["git status", "git diff *"]
  deny:  ["rm -rf *", "docker rm *"]
  ask:   ["kubectl *"]
```

A **classifier** labels every shell/tool call as:
- `safe` — allowlisted shell verbs (`ls`, `cat`, `pwd`…) or tools annotated read-only (MCP annotations feed this directly);
- `destructive` — recognized dangerous patterns (`rm -rf`, force-push…);
- `unknown` — everything else.

Applies to `shell` and `run_background_job` invocations primarily.

## Layer 2 — Safety modes

| Mode | safe | destructive | unknown | Notes |
|------|------|-------------|---------|-------|
| `strict` | ask | ask | ask | Maximum oversight |
| `balanced` *(default)* | allow | ask | ask | Daily driver |
| `restricted` | allow | **deny** | **deny** | Headless fail-closed (CI/servers) |
| `autonomous` | allow | allow | allow | Legacy `--yolo` |

Selection precedence: `--safety` flag > `safety_policy` on API session create/PATCH > per-agent `safety:` > `runtime.safety` global. In-TUI prompt escalation keys (B/A) let the user shift behavior mid-run. Approvals can be remembered ("always allow this pattern") writing back into the rule set.

## Layer 3 — Hook enforcement (Ch. 12 detail)

`tool_input_transform` → `tool_guard` → `permission_request` → approval UI → `tool_response_transform`. Even auto-approved calls pass transforms (secret redaction runs regardless of mode). Evaluator-backed guards produce OpenAI-style approve/reject verdicts from an LLM judge.

## Layer 4 — Real isolation (`pkg/sandbox`)

Docs are explicit: **restricted mode is a policy boundary, not a security boundary** — for true isolation use sandboxing:
- OS-level: `sandbox/kit`, darwin/linux implementations (seatbelt/namespaces-style confinement).
- **Cloud sandboxes**: `sandbox_cloud.go` + loginkit — remote execution environments; templates & Auto-Kit v3 provisioning.
- Tool execution can also target **remote runtimes** (`runtime/remote`) — agent logic local, side effects elsewhere.

## Supporting protections

- `readonly: true` — filters toolsets/agents to non-mutating tools (enforced in teamloader).
- `agentsignore` files + `.agentignore` for distribution.
- `redact_secrets` tri-state — auto-injects three hooks scrubbing secrets from tool args, outgoing messages, and outputs before persistence/UI/next LLM call (tri-state `auto` detects likely secret presence).
- Server hardening: `pkg/httpsec`, `pkg/servesafety`, `pkg/protect` guard `serve api/mcp/a2a` surfaces (auth, origin checks, resource limits); unix-socket listeners default on some platforms.

## Threat-model takeaway

Docker Agent assumes the **model may be prompt-injected**; therefore: least-privilege defaults (balanced), fail-closed option (restricted), annotation-driven classification, hook-based scrubbing, and optional hard isolation form defense-in-depth rather than one gate.

Next: [Chapter 12 — Hooks System](12-hooks-system.md)
