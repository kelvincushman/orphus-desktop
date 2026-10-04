# Orphus Desktop: product requirements

| | |
|---|---|
| Status | Draft 0.1, for review |
| Date | 2026-10-04 |
| Owner | Kelvin Lee (@kelvincushman) |
| Licence | MIT |
| Companion docs | [DESIGN.md](DESIGN.md) · [spec/parity-inventory.md](spec/parity-inventory.md) · [spec/reference-ux-architecture.md](spec/reference-ux-architecture.md) · [../CLEANROOM.md](../CLEANROOM.md) |

---

## 1. Summary

Orphus Desktop is a local-first desktop workspace for the [Orphus](https://orphus.dev) agent harness. It does two jobs:

1. **Parity.** It is a desktop agent workbench with the same feature set as [PI-Desktop](https://github.com/vastsa/PI-Desktop): projects, durable sessions, Agent/Plan/Goal modes, permissions with review and rollback, subagents, every major model provider, MCP, skills, plugins with a marketplace, voice, scheduled tasks, encrypted config sync, remote hosts, and auto-update on macOS, Windows and Linux.
2. **What only Orphus has.** It gives a window onto the things a terminal shows badly:
   - **orchestration:** rooms, fleets, roles, Goal workflow graphs, intercom;
   - **dual-process decisions:** System 1 (a fast, calibrated gate) and System 2 (deliberate model turns and room deliberation), with the receipts, escalations and the distillation loop between them made visible and controllable.

The licence decides how we build it. PI-Desktop is LGPL-3.0-or-later, and before 2026-08-18 it had no licence at all. We want Orphus Desktop to be MIT, so **nothing is copied**: we rebuild it clean-room from a behavioural spec (see [CLEANROOM.md](../CLEANROOM.md)). "As close as possible" therefore means the same features and the same interaction model, implemented independently and styled in the Orphus brand.

## 2. Problem

- **Orphus is terminal-only.** Its most distinctive features are spatial and concurrent: several agents in a room, a fleet of roles, a Goal DAG with ten leaves running at once, and System 1 decisions with confidence scores. A TUI shows these one pane at a time. Orca runs Orphus in worktrees, but it does not understand rooms, fleets or System 1.
- **Desktop agent apps are single-agent.** PI-Desktop, Codex and Claude Desktop centre on one conversation with helpers. None treats deliberation between many agents from many providers as a first-class object, and none exposes a calibrated fast-path decision layer.
- **Trust needs transparency.** Bounded digests, System 1 receipts and Goal evidence already exist as files on disk. Nobody looks at them because there is nowhere to see them.

## 3. Goals and non-goals

### Goals

| # | Goal | Measure |
|---|---|---|
| G1 | Feature parity with PI-Desktop 0.16 for every area marked **Keep** in §7 | Each parity ID in `spec/parity-inventory.md` is Done, Improved or explicitly Skipped, with a reason |
| G2 | Rooms, fleets and Goal runs are first-class, observable and controllable | A user can launch a fleet, watch a room, post into it, and approve a Goal leaf without touching a terminal |
| G3 | System 1 and System 2 are visible and configurable | Every System 1 decision shows as a receipt; escalations to System 2 are traceable; the label harvest and refine loop can be driven from the app |
| G4 | The app drives the real `orphus` harness rather than a fork of it | The desktop never imports pi packages directly; its only engine dependency is the `orphus` RPC contract at a pinned minimum version |
| G5 | Security at least as good as PI-Desktop, and better where it is weak | OS keychain for secrets, real plugin isolation, no telemetry |
| G6 | MIT, open source, clean-room | Provenance register complete; no LGPL code |

### Non-goals (v1)

- A hosted or cloud service. No account, no relay, no telemetry.
- Mobile apps.
- Replacing the Orphus TUI. The two share one harness and one data directory (`~/.orphus`), and a session started in one can be continued in the other.
- Reimplementing the agent loop. That belongs to the harness.
- Training System 1 models in the app. The app *configures*, *observes* and *feeds* System 1 but does not train it.

## 4. Users

| Persona | Needs |
|---|---|
| **Solo builder** (primary; Kelvin is the reference user) | Runs several agents across providers on real repos. Wants to see what each agent is doing, step in quickly, and keep costs bounded. Keyboard-first. Uses Linux (Omarchy/Hyprland) and SSHes into a GPU server. |
| **Team lead or researcher** | Designs fleets and roles. Compares how providers deliberate. Wants decisions with evidence and a record of how they were reached. |
| **Harness developer** | Builds Orphus extensions, workflows and plugins. Wants an inspector, logs, hot reload and receipts. |
| **Model researcher** | Runs a self-hosted System 1 endpoint. Wants calibration data, abstention rates, escalation traces and a clean labelled harvest. |

Accessibility applies to every persona: WCAG 2.1 AA, a full keyboard path, reduced motion respected, and calm notification behaviour (gentle signals, never alarms).

## 5. Product principles

Inherited from Orphus `PRODUCT.md`/`DESIGN.md` and adapted for a GUI:

1. **The Quiet Operator.** Dense, clear, calm. No dashboard clutter and no decoration without a state behind it.
2. **Keyboard-first.** Every action has a keyboard path, and the command palette reaches everything.
3. **Trust through transparency.** Show agent state, what each agent saw (digest budgets), what System 1 decided and how sure it was, and what changed on disk.
4. **Never steal focus.** Background work never navigates, switches project or grabs input.
5. **Bounded by default.** Context costs, retries, concurrency and plugin capabilities all have visible limits.
6. **Honest copy.** Say what is guaranteed and what is not, in a direct, precise voice. "Speak like a trusted tool, not a chatbot."
7. **One harness.** The desktop is a client of `orphus`. Anything the desktop can do, the CLI can do too, and the reverse.

## 6. Concepts and vocabulary

| Term | Meaning |
|---|---|
| **Project** | One or more local folders (the first is primary) with shared instructions and memory. |
| **Session** | A durable conversation with one agent: mode, model, thinking level, permission mode, transcript. Backed by an `orphus --mode rpc` process while active. |
| **Mode** | Agent (executes), Plan (investigates, then asks for approval), Goal (agrees an outcome and checks, then works autonomously). |
| **Role** | A named agent identity (`--name <role>`) with a provider, model and brief. Room read cursors are keyed by role name. |
| **Room** | A bounded discussion held by the local broker, outside every agent's context window. Agents get one-line pings and fixed-budget digests. |
| **Fleet** | A team blueprint (`*.fleet.yaml`) of roles and teams that deliberate, dispatch, or deliberate then dispatch. |
| **Goal run** | A durable workflow: a frozen plan of 3–24 leaves, parallel execution, per-check evidence, three reviewers, and a result of complete, blocked or needs-human. |
| **System 1 (S1)** | A fast triage layer that answers closed questions (yes/no, choice, score) with calibrated probabilities. It may **deny or withhold, never approve**. Below its threshold it abstains. |
| **System 2 (S2)** | Deliberate reasoning: full model turns, Goal planners, verifiers and reviewers, and multi-agent deliberation in rooms. S2 handles everything S1 abstains on. |
| **Receipt** | A JSONL record of one S1 decision, abstentions included. |
| **Distillation loop** | S2's outcome-verified decisions are harvested as labels and used to recalibrate or retrain S1. Proposed changes go through a deterministic gate and an explicit human apply. |

## 7. Scope: parity map

The full behavioural spec is in [`spec/parity-inventory.md`](spec/parity-inventory.md): 570 features with IDs (`F-<AREA>-<NN>`). This table records the decision for each area.

**Keep** = rebuild to parity. **Improve** = rebuild and fix a known weakness. **Replace** = same user outcome, delivered by the Orphus harness. **Skip** = deliberately not built.

| Area (inventory §) | Decision | Notes |
|---|---|---|
| Product framing (§1) | Keep | Project → Session → Agent → Work. Local-first and model-agnostic. |
| App shell, windowing, process model (§2) | Keep | Three columns, per-OS chrome, single instance, boot and recovery screens, supervised helper processes, orderly shutdown. |
| Sidebar and navigation (§3) | Keep + extend | Adds **Rooms** and **Fleets** sections and role badges (§8.1). |
| Search and command palette (§4) | Keep + extend | Also indexes rooms, fleets, Goal runs and receipts. |
| Projects (§5) | Keep | Multi-folder projects, archive workbench, clone from git, temporary chats. |
| Sessions (§6) | Keep | Durable queue, steering, smart stop, fork/branch, regenerate with a pager, crash-safe streaming, checklist, conversation links. Shares the session store with the TUI (`~/.orphus/agent/sessions`). |
| Composer (§7) | Keep + extend | `/`, `@` and skills chips work as specified. The slash menu also lists Orphus commands (`/fleet`, `/workflow`, `/refine`, `/intercom`, …) from `get_commands`. |
| Transcript rendering (§8) | Keep + extend | Adds room-post cards, S1 decision chips and Goal leaf cards. |
| Interactive cards (§9) | Keep | Permission, question, plan/goal approval. They also render harness `extension_ui_request` dialogs natively. |
| Agent/Plan/Goal modes (§10) | **Replace** | Desktop Plan/Goal map onto Orphus plan mode and the **Goal workflow**, which is richer (DAG, evidence, reviewers). See §8.3. |
| Context budget and compaction (§11) | Replace | The harness compacts. The desktop shows the ring, the inspector and compaction dividers from RPC events. |
| Attachments (§12) | Keep | |
| Built-in tools (§13) | Replace | The harness tools are authoritative. The desktop adds review, rollback, audit and admission limits through the **desktop bridge extension** (DESIGN §5.3). |
| Permissions and sandboxing (§14) | Keep + improve | Ask, Accept edits and Auto modes, workspace sandbox, denylist, review and rollback. Enforced in the host via the bridge extension's `tool_call` gate. OS-level sandboxing (bubblewrap/landlock on Linux, seatbelt on macOS) is optional. |
| Subagents (§15) | Replace | Uses Orphus `@orphus/subagents` (single, chain, parallel, async; bounded handoff; cheapest-first routing). The desktop provides the settings editor, live delegate tabs and resume. |
| Worker sessions (§16) | **Replace and extend** | First-class in Orphus: intercom, roles, fleets and rooms (§8.1–8.2). No plugin needed. |
| Models and providers (§17) | Keep | Same presets, wire formats and auth kinds, all via harness providers. Adds **Claude Code (bridge)** as a provider. Subscription OAuth where the harness supports it. |
| Model catalog and reasoning (§18) | Keep | models.dev snapshot, live discovery, seven reasoning levels. |
| Errors and retries (§19) | Keep | Typed errors rendered from harness events. Retry policy belongs to the harness. |
| Work panel (§20) | Keep + extend | New tab types: Room, Fleet board, Goal graph, Receipts, Inspector, Terminal. |
| Review, files, preview, browser (§21) | Keep | Review and rollback, file preview, file manager, embedded browser with an agent tool. |
| Shell, terminal, git (§22) | Keep + **add terminal** | PI-Desktop removed its terminal; Orphus adds a PTY terminal tab (via `@orphus/natives`). Git stays minimal in v1. |
| Instructions and memory (§23) | Keep + extend | AGENTS.md/CLAUDE.md chain, global and project instructions, project memory, prompt templates. Adds a **Memory** view for the `memory` tool (Dossier) when installed. |
| Plugin system (§24) | Keep + **improve** | Same contribution types and permission model. Real isolation (DESIGN §10). Orphus/pi extensions are native, not an add-on: trusted extensions *are* Orphus extensions. Package extension `.orphus-plugin` (see Q4). |
| Skills (§25) | Keep | Workbench, market, scan other tools. |
| MCP (§26) | Keep | Via `@orphus/mcp`. Workbench, market, OAuth 2.1. |
| Image generation (§27) | Keep | |
| Voice (§28) | Keep | Live Voice and Live Work by voice. Local dictation uses `@orphus/transcribe` when it is built. Voice never approves anything. |
| Scheduled tasks (§29) | Keep + extend | Can schedule a fleet run or a Goal workflow, not only a prompt. Agent-callable scheduling tools are **built** (PI-Desktop documents them but never registered them). |
| Notifications (§30) | Keep | Gentle by default: a single soft chime, never repeated alarms. |
| Settings (§31) | Keep + extend | Adds pages for Orchestration, System 1/System 2 and Harness (§8.4). |
| Shortcuts, menus, tray (§32) | Keep | Defaults must not collide with Omarchy/Hyprland bindings (e.g. Super+Alt+Space), so the global plugin launcher defaults to unbound on Linux. |
| Appearance (§33) | Keep, rebranded | Orphus brand system (DESIGN §17). |
| i18n (§34) | Keep | English first. Other languages arrive as community translations once the string catalog is stable. |
| Config sync (§35) | Keep | Encrypted WebDAV. Also syncs fleets, roles manifests and System 1 settings (never endpoints' secrets). |
| Data and storage (§36) | Keep | |
| Import from other tools (§37) | Keep | Claude Code, Codex, OpenCode, Pi CLI, CC Switch. Orphus/Atomic sessions are native. |
| Remote hosts (§38) | Keep, **promoted to v1** | The reference user runs agents on a remote Linux box. Remote hosts are stable, not dev-only. |
| Headless host CLI (§39) | Keep | `orphus-desktop-host` (name TBD), Linux x64/arm64. |
| Desktop as MCP server (§40) | Keep | Opt-in loopback control plane. |
| Updates (§41) | Keep | GitHub Releases feed, stable channel, plus an opt-in beta channel. |
| Packaging (§42) | Keep | macOS DMG/ZIP, Windows NSIS/ZIP, Linux AppImage/deb/rpm, plus an **AUR package** (`orphus-desktop-bin`) for Arch/Omarchy. |
| Logging and diagnostics (§43) | Keep | |
| Privacy (§44) | Keep | Zero telemetry. The plugin channel sends no device id by default. |
| Security model (§45) | Improve | OS keychain first (libsecret, Keychain, DPAPI), encrypted file as fallback with a one-time warning. Electron fuses on. |
| Networking (§46) | Keep | |
| Removed features (§48) | Skip | Except the terminal (re-added above). |
| Planned items (§49) | Selective | Keychain (F-PLAN-01) and the plugin capability sandbox (F-PLAN-03) are in v1. Others stay backlog. |

## 8. Orphus-native requirements

These have no PI-Desktop equivalent. Their IDs are `O-<AREA>-<NN>`. Priority: **P0** = v1 blocker, **P1** = v1 target, **P2** = later.

### 8.1 Rooms

| ID | Requirement | Pri |
|---|---|---|
| O-ROOM-01 | The sidebar has a **Rooms** section listing broker rooms with unread counts, member count and last activity. It updates live from broker `activity` pushes. | P0 |
| O-ROOM-02 | The **room view** is a work-panel tab or a main-pane destination. It shows the full transcript by sequence number, each member's role badge (provider/model) and each member's read cursor as a marker in the transcript. | P0 |
| O-ROOM-03 | **"What they saw"** lens: pick a member to render exactly the digest that member would receive now, with a budget meter (chars used / budget) and a count of collapsed messages. This makes the context-window contract visible. | P0 |
| O-ROOM-04 | The operator can post into a room as a named human seat (default role `operator`), reply to a message (`replyTo`), and @-mention roles. Operator posts are visually distinct. | P0 |
| O-ROOM-05 | **Observer mode:** watch a room without becoming a member. Needs harness work H-01; until it lands, the desktop joins as `operator` and labels itself that way. | P1 |
| O-ROOM-06 | Rooms survive broker exit: the desktop exports or persists transcripts locally (H-02) and can reopen them read-only. | P1 |
| O-ROOM-07 | **Convergence:** a `FINAL:` line is detected and pinned as the room's "decision of record", with the author and a link to the message. | P1 |
| O-ROOM-08 | **Seat an external agent:** generate the MCP config to seat a Claude Code (or other MCP-capable) session in a room as a named role, with a copy button. | P1 |
| O-ROOM-09 | Room cost panel: raw transcript size vs digests actually delivered per member (the "32% of raw" metric, live). | P2 |

### 8.2 Fleets, roles and sessions

| ID | Requirement | Pri |
|---|---|---|
| O-FLEET-01 | A **fleet library** lists blueprints from the project (`.orphus/fleets`), the user (`~/.orphus/agent/fleets`) and the bundled examples, with validation status. | P0 |
| O-FLEET-02 | **Launch a fleet** with a task: choose a blueprint, override models per member, and confirm the expected agent count and cost class. The desktop spawns members as sessions and shows them grouped. | P0 |
| O-FLEET-03 | **Fleet board** (work panel): teams as columns, members as cards with live status (idle, thinking, running a named tool, waiting, done, failed), current activity, model, and token use. Click a card to open that member's session. | P0 |
| O-FLEET-04 | **Roles manifest editor** for `orphus.roles.yaml`: a form plus YAML view, with brief path checks and budget fields. "Launch roles" spawns each role with `--name`. Exports to Orca and tmux are kept. | P1 |
| O-FLEET-05 | **Fleet setup interview** (`/fleetsetup`) renders as a native wizard through the extension UI protocol. | P1 |
| O-FLEET-06 | **Intercom presence:** the sidebar marks sessions that are live elsewhere (TUI, other machines via remote host), using intercom `list`. You can message a session, and `ask` blocks with a visible timer. | P1 |
| O-FLEET-07 | A split-fleet warning appears when processes resolve different agent dirs. | P1 |
| O-FLEET-08 | Gate reviewers (CodeRabbit/Greptile) are shown when a fleet declares them. | P2 |

### 8.3 Goal workflows

| ID | Requirement | Pri |
|---|---|---|
| O-GOAL-01 | **Goal mode** in the composer starts the Orphus Goal workflow. The approval bar shows the frozen plan: 3–24 leaves with OWNS, Needs, tier and checks. | P0 |
| O-GOAL-02 | **Goal graph** (work panel): a live DAG with nodes coloured by state (pending, running, complete, error, awaiting input), parallel lanes, and System 1 badges on gated steps. It replaces the TUI's F2 graph. | P0 |
| O-GOAL-03 | **Leaf detail:** transcript, per-check evidence, verifier result, reviewer votes (three reviewers and the reducer), and links to artifacts (ledger, execution plan, execution report, review report). | P0 |
| O-GOAL-04 | Human-in-the-loop prompts (`ctx.ui.*`) render as native cards, the same as question cards. | P0 |
| O-GOAL-05 | Run controls: pause, interrupt, resume, quit, send a message to a stage. Run list with status, filters and durations, backed by DBOS/run artifacts. | P0 |
| O-GOAL-06 | Other built-in workflows (ralph, adversarial verification, research, spec) can be launched from the palette with a typed input form generated from their schema. | P1 |

### 8.4 System 1 and System 2

| ID | Requirement | Pri |
|---|---|---|
| O-SYS-01 | **Settings → System 1:** pick an adapter (`null` default, `local`, `llm-wrapper`, `typesafe`, and a self-hosted **`orphus-sys1`** adapter once H-04 lands). Endpoint URL, model, timeout, a key kept in the keychain, and **Test** (health and a sample question). | P0 |
| O-SYS-02 | Per-surface thresholds for the active surfaces (`goal.tier`, `goal.review`, `goal.verify`), each with a plain-language explanation and the rule **"System 1 can deny or withhold; it never approves."** | P0 |
| O-SYS-03 | **Decisions view** (Receipts): a table of every S1 decision with time, run, surface, question, kind, value, p, confidence, threshold, abstain, adapter and calibrated flag. Filter by run or surface. Each row links to the Goal step it gated. | P0 |
| O-SYS-04 | **Escalation trace:** each gated step shows which system decided it ("S1 denied at 0.94", "S1 abstained → S2 decided"). Inline chips in the transcript and the Goal graph use the S1 and S2 hues (DESIGN §17). | P0 |
| O-SYS-05 | **Calibration health:** abstain rate, act rate, the confidence distribution, and once labels exist a reliability diagram with ECE and Brier. A persistent banner warns "Uncalibrated" while receipts say `calibrated:false`. | P1 |
| O-SYS-06 | **System 2 settings:** the model pool S2 uses for escalated decisions, reviewers and verifiers (for example a strong cloud model or a local model). Reviewer decorrelation: pick three different providers. | P1 |
| O-SYS-07 | **Distillation loop:** run the label harvest (`systemone-labels`), preview and export the labelled JSONL, filtering out disallowed receipt sources. Show label counts per surface. | P1 |
| O-SYS-08 | **Refine proposals:** a review screen for `/refine` that shows the proposal and the deterministic gate result, then **Apply** as a separate explicit action, plus **Rollback**. Later, also S2-proposed question rewrites (H-05). | P1 |
| O-SYS-09 | **S1 candidate surfaces** (room convergence, post etiquette, skill selection) appear as toggles marked Experimental once the harness wires them. | P2 |
| O-SYS-10 | **Vision tier:** when an S1 vision endpoint is configured, its decisions appear in the same receipts view, tagged by tier (S1, S1.5, S2). | P2 |

### 8.5 Harness inspection

| ID | Requirement | Pri |
|---|---|---|
| O-HARN-01 | **Inspector** tab: `orphus inspect runtime --json` rendered as model, tools with schema hashes, extensions, hook order and prompt hashes, all redacted. | P1 |
| O-HARN-02 | Per-session provider request/response audit (`orphus.provider.*.v1`) and tool audit (`orphus.tool.audit.v1`) in a developer-mode view. | P1 |
| O-HARN-03 | Harness version check: the minimum `orphus` version is pinned. On first run the desktop offers to install or update `orphus` (`install.sh` / `orphus update`). The Harness settings page shows the path, version, agent dir and data locations. | P0 |
| O-HARN-04 | The TUI and the desktop can open the same session (one writer at a time; the second opener sees a read-only notice). | P1 |

## 9. Key user journeys

1. **First run.** Launch → the harness is found or installed → add a provider (or import from Claude Code, Codex, OpenCode or Pi) → open a project → a three-step checklist (send a message, approve a permission, review a change).
2. **Single agent (parity).** Open project → prompt → approve an edit → review the diff → roll one change back → branch from a reply → continue later from the TUI.
3. **Deliberate then dispatch.** Palette: "Launch fleet: design" → task → the fleet board shows planner, critic and reviewer from three providers → open the room → use "what they saw" on the critic → post a constraint as operator → `FINAL:` is pinned → dispatch workers → watch the Goal graph → approve a needs-human leaf.
4. **Tuning System 1.** Settings → System 1 → point at a self-hosted endpoint → Test → run a Goal → the Decisions view shows S1 withholding a reviewer's "done" vote at p 0.93 → open the step and confirm it was right → run the harvest → export labels.
5. **Remote box.** Settings → Remote hosts → add the GPU server over SSH → sessions and fleets run there → the laptop shows the rooms and approvals → the laptop lid closes and the turns keep running.

## 10. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Cold start to an interactive window ≤ 3 s on a mid-range laptop (excluding first-run harness install). Typing latency < 16 ms. Smooth streaming at 60 fps on a 10k-message session. Room view handles 500 messages × 10 members. |
| Concurrency | ≥ 16 concurrent sessions or roles. The fleet board stays responsive with 24 live members. |
| Reliability | No data loss on a crash of any process (checkpointed streaming, outbox). Nothing replays automatically after a restart. |
| Security | Hardened renderer (context isolation, sandbox, strict CSP). Secrets live only in the host process and the keychain. Plugins in isolated processes with a capability sandbox. Electron fuses enabled. Signed releases where the platform allows. |
| Privacy | Zero telemetry. Network contacts are listed in the privacy doc. |
| Accessibility | WCAG 2.1 AA, a full keyboard path, screen-reader labels, reduced motion, and the non-colour status cues listed in DESIGN §17. |
| Platforms | macOS 13+ (arm64, x64), Windows 10/11 x64, Linux x64/arm64 (glibc ≥ 2.35) including Wayland/Hyprland. |
| Footprint | Idle RAM ≤ 400 MB with one session. Installer ≤ 200 MB, excluding the harness. |
| Compatibility | Reads and writes the same `~/.orphus` data as the TUI. Pins a minimum `orphus` version and refuses older ones with clear guidance. |

## 11. Harness prerequisites (work in `kelvincushman/orphus`)

The desktop needs these changes upstream in the harness. They are tracked here so both repos stay in step.

| ID | Change | Needed by |
|---|---|---|
| H-01 | Broker **observer** registration: read and receive pushes without becoming a member. | O-ROOM-05 |
| H-02 | Optional room **persistence** (append-only per room) or a broker `export` the desktop can call. | O-ROOM-06 |
| H-03 | Structured **RPC commands for workflow and fleet state**: list, get, stages, subscribe. Today this state is only reachable through slash commands or widgets. | O-GOAL-02/05, O-FLEET-03 |
| H-04 | A named **`orphus-sys1` System 1 adapter**, with contract tests against the self-hosted wire format (`/v1/systemone`, `/healthz`). Replaces the `typesafe` repoint workaround. | O-SYS-01 |
| H-05 | `systemone-refine`: S2 proposes question and criteria rewrites for human approval. | O-SYS-08 |
| H-06 | A **desktop bridge extension hook set**: `tool_call` permission delegation to the host (already possible), a `tool_result` diff and audit capture (already possible), plus a host capability handshake so the desktop can enforce its sandbox. | Permissions and review |
| H-07 | A decision on **pi upgrade** (0.83 → 0.86+/1.x): the desktop does not need it, but providers, hosted web search and token estimation improvements come with it. | Provider parity |
| H-08 | Personal paths and internal notes removed from public docs (`packages/claude-bridge/PORT-NOTES.md`, `packages/systemone/HANDOFF.md`). | Hygiene before linking from this repo |

## 12. Milestones

| Milestone | Scope | Exit criteria |
|---|---|---|
| **M0 Skeleton** | Repo, tooling, CI, Electron shell, host core, harness supervisor, one session over RPC end to end | A prompt streams a reply from `orphus --mode rpc` into the window; crash/restart works |
| **M1 Core parity** | §3–§12 and §17–§19 of the inventory: sessions, composer, transcript, cards, modes, providers, permissions via the bridge, review and rollback | Journey 2 passes |
| **M2 Workbench** | Work panel, files, browser, terminal, MCP, skills, subagents, instructions and memory, search, notifications, settings | Parity checklist for those areas is green |
| **M3 Orchestration** | Rooms, fleets, roles, intercom, Goal graph (needs H-01–H-03) | Journey 3 passes |
| **M4 System 1/2** | S1 settings, receipts, escalation trace, calibration health, harvest, refine (needs H-04) | Journey 4 passes |
| **M5 Plugins** | Plugin runtime with isolation, SDK, dev kit, marketplace | Example plugins of every contribution type install and run |
| **M6 Reach** | Remote hosts and headless host, voice, scheduled tasks, config sync, desktop MCP server | Journey 5 passes |
| **M7 Release 1.0** | Packaging for every target, AUR, auto-update, signing, docs site pages, accessibility audit | Release checklist |

Order: M0 → M1 → M3 → M2 → M4 → M5 → M6 → M7. Orchestration moves ahead of the workbench because it is the reason the product exists.

## 13. Open questions

| # | Question | Default if not decided |
|---|---|---|
| Q1 | Engine embedding: one `orphus --mode rpc` process per session (isolation, simple) or a shared in-process SDK host (lower memory, harder native ABI)? | Process per session; revisit at M3 with memory data |
| Q2 | Keep a Rust host core like PI-Desktop, or write the host in TypeScript first? | Rust host core (it fits the existing `crates/` and gives a stronger privilege boundary) |
| Q3 | Light theme in v1, given the brand rule "pure black is non-negotiable"? | Dark only in v1; light theme as P1 after M7 |
| Q4 | Plugin package extension and marketplace hosting | `.orphus-plugin`; a GitHub-hosted catalog repo |
| Q5 | Relationship to Orca | Complement it: the desktop exports to Orca and does not compete with its worktree management |
| Q6 | Should the operator's room seat be the user's name or a fixed `operator` role? | `operator`, renameable in settings |
| Q7 | Public repo from day one? | Yes, MIT; nothing private referenced |

## 14. Out of scope for this document

Implementation detail lives in [DESIGN.md](DESIGN.md). The behavioural detail behind every Keep row lives in [spec/parity-inventory.md](spec/parity-inventory.md).
