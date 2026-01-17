# Specification: Thoth — Governance-Indexed Coordination for Concurrent Coding Agents

This document specifies a system that coordinates many concurrent coding agents in a **VCS repository** by introducing a **Thoth control plane** (a governance-indexed coordination layer). The **VCS is the source of truth** for code history (whether Git, JJ, or another backend via adapter). Thoth provides **deterministic binding** from code changes to **governed resources**, injects **relevant rationale/invariants** into agent context, enforces **leases and checks**, and exposes a **queryable architecture graph** for fast agent research.

This spec is written in an **executable-spec style**: it defines **artifacts**, **schemas**, **contracts**, **state machines**, **sequence flows**, and **acceptance tests**.

---

## 1. Goals and Non-Goals

### Goals

1. **Prevent high-cost conflicts** among concurrent agents by coordinating at the level of **architectural surfaces** (interfaces, protocols, schemas, non-mergeables, critical code regions), not just text merges.
2. Provide **fast, deterministic retrieval** of "what governs what" for agents and reviewers.
3. Provide **hard enforcement** where required: leases for non-mergeables/serialized surfaces, targeted checks for invariants, review gates.
4. Provide an **agent-friendly documentation substrate**: from any handle (path, symbol, keyword) agents can deterministically discover relevant subsystems/resources and recursively expand local architecture.
5. Keep the system **minimally intrusive**: most code remains "free," while Thoth applies only to declared resources and policies.

### Non-Goals

1. Not a replacement for VCS, code review, or CI.
2. Not a full-program semantic verifier. The system enforces **declared invariants** via **declared checks**, not via theorem proving.
3. Not a requirement that "all code must be modeled." Coverage is selective.

---

## 2. Core Concepts

**Resource**: A named governed surface. Examples: `user_proto`, `wal_subsystem`, `public_api_pkg_user`, `design_tokens`, `logo_psd`.

**Bindings**: Deterministic mapping from changes to resources using:

* **paths** (glob patterns)
* **symbols** (language-aware fully-qualified names extracted via tree-sitter)
* **regions** (tagged begin/end blocks with stable IDs)

**Invariant**: A must-hold property associated with a resource, referenced as `INVARIANT-XXXX`, documented succinctly and backed by checks.

**ADR capsule**: A short rationale summary `ADR-XXXX` used for context injection; full ADR lives in `docs/adr/`.

**Lease**: A time-bounded exclusive right to modify a resource (or a subset), primarily for non-mergeables or serialized interfaces.

**Checks**: Named executable commands that validate invariants and compatibility constraints for resources.

**Thoth graph**: Typed nodes/edges enabling deterministic exploration: `resource -> regions/symbols/invariants/checks/adrs/deps/revisions`.

---

## 3. Repository Layout and Authoritative Files

The following file paths are normative.

```
.governance/
  governance.hjson                # authoritative bindings + policies (HJSON format)
  invariants/
    INVARIANT-0001.md             # short invariant statements + verification pointers
  adr_capsules/
    ADR-0001.md                   # short capsule; must link to full ADR
  regions/
    README.md                     # rules for region tagging
  treesitter/
    queries/                      # custom tree-sitter queries per language (optional)
docs/
  adr/
    ADR-0001.md                   # full ADR narrative
tools/
  thoth/                          # CLI (or package + wrapper) implementing this spec
  constraints/                    # executable checks referenced from governance.hjson
```

Generated artifacts (optional to commit; must be reproducible):

```
.governance/
  index.sqlite                    # resource graph + history index (recommended)
  index.json                      # alternative to sqlite for minimal install
```

---

## 4. Thoth Manifest Schema (`.governance/governance.hjson`)

This is the single source of truth used by **workers**, **PM bot**, **review agents**, and **CI**. The manifest uses [HJSON](https://hjson.github.io/) format for human-friendly authoring (comments, trailing commas, unquoted keys).

### 4.1 Top-Level Schema

```hjson
{
  version: 1

  resources: {
    <resource_id>: {
      description: <string>
      owners: [<team_or_role>, ...]
      severity: advisory | gated | serialized
      lease: {
        mode: none | exclusive
        ttl_seconds: <int>              // required if exclusive
        scope: resource | regions | paths  // optional; default resource
      }
      bindings: {
        paths: [<glob>, ...]
        symbols: [
          {
            lang: go | ts | proto | py | java | rust | ...
            kind: interface | type | func | service | message | class | module | method | ...
            pattern: <string>           // tree-sitter query pattern or FQN match
            fqname: <string>            // optional: exact fully-qualified name
          }
        ]
        regions: [<THOTH-XXXX>, ...]
      }
      invariants: [INVARIANT-XXXX, ...]
      adrs: [ADR-XXXX, ...]
      checks: [<check_id>, ...]
      deps: [<resource_id>, ...]        // optional explicit resource dependency edges
      tags: [<string>, ...]             // for discovery via map/find
      doc_entrypoints: {
        paths: [<path>, ...]            // optional curated entry points for describe()
        symbols: [<fqname>, ...]        // optional curated entry points
      }
    }
  }

  checks: {
    <check_id>: {
      cmd: <string>                     // shell command; executed in repo root
      timeout_seconds: <int>
      cacheable: true | false
    }
  }

  // VCS adapter configuration
  vcs: {
    adapter: jj | git                   // default: auto-detect
  }
}
```

**Contract**: `resources[*].checks[*]` MUST exist in `checks`.

---

## 5. VCS Abstraction Layer

Thoth treats the VCS as a pluggable backend. The **VCS is the source of truth** for code history. Thoth does not replace or wrap the VCS; it reads from it via an adapter.

### 5.1 Supported Backends

| Backend | Adapter | Revision Identifier | Diff Source |
|---------|---------|---------------------|-------------|
| **JJ (Jujutsu)** | `jj` | Change ID (`@`, revset expression) | `jj diff`, working copy |
| **Git** | `git` | Commit SHA, ref name | `git diff`, index/staging |

### 5.2 VCS Adapter Contract

The adapter MUST implement these operations:

```
vcs.current_rev() -> RevisionID
vcs.diff(base: RevisionID?, target: RevisionID?) -> Diff
vcs.file_at_rev(path: Path, rev: RevisionID) -> Bytes
vcs.log(revset: string, limit: int) -> [RevisionInfo]
vcs.changed_paths(base: RevisionID, target: RevisionID) -> [Path]
```

**RevisionID**: An opaque string representing a revision. For JJ, this is a change ID or commit ID. For Git, this is a commit SHA.

### 5.3 JJ-Specific Semantics

When using the JJ adapter:

* **Revsets** are the query language for selecting revisions (e.g., `@`, `main..@`, `ancestors(@, 10)`)
* **Change IDs** are the stable identifiers (preferred over commit IDs which can change during rewrites)
* **Working copy** diff is obtained via `jj diff` without arguments
* **Immutable revisions** are respected; Thoth never mutates VCS state

### 5.4 CLI Revision Arguments

CLI commands accepting revisions use the `rev:` prefix with backend-appropriate syntax:

```bash
# JJ examples
thoth touch rev:@              # working copy changes
thoth touch rev:main..@        # all changes since main
thoth history <resource> rev:ancestors(@,50)

# Git examples
thoth touch rev:HEAD           # HEAD commit
thoth touch rev:main..HEAD     # all changes since main
thoth touch staged             # staged changes (git-specific)
```

---

## 6. Tree-Sitter Symbol Bindings

Symbol bindings provide language-aware change detection. Thoth uses **tree-sitter** for parsing and symbol extraction, enabling precise detection of when governed symbols are modified.

### 6.1 Symbol Extraction Pipeline

```
Source File -> Tree-sitter Parse -> AST -> Query Execution -> Symbol Table
```

1. **Parse**: Tree-sitter parses source into a concrete syntax tree
2. **Query**: Tree-sitter queries extract named nodes matching symbol patterns
3. **Resolve**: Extracted names are resolved to fully-qualified names (FQNs)
4. **Index**: FQNs are stored in the symbol table with file location

### 6.2 Built-in Language Support

| Language | Symbol Kinds | FQN Format |
|----------|--------------|------------|
| **Go** | `package`, `interface`, `struct`, `func`, `method`, `type` | `module/path.Package.Symbol` |
| **TypeScript** | `class`, `interface`, `function`, `type`, `method`, `export` | `@scope/pkg:path/file.Symbol` |
| **Python** | `class`, `function`, `method`, `module` | `package.module.Symbol` |
| **Rust** | `struct`, `enum`, `trait`, `impl`, `fn`, `mod` | `crate::module::Symbol` |
| **Protobuf** | `service`, `message`, `enum`, `rpc` | `package.Service.Method` |
| **Java** | `class`, `interface`, `method`, `field` | `com.pkg.Class.method` |

### 6.3 Symbol Binding Specification

In `governance.hjson`, symbol bindings can be specified in two ways:

**Exact FQN match**:
```hjson
symbols: [
  { lang: go, kind: interface, fqname: "github.com/org/repo/pkg/api.UserService" }
]
```

**Pattern match** (tree-sitter query):
```hjson
symbols: [
  { lang: go, kind: interface, pattern: ".*Service$" }  // regex on symbol name
]
```

**Custom query** (advanced):
```hjson
symbols: [
  {
    lang: go
    kind: func
    query: '''
      (function_declaration
        name: (identifier) @name
        (#match? @name "^Handle.*"))
    '''
  }
]
```

### 6.4 Tree-Sitter Query Files

Custom queries can be placed in `.governance/treesitter/queries/<lang>.scm`:

```scheme
; .governance/treesitter/queries/go.scm
; Extract all interface methods
(method_spec
  name: (field_identifier) @method.name) @method.def
```

### 6.5 Symbol Change Detection

When `thoth touch` analyzes a diff:

1. Parse old and new versions of changed files
2. Extract symbols from both versions
3. Compute symbol-level diff (added, removed, modified)
4. Match modified symbols against resource bindings
5. Report touched resources with `reason.type = "symbol"`

A symbol is considered **modified** if:
- Its signature changed (parameters, return type)
- Its body changed (for functions/methods)
- It was renamed (detected as remove + add)

---

## 7. Invariant Document Schema (`.governance/invariants/INVARIANT-XXXX.md`)

Normative content sections:

* **Statement** (1–3 sentences, testable)
* **Why** (1–3 sentences)
* **Scope** (what code/resource it applies to)
* **Verification** (named checks + what they prove)
* **Allowed changes / escape hatch** (what kinds of edits are acceptable without redesign)

The first line MUST contain the invariant ID and a title.

---

## 8. ADR Capsules and ADRs

**ADR capsule** (`.governance/adr_capsules/ADR-XXXX.md`) is mandatory if referenced by Thoth.

Required sections:

* **Decision** (bullets allowed, short)
* **Rationale**
* **Constraints / Non-goals**
* **Pointers** (path to full ADR, key code entry points)

**Full ADR** (`docs/adr/ADR-XXXX.md`) is free-form but SHOULD contain a "Consequences" section.

**Contract**: capsule MUST link to full ADR path.

---

## 9. Region Tagging (begin/end blocks)

Used for code blocks not cleanly identified as symbols.

**Syntax (language-agnostic)**

* Begin: `THOTH-BEGIN resource=<resource_id> id=THOTH-XXXX`
* End: `THOTH-END id=THOTH-XXXX`

Example:

```go
// THOTH-BEGIN resource=wal_subsystem id=THOTH-0192
// ... critical loop ...
// THOTH-END id=THOTH-0192
```

**Contracts**

1. `id` MUST be unique repo-wide.
2. Begin and end MUST match and be properly nested (nesting allowed only if explicitly enabled; default: no nesting).
3. Scanner MUST canonicalize region content for hashing (see §13).

---

## 10. Tooling: `thoth` CLI and Library Contracts

`thoth` is the only required integration point. All bots call it rather than re-implementing logic.

The CLI is designed to be **LLM-friendly**: stable verbs, predictable JSON, minimal flags, and "handles" accepted everywhere.

### 10.1 Commands (Normative)

1. **`thoth map [<scope>]`**

   Default entry point for architecture discovery. Returns a structured map of all governed resources, optionally filtered by scope.

   `<scope>` forms (optional):
   * `--tags=<t1,t2>` — filter by tags
   * `--severity=<level>` — filter by severity
   * `--path=<glob>` — filter by path binding overlap

   Output: Hierarchical JSON of resources with summaries, tags, severity, and relationship edges.

2. **`thoth touch <what>`**

   Classify what a change would touch.

   `<what>` forms (exactly one):
   * `working` — working copy changes (JJ: `jj diff`, Git: unstaged)
   * `staged` — staged changes (Git-specific; JJ equivalent: `working`)
   * `rev:<revset>` — VCS revision(s) specified by revset/ref
   * `paths:<p1,p2,...>` — explicit paths
   * `patch:<file>` — patch file

   Output: JSON with touched resources and reasons.

3. **`thoth brief <resources>`**

   Produce deterministic context bundle for injection.

   `<resources>`: comma-separated resource ids (e.g., `wal_subsystem,user_proto`)

   Output includes:
   * invariant **statements** (not full docs)
   * ADR capsules (not full ADRs)
   * required checks
   * lease requirements
   * curated entry points

4. **`thoth lease acquire <resource_id> --holder=<id> [--ttl=<sec>]`**

   Output: success with lease token OR failure with current holder and expiry.

5. **`thoth lease release <resource_id> --token=<token>`**

6. **`thoth lease status [<resource_id>]`**

   Show lease status for one or all resources.

7. **`thoth verify <resources> [--changed-only]`**

   Runs required checks; outputs machine-readable results.

8. **`thoth find <handle>`**

   Handle types: `path:...`, `symbol:...`, `tag:...`, `kw:...` (keyword).

   **Discovery is deterministic**: searches manifest tags, resource descriptions, bindings, and capsule text. No embeddings or vector search.

   Output: ranked list of matching resources with 1-line summaries.

9. **`thoth show <resource_id>`**

   Output: capsule view combining bindings, invariants, checks, ADR capsule pointers, entry points.

10. **`thoth walk <node_id> --edges=<types> --depth=<n>`**

    Deterministic graph traversal for local architecture.

11. **`thoth history <resource_id> [--since=<duration>] [--limit=<N>] [--rev=<revset>]`**

    Output: VCS revisions affecting the resource via bindings (regions/symbols/paths).

12. **`thoth index build`**

    Builds/rebuilds `.governance/index.sqlite` deterministically.

13. **`thoth index symbols [--lang=<lang>] [--path=<glob>]`**

    Extract and display symbol table for debugging tree-sitter bindings.

### 10.2 CLI Contract

* Every command accepts `--json` (default true for bot usage)
* Output schemas are stable; breaking changes require version bump
* Human formatting via `--pretty`
* Exit codes: 0 = success, 1 = error, 2 = policy violation (checks failed, lease denied)

### 10.3 JSON Output Schemas

**`thoth map` output**:
```json
{
  "version": 1,
  "resources": [
    {
      "resource_id": "wal_subsystem",
      "description": "Write-ahead log subsystem",
      "severity": "serialized",
      "tags": ["storage", "critical"],
      "bindings_summary": { "paths": 3, "symbols": 5, "regions": 1 },
      "deps": ["storage_engine"],
      "invariants_count": 2,
      "checks_count": 1
    }
  ],
  "edges": [
    { "src": "wal_subsystem", "dst": "storage_engine", "type": "depends-on" }
  ]
}
```

**`thoth touch` output**:
```json
{
  "vcs": { "adapter": "jj", "rev": "@" },
  "inputs": { "what": "working" },
  "touched": [
    {
      "resource_id": "wal_subsystem",
      "severity": "gated",
      "reasons": [
        { "type": "path", "value": "pkg/storage/wal/segment.go" },
        { "type": "region", "value": "THOTH-0192" },
        { "type": "symbol", "value": "pkg/storage/wal.WAL.Append", "change": "modified" }
      ]
    }
  ],
  "unknown": [
    { "path": "pkg/utils/helper.go", "note": "unbound" }
  ]
}
```

**`thoth brief` output**:
```json
{
  "resources": [
    {
      "resource_id": "wal_subsystem",
      "severity": "gated",
      "lease": { "mode": "exclusive", "ttl_seconds": 300 },
      "invariants": [
        { "id": "INVARIANT-0012", "statement": "...", "verification": ["wal_determinism"] }
      ],
      "adrs": [
        { "id": "ADR-0017", "capsule_path": ".governance/adr_capsules/ADR-0017.md" }
      ],
      "checks": ["wal_determinism", "no_cross_layer_imports"],
      "entrypoints": { "paths": ["pkg/storage/wal/README.md"], "symbols": ["..."] }
    }
  ]
}
```

---

## 11. Lease Authority Model

Thoth supports two normative deployment modes for leases, each with specific semantics for concurrency control.

### 11.1 Mode A: Local-Only Leases (Single-Host)

For single-machine or single-orchestrator deployments where only one agent coordinator runs at a time.

**Characteristics**:
* Leases stored in local SQLite database
* File: `.governance/leases.sqlite` or in-memory
* No network coordination required
* Sufficient when one orchestrator serializes all agent work

**Semantics**:
* CAS (compare-and-swap) via SQLite transactions
* Lease expiry checked against local monotonic clock
* On orchestrator restart, all leases are considered expired

**Configuration**:
```hjson
{
  leases: {
    mode: local
    db_path: .governance/leases.sqlite  // optional; default in-memory
  }
}
```

### 11.2 Mode B: Distributed Leases (Multi-Host)

For deployments with multiple orchestrators or globally distributed agents requiring coordination.

**Characteristics**:
* Leases stored in **Turso/libSQL** (recommended) or compatible transactional store
* Lease service provides HTTP/gRPC interface
* Global consistency via database transactions

**Architecture**:
```
Agent -> thoth CLI -> Lease Service (HTTP) -> Turso/libSQL
```

**Turso/libSQL Requirements**:
* Database URL configured via environment or config
* Table schema (normative):

```sql
CREATE TABLE leases (
  resource_id TEXT PRIMARY KEY,
  holder_id TEXT NOT NULL,
  token TEXT UNIQUE NOT NULL,
  acquired_at INTEGER NOT NULL,  -- Unix timestamp ms
  expires_at INTEGER NOT NULL,   -- Unix timestamp ms
  scope TEXT,                    -- 'resource' | 'regions' | 'paths'
  metadata TEXT                  -- JSON blob for extensions
);

CREATE INDEX idx_leases_expires ON leases(expires_at);
```

**Configuration**:
```hjson
{
  leases: {
    mode: distributed
    service_url: "https://lease.example.com"  // or libsql://...
    turso_url: "libsql://mydb-org.turso.io"
    turso_auth_token_env: "TURSO_AUTH_TOKEN"  // env var name
  }
}
```

### 11.3 Lease State Machine (Normative)

States: `FREE`, `HELD(holder, token, expiry)`

Transitions:

* `FREE -> HELD` on successful acquire (precondition: no active lease)
* `HELD -> FREE` on release by valid token
* `HELD -> FREE` on expiry (expiry checked on any access)
* `HELD -> HELD` on renew by valid token (extends expiry)
* `HELD -> HELD(new)` only via **steal** with admin role (optional)

**Acquire operation** (atomic):
```
IF NOT EXISTS lease for resource_id OR lease.expires_at < now():
  INSERT/UPDATE lease(resource_id, holder, token=random(), expires_at=now()+ttl)
  RETURN success(token)
ELSE:
  RETURN failure(held_by=lease.holder, expires_at=lease.expires_at)
```

**Release operation** (atomic):
```
DELETE FROM leases WHERE resource_id = ? AND token = ?
RETURN affected_rows > 0
```

---

## 12. Deterministic Discovery (No Embeddings)

Discovery in Thoth is **deterministic and reproducible**. There is no vector search, no embeddings, and no ML-based semantic retrieval.

### 12.1 Discovery Mechanisms

1. **`map`**: Primary entry point. Returns full resource graph filtered by scope.
2. **`find`**: Searches across manifest metadata using exact/substring/regex matching.
3. **`walk`**: Traverses explicit graph edges.
4. **`show`**: Displays single resource with all metadata.

### 12.2 `find` Search Algorithm

Given a handle, `find` searches in order:

1. **Exact match** on resource_id
2. **Tag match** on `resources[*].tags`
3. **Binding match** on paths/symbols/regions
4. **Text search** on `description`, invariant statements, ADR capsule text

Text search uses:
* Case-insensitive substring match
* Optional regex with `--regex` flag
* No stemming, no synonyms, no semantic expansion

Results are ranked by:
1. Match type (exact > tag > binding > text)
2. Severity (serialized > gated > advisory)
3. Alphabetical (stable tiebreaker)

### 12.3 Rationale for No Embeddings

* **Reproducibility**: Same query always returns same results
* **Debuggability**: Search behavior is inspectable and explainable
* **Simplicity**: No model dependencies, no drift, no index staleness
* **Speed**: Manifest is small; linear scan is fast enough

If semantic search is desired, it should be implemented as a separate tool that outputs resource IDs, which can then be passed to `thoth brief`.

---

## 13. Indexing: Deterministic Graph + Revision History

The graph is materialized in `.governance/index.sqlite` (preferred) or computed on demand.

### 13.1 Required Tables (SQLite)

```sql
-- Core entities
CREATE TABLE resource (
  resource_id TEXT PRIMARY KEY,
  description TEXT,
  severity TEXT CHECK(severity IN ('advisory', 'gated', 'serialized')),
  owners TEXT,  -- JSON array
  tags TEXT     -- JSON array
);

-- Bindings
CREATE TABLE binding_path (resource_id TEXT, glob TEXT);
CREATE TABLE binding_symbol (
  resource_id TEXT,
  lang TEXT,
  kind TEXT,
  fqname TEXT,
  pattern TEXT,
  query TEXT
);
CREATE TABLE binding_region (resource_id TEXT, region_id TEXT);

-- Invariants and ADRs
CREATE TABLE invariant (invariant_id TEXT PRIMARY KEY, statement TEXT, doc_path TEXT);
CREATE TABLE resource_invariant (resource_id TEXT, invariant_id TEXT);
CREATE TABLE adr (adr_id TEXT PRIMARY KEY, capsule_path TEXT, full_path TEXT);
CREATE TABLE resource_adr (resource_id TEXT, adr_id TEXT);

-- Checks
CREATE TABLE check_def (check_id TEXT PRIMARY KEY, cmd TEXT, timeout_seconds INT, cacheable INT);
CREATE TABLE resource_check (resource_id TEXT, check_id TEXT);

-- Graph edges
CREATE TABLE edge (src TEXT, dst TEXT, edge_type TEXT);

-- Regions
CREATE TABLE region (
  region_id TEXT PRIMARY KEY,
  resource_id TEXT,
  file_path TEXT,
  start_line INT,
  end_line INT
);

-- Symbol table (from tree-sitter)
CREATE TABLE symbol (
  id INTEGER PRIMARY KEY,
  fqname TEXT UNIQUE,
  lang TEXT,
  kind TEXT,
  file_path TEXT,
  start_line INT,
  end_line INT
);
CREATE INDEX idx_symbol_fqname ON symbol(fqname);
CREATE INDEX idx_symbol_file ON symbol(file_path);

-- Revision tracking
CREATE TABLE region_snapshot (region_id TEXT, rev_id TEXT, content_hash TEXT, canonical_len INT);
CREATE TABLE resource_revision (resource_id TEXT, rev_id TEXT);
```

### 13.2 Canonical Hashing for Regions (Normative)

When scanning region `THOTH-XXXX`, canonical content is:

* Lines strictly between begin/end markers
* Normalize line endings to `\n`
* Trim trailing whitespace per line
* Do not remove internal whitespace
* Hash algorithm: SHA-256 of canonical UTF-8 bytes

Hash is used to map revisions to region changes deterministically and to detect drift.

---

## 14. Agent Integration Contracts

There are three agent roles: **PM bot**, **worker bots**, **review bots**. Each interacts with `thoth` in specific ways.

### 14.1 Worker Bot "Write Tool" Hook Contract (Normative)

Whenever a worker proposes to write/patch files:

1. Compute diff source (working copy or proposed patch)
2. Call `thoth touch working` (or appropriate `<what>`)
3. Call `thoth brief <touched_resource_ids>`
4. If any touched resource has `lease.mode=exclusive`, acquire lease **before** applying write
5. Inject brief bundle into the agent's next planning step
6. On "ready to push/review," run `thoth verify <touched_resource_ids>`
7. Attach `touch/brief/verify` outputs as artifacts to the change request

**Hard rule**: a worker MUST NOT push changes touching a `serialized` resource without satisfying lease and checks.

### 14.2 PM Bot Scheduling Contract

When PM bot generates tasks from backlog items:

* PM bot MUST predict candidate touched paths/symbols (best effort)
* PM bot MUST run `thoth touch paths:<...>` to attach `resource_ids` to tasks
* PM bot MUST detect conflicting tasks by overlapping `resource_ids` where `severity in {gated, serialized}` and **write intent** overlaps
* On conflict, PM bot MUST create a "reconciliation task" whose output is either:
  * an ADR update/capsule update and/or
  * an interface spec change
  * plus any required baseline change merged first

Tasks depending on the reconciled surface MUST be blocked on the reconciliation task.

**Write intent model (minimal)**:
Each task has declared intents:
* `reads: [resource_id...]`
* `writes: [resource_id...]`

PM bot assigns `writes` conservatively based on touch results and task type.

### 14.3 Review Bot Contract

Before approval:

* Review bot MUST run `thoth touch` on the diff and load `thoth brief`
* Review bot MUST verify required checks passed
* If change touches `serialized` resources, verify lease compliance metadata exists

---

## 15. Sequence Flows

### 15.1 Worker Planning + Write + Checks

```
WorkerAgent            thoth              LeaseSvc          VCS
    |                   |                  |                |
    |-- propose patch -->|                  |                |
    |                   |-- touch working ->|                |
    |                   |<- touched --------|                |
    |                   |-- brief --------->|                |
    |                   |<- bundle ---------|                |
    |-- (inject bundle; plan)               |                |
    |-- if lease needed: lease acquire ---->|                |
    |                                       |<-- CAS ------->|
    |                                       |--- token/deny->|
    |-- apply write/patch ---------------------------------> VCS
    |-- verify --------------------------->|                 |
    |                   |-- run checks ------------------->  |
    |                   |<- results -------------------------|
    |-- push + artifacts ---------------------------------> Review
```

### 15.2 PM Bot Conflict Detection

```
Backlog     PMBot               thoth             Tracker
  |          |                   |                  |
  |-- item ->|                   |                  |
  |          |-- decompose tasks |                  |
  |          |-- touch plans ---->|                  |
  |          |<- resources -------|                  |
  |          |-- detect write overlap on gated/serialized   |
  |          |-- create reconcile task ----------------->   |
  |          |-- block dependent tasks ----------------->   |
```

### 15.3 Review + Merge Gate

```
ReviewBot             thoth              VCS
    |                 |                 |
    |-- touch rev:X ->|                 |
    |<- resources ----|                 |
    |-- verify ------>|-- run checks -->|
    |<- results ------|                 |
    |-- enforce policy (lease/serialization)
    |-- approve/merge ----------------->|
```

---

## 16. Research Flows: Deterministic Architecture Discovery

Agents should not guess resource IDs. They should use `map` or `find`, then walk.

**Recommended flow**:

```
Agent needs to understand architecture
    |
    v
thoth map                    -> overview of all resources
    |
    v
thoth find <handle>          -> candidate resources for specific query
    |
    v
thoth show <resource>        -> capsule view
    |
    v
thoth walk <resource> --edges=depends-on,constrained-by --depth=2
    |
    v
Read entry points / thoth history as needed
```

**Contract**: `show` and `map` MUST be bounded and fast; they MUST NOT dump entire files.

---

## 17. Policies and Severity Semantics

**Severity = advisory**

* Inject context if touched
* Checks optional (may run if cheap)
* Never blocks

**Severity = gated**

* Inject context
* Required checks MUST pass before "ready for review"
* No lease unless specified

**Severity = serialized**

* Inject context
* Required checks MUST pass
* Exclusive lease required for writes
* Merge-queue serialization at review time

---

## 18. Acceptance Tests (Executable)

These are normative behaviors; implement as integration tests.

1. **Classify by path**
   - Given governance with `paths: ["pkg/storage/wal/**"]`
   - When diff touches `pkg/storage/wal/x.go`
   - Then `thoth touch` returns `wal_subsystem` with `reason.type = "path"`

2. **Classify by symbol (tree-sitter)**
   - Given binding for Go interface `...UserService`
   - When diff changes its signature
   - Then `thoth touch` returns `user_service_api` with `reason.type = "symbol"`

3. **Classify by region**
   - Given region tags for `THOTH-0192` bound to `wal_subsystem`
   - When diff changes region body
   - Then `thoth touch` returns `wal_subsystem` with `reason.type = "region"`

4. **Context injection content**
   - Given resource with invariants and ADR capsule
   - When `thoth brief <id>`
   - Then output includes invariant **statements** and capsule paths, not full ADR text

5. **Lease blocks second writer (distributed mode)**
   - Given `serialized` resource with exclusive lease in Turso
   - When holder A acquires lease
   - Then holder B acquire attempt fails with `held_by=A, expires_at=...`

6. **Check enforcement**
   - Given `gated` resource with required check
   - When check fails
   - Then `thoth verify` returns exit code 2

7. **History correctness**
   - Given two revisions; only one touches `THOTH-0192`
   - When `thoth history wal_subsystem`
   - Then output includes only the touching revision

8. **Map as entry point**
   - When `thoth map`
   - Then returns all resources with severity, tags, binding counts

9. **Find is deterministic**
   - Given query `kw:"write ahead log"`
   - When run twice
   - Then results are identical in order

10. **Symbol extraction (tree-sitter)**
    - Given Go file with interface `UserService`
    - When `thoth index symbols --lang=go`
    - Then symbol table includes FQN for `UserService`

---

## 19. Implementation Phases

**Phase 0 (MVP, single machine)**

* `governance.hjson` + `thoth touch/brief/verify/map`
* Path-based bindings only
* Local-only leases (SQLite)
* JJ or Git adapter (auto-detect)

**Phase 1 (agent-safe)**

* Tree-sitter symbol extraction for primary languages (Go, TypeScript)
* Region tags + hashing
* Distributed leases (Turso/libSQL)
* `find/show/history`

**Phase 2 (documentation substrate)**

* `index.sqlite` build + graph `walk`
* Full tree-sitter language support
* PM bot conflict detection using write intent overlaps

---

## 20. Notes on Pragmatic Coding

The system avoids rigidity by:

* Keeping coverage selective
* Using **severity tiers**
* Allowing **overrides** (optional): a structured "override note" attached to the change, requiring owner acknowledgment for serialized resources
* Making `map` the default entry point so agents always have architectural context available
