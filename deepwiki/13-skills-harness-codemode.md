# Chapter 13 — Skills, Coding Harnesses & Code Mode

Expert: **Tools & MCP Expert**. Sources: `pkg/skills/`, `pkg/harness/`, `pkg/codingharness/`, `pkg/tools/codemode/`, `docs/features/skills`, `docs/concepts/coding-agents`.

## Skills (`pkg/skills`)

Portable, discoverable **on-demand instruction packages** (Anthropic "Agent Skills" style):

- A skill = folder with `SKILL.md` + frontmatter (name, description, when-to-use) + optional resources. This very repo uses them: `.agents/skills/triage-prs`, `bump-config-version`, `bump-go-dependencies`.
- Discovery from standard dirs (`.agents/skills/` walking up tree); remote sources: URLs/GitHub archives with caching; config `skills: true | ["local", url, name…]` filters what loads.
- **Progressive disclosure**: only metadata is injected into the system prompt; full instructions load when the model invokes the skill — token economy again.
- Runtime executes skills in isolated sub-sessions (`runtime/skill_runner.go`); `skill_content_guard` hook vets content before delivery (prompt-injection defense for third-party skills).

## Coding harnesses (`pkg/harness` port + `pkg/codingharness` drivers)

Let an agent delegate its *entire inner loop* to an external coding CLI while docker-agent supplies orchestration:

```yaml
agents:
  coder:
    harness: claude-code     # also: codex, opencode, pi, command-code
    effort: high             # per-agent override
```

Provider interface (from `pkg/harness/provider.go`):

```go
type Provider interface {
    Name() string
    Run(ctx, prompt Prompt, handle func(runtime.Event)) (*Result, error)
    Resume(ctx, handle SessionHandle, prompt Prompt, handleFn func(runtime.Event)) (*Result, error)
}
```

Drivers stream text/reasoning/tool events back into the normal event taxonomy (frontends don't know the difference), report Usage (cost/tokens) into budgets, and bridge permissions (`claudepermissions.go`). Value proposition: run Claude subscription auth without API keys, wrap vendor CLIs with your teams, hooks, and OCI distribution.

## Code Mode (`pkg/tools/codemode`)

Collapses N individual tools into ONE meta-tool:

- Tool `run_tools_with_javascript`: wrapped tools become Promise-returning JS functions, documented as TypeScript interfaces inside the tool description.
- Top-level await allowed; `Promise.all([...])` runs independent calls concurrently → **many model round-trips collapse into a single turn** (huge latency/token win for multi-step file/API work).
- Enable per-agent `code_mode_tools: true` or globally `--code-mode-tools`; works alongside deferred tool catalog (Ch. 6). HEAD commit shows ongoing schema-refactor work (maps-clone) in this package.

## Related capabilities

- **Plan/Tasks/Todo** toolsets maintain visible task lists (`PlanChangedEvent` renders TUI sidebar).
- **Structured output** (`builtin/structuredoutput` + `runtime/structured_output.go`): schema-forced final answers via tool-call mode with reminder injection — reliable JSON for pipelines.
- **Webhook / scheduler** toolsets enable outbound automation and time-based agent runs.

Next: [Chapter 14 — RAG & Evaluation](14-rag-evaluation.md)
