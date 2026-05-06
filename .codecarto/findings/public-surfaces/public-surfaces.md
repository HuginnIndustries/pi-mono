# Public Surfaces — pi-mono

Catalog of every boundary the outside world touches. Append-only across phases;
each phase adds a dated section.

---

## 2026-05-06 — architecture phase

### Binaries

| Name | Source | Mode |
|---|---|---|
| `pi` | `packages/coding-agent` (`package.json` `bin: pi`) | Node CLI; also distributed as Bun standalone binary for 5 platforms |

### npm Packages

| Package | Version | Distribution | Notes |
|---|---|---|---|
| `@mariozechner/pi-ai` | 0.73.0 | ESM/CJS | Provider abstraction + model catalog. ~70 named exports; `./oauth` subpath for oauth helpers. *(observed fact — `packages/ai/package.json`)* |
| `@mariozechner/pi-agent-core` | 0.73.0 | ESM | Agent runtime. Flat `main`/`types`; no `exports` map. *(observed fact)* |
| `@mariozechner/pi-tui` | 0.73.0 | ESM | Terminal UI. ~12 widgets surfaced through `src/index.ts:1-106`. *(observed fact)* |
| `@mariozechner/pi-web-ui` | 0.73.0 | ESM-only, unbundled `tsc` | Lit web components. Single `"."` and `"./app.css"` exports. ~75 named exports incl. ~30 custom elements. *(observed fact)* |
| `@mariozechner/pi-coding-agent` | 0.73.0 | ESM + Bun binary | Owns the `pi` CLI + extension API. *(observed fact)* |

### Library Public Types (load-bearing)

| Type | Package | File:line | Why load-bearing |
|---|---|---|---|
| `AssistantMessageEvent` (12-variant union) | pi-ai | `src/types.ts:269-281` | Every layer above pi-ai consumes this stream. Terminal events are `done` and `error`. |
| `KnownApi` (provider discriminator) | pi-ai | `src/api-registry.ts` | Selects provider implementation by `model.api` field, not by model id. |
| `Agent` class | pi-agent-core | `src/agent.ts:158` | Core loop driver. |
| `agent-loop` | pi-agent-core | `src/agent-loop.ts` | Iteration semantics. |
| `streamProxy` | pi-agent-core | `src/proxy.ts:116` | Optional HTTP/SSE delivery of agent events to a remote consumer. |
| `Component` interface | pi-tui | `src/tui.ts:17-41` | Widget API. |
| `TUI` class | pi-tui | `src/tui.ts:217` | Render loop driver. |
| `pi-chat-panel` custom element | pi-web-ui | `src/ChatPanel.ts:17` | Flagship browser surface. |

### Network / RPC Interfaces

| Surface | Source | Wire format |
|---|---|---|
| LLM provider HTTPS endpoints | `packages/ai/src/providers/*` | Provider-native (Anthropic Messages, OpenAI Responses, Google Generative AI, Vertex, Bedrock Converse Stream, Mistral Conversations, Azure OpenAI, OpenAI Codex Responses, OpenAI Completions). ~15 OpenAI-compatible vendor URL branches at `openai-completions.ts:1030+`. |
| `streamProxy` server `/api/stream` (optional) | `packages/agent/src/proxy.ts:116` | HTTP POST + SSE; framing `data: {ProxyAssistantMessageEvent}\n`. Event union at `src/proxy.ts:36-57`. *(needs runtime fixture capture — see arch-OQ3)* |
| `rg` / `fd` shell-out | `packages/coding-agent/src/utils/tools-manager.ts` | Auto-downloads ripgrep/fd binaries on first use; network side effect on startup. *(portability hazard)* |
| Browser HTTP `fetch` (model discovery, attachment, document extraction) | `packages/web-ui` | Direct fetch calls; CORS-proxy URL rewriter in `proxy-utils.ts`. |

### File Formats

| Format | Path | Schema location | Notes |
|---|---|---|---|
| Credential store | `~/.pi/agent/auth.json` | `packages/coding-agent/src/core/auth-storage.ts` | Mode 0600 enforced. *(observed fact)* OAuth refresh uses `proper-lockfile`. |
| Settings | `~/.pi/agent/settings.json` | `packages/coding-agent/src/core/config.ts` | User config. |
| Models cache | `~/.pi/agent/models.json` | TBD (defer to protocols) | Model overrides + catalog cache. |
| Session JSONL | `~/.pi/agent/sessions/--<encoded-cwd>--/<ts>_<sessionId>.jsonl` | TBD (defer to protocols, see arch-CF4) | Format version 3, append-only with full-rewrite for branch edits. *(observed fact)* |
| Debug log | `~/.pi/agent/pi-debug.log` | n/a | Diagnostic trace. |

### Browser Custom Elements (subset)

The `pi-web-ui` package registers ~30 custom elements; `pi-chat-panel`
(`src/ChatPanel.ts:17`) is the flagship. Full enumeration deferred to the
contracts phase. *(see arch-CF1 / contracts phase)*

### User-Facing Modes

| Mode | Entry | Notes |
|---|---|---|
| Interactive TUI | `pi` (default) | Differential render via `pi-tui`; synchronized output mode 2026; Kitty/iTerm2 image escapes. |
| Non-interactive RPC (in-process) | `Agent` class | Programmatic driver from another Node process. |
| Headless / scripted | `pi` with flags | Defer enumeration to contracts. |

### Slash Commands & Custom Prompts (project-scoped)

| Path | Purpose |
|---|---|
| `.pi/prompts/cl.md` | Changelog auditor *(observed fact — arch-build extraction)* |
| `.pi/prompts/is.md` | Issue analyzer |
| `.pi/prompts/pr.md` | PR reviewer |
| `.pi/prompts/wr.md` | Wrap-it utility prompt |

These ship as part of the pi-mono repo's own `.pi/` and serve as both
runtime extensions for users running `pi` here AND as canonical examples
for what a project-scoped extension looks like.
