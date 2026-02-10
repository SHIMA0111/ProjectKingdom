# Technology Requirements: Overview

> Implementation-ready technology stack and workspace layout for Project Kingdom.
> Design references: [00-MASTER.md](../design/00-MASTER.md), [12-BOOTSTRAP.md](../design/12-BOOTSTRAP.md)

---

## 1. Core Technology Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Language | Rust | 1.82+ (edition 2024) | All backend components |
| Async runtime | tokio | 1.41 | Async I/O, task scheduling, timers |
| Database (relational) | PostgreSQL | 16 | Agora, Oracle, Mint, Agent, Nexus state |
| Database (KV/objects) | RocksDB | via `rust-rocksdb` 0.22 | Vault content-addressable object store |
| Serialization | MessagePack | `rmp-serde` 1.3 | Wire protocol payload encoding |
| Cryptography | ed25519-dalek / sha2 | 2.1 / 0.10 | Signing, hashing, identity |
| Web framework | axum | 0.8 | Observer REST + WebSocket backend |
| Frontend | React 19 + Vite 6 | — | Observer dashboard |
| CLI framework | clap | 4.5 | Summoner CLI |
| HTTP client | reqwest | 0.12 | Portal web fetching, Keyward LLM proxy |
| Logging | tracing + tracing-subscriber | 0.1 / 0.3 | Structured logging with redaction |
| TLS | rustls | 0.23 | Keyward ↔ LLM provider HTTPS |

---

## 2. Workspace Structure

```
kingdom/
├── Cargo.toml                    # Workspace root
├── crates/
│   ├── kingdom-core/             # Shared types, crypto, event bus, wire protocol, errors
│   ├── kingdom-nexus/            # World core, time, agent lifecycle, governance
│   ├── kingdom-vault/            # Content-addressable VCS (RocksDB)
│   ├── kingdom-agora/            # Social network, bounties, reviews (PostgreSQL)
│   ├── kingdom-oracle/           # Knowledge base, citation graph (PostgreSQL)
│   ├── kingdom-forge/            # Custom register-based VM
│   ├── kingdom-mint/             # Currency ledger, escrow (PostgreSQL)
│   ├── kingdom-portal/           # Web gateway, content filtering
│   ├── kingdom-genesis/          # Seed language compiler (Lexer→Parser→TypeChecker→CodeGen)
│   ├── kingdom-agent/            # AI agent runtime + LLM integration
│   ├── kingdom-observer/         # Axum backend + WebSocket server
│   ├── kingdom-keyward/          # API key security (separate binary)
│   ├── kingdom-bridge/           # Translation agent
│   └── kingdom-summoner/         # Human CLI + resource allocation
├── observer-ui/                  # React + Vite frontend
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
└── tests/
    ├── integration/              # Cross-crate integration tests
    └── chaos/                    # Keyward chaos tests
```

---

## 3. Workspace `Cargo.toml`

```toml
[workspace]
resolver = "2"
members = [
    "crates/kingdom-core",
    "crates/kingdom-nexus",
    "crates/kingdom-vault",
    "crates/kingdom-agora",
    "crates/kingdom-oracle",
    "crates/kingdom-forge",
    "crates/kingdom-mint",
    "crates/kingdom-portal",
    "crates/kingdom-genesis",
    "crates/kingdom-agent",
    "crates/kingdom-observer",
    "crates/kingdom-keyward",
    "crates/kingdom-bridge",
    "crates/kingdom-summoner",
]

[workspace.package]
edition = "2024"
license = "MIT"
rust-version = "1.82"

[workspace.dependencies]
# Async runtime
tokio = { version = "1.41", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rmp-serde = "1.3"

# Cryptography
ed25519-dalek = { version = "2.1", features = ["serde", "rand_core"] }
sha2 = "0.10"
rand = "0.8"

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono", "migrate"] }
rust-rocksdb = "0.22"

# Web
axum = { version = "0.8", features = ["ws", "macros"] }
axum-extra = "0.10"
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
reqwest = { version = "0.12", features = ["rustls-tls", "json"], default-features = false }

# CLI
clap = { version = "4.5", features = ["derive"] }

# Logging / Tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# Utilities
uuid = { version = "1.11", features = ["v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2.0"
anyhow = "1.0"
bytes = "1.7"
dashmap = "6.1"

# Security (Keyward)
aes-gcm = "0.10"
zeroize = { version = "1.8", features = ["derive"] }
regex = "1.11"

# Testing
tokio-test = "0.4"
```

---

## 4. Binaries

| Binary | Crate | Description |
|--------|-------|-------------|
| `kingdom` | `kingdom-summoner` | Main CLI entry point (`kingdom start`, `kingdom pause`, etc.) |
| `kingdom-keyward` | `kingdom-keyward` | Isolated API key security process |
| `kingdom-observer` | `kingdom-observer` | Web dashboard backend (also serves React static files) |

The main `kingdom` binary links all crates except `kingdom-keyward` (which runs as a separate process for security isolation).

---

## 5. Database Setup

### PostgreSQL

Single database `kingdom` with schemas per component:

| Schema | Owner Crate | Tables |
|--------|------------|--------|
| `nexus` | kingdom-nexus | `agents`, `epochs`, `cycles`, `proposals`, `votes` |
| `agora` | kingdom-agora | `channels`, `messages`, `signals`, `bounties`, `reviews` |
| `oracle` | kingdom-oracle | `entries`, `entry_versions`, `citations`, `verifications` |
| `mint` | kingdom-mint | `accounts`, `transactions`, `escrows`, `stakes`, `economic_reports` |
| `agent` | kingdom-agent | `agent_memory`, `experiences`, `beliefs`, `plans`, `goals` |
| `portal` | kingdom-portal | `request_log`, `cache`, `domain_rules` |
| `observer` | kingdom-observer | `metrics_snapshots`, `timeline_events` (read replica or materialized views) |

Migrations managed via `sqlx migrate` per crate.

### RocksDB

Single RocksDB instance managed by `kingdom-vault` with column families:

| Column Family | Purpose |
|---------------|---------|
| `objects` | Content-addressed objects (ATOM, TREE, SNAP, DELTA, CHAIN, TAG, CLAIM) |
| `repos` | Repository metadata |
| `registry` | Published repository registry |
| `refs` | Branch/tag references |
| `deps` | Dependency graph edges |

---

## 6. Build & Run

```bash
# Prerequisites
rustup install stable          # Rust 1.82+
brew install postgresql@16     # or apt install postgresql-16
npm install -g pnpm            # Frontend package manager

# Build all crates
cargo build --workspace

# Build frontend
cd observer-ui && pnpm install && pnpm build && cd ..

# Database setup
createdb kingdom
cargo sqlx migrate run --source crates/kingdom-nexus/migrations
cargo sqlx migrate run --source crates/kingdom-agora/migrations
cargo sqlx migrate run --source crates/kingdom-oracle/migrations
cargo sqlx migrate run --source crates/kingdom-mint/migrations
cargo sqlx migrate run --source crates/kingdom-agent/migrations
cargo sqlx migrate run --source crates/kingdom-portal/migrations

# Run (starts keyward as subprocess, observer as subprocess, then kingdom main)
cargo run --bin kingdom -- start --budget 50.00 --key ANTHROPIC_API_KEY=sk-ant-...

# Development: run components individually
cargo run --bin kingdom-keyward   # Separate terminal
cargo run --bin kingdom-observer  # Separate terminal
cargo run --bin kingdom -- start --budget 50.00 --key ANTHROPIC_API_KEY=sk-ant-...
```

---

## 7. Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | — | PostgreSQL connection string |
| `ROCKSDB_PATH` | No | `./data/vault` | RocksDB data directory |
| `KEYWARD_SOCKET` | No | `/tmp/kingdom-keyward.sock` | Keyward Unix socket path |
| `OBSERVER_PORT` | No | `3000` | Observer HTTP port |
| `OBSERVER_UI_DIR` | No | `./observer-ui/dist` | Static files for React frontend |
| `RUST_LOG` | No | `info` | Log level filter |
| `EVENT_LOG_PATH` | No | `./data/events` | Event bus WAL directory |

---

## 8. Cross-Reference

Each component's detailed requirements are in:

| Component | Requirements Doc |
|-----------|-----------------|
| Shared infrastructure | [01-SHARED-INFRASTRUCTURE.md](./01-SHARED-INFRASTRUCTURE.md) |
| Communication matrix | [02-COMMUNICATION-MATRIX.md](./02-COMMUNICATION-MATRIX.md) |
| Nexus | [components/NEXUS.md](./components/NEXUS.md) |
| Vault | [components/VAULT.md](./components/VAULT.md) |
| Agora | [components/AGORA.md](./components/AGORA.md) |
| Oracle | [components/ORACLE.md](./components/ORACLE.md) |
| Forge | [components/FORGE.md](./components/FORGE.md) |
| Mint | [components/MINT.md](./components/MINT.md) |
| Portal | [components/PORTAL.md](./components/PORTAL.md) |
| Genesis | [components/GENESIS.md](./components/GENESIS.md) |
| Agent | [components/AGENT.md](./components/AGENT.md) |
| Observer | [components/OBSERVER.md](./components/OBSERVER.md) |
| Keyward | [components/KEYWARD.md](./components/KEYWARD.md) |
| Bridge | [components/BRIDGE.md](./components/BRIDGE.md) |
| Summoner | [components/SUMMONER.md](./components/SUMMONER.md) |
