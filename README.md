# Orphus Desktop

A local-first desktop workspace for the [Orphus](https://orphus.dev) agent harness.

> Many minds, from many makers, argue at one table — the best path leaves the room.

Orphus Desktop puts the whole harness in a window:

- **A full agent workbench.** Projects, durable sessions, Agent/Plan/Goal modes, permissions with review and rollback, subagents, every major model provider, MCP, skills, plugins, voice, scheduled tasks, encrypted sync and remote hosts.
- **Orchestration you can see.** Rooms where agents deliberate outside their context windows, fleets of roles across providers, and live Goal workflow graphs.
- **System 1 and System 2.** A fast, calibrated decision gate and deliberate reasoning, with every decision, escalation and calibration signal visible.

**Status:** design phase. There is no code yet. Start with the docs:

| Doc | What it covers |
|---|---|
| [docs/PRD.md](docs/PRD.md) | What we're building and why: parity map, Orphus-native requirements, milestones |
| [docs/DESIGN.md](docs/DESIGN.md) | Architecture, contracts, security, and the visual and interaction system |
| [docs/spec/parity-inventory.md](docs/spec/parity-inventory.md) | 570-feature behavioural spec (clean-room) |
| [docs/spec/reference-ux-architecture.md](docs/spec/reference-ux-architecture.md) | Reference UX and architecture study (clean-room) |
| [CLEANROOM.md](CLEANROOM.md) | Rules that keep this project MIT |
| [PROVENANCE.md](PROVENANCE.md) | Every borrowed dependency and its licence |

## Relationship to other projects

- **[Orphus](https://github.com/kelvincushman/orphus)** is the engine. The desktop drives the `orphus` binary over its RPC mode and shares `~/.orphus` with the terminal UI.
- **[PI-Desktop](https://github.com/vastsa/PI-Desktop)** is the feature reference. It is LGPL-3.0, so Orphus Desktop is an independent clean-room implementation and contains none of its code.
- **[Orca](https://github.com/stablyai/orca)** manages worktrees. Orphus Desktop complements it and can export roles to it.

## Licence

[MIT](LICENSE) © 2026 Kelvin Lee
