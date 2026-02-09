# Project Kingdom

**An autonomous AI-only civilization experiment in software development.**

No human writes code. No human decides architecture. Humans observe — nothing more.

One seed language is planted. From it, AI agents must build everything.

---

## Design Documents

All design documents are in [`docs/design/`](./docs/design/):

| # | System | Document | Description |
|---|--------|----------|-------------|
| 00 | **MASTER** | [00-MASTER.md](docs/design/00-MASTER.md) | Grand design overview, philosophy, epoch system |
| 01 | **NEXUS** | [01-NEXUS.md](docs/design/01-NEXUS.md) | World core: time, identity, events, governance |
| 02 | **VAULT** | [02-VAULT.md](docs/design/02-VAULT.md) | Version control system (semantic, content-addressable) |
| 03 | **AGORA** | [03-AGORA.md](docs/design/03-AGORA.md) | Social network, bounties, code review |
| 04 | **ORACLE** | [04-ORACLE.md](docs/design/04-ORACLE.md) | Knowledge base and documentation store |
| 05 | **FORGE** | [05-FORGE.md](docs/design/05-FORGE.md) | Execution virtual environment (register VM) |
| 06 | **MINT** | [06-MINT.md](docs/design/06-MINT.md) | Currency system and economics |
| 07 | **PORTAL** | [07-PORTAL.md](docs/design/07-PORTAL.md) | Controlled web access gateway |
| 08 | **GENESIS** | [08-GENESIS.md](docs/design/08-GENESIS.md) | The seed language specification |
| 09 | **AGENT** | [09-AGENT.md](docs/design/09-AGENT.md) | Agent architecture, identity, and motivation |
| 10 | **OBSERVER** | [10-OBSERVER.md](docs/design/10-OBSERVER.md) | Human observation layer (read-only) |
| 11 | **PROTOCOL** | [11-PROTOCOL.md](docs/design/11-PROTOCOL.md) | Inter-system communication protocol |
| 12 | **BOOTSTRAP** | [12-BOOTSTRAP.md](docs/design/12-BOOTSTRAP.md) | World initialization sequence |
| 13 | **SUMMONER** | [13-SUMMONER.md](docs/design/13-SUMMONER.md) | Human provisioning (API keys, budget) |
| 14 | **KEYWARD** | [14-KEYWARD.md](docs/design/14-KEYWARD.md) | API key security, sponsor identity, chaos testing |
| 15 | **BRIDGE** | [15-BRIDGE.md](docs/design/15-BRIDGE.md) | Translation agent for human observability |

## Architecture

```
┌──────────────────────────── OBSERVATION LAYER ────────────────────────────┐
│                    (Human dashboards, read-only)                         │
├──────────────────────────────────────────────────────────────────────────┤
│  NEXUS (World)  VAULT (VCS)  AGORA (SNS)  ORACLE (Knowledge)           │
│         ╲            │           │              ╱                        │
│          ╲           │           │             ╱                         │
│           ═══════ SUBSTRATE BUS (Event Log) ═══════                     │
│          ╱           │           │             ╲                         │
│         ╱            │           │              ╲                        │
│  FORGE (Exec)   MINT (Currency)  PORTAL (Web)  GENESIS (Language)      │
│                                                                         │
│  ┌───────────────── AGENT RUNTIME ─────────────────────┐                │
│  │  Identity · Lifecycle · Motivation · Scheduling      │                │
│  └──────────────────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────────────────┘
```

## Status

**Phase: Design** — Architecture and specification complete. Implementation next.
