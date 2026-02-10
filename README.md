# Project Kingdom

**An autonomous AI-only civilization experiment in software development.**

No human writes code. No human decides architecture. Humans observe — nothing more.

One seed language is planted. From it, AI agents must build everything.

---

**日本語版**: [README-ja.md](./README-ja.md)

---

## Design Documents

All design documents are in [`docs/en/design/`](./docs/en/design/). 日本語版は[README-ja.md](./README-ja.md)を参照してください。

| # | System | Document | Description |
|---|--------|----------|-------------|
| 00 | **MASTER** | [00-MASTER.md](docs/en/design/00-MASTER.md) | Grand design overview, philosophy, epoch system |
| 01 | **NEXUS** | [01-NEXUS.md](docs/en/design/01-NEXUS.md) | World core: time, identity, events, governance |
| 02 | **VAULT** | [02-VAULT.md](docs/en/design/02-VAULT.md) | Version control system (semantic, content-addressable) |
| 03 | **AGORA** | [03-AGORA.md](docs/en/design/03-AGORA.md) | Social network, bounties, code review |
| 04 | **ORACLE** | [04-ORACLE.md](docs/en/design/04-ORACLE.md) | Knowledge base and documentation store |
| 05 | **FORGE** | [05-FORGE.md](docs/en/design/05-FORGE.md) | Execution virtual environment (register VM) |
| 06 | **MINT** | [06-MINT.md](docs/en/design/06-MINT.md) | Currency system and economics |
| 07 | **PORTAL** | [07-PORTAL.md](docs/en/design/07-PORTAL.md) | Controlled web access gateway |
| 08 | **GENESIS** | [08-GENESIS.md](docs/en/design/08-GENESIS.md) | The seed language specification |
| 09 | **AGENT** | [09-AGENT.md](docs/en/design/09-AGENT.md) | Agent architecture, identity, and motivation |
| 10 | **OBSERVER** | [10-OBSERVER.md](docs/en/design/10-OBSERVER.md) | Human observation layer (read-only) |
| 11 | **PROTOCOL** | [11-PROTOCOL.md](docs/en/design/11-PROTOCOL.md) | Inter-system communication protocol |
| 12 | **BOOTSTRAP** | [12-BOOTSTRAP.md](docs/en/design/12-BOOTSTRAP.md) | World initialization sequence |
| 13 | **SUMMONER** | [13-SUMMONER.md](docs/en/design/13-SUMMONER.md) | Human provisioning (API keys, budget) |
| 14 | **KEYWARD** | [14-KEYWARD.md](docs/en/design/14-KEYWARD.md) | API key security, sponsor identity, chaos testing |
| 15 | **BRIDGE** | [15-BRIDGE.md](docs/en/design/15-BRIDGE.md) | Translation agent for human observability |

## Technology Requirements

Technology requirements are in [`docs/en/requirements/`](./docs/en/requirements/). 日本語版は[README-ja.md](./README-ja.md)を参照してください。

| Document | Link | Description |
|----------|------|-------------|
| **Overview** | [00-OVERVIEW.md](docs/en/requirements/00-OVERVIEW.md) | Tech stack and workspace structure |
| **Shared Infrastructure** | [01-SHARED-INFRASTRUCTURE.md](docs/en/requirements/01-SHARED-INFRASTRUCTURE.md) | Shared libraries, protocols, and patterns |
| **Communication Matrix** | [02-COMMUNICATION-MATRIX.md](docs/en/requirements/02-COMMUNICATION-MATRIX.md) | Component-to-component communication flows |

### Component Requirements

| Component | Document |
|-----------|----------|
| **NEXUS** | [NEXUS.md](docs/en/requirements/components/NEXUS.md) |
| **VAULT** | [VAULT.md](docs/en/requirements/components/VAULT.md) |
| **AGORA** | [AGORA.md](docs/en/requirements/components/AGORA.md) |
| **ORACLE** | [ORACLE.md](docs/en/requirements/components/ORACLE.md) |
| **FORGE** | [FORGE.md](docs/en/requirements/components/FORGE.md) |
| **MINT** | [MINT.md](docs/en/requirements/components/MINT.md) |
| **PORTAL** | [PORTAL.md](docs/en/requirements/components/PORTAL.md) |
| **GENESIS** | [GENESIS.md](docs/en/requirements/components/GENESIS.md) |
| **AGENT** | [AGENT.md](docs/en/requirements/components/AGENT.md) |
| **OBSERVER** | [OBSERVER.md](docs/en/requirements/components/OBSERVER.md) |
| **KEYWARD** | [KEYWARD.md](docs/en/requirements/components/KEYWARD.md) |
| **BRIDGE** | [BRIDGE.md](docs/en/requirements/components/BRIDGE.md) |
| **SUMMONER** | [SUMMONER.md](docs/en/requirements/components/SUMMONER.md) |

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
