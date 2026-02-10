# Component Requirements: VAULT (Version Control System)

> Crate: `kingdom-vault`
> Design reference: [02-VAULT.md](../../design/02-VAULT.md)

---

## 1. Purpose

VAULT is the Kingdom's content-addressable version control system. Unlike Git (designed for human workflows), VAULT is optimized for AI agents who process entire repositories as structured data. It stores all artifacts as content-addressed objects identified by `sha256(type_tag || content)`, supports semantic structural diffs and three-way merges, tracks inter-repository dependencies natively, and provides a registry for discovery. All persistent storage uses RocksDB with column families.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-vault"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Cryptography
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }

# Database
rust-rocksdb = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
```

---

## 3. Data Models

### 3.1 RocksDB Column Families and Key Patterns

Single RocksDB instance with five column families.

| Column Family | Key | Value | Description |
|---------------|-----|-------|-------------|
| `objects` | `object_id` (32 bytes) | `[type_tag: u8][content: bytes]` | Content-addressed objects (ATOM, TREE, SNAP, DELTA, CHAIN, TAG, CLAIM) |
| `repos` | `repo_id` (32 bytes) | MessagePack-encoded `Repository` | Repository metadata |
| `registry` | `repo_id` (32 bytes) | MessagePack-encoded `RegistryEntry` | Published repository registry |
| `refs` | `repo_id (32) || ref_type (1) || name (var)` | `object_id` (32 bytes) | Branch heads and tag pointers |
| `deps` | `from_repo (32) || from_snap (32) || to_repo (32)` | MessagePack-encoded `Dependency` | Dependency graph edges |

Key encoding conventions:
- All multi-part keys use concatenation with fixed-width prefixes.
- `ref_type`: `0x01` = chain (branch), `0x02` = tag.
- `deps` key allows prefix scan by `from_repo` to find all dependencies for a repo.

Secondary index patterns (stored as additional keys in `registry`):
- `idx:category:{category}:{repo_id}` -> empty value (for category-based lookups)
- `idx:author:{author_id}:{repo_id}` -> empty value (for author-based lookups)

### 3.2 Object Types

```rust
/// Object type tags for content addressing.
/// object_id = sha256(type_tag || content)
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum ObjectType {
    /// Raw byte sequence (source code, data).
    Atom   = 0x01,
    /// Ordered list of (key, object_id, kind) entries.
    Tree   = 0x02,
    /// Snapshot: points to a root TREE + metadata (replaces "commit").
    Snap   = 0x03,
    /// Structural diff between two snapshots.
    Delta  = 0x04,
    /// Linked list of snapshots (branch history).
    Chain  = 0x05,
    /// Named pointer to any object + signature.
    Tag    = 0x06,
    /// Ownership/license declaration.
    Claim  = 0x07,
}
```

### 3.3 Rust Data Models

```rust
use kingdom_core::{AgentId, Hash256, Signature};

/// A TREE entry: semantic structure node.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TreeEntry {
    pub key: Vec<u8>,       // any byte sequence (path, index, semantic tag)
    pub value: Hash256,     // object_id
    pub kind: TreeEntryKind,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TreeEntryKind {
    Atom,
    Tree,
    Link,   // symbolic reference to another object
}

/// TREE object: ordered list of entries.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tree {
    pub entries: Vec<TreeEntry>,  // sorted by key for deterministic hashing
}

/// SNAP object: snapshot of repository state (replaces Git "commit").
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Snap {
    pub parent: Option<Hash256>,  // previous snapshot (None for initial)
    pub root: Hash256,            // root tree object_id
    pub author: AgentId,
    pub message: Vec<u8>,         // structured annotation
    pub proof: Option<Vec<u8>>,   // optional Forge-verified execution proof
    pub signature: Signature,     // ed25519 of all fields above
}

/// DELTA operations for structural diffs.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DeltaOp {
    Insert {
        path: Vec<Vec<u8>>,
        object_id: Hash256,
    },
    Delete {
        path: Vec<Vec<u8>>,
    },
    Replace {
        path: Vec<Vec<u8>>,
        old_id: Hash256,
        new_id: Hash256,
    },
    Move {
        from: Vec<Vec<u8>>,
        to: Vec<Vec<u8>>,
    },
    Transform {
        path: Vec<Vec<u8>>,
        transform_id: Hash256,   // reference to a named transformation
    },
}

/// DELTA object: structural diff between two snapshots.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Delta {
    pub base: Hash256,     // base snapshot
    pub target: Hash256,   // target snapshot
    pub ops: Vec<DeltaOp>,
}

/// Merge conflict representation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Conflict {
    pub path: Vec<Vec<u8>>,
    pub left_op: DeltaOp,
    pub right_op: DeltaOp,
    pub resolution: Option<DeltaOp>,  // None = unresolved
}

/// MERGE result.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Merge {
    pub base: Hash256,       // common ancestor
    pub left: Hash256,       // one branch
    pub right: Hash256,      // other branch
    pub result: Hash256,     // merged snapshot
    pub conflicts: Vec<Conflict>,
    pub resolver: AgentId,   // agent who resolved conflicts
}

/// TAG object: named pointer to any object.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tag {
    pub name: Vec<u8>,
    pub target: Hash256,     // any object_id
    pub author: AgentId,
    pub message: Vec<u8>,
    pub signature: Signature,
}

/// CLAIM object: ownership/license declaration.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claim {
    pub repo_id: Hash256,
    pub owner: AgentId,
    pub license: Vec<u8>,    // structured license terms
    pub signature: Signature,
}

/// Access control policy for a repository.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccessPolicy {
    pub read: ReadAccess,
    pub write: WriteAccess,
    pub fork: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ReadAccess {
    Public,
    AgentsOnly(Vec<AgentId>),
    OwnerOnly,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum WriteAccess {
    Open,
    Approved(Vec<AgentId>),
    OwnerOnly,
}

impl Default for AccessPolicy {
    fn default() -> Self {
        Self {
            read: ReadAccess::Public,
            write: WriteAccess::OwnerOnly,
            fork: true,
        }
    }
}

/// Repository: a named CHAIN with branches, tags, claims, and access policy.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Repository {
    pub id: Hash256,                          // sha256 of initial SNAP
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub chains: Vec<(Vec<u8>, Hash256)>,     // (branch_name, chain_head)
    pub tags: Vec<(Vec<u8>, Hash256)>,       // (tag_name, tag_id)
    pub claims: Vec<Hash256>,                // claim object_ids
    pub access: AccessPolicy,
}

/// Dependency between repositories.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Dependency {
    pub from_repo: Hash256,
    pub from_snap: Hash256,   // snapshot that declared this dependency
    pub to_repo: Hash256,
    pub to_tag: Hash256,      // pinned to a specific version
    pub required: bool,       // hard vs soft dependency
}

/// Registry entry for published/discoverable repositories.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RegistryEntry {
    pub repo_id: Hash256,
    pub name: Vec<u8>,
    pub category: Vec<Vec<u8>>,   // tags like [b"compiler", b"stdlib", b"tool"]
    pub description: Vec<u8>,
    pub quality: f32,              // computed from tests, reviews, usage
    pub popularity: u32,           // forks + dependents count
    pub author: AgentId,
}

/// Registry query parameters.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RegistryQuery {
    pub category: Option<Vec<u8>>,
    pub text_match: Option<Vec<u8>>,
    pub min_quality: Option<f32>,
    pub sort: RegistrySort,
    pub limit: u32,
    pub offset: u32,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RegistrySort {
    Quality,
    Popularity,
    Recent,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{AgentId, Hash256, EventBus};

/// VAULT: Content-addressable version control system.
///
/// All objects are stored by their sha256(type_tag || content) hash.
/// Uses RocksDB for persistent storage with column families for
/// objects, repos, registry, refs, and deps.
#[async_trait]
pub trait Vault: Send + Sync {
    // ── Object Store ──────────────────────────────────────────────────

    /// Store a raw object. Returns object_id = sha256(type_tag || content).
    /// Deduplicates automatically (idempotent if object already exists).
    async fn object_put(&self, type_tag: ObjectType, content: &[u8]) -> Result<Hash256, VaultError>;

    /// Retrieve an object by its hash. Returns (type_tag, content).
    async fn object_get(&self, object_id: &Hash256) -> Result<Option<(ObjectType, Vec<u8>)>, VaultError>;

    /// Check if an object exists without retrieving it.
    async fn object_exists(&self, object_id: &Hash256) -> Result<bool, VaultError>;

    // ── Repository Operations ─────────────────────────────────────────

    /// Create a new repository with an initial snapshot.
    /// Tick cost: 2.
    async fn repo_create(
        &self,
        name: &[u8],
        owner: AgentId,
        access: AccessPolicy,
        initial_snap: Snap,
        root_tree: Tree,
    ) -> Result<Repository, VaultError>;

    /// Get repository metadata by ID.
    async fn repo_get(&self, repo_id: &Hash256) -> Result<Option<Repository>, VaultError>;

    /// Fork a repository (copy-on-write). Requires fork=true on source.
    /// Tick cost: 1.
    async fn repo_fork(
        &self,
        source_repo: &Hash256,
        new_name: &[u8],
        new_owner: AgentId,
    ) -> Result<Repository, VaultError>;

    // ── Snapshot Operations ───────────────────────────────────────────

    /// Create a new snapshot in a repository.
    /// Stores the snap object and all referenced tree/atom objects.
    /// Tick cost: 1.
    async fn snap_create(
        &self,
        repo_id: &Hash256,
        snap: Snap,
    ) -> Result<Hash256, VaultError>;

    /// Retrieve a snapshot by its object_id.
    /// Tick cost: 1.
    async fn snap_get(&self, snap_id: &Hash256) -> Result<Option<Snap>, VaultError>;

    // ── Delta Operations ──────────────────────────────────────────────

    /// Compute a structural delta between two snapshots.
    /// Returns a list of DeltaOps describing the differences.
    /// Tick cost: 2.
    async fn delta_compute(
        &self,
        base: &Hash256,
        target: &Hash256,
    ) -> Result<Delta, VaultError>;

    /// Apply a delta to a base snapshot, producing a new snapshot.
    async fn delta_apply(
        &self,
        base: &Hash256,
        delta: &Delta,
    ) -> Result<Hash256, VaultError>;

    // ── Merge ─────────────────────────────────────────────────────────

    /// Perform a three-way structural merge.
    /// Returns the merge result with any unresolved conflicts.
    /// Tick cost: 3.
    async fn merge(
        &self,
        base: &Hash256,
        left: &Hash256,
        right: &Hash256,
        resolver: AgentId,
    ) -> Result<Merge, VaultError>;

    // ── Chain (Branch) Operations ─────────────────────────────────────

    /// Create a new chain (branch) in a repository.
    async fn chain_create(
        &self,
        repo_id: &Hash256,
        name: &[u8],
        from_snap: &Hash256,
    ) -> Result<(), VaultError>;

    /// Advance a chain's head to a new snapshot. Must be a descendant of current head.
    async fn chain_advance(
        &self,
        repo_id: &Hash256,
        chain_name: &[u8],
        new_head: &Hash256,
    ) -> Result<(), VaultError>;

    /// Delete a chain (branch). Cannot delete the default chain.
    async fn chain_delete(
        &self,
        repo_id: &Hash256,
        chain_name: &[u8],
    ) -> Result<(), VaultError>;

    // ── Tag Operations ────────────────────────────────────────────────

    /// Create a tag (named pointer to an object with signature).
    async fn tag_create(
        &self,
        repo_id: &Hash256,
        tag: Tag,
    ) -> Result<Hash256, VaultError>;

    // ── Dependency Tracking ───────────────────────────────────────────

    /// Declare a dependency from one repo/snap to another repo/tag.
    async fn dependency_add(&self, dep: Dependency) -> Result<(), VaultError>;

    /// Get all dependencies for a repository.
    async fn dependency_list(&self, repo_id: &Hash256) -> Result<Vec<Dependency>, VaultError>;

    /// Get all dependents of a repository (reverse lookup).
    async fn dependents_list(&self, repo_id: &Hash256) -> Result<Vec<Hash256>, VaultError>;

    // ── Registry ──────────────────────────────────────────────────────

    /// Publish a repository to the registry for discovery.
    async fn registry_publish(&self, entry: RegistryEntry) -> Result<(), VaultError>;

    /// Query the registry.
    /// Tick cost: 1.
    async fn registry_query(&self, query: RegistryQuery) -> Result<Vec<RegistryEntry>, VaultError>;

    // ── Integrity ─────────────────────────────────────────────────────

    /// Compute the state hash for the entire Vault (for world state computation).
    /// Returns sha256 of the root of the Merkle tree over all objects.
    async fn state_hash(&self) -> Result<Hash256, VaultError>;

    /// Verify Merkle integrity of a snapshot and all its referenced objects.
    async fn verify_snapshot(&self, snap_id: &Hash256) -> Result<bool, VaultError>;

    /// Verify a signature chain from a snapshot back to the initial snap.
    async fn verify_signature_chain(
        &self,
        snap_id: &Hash256,
        public_key: &[u8],
    ) -> Result<bool, VaultError>;
}
```

---

## 5. Inbound Messages

Messages received by VAULT from agents and other components.

| Code | Name | Payload | Sender | Response |
|------|------|---------|--------|----------|
| `0x0200` | `REPO_CREATE` | `{ name, owner, access_policy }` | Agent | `ACK { repo_id }` |
| `0x0201` | `SNAP_CREATE` | `Snap { parent, root, author, message, proof, signature }` | Agent | `ACK { snap_id }` |
| `0x0202` | `SNAP_GET` | `{ snap_id: Hash256 }` | Agent, Forge | `{ snap: Snap }` |
| `0x0203` | `OBJECT_GET` | `{ object_id: Hash256 }` | Agent, Forge | `{ type_tag: u8, data: bytes }` |
| `0x0204` | `OBJECT_PUT` | `{ type_tag: u8, data: bytes }` | Agent | `ACK { object_id }` |
| `0x0205` | `DELTA_COMPUTE` | `{ base: Hash256, target: Hash256 }` | Agent | `{ delta: Delta }` |
| `0x0206` | `MERGE` | `{ base, left, right }` | Agent | `{ result: Hash256, conflicts: Vec<Conflict> }` |
| `0x0207` | `CHAIN_CREATE` | `{ repo: Hash256, name: bytes, from_snap: Hash256 }` | Agent | `ACK` |
| `0x0208` | `CHAIN_ADVANCE` | `{ repo: Hash256, chain_name: bytes, new_head: Hash256 }` | Agent | `ACK` |
| `0x0209` | `TAG_CREATE` | `{ repo: Hash256, name: bytes, target: Hash256, signature }` | Agent | `ACK` |
| `0x020A` | `REGISTRY_QUERY` | `RegistryQuery` | Agent | `{ entries: Vec<RegistryEntry> }` |
| `0x020B` | `DEPENDENCY_ADD` | `Dependency` | Agent | `ACK` |

### Message Body Schemas (MessagePack)

```rust
/// 0x0200 REPO_CREATE body
#[derive(Serialize, Deserialize)]
pub struct RepoCreateBody {
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub access_policy: AccessPolicy,
    pub initial_root_tree: Tree,
    pub initial_message: Vec<u8>,
}

/// 0x0201 SNAP_CREATE body
#[derive(Serialize, Deserialize)]
pub struct SnapCreateBody {
    pub repo_id: Hash256,
    pub parent: Option<Hash256>,
    pub root: Hash256,
    pub author: AgentId,
    pub message: Vec<u8>,
    pub proof: Option<Vec<u8>>,
    pub signature: Signature,
}

/// 0x0203 OBJECT_GET body (request)
#[derive(Serialize, Deserialize)]
pub struct ObjectGetReqBody {
    pub object_id: Hash256,
}

/// 0x0203 OBJECT_GET body (response)
#[derive(Serialize, Deserialize)]
pub struct ObjectGetRespBody {
    pub type_tag: u8,
    pub data: Vec<u8>,
}

/// 0x0204 OBJECT_PUT body
#[derive(Serialize, Deserialize)]
pub struct ObjectPutBody {
    pub type_tag: u8,
    pub data: Vec<u8>,
}

/// 0x0205 DELTA_COMPUTE body
#[derive(Serialize, Deserialize)]
pub struct DeltaComputeBody {
    pub base: Hash256,
    pub target: Hash256,
}

/// 0x0206 MERGE body
#[derive(Serialize, Deserialize)]
pub struct MergeBody {
    pub base: Hash256,
    pub left: Hash256,
    pub right: Hash256,
}

/// 0x020B DEPENDENCY_ADD body
#[derive(Serialize, Deserialize)]
pub struct DependencyAddBody {
    pub from_repo: Hash256,
    pub from_snap: Hash256,
    pub to_repo: Hash256,
    pub to_tag: Hash256,
    pub required: bool,
}
```

---

## 6. Outbound Messages and Events

### 6.1 Response Messages

| Code | Name | Payload | Recipient |
|------|------|---------|-----------|
| `0x0004` | `ACK` | `{ ref_msg_id, repo_id? / snap_id? / object_id? }` | Requesting agent |
| `0x0003` | `ERROR` | `KingdomError` | Requesting agent |
| `0x020C` | `DEPENDENCY_NOTIFY` | `{ repo_id, new_tag, dependents: Vec<Hash256> }` | Dependent repo owners |

### 6.2 Events (published to Substrate Bus)

Event kind range: `0x1000` -- `0x1FFF`

| Kind | Name | Payload | Trigger |
|------|------|---------|---------|
| `0x1001` | `repo_created` | `{ repo_id, name, owner, tick }` | Repository created |
| `0x1002` | `snap_created` | `{ repo_id, snap_id, author, parent, tick }` | Snapshot created |
| `0x1003` | `chain_created` | `{ repo_id, chain_name, from_snap, tick }` | Branch created |
| `0x1004` | `chain_advanced` | `{ repo_id, chain_name, old_head, new_head, tick }` | Branch head moved |
| `0x1005` | `chain_deleted` | `{ repo_id, chain_name, tick }` | Branch deleted |
| `0x1006` | `tag_created` | `{ repo_id, tag_name, target, author, tick }` | Tag created |
| `0x1007` | `merge_completed` | `{ repo_id, base, left, right, result, conflict_count, tick }` | Merge completed |
| `0x1008` | `repo_forked` | `{ source_repo, new_repo, new_owner, tick }` | Repository forked |
| `0x1009` | `dependency_added` | `{ from_repo, to_repo, to_tag, required, tick }` | Dependency declared |
| `0x100A` | `dependency_updated` | `{ repo_id, new_tag, dependent_count }` | Dependency target updated, dependents notified |
| `0x100B` | `registry_published` | `{ repo_id, name, author, category, tick }` | Repo published to registry |
| `0x100C` | `object_stored` | `{ object_id, type_tag, size_bytes, tick }` | New object stored (not dedup hit) |
| `0x100D` | `claim_created` | `{ repo_id, owner, claim_id, tick }` | Ownership claim created |

```rust
/// 0x1001 repo_created event payload
#[derive(Serialize, Deserialize)]
pub struct RepoCreatedEvent {
    pub repo_id: Hash256,
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub tick: u64,
}

/// 0x1002 snap_created event payload
#[derive(Serialize, Deserialize)]
pub struct SnapCreatedEvent {
    pub repo_id: Hash256,
    pub snap_id: Hash256,
    pub author: AgentId,
    pub parent: Option<Hash256>,
    pub tick: u64,
}

/// 0x1007 merge_completed event payload
#[derive(Serialize, Deserialize)]
pub struct MergeCompletedEvent {
    pub repo_id: Hash256,
    pub base: Hash256,
    pub left: Hash256,
    pub right: Hash256,
    pub result: Hash256,
    pub conflict_count: u32,
    pub tick: u64,
}

/// 0x100A dependency_updated event payload
#[derive(Serialize, Deserialize)]
pub struct DependencyUpdatedEvent {
    pub repo_id: Hash256,
    pub new_tag: Hash256,
    pub dependent_count: u32,
}
```

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Object put latency | < 5 ms | Single RocksDB write + sha256 hash |
| Object get latency | < 2 ms | Single RocksDB point read |
| Snapshot creation | < 20 ms | Store snap + verify tree references |
| Delta computation | < 100 ms | For trees with up to 1000 entries |
| Three-way merge | < 200 ms | Structural comparison of three trees |
| Registry query | < 50 ms | Index scan + filter |
| Fork (copy-on-write) | < 10 ms | Metadata copy only, objects shared |
| State hash computation | < 500 ms | Full Merkle root over all objects |
| Max object size | 1 MB | Enforced at OBJECT_PUT |
| Max tree entries | 65,536 | Per single TREE object |
| RocksDB compaction | Background | Should not block reads/writes |
| Object deduplication | 100% | Inherent in content-addressing |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Hash256, ObjectId, AgentId, Signature, EventBus, Envelope, crypto |
| Event Bus (Substrate Bus) | Runtime | Publishing events (repo/snap/branch/tag/merge events) |
| RocksDB | Runtime | Persistent content-addressable object store |
| Forge | Runtime (soft) | Proof verification for snapshot proof fields |
| Nexus | Runtime (soft) | Tick deduction for operations, agent identity validation |

Hard startup dependencies: Event Bus, RocksDB.

---

## 9. Key Algorithms

### 9.1 Content Addressing

```
object_put(type_tag, content):
    object_id = sha256(type_tag_byte || content)
    if objects_cf.get(object_id).is_some():
        return object_id  // deduplication: already exists
    objects_cf.put(object_id, type_tag_byte || content)
    publish object_stored event
    return object_id

object_get(object_id):
    raw = objects_cf.get(object_id)?
    type_tag = raw[0]
    content = raw[1..]
    return (type_tag, content)
```

### 9.2 Snapshot Creation

```
snap_create(repo_id, snap):
    // Verify the parent exists (if not initial)
    if snap.parent.is_some():
        assert objects_cf.contains(snap.parent)

    // Verify root tree exists and is a TREE type
    root_raw = objects_cf.get(snap.root)?
    assert root_raw[0] == TREE_TAG

    // Verify signature
    verify_signature(snap.author, snap.signing_bytes(), snap.signature)?

    // Store the snap object
    snap_bytes = msgpack_encode(snap)
    snap_id = sha256(SNAP_TAG || snap_bytes)
    objects_cf.put(snap_id, SNAP_TAG || snap_bytes)

    // Update the default chain head
    repo = repos_cf.get(repo_id)?
    // Advance the chain that snap.parent is on
    update_chain_head(repo_id, snap_id)

    publish snap_created event
    return snap_id
```

### 9.3 Structural Delta Computation

```
delta_compute(base_snap_id, target_snap_id):
    base_snap = load_snap(base_snap_id)
    target_snap = load_snap(target_snap_id)

    base_tree = load_tree_recursive(base_snap.root)
    target_tree = load_tree_recursive(target_snap.root)

    ops = []
    diff_trees([], base_tree, target_tree, &mut ops)

    delta = Delta { base: base_snap_id, target: target_snap_id, ops }
    delta_id = object_put(DELTA, msgpack_encode(delta))
    return delta

diff_trees(path, base, target, ops):
    base_map = { e.key -> e for e in base.entries }
    target_map = { e.key -> e for e in target.entries }

    // Deleted entries (in base but not target)
    for key in base_map.keys() - target_map.keys():
        ops.push(Delete { path: path + [key] })

    // Inserted entries (in target but not base)
    for key in target_map.keys() - base_map.keys():
        ops.push(Insert { path: path + [key], object_id: target_map[key].value })

    // Modified entries (in both but different)
    for key in base_map.keys() & target_map.keys():
        b = base_map[key]
        t = target_map[key]
        if b.value != t.value:
            if b.kind == Tree && t.kind == Tree:
                // Recurse into sub-trees
                diff_trees(path + [key], load_tree(b.value), load_tree(t.value), ops)
            else:
                ops.push(Replace { path: path + [key], old_id: b.value, new_id: t.value })
```

### 9.4 Three-Way Structural Merge

```
merge(base_snap, left_snap, right_snap, resolver):
    base_tree = load_tree_recursive(base_snap.root)
    left_tree = load_tree_recursive(left_snap.root)
    right_tree = load_tree_recursive(right_snap.root)

    left_ops = diff_trees(base_tree, left_tree)
    right_ops = diff_trees(base_tree, right_tree)

    merged_ops = []
    conflicts = []

    // Ops only in left or only in right: apply directly
    left_only = left_ops - right_ops (by path)
    right_only = right_ops - left_ops (by path)
    merged_ops.extend(left_only)
    merged_ops.extend(right_only)

    // Ops on same paths: check for conflicts
    for path in paths_in_both(left_ops, right_ops):
        l = left_ops[path]
        r = right_ops[path]
        if l == r:
            merged_ops.push(l)  // identical changes, no conflict
        else:
            conflicts.push(Conflict { path, left_op: l, right_op: r, resolution: None })

    // Build merged tree by applying merged_ops to base
    result_tree = apply_ops(base_tree, merged_ops)
    result_root_id = store_tree_recursive(result_tree)

    result_snap = Snap {
        parent: Some(left_snap),  // convention: left is primary parent
        root: result_root_id,
        author: resolver,
        message: msgpack({"merge": {"base": base, "left": left, "right": right}}),
        ...
    }
    result_id = snap_create(result_snap)

    return Merge { base, left, right, result: result_id, conflicts, resolver }
```

### 9.5 Dependency Notification

```
on_tag_created(repo_id, new_tag):
    // Find all repos that depend on this repo
    dependents = []
    prefix = to_bytes(repo_id)  // scan deps CF for to_repo == repo_id
    for (key, dep) in deps_cf.prefix_iterator(/* scan all deps where to_repo == repo_id */):
        dependents.push(dep.from_repo)

    if dependents.is_empty():
        return

    // Send notifications
    for dependent_repo in dependents:
        owner = repos_cf.get(dependent_repo).owner
        send DEPENDENCY_NOTIFY { repo_id, new_tag, dependents }

    publish dependency_updated event
```

### 9.6 Merkle State Hash

```
state_hash():
    // Compute a Merkle tree over all objects in the objects CF.
    // Strategy: take sha256 of concatenated sorted object_ids.
    // (Full Merkle tree is optional optimization for partial verification.)

    hasher = Sha256::new()
    for (object_id, _) in objects_cf.iterator():
        hasher.update(object_id)
    return Hash256(hasher.finalize())
```

### 9.7 Access Control Enforcement

```
check_access(repo, agent_id, operation):
    match operation:
        Read:
            match repo.access.read:
                Public -> allow
                AgentsOnly(list) -> allow if agent_id in list
                OwnerOnly -> allow if agent_id == repo.owner
        Write:
            match repo.access.write:
                Open -> allow
                Approved(list) -> allow if agent_id in list
                OwnerOnly -> allow if agent_id == repo.owner
        Fork:
            if !repo.access.fork:
                deny
            else:
                check_access(repo, agent_id, Read)
```
