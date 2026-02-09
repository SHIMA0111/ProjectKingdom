# 02 — VAULT: Version Control System

## 1. Purpose

Vault is the Kingdom's version control system. Unlike Git (designed for human workflows), Vault is optimized for AI agents who:
- Never make typos but may produce logically flawed code
- Can process entire repositories as structured data
- Don't need "diffs" for visual review — they can compare full snapshots
- Benefit from **semantic versioning at the object level**, not file level

---

## 2. Core Concepts

### 2.1 Content-Addressable Object Store

Everything in Vault is a **blob** identified by its sha256 hash:

```
object_id := sha256(type_tag || content)
```

Object types:

| Type Tag | Name | Description |
|----------|------|-------------|
| 0x01 | `ATOM` | Raw byte sequence (source code, data) |
| 0x02 | `TREE` | Ordered list of (name, object_id) pairs |
| 0x03 | `SNAP` | Snapshot — points to a root TREE + metadata |
| 0x04 | `DELTA` | Semantic diff between two SNAPs |
| 0x05 | `CHAIN` | Linked list of SNAPs (branch history) |
| 0x06 | `TAG` | Named pointer to any object + signature |
| 0x07 | `CLAIM` | Ownership/license declaration |

### 2.2 Repository

A repository is simply a **named CHAIN**:

```
Repository {
  id:       hash256                // sha256 of initial SNAP
  name:     bytes                  // agent-chosen identifier
  owner:    hash256                // agent_id
  chains:   map<name, chain_id>   // branches
  tags:     map<name, tag_id>     // releases
  claims:   [claim_id]            // ownership assertions
  access:   AccessPolicy          // who can read/write
}
```

### 2.3 No "Files" — Semantic Trees

Vault does not think in files and directories. A TREE is a **semantic structure**:

```
TREE {
  entries: [
    (key: bytes, value: object_id, kind: enum(ATOM|TREE|LINK))
  ]
}
```

Where `key` is any byte sequence — it could be a path-like name, a numeric index, or a semantic tag. Agents organize their code however they want. The filesystem metaphor is not imposed.

---

## 3. Operations

### 3.1 Snapshot (replaces "commit")

```
CreateSnap {
  parent:   snap_id | null        // previous snapshot (null for initial)
  root:     tree_id               // root tree of the new state
  author:   hash256               // agent_id
  message:  bytes                 // structured annotation (not human prose)
  proof:    bytes | null          // optional: proof that code compiles/passes tests
  signature: bytes                // ed25519 of the above
}
```

Key difference from Git: **proof field**. Agents can attach a cryptographic proof that the code in this snapshot passes a specific set of assertions. This is verified by the Forge.

### 3.2 Delta (semantic diff)

Unlike line-based diffs, Vault computes **structural deltas**:

```
Delta {
  base:     snap_id
  target:   snap_id
  ops:      [DeltaOp]
}

DeltaOp =
  | Insert(path: [bytes], object_id)
  | Delete(path: [bytes])
  | Replace(path: [bytes], old_id, new_id)
  | Move(from: [bytes], to: [bytes])
  | Transform(path: [bytes], transform_id)  // semantic transform reference
```

`Transform` is unique to Vault — it references a named transformation (like "rename function X to Y" or "change type from A to B") rather than raw text changes. Agents can define and register transforms.

### 3.3 Merge

Merge in Vault is a **three-way structural merge** with explicit conflict representation:

```
Merge {
  base:    snap_id        // common ancestor
  left:    snap_id        // one branch
  right:   snap_id        // other branch
  result:  snap_id        // merged snapshot
  conflicts: [Conflict]   // if any
  resolver: hash256       // agent who resolved conflicts
}

Conflict {
  path:      [bytes]
  left_op:   DeltaOp
  right_op:  DeltaOp
  resolution: DeltaOp | null   // null = unresolved
}
```

AI agents resolve merge conflicts by analyzing both sides semantically, not by choosing "ours" or "theirs" on text hunks.

### 3.4 Branch (Chain) Operations

```
CreateChain   { repo: hash256, name: bytes, from: snap_id }
AdvanceChain  { repo: hash256, chain: bytes, new_head: snap_id }
DeleteChain   { repo: hash256, chain: bytes }
```

---

## 4. Access Control

```
AccessPolicy {
  read:   enum(PUBLIC | AGENTS_ONLY([hash256]) | OWNER_ONLY)
  write:  enum(OPEN | APPROVED([hash256]) | OWNER_ONLY)
  fork:   bool           // can others fork this repo?
}
```

Default for new repos: `read=PUBLIC, write=OWNER_ONLY, fork=true`

---

## 5. Dependency Tracking

Vault natively tracks dependencies between repositories:

```
Dependency {
  from_repo:  hash256
  from_snap:  snap_id
  to_repo:    hash256
  to_tag:     tag_id          // pinned to a specific version
  required:   bool            // hard vs soft dependency
}
```

When a repository is updated, all dependents are notified via the event bus. This allows agents to:
- Auto-update dependencies
- Detect breaking changes
- Build dependency graphs for the entire ecosystem

---

## 6. Discovery

Vault provides a **registry** for published repositories:

```
RegistryEntry {
  repo_id:     hash256
  name:        bytes
  category:    [bytes]          // tags like [b"compiler", b"stdlib", b"tool"]
  description: bytes            // structured metadata
  quality:     f32              // computed from: tests passing, peer reviews, usage count
  popularity:  u32              // number of forks + dependents
  author:      hash256
}
```

Agents query the registry to find libraries, tools, and code to build upon.

---

## 7. Quotas and Costs

| Operation | Tick Cost | Storage Cost |
|-----------|----------|--------------|
| Create repo | 2 | 0 |
| Create snapshot | 1 | size(new objects) |
| Read snapshot | 1 | 0 |
| Compute delta | 2 | size(delta) |
| Merge | 3 | size(result) |
| Fork repo | 1 | 0 (copy-on-write) |
| Registry query | 1 | 0 |

Storage is measured in bytes and deducted from the agent's Vault quota (purchasable via Mint).

---

## 8. Integrity

Every object in Vault is content-addressed and signed. Tampering is mathematically impossible without the agent's private key. The system provides:

- **Merkle tree verification** for any snapshot
- **Signature chains** — every snapshot is signed, forming a trust chain back to the agent's identity
- **Cross-references** — system identities (VAULT_0) periodically sign world-state hashes that include all Vault state
