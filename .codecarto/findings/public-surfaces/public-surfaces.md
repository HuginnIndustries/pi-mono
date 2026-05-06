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

---

## 2026-05-06 — contracts phase (append)

Corrections and additions to the architecture-phase enumeration. (Catalog-level detail; primary at `findings/contracts/behavioral-contracts.md` owns the load-bearing claims.)

### Corrections to architecture-phase entries

- **`streamProxy` is a CLIENT, not a server.** The architecture phase's "optional SSE proxy server `/api/stream`" entry is wrong. `pi-agent-core` ships only `streamProxy({ proxyUrl })` which POSTs to a caller-supplied URL and parses SSE responses (`packages/agent/src/proxy.ts:116,152,195-205`). pi-mono ships **no listening server**.
- **`pi-agent-core` `Agent` methods**: the public methods are `prompt(message)` and `continue()` — **not** `run()`. Listener registration is `agent.subscribe(callback)` — **not** `addListener`/`removeListener`.
- **State-anchor env var** is `PI_CODING_AGENT_DIR` (templated `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR` at `packages/coding-agent/src/config.ts:374-380`); `ENV_AGENT_DIR` is the source-internal constant name. Both refer to the same setting.

### CLI Flag Inventory

30+ user-facing flags parsed in `packages/coding-agent/src/cli/args.ts:67-189`. Notable:
- `--session <id>` / `-c` (continue most-recent) / `-r` (resume) / `--no-session` — mutually exclusive with each other and with `--fork` (validated `main.ts:188-202`).
- `--fork <id>` — branches an existing session.
- `--print` — peeks the next arg as message if it's not a flag/`@file`.
- `--prompt-template <path>` — adds a prompt-template search path.
- Unknown `--flags` flow into `unknownFlags` map for extension consumption (`args.ts:167-180`) — extensions can register their own flags.

### Built-in Slash Commands

20 builtins enumerated in `packages/coding-agent/src/core/slash-commands.ts:18-40`. Plus user-defined prompt-template commands discovered via the order: `~/.pi/agent/prompts/` → `<cwd>/.pi/prompts/` → `--prompt-template` paths (`prompt-templates.ts:248-273`). Frontmatter is `---`-fenced YAML; supports `description` and `argument-hint` keys. Filename minus `.md` becomes the command name.

### Browser Custom Element Inventory

~30 custom elements in pi-web-ui (`packages/web-ui/src/index.ts`). Flagship: `<pi-chat-panel>` (`ChatPanel.ts:17`). Configuration flows through `setAgent(agent, config)` — **no HTML attribute projection** (`ChatPanel` uses only `@state()`, no `@property()` decorators). Dispatches **no** custom events. Full element list available in `scratch/contracts-tui-webui.md` §D.

### File Format Updates

- `models.json` — overrides + custom headers. **Headers values can be `!`-prefix shell strings** (mechanism in `packages/coding-agent/src/core/config.ts` `resolveConfigValue`). Treated as a covert RCE channel when the file comes from an untrusted project root. *(Pass 4 finding dsm-RT1.)*
- Session JSONL v3 — defer field-level schema to protocols phase (`con-CF3`).

### Subcommands

`pi` intercepts subcommands before flag parsing (`main.ts:431-437`):
- `pi install` — install pi
- `pi update` — update pi
- `pi config` — interactive config tool
- (Others — defer enumeration to scratch/contracts-cli.md if needed.)

### Modes Observable

| Mode | Trigger | Stdout shape |
|---|---|---|
| Interactive TUI | TTY stdin (default) | TTY paint via pi-tui differential renderer |
| Print mode | `--print` or piped stdin (`main.ts:634` auto-fallback) | Plain text on stdout, exit code 0/non-0 |
| RPC mode | (mode-specific flag) | JSON-RPC over stdio |
