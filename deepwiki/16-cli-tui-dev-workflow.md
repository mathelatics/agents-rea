# Chapter 16 — CLI, TUI & Developer Workflow

Expert: **DevOps/Ecosystem Expert**. Sources: `cmd/root/`, `pkg/tui/`, `docs/features/cli` (911 lines), `docs/features/tui`, `Taskfile.yml`, `.github/`, `e2e/`.

## CLI command surface (`docker agent …`)

| Command | Purpose |
|---------|---------|
| `run <config\|oci-ref>` | Main entry: TUI by default; `-a` pick agent; `--exec "prompt"` headless one-shot; flags `--worktree`, `--safety/--yolo`, `--code-mode-tools`, `--flavor`, model overrides |
| `new` | Interactive agent generator (`--model` to skip prompt) |
| `getting-started` | Guided first-run wizard |
| `models` / `toolsets` | Browse catalog info (context windows, pricing; available toolsets) |
| `setup` / `doctor` | Provider auth setup; environment diagnostics |
| `serve {api,mcp,a2a,acp,chat}` | Server modes (Ch. 15) |
| `board` | Multi-session dashboard |
| `share push/pull` | OCI distribution (Ch. 15) |
| `sessions [diff]` | Browse/diff/resume sessions (incl. into worktrees) |
| `eval` | Evaluation runs (Ch. 14) |
| `sandbox` | Sandbox/template management |
| `plans` | Inspect saved plans |
| `debug {auth,oauth,tool}` | Credential and tool debugging |
| `alias` | Named shortcuts for configs/refs |
| `version` / `completion` / self-update (`pkg/selfupdate`) | Utility |

Global OTel env vars documented in `docs/reference/environment-variables`.

## TUI (`pkg/tui`, `pkg/leantui`)

Rich terminal experience driven purely by events: slash commands, file attachments + generated-media display, runtime model switching, editable messages + error recovery, session browser with tabs & titles, usage/cost gauge, keyboard shortcuts, history search, theming, tool-permission dialogs, notifications, plan sidebar, and **recorder/replay** (`pkg/recording`, `pkg/replay`) for deterministic debugging of live streams.

## Developer workflow

- **Taskfile**: `task build` (Buildx → `build/docker-agent`), `test`, `lint` (strict golangci-lint), `format`, `dev` (parallel watch), `cross` (6-platform matrix), `build-image`/`push-image` (`docker/docker-agent`), `generate-ts`.
- **Repo dogfooding**: `golang_developer.yaml` + `.agents/skills/` mean maintainers fix the repo *with the product itself*.
- **Testing philosophy** (AGENTS.md): prefer real code paths over mocks; use exported identifiers; provider checklist ensures new vendors get capability tables + fake-provider e2e tests; e2e suite covers binary/TUI/API/MCP/A2A/proxy.
- **Comment philosophy**: terse *why*-comments only — visible throughout the codebase.
- CI: GitHub Actions (.github/) incl. docs check-links, TS SDK freshness, cross-builds; Dockerfile multi-stage; devcontainer provided.

## Getting oriented fast (recommended path)

```sh
git clone https://github.com/docker/docker-agent && cd docker-agent
task build
./build/docker-agent getting-started        # or: run an example
./build/docker-agent run examples/multi_agent/... 
```

Then re-read Chapters 3→5 with a live session open — config → team load → event stream is the spine that makes every other chapter click.

---

# Epilogue — The Team's Synthesis

Docker Agent's engineering identity, distilled:

1. **Declarative-first** — YAML/HCL + versioned migrations + JSON schema; agents are data, not code.
2. **Event-stream-centric runtime** — every frontend is a dumb consumer of `RunStream`.
3. **Port/adapter purity** around providers — lean, swappable, WASM-capable.
4. **Layered autonomy** — permissions × safety modes × hook phases × sandbox = defense-in-depth against prompt injection.
5. **Prompt-economy obsession** — frozen prefixes, deferred tools, compaction controllers, code-mode batching, shared budget pots.
6. **Distribution-native** — agents as signed OCI artifacts you `docker agent run` like images.

Anyone who reads these 16 chapters against `rough/docker-agent` can navigate, extend (new provider/tool/hook), and operate the system end-to-end. ∎

[← Back to Table of Contents](README.md)
