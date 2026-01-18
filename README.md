# BedrockDB

**BedrockDB** is a **disk-based, log-structured key-value database** built from first principles to explore **database internals, durability guarantees, and distributed systems tradeoffs**. It is designed with a strong emphasis on **write throughput, crash safety, and predictable performance**, while using memory purely as an optimization layer.

> Disk is the source of truth. Memory accelerates, but never defines correctness.

---

## Why BedrockDB?

Modern databases hide complexity behind abstractions. BedrockDB intentionally exposes and documents those complexities to answer questions like:

* How do databases guarantee durability?
* Why do LSM trees favor writes but complicate reads?
* Why is compaction the real bottleneck?
* How do replicas recover after failures?

BedrockDB is both:

* A **practical storage engine**, and
* A **learning-oriented database internals project**

---

## High-Level Architecture

```
┌───────────────────────────────────────────┐
│                 Client                    │
│        (gRPC / REST / CLI)                │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│               Query Layer                 │
│  - Simple SQL-like DSL                    │
│  - Point lookups                          │
│  - Range scans                            │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│             Storage Engine                │
│                                           │
│  ┌───────────────┐     ┌───────────────┐  │
│  │   MemTable    │◀───▶│   Bloom       │  │
│  │ (In-Memory)   │     │   Filters     │  │
│  └───────┬───────┘     └───────────────┘  │
│          │                                  │
│          ▼                                  │
│  ┌─────────────────────────────────────┐  │
│  │            SSTables (Disk)           │  │
│  │  - Immutable                          │  │
│  │  - Sorted                             │  │
│  │  - Indexed                            │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │          Compaction Engine           │  │
│  │  - Size-tiered / Leveled              │  │
│  │  - Tombstone cleanup                  │  │
│  └─────────────────────────────────────┘  │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│        Write-Ahead Log (WAL)               │
│  - Append-only                            │
│  - fsync for durability                  │
│  - Crash recovery                        │
└─────────────────────┬─────────────────────┘
                      │
┌─────────────────────▼─────────────────────┐
│          Replication Layer                │
│  - Leader–Follower                       │
│  - WAL shipping                          │
│  - Follower catch-up                     │
└───────────────────────────────────────────┘
```

---

## Storage Model

BedrockDB is **not an in-memory database**.

### Disk (Source of Truth)

* **Write-Ahead Log (WAL)**

  * Every mutation is appended to disk
  * fsync determines durability
* **SSTables**

  * Immutable, sorted on-disk files
  * Persisted across restarts

If the process crashes, data is recovered entirely from disk.

### Memory (Performance Optimization)

* **MemTable**: In-memory sorted buffer for recent writes
* **Bloom Filters**: Reduce unnecessary disk reads
* **Caches**: Optional and rebuildable

All in-memory structures can be reconstructed from disk.

---

## Write Path

```
Client
 → WAL (append + fsync)
 → MemTable (in-memory)
 → Async flush to SSTables
 → Background compaction
```

**Durability guarantee**: A write is acknowledged only after the WAL is safely persisted to disk.

---

## Read Path

```
Client
 → MemTable
 → Bloom Filter
 → SSTables (newest → oldest)
```

Reads are optimized to avoid disk access when possible, but correctness never depends on memory.

---

## Compaction

BedrockDB uses **LSM-tree compaction** to:

* Merge immutable SSTables
* Remove overwritten keys
* Clean up tombstones
* Control read amplification

Compaction runs in the background and is carefully throttled to avoid write stalls.

> Compaction is the real cost of write optimization.

---

## Replication & Consistency

BedrockDB follows a **leader–follower replication model**:

* All writes go to the leader
* WAL entries are shipped to followers
* Followers replay WAL for consistency
* Followers can serve read traffic (configurable)

Consistency guarantees are explicit and configurable, favoring clarity over hidden behavior.

---

## Crash Recovery

On restart:

```
1. Read WAL from disk
2. Replay entries into MemTable
3. Resume normal operation
```

Recovery is deterministic and does not require external coordination.

---

## Observability

BedrockDB exposes:

* Write / read latency metrics
* Compaction statistics
* Replication lag
* Health endpoints

Every subsystem is measurable and inspectable.

---

## What BedrockDB Is NOT

* Not a production-ready database
* Not optimized for low-latency in-memory workloads
* Not feature-complete SQL engine

This project prioritizes **correctness, transparency, and learning** over features.

---

## Project Goals

* Understand real-world database internals
* Make performance tradeoffs explicit
* Build failure-tested systems
* Serve as a strong portfolio signal for backend / database roles

---

## Roadmap (High Level)

* Secondary indexes
* Leveled compaction
* Snapshot isolation
* Multi-leader replication
* Pluggable storage formats

---

## Author

Built by an engineer with a focus on **databases, distributed systems, and performance engineering
