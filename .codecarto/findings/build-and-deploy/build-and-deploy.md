# Build and Deploy — pi-mono

Catalog of build pipeline, packaging, and deployment. Append-only across phases;
each phase adds a dated section.

---

## 2026-05-06 — architecture phase

### Workspace Topology

- npm workspaces (root `package.json`).
- 5 published packages under `packages/*`.
- All five at `0.73.0` lockstep.
- `scripts/sync-versions.js:36-45` exits 1 if any version diverges. Consumed by
  `npm run version:*` and `release.mjs`. *(observed fact)*

### Build Order (Sequential, Load-Bearing)

```
tui → ai → agent → coding-agent → web-ui
```

Why ordered:
- `pi-web-ui` uses stock `tsc` while the others use `tsgo`.
- Downstream packages need upstream `dist/*.d.ts`.
- Documented in `README.md:74`.

**Treat as portability hazard** for any reimplementation that flattens or
parallelizes the build. *(observed fact, portability hazard)*

### TypeScript Configuration

- Root `tsconfig.base.json` + `tsconfig.json` (workspace root).
- Per-package `tsconfig.json`.
- Compiler: `tsgo` (Microsoft's Go-based TypeScript compiler) for fast
  incremental checks; stock `tsc` only for `pi-web-ui`. *(observed fact)*

### Linting / Formatting

| Tool | Config | Notes |
|---|---|---|
| Biome 2.3.5 | `biome.json` | Tab indents, 120-char lines; excludes `models.generated.ts`. |
| Pre-commit (Husky) | `.husky/pre-commit` | Runs `npm run check` (format + lint + tsgo + browser smoke). Re-stages files. |

### CI Workflows (`.github/workflows/`)

| Workflow | Trigger | Job |
|---|---|---|
| `ci.yml` | push, PR to main | `npm install`, build, check, test on Node 22 |
| `build-binaries.yml` | tag push, manual dispatch | Bun cross-compile to 5 platforms; upload to GitHub Release |
| `issue-gate.yml` | new issue | Currently in **"refactor mode"** auto-closing all issues until 2026-05-17 (`issue-gate.yml:20`) |
| `pr-gate.yml` | new PR | Auto-closes new-contributor PRs without approval |
| `approve-contributor.yml` | maintainer comment with `lgtm`/`lgtmi` | Adds contributor to `.github/APPROVED_CONTRIBUTORS` |
| `openclaw-gate.yml` | (defer to closer read in protocols/contracts) | Maintainer governance gate |

The contributor-gate trio + `.github/APPROVED_CONTRIBUTORS` text file is a
**complete soft-moderation system**. *(observed fact)*

### Release Process

`scripts/release.mjs`:

- Operator-driven (run locally), not CI-driven.
- Uses operator's `~/.npmrc` to `npm publish` all five packages.
- After tag push, CI's `build-binaries.yml` attaches Bun binaries to the
  GitHub Release.

### Distribution Channels

| Channel | Mechanism | Targets |
|---|---|---|
| npm packages | `release.mjs` publish | 5 packages, ESM (web-ui ESM-only) |
| Bun standalone binaries | `build-binaries.yml` + `scripts/build-binaries.sh` | 5 platforms |
| Browser-importable | `pi-web-ui` ESM + `pi-ai` browser-safe build (Node-only providers like Bedrock dynamic-imported behind gates) | Modern browsers (custom elements, ES2022+) |

### Bun Binary Build Mechanics

- `scripts/build-binaries.sh` runs `bun build --compile` for each platform.
- `koffi` externalized (FFI for Windows VT input).
- Windows `.node` shipped beside the binary.
- `photon_rs_bg.wasm` bundled beside the binary.
- Binary loads `photon_rs_bg.wasm` via a global `fs.readFileSync` monkey-patch
  in `packages/coding-agent/src/utils/photon.ts`. **Brittle vs. photon
  upgrades.** *(portability hazard, routed to defect-scan as arch-CF6)*

### Hermetic Test Runner

`test.sh`:
1. Backs up `~/.pi/agent/auth.json`.
2. Unsets ~30 provider envvars.
3. Runs `npm test` (vitest).
4. Restores `auth.json`.

`pi-test.sh --no-env` does the equivalent for interactive runs.
`pi-test.ps1` is the PowerShell counterpart for Windows. *(observed fact)*

### Other Scripts (`scripts/`)

| Script | Purpose |
|---|---|
| `release.mjs` | Lockstep bump + publish. |
| `sync-versions.js` | Verify lockstep; fail on drift. |
| `profile-coding-agent-node.mjs` | Node profiler harness for the `pi` CLI. |
| `edit-tool-stats.mjs` | Aggregates edit-tool usage stats. |
| `build-binaries.sh` | Bun cross-compile orchestration. |

### AGENTS.md Rules That Affect Build / Analysis Discipline

- No inline imports (`AGENTS.md:14`).
- No `models.generated.ts` edits (line 19).
- Lockstep versioning (line 177).
- Git safety section for parallel agents (lines 196–246) — relevant when
  running multi-agent CodeCartographer sessions.

### Known Build Hazards

- Photon WASM monkey-patch (cited above; carry-forward `arch-CF6`).
- undici timeouts disabled at CLI startup (`coding-agent/src/cli.ts:20`)
  *(defect-scan carry-forward `arch-CF7`)*.
- Build order requires sequential execution; parallelizing breaks `.d.ts`
  generation for downstream packages.
- `rg` / `fd` auto-download is a network side effect on the first `pi` run
  in any new environment.
