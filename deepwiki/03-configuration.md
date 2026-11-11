# Chapter 3 — Configuration System

Expert: **Config/Schema Expert**. Sources: `pkg/config/`, `agent-schema.json`, `docs/configuration/`.

## Config sections (from `pkg/config/latest/types.go` `Config` struct)

| Section | Purpose |
|---------|---------|
| `version` | Integer schema version (currently **16**) |
| `agents` | Map of named agents; root agent = first key or explicit `root` |
| `providers` | Per-provider auth/settings: API keys, base URLs, subscription auth, headers |
| `models` | Named, reusable model definitions (referenced by agents; supports `first_available` lists) |
| `evaluators` | Named LLM-as-judge evaluators (used by evals, routing, tool_guard hooks) |
| `mcps` | Named MCP server definitions (local command, remote URL, or `docker:<catalog-ref>`) |
| `rag` | Named RAG pipeline definitions (chunking, embedding, fusion, rerank) |
| `commands` | Named `/slash` prompt groups users can invoke in the TUI |
| `skills` | Skill sources (local dirs, URLs, names) |
| `toolsets` | Reusable named toolset definitions referenced by agents |
| `metadata` | author/version/description/homepage/license — feeds OCI distribution labels |
| `permissions` | Rule lists: allow / deny / ask patterns (esp. shell commands) |
| `runtime` | Global defaults incl. safety mode, shared behaviors |
| `budget` / `budgets` | Whole-run cost caps; named budget "pots" shared across sub-sessions |
| `flavors` | Named YAML patches applied at load (`--flavor x`) for env-specific variants |

Plus **HCL** as an alternative syntax (see below) and per-agent fields detailed in Ch. 4.

## Loading pipeline (`config.Load(ctx, source, opts...)`)

1. **Source resolution** — file path, URL, OCI image ref, or raw bytes; discovery of default config locations when none given.
2. **Env-var expansion** — `${VAR}` style with warnings collected (`expansion_warnings.go`) rather than hard-failing on missing vars.
3. **Version detection** — read declared `version:`; dispatch to the *frozen* parser for that version (`pkg/config/v0…v15`). Unversioned files go through legacy heuristics.
4. **Ordered upgrades** — `legacy.Register` chains upgraders v0→v1→…→v15; the final step clones v15 through JSON into `latest.Config`. Upgraders preserve semantics while renaming/restructuring.
5. **Reference resolution** — agents opt into shared groups via `use_commands`, `use_skills`, `use_toolsets`, and by-name references to `models`, `mcps`, `rag`, `budgets`; resolution merges them into each agent (`resolveToolsetDefinitions` etc.).
6. **Validation** — against `agent-schema.json` semantics; unknown fields surface as warnings/errors with helpful fuzzy suggestions (`pkg/fuzzy`).

Other notable features:
- **`instruction_file:`** lets long prompts live in separate markdown files — single path or list (concatenated in order). Keeps YAML diff-friendly.
- **Overrides system** (`overrides.go`) — CLI flags and flavor patches layer over file values.
- **Aliases** — `docker agent alias` maps short names to configs/OCI refs.

## HCL support (`docs/configuration/hcl`)

Block-based alternative close to Terraform but simpler: heredocs for instructions, `file()` functions for loading external content, repeated blocks become lists. Parsed by `pkg/config/hcl` into the same internal structures as YAML.

## Practical example (annotated)

```yaml
version: 16
metadata:
  author: acme
  version: 1.0.0
models:
  smart:
    model: anthropic/claude-sonnet-4-5
    params: { temperature: 0.2 }
mcps:
  gh:
    type: docker
    ref: docker:github        # catalog image-based MCP server
permissions:
  allow: ["git status", "git diff"]
  deny:  ["rm -rf /*"]
runtime:
  safety: balanced
agents:
  root:
    model: smart
    instruction_file: ./prompts/root.md
    use_toolsets: [searchers]
    sub_agents: [researcher, writer]
    fallback_models: [openai/gpt-5-mini]
budget:
  max_cost_usd: 2.00
```

## Gotchas the team observed

- Referencing a nonexistent named model/MCP/toolset is a **load-time error**, not runtime — good failure locality.
- `first_available` model lists make one agent portable across environments (e.g., DMR locally, OpenAI in CI) without flavors.
- Flavors patch *before* reference resolution, so a flavor can swap which named model `smart` points to.

Next: [Chapter 4 — The Agent Model](04-agent-model.md)
