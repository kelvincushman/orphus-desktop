# Orphus Desktop: design document

| | |
|---|---|
| Status | Draft 0.1, for review |
| Date | 2026-10-04 |
| Requirements | [PRD.md](PRD.md) |
| Clean-room input | [spec/reference-ux-architecture.md](spec/reference-ux-architecture.md) (describes PI-Desktop; do not copy its code) |

This document covers how Orphus Desktop is built (Part A) and how it looks and behaves (Part B). Wherever it follows PI-Desktop's shape, the shape comes from the behavioural spec, not from its code.

---

# Part A: Architecture

## 1. Constraints

1. **The harness is the engine.** Agent loop, tools, providers, compaction, subagents, workflows and System 1 all run in `orphus`. The desktop never imports `@earendil-works/pi-*`, so harness pi upgrades never break the desktop.
2. **One data home.** The desktop and the TUI share `~/.orphus` (or `ORPHUS_CODING_AGENT_DIR`). The desktop adds its own state under `~/.orphus/desktop/`.
3. **Least privilege per process.** The renderer is unprivileged. Secrets live in one process. Plugins run in their own sandboxed processes.
4. **Contracts first.** Every process boundary has a versioned, schema-typed protocol in `packages/protocol`, written before the code that uses it.
5. **MIT and clean-room** (see [CLEANROOM.md](../CLEANROOM.md)).

## 2. Process topology

```
┌───────────────────────────────────────────────────────────────────────┐
│ Renderer (React, sandboxed, contextIsolation, no Node, strict CSP)    │
│   preload: one allowlisted bridge { invoke, on }                      │
└───────────────▲───────────────────────────────────────────────────────┘
                │ Electron IPC (allowlisted channels, schema-checked)
┌───────────────┴───────────────────────────────────────────────────────┐
│ Main (thin orchestrator, TypeScript)                                  │
│  windows · tray · menus · updater · IPC router · supervisor           │
│  OAuth flows · plugin broker · remote-host router · voice adapters    │
└───┬───────────────────┬──────────────────────┬──────────────────┬─────┘
    │ stdio JSON-RPC    │ stdio JSONL (RPC)    │ unix sockets     │ utilityProcess
    ▼                   ▼                      ▼                  ▼
┌──────────────┐  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────┐
│ host-core    │  │ orphus --mode rpc │  │ Orphus brokers   │  │ plugin hosts │
│ (Rust)       │  │ one per session / │  │ roundtable.sock  │  │ one per      │
│ SQLite index │  │ role              │  │ intercom.sock    │  │ plugin,      │
│ secrets      │  │  + desktop-bridge │  │ (owned by the    │  │ sandboxed    │
│ policy       │◄─┼── extension ──────┼─►│  harness)        │  │              │
│ review/audit │  │  (tool gate,      │  └──────────────────┘  └──────────────┘
│ queue/sched. │  │   diff capture)   │
└──────────────┘  └───────────────────┘
        ▲
        │ HTTP (configured)          Optional: loopback MCP control server (Main),
  System 1 endpoint                  headless host on a remote box (SSH tunnel + WS)
```

### 2.1 Responsibilities

| Process | Owns | Never does |
|---|---|---|
| **Renderer** | UI, view state, optimistic projections | Touch disk, the network or secrets |
| **Main** | Window lifecycle, IPC routing, supervising every child, OAuth browser flows, plugin broker, update, tray, deep links | Hold secrets beyond a single request; business logic |
| **host-core** (Rust) | SQLite index (sessions, projects, providers, queue, schedules, receipts index, audit, inbox, usage); secrets (keychain + encrypted fallback); permission policy; workspace sandbox checks; review snapshots and rollback; admission limits; config sync | Run models or agent loops |
| **Engine** (`orphus --mode rpc`) | Agent loop, tools, providers, compaction, subagents, workflows, System 1 calls, session JSONL | Prompt the user directly. All UI goes through `extension_ui_request` to Main |
| **desktop-bridge extension** (TypeScript, loaded into each engine) | Asks host-core about every `tool_call` (allow, deny, or ask the user); records before/after for review; emits audit; enforces the sandbox decision | Hold policy itself; it only enforces what host-core answers |
| **Brokers** | Rooms, intercom (harness-owned processes) | n/a |
| **Plugin host** | One plugin's main module | Reach anything except through the broker |

### 2.2 Why this shape

- **Rust host-core** mirrors the privilege split proven in the reference app, and it fits Orphus's existing `crates/`. One writer for SQLite. Secrets never enter Node processes that run third-party code. *Alternative considered:* host logic in Main (TypeScript). Faster to build, but Main would then hold secrets and policy next to the plugin broker. Rejected for v1 (PRD Q2).
- **Engine as a subprocess** (`orphus --mode rpc`) rather than the in-process SDK. The RPC contract is documented and stable. The engine binary is Bun-compiled and self-contained, which avoids rebuilding Rust natives for Electron's Node ABI. A crash takes down one session, not the app. *Cost:* about 60–120 MB per live session. Mitigation: idle sessions are parked (process exits, state is in JSONL) and resumed on demand (PRD Q1).
- **Bridge extension** rather than reimplementing tools. Orphus extensions can block `tool_call` and transform `tool_result`. That is enough to make the harness's own tools obey desktop policy without forking them.

## 3. Repository layout

```
orphus-desktop/
  apps/
    desktop/            Electron app: main/, preload/, renderer/ (React)
    host/               headless host for remote boxes (Node, no UI)
  crates/
    host-core/          Rust: SQLite, secrets, policy, review, sync
  packages/
    protocol/           schemas + types for every boundary (TypeBox), versioned
    engine-client/      typed RPC client for `orphus --mode rpc`
    desktop-bridge/     the Orphus extension loaded into every engine
    broker-client/      roundtable + intercom socket clients (length-prefixed JSON)
    orphus-data/        readers for receipts, run artifacts, fleets, roles manifests
    plugin-sdk/         public SDK for plugin authors (no deps)
    plugin-devkit/      `orphus-plugin` CLI: init, check, pack, publish
    ui/                 design tokens + component library
    i18n/               string catalog (English source)
  docs/                 PRD, DESIGN, spec/, ADRs (docs/adr/NNNN-*.md)
```

Tooling follows the Orphus harness: npm workspaces (with the harness's `.npmrc` supply-chain gates), TypeScript strict, Biome, Vitest, cargo for `crates/`, electron-builder for packaging, and Playwright-for-Electron for end-to-end tests.

## 4. Contracts

| Boundary | Transport | Contract |
|---|---|---|
| Renderer ↔ Main | Electron IPC | `protocol/ipc`: a channel allowlist, one schema per channel, checked in both preload and Main |
| Main ↔ host-core | stdio, newline-delimited JSON-RPC 2.0 | `protocol/host`: version handshake (exact match, else a recovery screen), deadlines per method, 64 MiB frame cap |
| Main ↔ engine | stdio JSONL (Orphus RPC) | Owned by the harness (`packages/coding-agent/docs/rpc.md`). Pinned by minimum `orphus` version. Strict LF framing; never use Node `readline` (it splits on U+2028/2029) |
| bridge ↔ host-core | via Main: the bridge sends `extension_ui_request`-style custom messages over the engine's stdio, and Main relays them to host-core | `protocol/bridge`: `gate.request{session, tool, args, cwd}` → `{decision: allow|deny|ask, reason}`; `review.record{…}`; `audit.append{…}` |
| Main ↔ brokers | unix socket / named pipe, 4-byte big-endian length + JSON | Owned by the harness (`packages/roundtable/types.ts`, intercom) |
| Main ↔ plugin host | utilityProcess MessagePort | `protocol/plugin`: capability-scoped API, typed errors |
| Main ↔ remote host | WebSocket over SSH tunnel, JSON-RPC with event cursors | `protocol/remote`: roles viewer/controller/approver/owner; resume from a cursor |

The engine RPC is the contract we do not own. `engine-client` wraps it, records the `orphus` version, and turns RPC events into the desktop's normalised session events. That lets us absorb harness changes in one place.

## 5. Engine integration

### 5.1 Lifecycle

- **Spawn:** `orphus --mode rpc --name <role?> --session <path?>`, with `cwd` set to the project root and `ORPHUS_CODING_AGENT_DIR` passed through. The bridge extension loads with `-e <bundled path>` (or the equivalent flag).
- **Supervisor:** restart with backoff, at most 3 restarts in 2 minutes per session, then the session is marked failed. When the engine exits mid-turn, the partial reply is promoted to an "aborted" row. Nothing replays.
- **Park and resume:** an engine idle for N minutes (default 10), with nothing queued and no room membership, exits. The next prompt respawns it on the same session file.
- **Version gate:** `orphus --version` must be ≥ the pinned minimum, otherwise first-run guidance offers to install or update.

### 5.2 Mapping parity features onto the harness

| Desktop feature | Harness mechanism |
|---|---|
| Prompt, queue, steer, stop | RPC `prompt` (`streamingBehavior: steer|followUp`), `steer`, `follow_up`, `abort`. The durable queue lives in host-core and is released turn by turn |
| Fork / branch / regenerate | RPC `fork`, `clone`, `get_tree`, `switch_session`. Variants are tracked in host-core |
| Model, thinking, compaction | `set_model`, `cycle_model`, thinking level, `compact`, `set_auto_compaction`; events `compaction_*` |
| Questions / confirms / selects | `extension_ui_request` → native cards → response on stdin |
| Slash menu | `get_commands` merged with desktop built-ins |
| Context ring and inspector | `get_session_stats`, usage on `message_*` events |
| Permission cards | bridge `tool_call` → host-core policy → `ask` → card → decision |
| Review and rollback | bridge snapshots before/after into host-core; rollback is executed by host-core, guarded by a post-edit hash |
| Plan / Goal | Orphus plan mode, and the Goal workflow via the `workflow` tool/command (structured once H-03 lands) |
| Subagents | `@orphus/subagents` events; delegate transcripts opened read-only |

### 5.3 desktop-bridge extension

A small Orphus extension, bundled with the app and loaded into every engine the desktop spawns:

1. On `session_start`, handshakes with the host (session id, capabilities).
2. On `tool_call`, classifies the call (read/search, write/edit, shell, MCP, plugin, delegate, external path). Asks the host and waits. Returns `{block:true, reason}` on deny. Pure reads inside the workspace short-circuit locally under a policy the host pushed, to avoid round-trips.
3. On `tool_result` for mutations, sends the path, before-hash and after-hash, and the diff to the host for review.
4. Emits audit rows. Never logs file contents.

If the bridge is missing or fails, the engine **fails closed**: the desktop refuses to send prompts to an engine whose handshake did not complete.

## 6. Orchestration integration

| Surface | Source | Desktop module |
|---|---|---|
| Rooms list, room transcript, pushes | roundtable broker socket (`register`, `rooms`, `fetch`, `activity`) | `broker-client` → `rooms` store |
| "What they saw" lens | re-runs the harness's deterministic digest over the fetched messages, using the member's cursor and the configured budgets | `orphus-data/digest` (the harness algorithm, imported as an MIT dependency, not reimplemented) |
| Operator posts | `join` + `post` as `operator` (observer once H-01 lands) | `rooms` |
| Fleets | blueprint files plus `fleet` tool (`list|get|validate`), `/fleet <name> <task>` | `fleets` |
| Roles | `orphus-roles --format json` → the desktop spawns each role itself | `roles` |
| Presence | intercom `list` (status idle/thinking/tool:x) | `presence` |
| Goal runs | run artifacts under `~/.orphus/workflows/runs/<id>/`, `workflow` tool with `format: json`, later H-03 subscriptions | `runs` |

The broker auto-exits 5 s after its last client leaves. While the room view is open the desktop holds a connection, and it says so in the room header ("Desktop is keeping this room alive").

## 7. System 1 / System 2 integration

```
            Goal step (tier / review / verify)
                         │
                 ┌───────▼────────┐  adapter: null | local | llm-wrapper |
                 │   System 1     │  typesafe | orphus-sys1 (H-04)
                 │  closed Q →    │
                 │  p, confidence │──► receipt (JSONL, per run)
                 └───┬────────┬───┘           │
          conf ≥ θ & │        │ abstain /     │ desktop tails + indexes
          denies     │        │ below θ       ▼
                     ▼        ▼          Decisions view, escalation chips,
               withhold /   System 2     calibration health
               fail step    (full model     │
                            turn, reviewers,│ harvest (systemone-labels)
                            room debate)    ▼
                                        labelled JSONL ──► recalibrate / retrain S1
                                                         (outside the app)
```

- **Config** is written to the harness's own config (`.orphus/extensions/workflow/config.json` or the user equivalent). The desktop never invents a parallel settings store. API keys go to the keychain and reach the engine as environment variables at spawn time.
- **Receipts** are tailed from `turn-N-systemone-receipts.jsonl` and indexed in host-core (run, surface, time) for fast filtering. The JSONL stays the source of truth.
- **Escalation trace:** a step whose receipt says `abstain:true`, or is below threshold, is marked "S2 decided". A step S1 denied is marked "S1 withheld/failed". Steps without a receipt are plain S2.
- **Calibration health** is computed locally from receipts plus harvested outcome labels: ECE, Brier, and a reliability diagram in 10 bins.
- **Distillation** runs the harness harvest script as a child process and previews its output. Export never includes receipts whose adapter is on the exclusion list.

## 8. Persistence

`~/.orphus/desktop/` (overridable with `ORPHUS_DESKTOP_DATA`):

| Path | Contents |
|---|---|
| `index.db` | SQLite (WAL): projects, provider rows (secret refs only), session index, queue, variants, schedules and runs, receipts index, notifications, audit, usage |
| `secrets/` | Encrypted fallback store (only when no keychain is available, with a one-time warning) |
| `review/` | Before-snapshots for rollback, content-addressed |
| `attachments/` | Content-addressed blobs |
| `plugins/` | Installed plugins, per-plugin data and logs |
| `rooms/` | Exported room transcripts (until H-02) |
| `logs/`, `crash/`, `cache/` | |

Session transcripts stay in the harness's session JSONL (`~/.orphus/agent/sessions/`), so the TUI and the desktop see one history. The desktop never rewrites harness files except through RPC. Schema migrations take a backup first. A database written by a newer app version blocks startup with a named recovery screen.

## 9. Security

- **Renderer:** contextIsolation, sandbox, nodeIntegration off, strict CSP, no `window.open` and no in-app navigation, only http/https/mailto handed to the OS, sanitised Markdown/Mermaid/SVG.
- **Electron fuses:** RunAsNode off (the engine is a separate binary, so this costs nothing), cookie encryption on, ASAR integrity on, `only-load-app-from-asar` on.
- **Secrets:** OS keychain via libsecret (Secret Service), macOS Keychain or Windows DPAPI. Encrypted file fallback with a warning. The renderer sees only "configured". Engines get keys as environment variables for their own process only.
- **Workspace policy:** path resolution with symlink re-check, an outside-workspace gate, a denylist (VCS object store, private keys and certs, `.env*` except example/template files, credential JSON, token-bearing rc files), ignore layers, and per-session scratch.
- **Optional OS sandbox for shell tools:** bubblewrap on Linux and `sandbox-exec` on macOS, run by the bridge as a wrapper for the engine's shell tool. Off by default, P2.
- **Plugins (an improvement over the reference):** see §10.
- **Remote hosts:** SSH only, pairing token exchanged for a device token kept in the keychain, approvals expire after 30 minutes, a permission ceiling for remote-started turns.

## 10. Plugin runtime

The contribution types and permission catalogue match the parity spec (inventory §24). Differences:

| Topic | Design |
|---|---|
| Isolation | Each plugin runs in an Electron `utilityProcess` started with the **Node permission model** (`--permission`, with `--allow-fs-read` and `--allow-fs-write` scoped to the plugin dir and its granted globs, no child processes, no workers, no addons unless granted). So raw `fs`/`net` cannot bypass the broker. Network goes only through the broker's allowlisted fetch. |
| Resources | Memory cap per plugin (`--max-old-space-size`), CPU watchdog (heartbeat; a stuck plugin is killed and restarted with backoff). |
| Trusted extensions | These are **Orphus extensions**, loaded into the engine with a separate high-risk grant. Local/dev only until publisher signatures exist. |
| Panels | Sandboxed renderer partitions per plugin, the bridge object only, no popups or webviews. |
| Package | `.orphus-plugin`: a stored ZIP with the manifest at the root, no symlinks or traversal, ≤ 2,000 files and 50 MiB. SHA-256 is verified against the catalog. ed25519 publisher signatures are planned (P1 after M5). |
| Marketplace | A GitHub-hosted catalog repo with JSON index and releases as downloads. No device identifiers sent. |

## 11. Remote hosts

`apps/host` is a headless Node build of Main's session machinery plus host-core and the engine, run on a remote Linux box. The desktop installs it over SSH (system `ssh`, so the user's config, agent and jump hosts work). Bootstrap and roles follow the parity spec (§38).

Additions for Orphus: rooms and fleets on the remote box are mirrored by tunnelling the broker sockets over the same SSH connection. Remote sessions also appear in intercom presence.

## 12. Testing

- **Unit:** Vitest (TypeScript) and cargo test (Rust).
- **Contract tests:** `engine-client` runs against a pinned `orphus` binary in CI (RPC golden transcripts). `broker-client` runs against a real broker.
- **E2E:** Playwright for Electron, covering journeys 1–5 from the PRD, with fake providers (a scripted OpenAI-compatible server).
- **Fault injection:** kill host-core, the engine or a plugin mid-turn, and assert no data loss and no replay.
- **Accessibility:** axe checks in E2E, plus a manual screen-reader pass before each milestone.
- **Clean-room check:** a CI job rejects a PR if it adds files matching PI-Desktop source fingerprints (an n-gram overlap check against a hashed corpus, which stores no source text).

## 13. Packaging and release

electron-builder targets: macOS DMG/ZIP (arm64, x64), Windows NSIS/ZIP (x64), Linux AppImage/deb/rpm (x64, arm64), plus an AUR `orphus-desktop-bin` PKGBUILD. The `orphus` harness is **not bundled** in v1: it is detected or installed on first run so it can update independently. A bundled fallback is an option for M7. Auto-update covers macOS, NSIS and AppImage. Other formats get a notice. Releases come from tag-triggered CI with checksums and signing where configured.

---

# Part B: Experience design

## 14. Information architecture

```
Window
├── Sidebar
│   ├── Pinned
│   ├── Rooms            ← new: live rooms, unread, member count
│   ├── Fleets           ← new: running fleets (expand → members)
│   ├── Sessions (temporary chats)
│   ├── Projects → sessions (role badge if --name)
│   └── Footer: Settings · Extensions · Scheduled · Runs · Notifications · Profile
├── Main pane (destinations)
│   ├── Home / Chat
│   ├── Room             ← new (also openable as a work-panel tab)
│   ├── Runs             ← new: Goal & workflow runs list
│   ├── Scheduled
│   ├── Extensions
│   └── Settings (full-window)
└── Work panel tabs
    Review · Files · Browser · Terminal · Plan/Goal artifact · Subagent ·
    Room · Fleet board · Goal graph · Decisions · Inspector · plugin views
```

Settings groups: **Preferences** (General, Appearance, Shortcuts, Voice) · **Agent** (Models, Instructions, Skills, MCP, Subagents) · **Orchestration** (Fleets, Roles, Rooms, Intercom) · **Decisions** (System 1, System 2, Refine) · **Workspace** (Projects, Storage, Sync, Remote hosts) · **System** (Harness, Network, Updates, Info) · **Extensions**.

## 15. Layout

- Three columns, as the parity spec describes: a resizable sidebar (default 275 px, 240–520, collapses to an icon rail), the main pane with a centred, draggable reading band (default 760 px), and the work panel (closed by default, min 244 px, default 360 px). The chat keeps a 450 px floor, and the sidebar auto-collapses first.
- One shared toolbar height across all chrome. No status bar: live state lives in the composer, the sidebar markers and the transcript.
- On Linux the window is frameless with drawn controls. Under tiling compositors (Hyprland) the drawn controls hide and the app draws no drop shadow. It follows `XDG_CURRENT_DESKTOP` with an override in Settings.

## 16. Key screens

### 16.1 Room

```
┌ #design · 4 members · Desktop keeping room alive ───────── [What they saw ▾] ┐
│  planner  (anthropic/opus)   ▸ cursor @ 41                                   │
│  critic   (openai/gpt)       ▸ cursor @ 38                                   │
│  reviewer (xai/grok)         ▸ cursor @ 44                                   │
│  operator (you)                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ 39  critic    The migration plan ignores the two writers on…                 │
│ 40  planner   ↳39  Agreed. Splitting into …                                  │
│ ── critic has read to here ───────────────────────────────────────────────── │
│ 41  operator  Constraint: no schema change before Friday.                    │
│ 44  reviewer  FINAL: ship plan B with the dual-write shim.   ◆ decision      │
├──────────────────────────────────────────────────────────────────────────────┤
│ [post as operator…]                                        @role  ↵ post     │
└──────────────────────────────────────────────────────────────────────────────┘
```

"What they saw" swaps the transcript for the selected member's digest: the newest messages verbatim, then headlines, then a collapsed count. A budget bar shows `1,840 / 2,000 chars`.

### 16.2 Fleet board

Teams are columns and members are cards. A card shows: role, model badge, a state dot with a text label, current activity ("tool: grep", "waiting on room #design"), elapsed time, tokens, and an open button. The header shows the mode (deliberate → dispatch), the round number, and Stop fleet.

### 16.3 Goal graph

A left-to-right DAG. Each node is a card: leaf title, tier, owner, a state colour plus a glyph (○ pending, ● running, ✓ complete, ✗ error, ? awaiting), an S1 chip where a gate ran, and check counters (`3/4 ✓`). Selecting a node opens leaf detail beneath it: evidence, verifier, reviewer votes and artifacts.

### 16.4 Decisions (System 1)

A dense table with sticky filters (run, surface, outcome: acted/abstained/denied), a confidence sparkline per surface, and an "Uncalibrated" banner while applicable. A row expands to show the question, criteria, probabilities as bars, the state hash, and a link to the step.

### 16.5 Chat (parity)

As specified in the parity inventory §7–§9: floating composer, process disclosure per turn, tool rows, inline review cards, docked cards above the composer, minimap rail. Orphus additions inline:
- **S1 chip** on gated steps: `S1 · withheld · 0.93` in the S1 hue, or `S1 abstained → S2` in the S2 hue.
- **Room post card** when the agent posts to or reads from a room. It shows the digest size, never the full digest.

## 17. Visual system

### 17.1 Direction

"The Quiet Operator": a matrix-green brand on pure black, with the calm density of a workstation. The reference app's restraint (one accent, colour only for meaning, tone instead of lines) carries over. Its neutral charcoal and ink accent are replaced by the Orphus identity.

### 17.2 Colour tokens (dark, v1 default)

| Token | Value | Use |
|---|---|---|
| `bg.base` | `#000000` | window ground; the brand rule is pure black |
| `bg.sidebar` | `#050805` | sidebar, one step off black |
| `bg.surface` | `#0a0f0a` | cards, composer, code blocks |
| `bg.raised` | `#0e150f` | hover, headers, menus |
| `bg.elevated` | `#131c14` | dialogs, popovers (with hairline + shadow) |
| `bg.inset` | `#030503` | code insets, tool output |
| `line` | `#1a231c` | hairlines on floating layers only |
| `text.primary` | `#e8efe9` | body |
| `text.secondary` | `#b4c0b6` | secondary |
| `text.muted` | `#8a978d` | captions, labels |
| `text.faint` | `#5c665e` | disabled, placeholders |
| `accent` | `#00ff41` | brand, focus ring, primary action fill, links. **Never body text** |
| `accent.dim` | `#1a9948` | accent borders, secondary accents |
| `accent.tint` | `rgba(0,255,65,0.06)` | selected rows, ghost hover |
| `on.accent` | `#000000` | text on green fills |
| `selection` | green ground, black text | |

Status colours (shared by graph nodes, sidebar markers, chips and fleet cards; all checked for AA on `bg.surface`):

| State | Token | Value | Glyph |
|---|---|---|---|
| pending | `status.pending` | `#7f849c` | ○ |
| running | `status.running` | `#f9e2af` | ● (pulses twice, then holds) |
| complete | `status.ok` | `#3ddc84` | ✓ |
| error | `status.error` | `#f38ba8` | ✗ |
| awaiting input | `status.awaiting` | `#89dceb` | ? |
| warning | `status.warn` | `#fab387` | ! |

Decision-system hues, used only for S1/S2 attribution:

| Token | Value | Meaning |
|---|---|---|
| `sys.one` | `#94e2d5` (teal) | System 1 decided (acted, withheld or failed) |
| `sys.two` | `#cba6f7` (mauve) | System 2 decided (S1 abstained or no gate) |

`status.ok` is deliberately a softer green than `accent`, so "done" never reads as "brand". Every status is also shown as a glyph and text, never colour alone.

Syntax highlighting is a green-tinted dark theme built from the same tokens (keywords `#89dceb`, strings `#a6e3a1`, numbers `#fab387`, comments `text.muted`, functions `#f9e2af`).

**Light theme (P1, see PRD Q3):** a paper `#f6f8f6` base, ink `#0b120c`, accent darkened to `#0a7d2c` for AA, same status hues darkened.

### 17.3 Typography

- **UI:** Inter (variable), 13 px base ("workstation" size), weights 400–600, no bold beyond 600.
- **Mono:** JetBrains Mono, ligatures off, for code, paths, tool output, IDs, room sequence numbers and command-like buttons (`$ launch`).
- **Display:** Archivo 900, tracking −0.045em, for the home hero and the empty-state wordmark only.
- **Labels:** mono 11 px uppercase, tracking 0.12em, `text.muted`. Section headers may use the brand's `// section` comment style.
- Fonts are bundled (self-hosted, offline), unlike the reference app. The UI font is user-switchable; mono stays fixed. A CJK fallback is always appended.
- Size presets: Compact 90%, Default 100%, Comfortable 115%, Large 125%, plus an 80–150% slider. These are independent of zoom.

### 17.4 Shape, density, elevation

- Radius ladder: 4 (inline code) → 5 (chips) → 6 (buttons, inputs, rows) → 8 (menus, code blocks) → 10 (cards, panels) → 14 (dialogs) → 18 (composer). Nested corners are concentric, and edges against the window stay square.
- Spacing base 4 px; rows 28 px; dense lists 24 px.
- Elevation comes from tone in dark mode. Shadows are used only on floating layers. In-flow regions are separated by tone and spacing, not borders.

### 17.5 Motion

- Three tiers: 90 ms (hover/press), 160 ms (enter/expand/toast), 240 ms (panels). Ease-out on enter, ease-in on exit.
- **Bounded idle motion:** status indicators pulse twice, then hold. Only live work keeps moving: the streaming cursor, a running tool, a live microphone, and the `∀` working glyph (an 88 ms luminance ramp, as in the TUI).
- **Matrix rain** appears only on the empty home hero, at opacity ≤ 0.12, about 11 fps. It is off under reduced motion, paused when hidden, and can be switched off in Appearance. Nowhere else.
- No parallax, bounce, long shimmer or theatrical boot.
- Reduced motion: transitions become instant, pulses stop, and smooth streaming and smooth scroll turn off.

### 17.6 Iconography and identity

- Line icons at 1.5 px stroke, plus the Unicode state glyphs `✓ ✗ → … ○ ● │ ◆`. No emoji in chrome.
- The ORPHUS block wordmark is one mark across the TUI, README, site and app. It appears on the splash and the empty home, unchanged.
- App icon: the favicon mark (a black rounded square with a green outlined square and a solid green centre square), rendered at all OS sizes. Tray icon: a monochrome template version.

### 17.7 Copy

- Direct, precise, confident, no filler. Sentence case. Buttons read as actions ("Launch fleet", "Post", "Approve (Accept edits)").
- Errors name what failed, why, and the next action. They never blame the user.
- Notifications are gentle: one soft chime, no repeats, no red badges for non-failures. "Needs your decision" rather than "ACTION REQUIRED".

## 18. Accessibility

- Every interactive element is reachable and operable by keyboard, with visible focus (green ring, keyboard focus only).
- Roles and labels on custom widgets: the graph is navigable as a tree, and the room transcript as a feed.
- Live regions for turn completion, approvals and room activity, throttled so screen readers are not flooded.
- Contrast AA minimum, and status never conveyed by colour alone.
- Reduced motion and high-contrast mode respected. `NO_COLOR` is honoured in any terminal surfaces.

## 19. Decisions log

ADRs live in `docs/adr/`. The initial set to write at M0:

1. ADR-0001 Clean-room and MIT
2. ADR-0002 Engine is `orphus --mode rpc`, one process per session
3. ADR-0003 Rust host-core owns persistence, secrets and policy
4. ADR-0004 desktop-bridge extension for permission and review
5. ADR-0005 Orphus brand tokens; dark-only v1
6. ADR-0006 Plugin isolation via the Node permission model in utilityProcess
