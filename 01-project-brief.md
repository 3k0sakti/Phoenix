# 01 — Project Brief

## 1. Context

Large storage systems such as Amazon Dynamo, Cassandra, Riak, Ceph, and HDFS assume that **something is always broken**. At scale, waiting for a human to fix every failed disk or flaky node is not an option, so these systems are designed to:

- **favor availability**: keep serving reads and writes during failures and partitions, accepting temporary inconsistency;
- **detect** missing or divergent replicas automatically;
- **repair** them in the background, without making foreground traffic unusable.

In this project you build such a system yourself and run it across **two physical computers**, so that the failures you heal from are real.

## 2. Learning objectives

After the project, each student can:

1. **Design a decentralized replicated store** using consistent hashing, virtual nodes, and failure-domain-aware replica placement.
2. **Explain the availability/consistency trade-off** (CAP, PACELC). Configure and measure quorum parameters `N`, `R`, `W`, and strict vs. sloppy quorums.
3. **Use vector clocks** to detect concurrent updates, and resolve conflicts without silently losing writes.
4. **Implement failure detection** with gossip, and reason about the trade-off between detection speed and false positives.
5. **Implement and compare self-healing mechanisms**: hinted handoff (transient failures), read repair (on demand), Merkle-tree anti-entropy (background), and re-replication (permanent failures).
6. **Handle deletes correctly** in an eventually consistent system (tombstones, garbage collection, the resurrection problem).
7. **Deploy, break, and measure** a multi-host containerized system. Quantify *time-to-heal*, repair cost, and availability.

## 3. Problem statement

Build **Phoenix**: a cluster of `M` storage nodes (default `M = 6`, three per computer) that stores **records** (key → JSON value). Every record is replicated on `N = 3` nodes.

Phoenix must satisfy these **goals**, in priority order:

| # | Goal | Meaning |
|---|---|---|
| G1 | **Durability** | An acknowledged write is never lost unless all nodes that stored it lose their disks permanently. |
| G2 | **Reachability** | Every record stays readable and writable while any single node is down, and on *both* sides of a partition between the two computers. |
| G3 | **No silent overwrites** | Concurrent updates are detected and kept as *siblings* until merged. Last-writer-wins must never be applied by accident. |
| G4 | **Convergence** | Once faults stop, all replicas of every record become identical within a bounded *time-to-heal*. |
| G5 | **Restored redundancy** | When a node is permanently gone, every record is brought back to `N` replicas, spread over both computers, without human action. |
| G6 | **No resurrection** | A deleted record does not reappear when a stale node returns. |

## 4. Requirements

### 4.1 Core functional requirements (all mandatory)

| ID | Requirement |
|---|---|
| F1 | **Consistent hashing ring** with virtual nodes. Each key has a *preference list* of `N` nodes. |
| F2 | **Zone-aware placement**: each key's `N` replicas span **both computers** (e.g., 2+1). |
| F3 | **Coordinator-based quorum operations**: any node can coordinate a `Get`/`Put`/`Delete`. Configurable `N, R, W` (default 3/2/2). |
| F4 | **Versioning with vector clocks**: a `Put` carries the causal *context* of the client's last read. Concurrent versions are kept as siblings and returned to the client for merging. |
| F5 | **Gossip-based membership and failure detection**: heartbeat counters gossiped every `T_gossip`. Node states are `ALIVE → SUSPECT → DEAD`. |
| F6 | **Sloppy quorum and hinted handoff**: if a preference-list node is unreachable, the write goes to the next healthy node with a *hint*. The hint is delivered when the target node returns. |
| F7 | **Read repair**: when a read finds stale or missing replicas, the coordinator pushes the merged newest version to them. |
| F8 | **Merkle-tree anti-entropy**: nodes periodically compare per-range Merkle trees with their replica peers and sync only the differing keys. |
| F9 | **Re-replication on permanent failure**: a node `DEAD` for longer than `T_dead` is removed from the ring view. Its ranges are re-replicated to restore `N` copies. When it comes back, it rejoins, catches up, and temporary replicas are garbage-collected. |
| F10 | **Deletes with tombstones**, garbage-collected after a grace period `T_gc`, with the resurrection risk documented. |
| F11 | **Persistence**: each node stores records and hints on a Docker volume, with fsync before acknowledging. A node that loses its volume rebuilds from its peers. |
| F12 | **Dashboard**: ring view, node states per computer, a **replication-health meter** (fully replicated / under-replicated / diverged keys), pending hints, and repair traffic. |
| F13 | **Workload generator** with history recording, plus a **healing checker** (see [05](05-testing-and-evaluation.md)). |
| F14 | **2-computer Docker deployment** with one documented command per computer (see [03](03-deployment-docker.md)). |

### 4.2 Extensions (implement **at least one**; two, or one ⭐, for a top grade)

| ID | Extension | Textbook link |
|---|---|---|
| E1 ⭐ | **Raft-based configuration service**: a small Raft group (your own implementation) holds the authoritative ring and membership, giving a strongly consistent control plane next to the AP data plane. Compare it with pure gossip. | Ch. 5, Ch. 8 |
| E2 | **CRDTs**: make records automatically mergeable (OR-Set shopping cart, PN-Counter, LWW-Map with HLC) instead of returning siblings to the client. | Ch. 7 |
| E3 | **Advanced failure detection**: SWIM (indirect probing) and/or a φ-accrual detector. Measure false positives vs. detection time under Wi-Fi jitter. | Ch. 8 |
| E4 ⭐ | **Erasure coding**: store records as Reed-Solomon fragments (e.g., k=4, m=2) instead of 3 full copies. Compare storage overhead, repair bandwidth, and read latency. | Ch. 7, Ch. 8 |
| E5 | **Client-centric consistency**: session guarantees (read-your-writes, monotonic reads) using session vector tokens, even when the client switches coordinators. | Ch. 7.3 |
| E6 | **Scrubbing and bit-rot repair**: per-record checksums, a background scrubber, and automatic repair of *corrupted* replicas, demonstrated by flipping bytes on disk. | Ch. 8 |
| E7 | **Elastic rebalancing**: add or remove a node while traffic runs, with bounded data movement and throttled streaming. Measure how many keys move. | Ch. 6 |
| E8 | **Repair throttling and priorities**: rate-limit repair to protect foreground p99 latency; repair the least-replicated keys first. Show the trade-off curve. | Ch. 3, Ch. 8 |

### 4.3 Non-functional requirements

| ID | Requirement |
|---|---|
| N1 | Runs on **2 physical computers** on the same LAN, with at least 2 nodes per computer. |
| N2 | **Reachability:** during a single-node failure, ≥ 99.9% of reads and writes succeed. During a partition of the two computers, ≥ 99% succeed on *each* side (sloppy quorum on). |
| N3 | **Time-to-heal:** after a node that was down for 60 s returns, all its replicas converge within **30 s** for 100k records (default settings). |
| N4 | **Restored redundancy:** after a permanent node loss, every key is back at `N` replicas within **5 min** for 100k records of 1 KB. |
| N5 | ≥ **2,000 ops/s** (mixed 80/20 read/write, 1 KB records) on the 2-computer cluster (sanity floor). |
| N6 | `make test` runs the automated suite (single machine, simulated network) in under 10 minutes. |

## 5. Out of scope

- Strong consistency / linearizability for data operations (except in E1's control plane).
- Byzantine faults (discussed only).
- Multi-key transactions, secondary indexes, query languages.

## 6. Deliverables

1. Git repository: source, `Dockerfile`s, compose files, tests, chaos scripts, and a README that lets another team reproduce the 2-computer setup.
2. Design document (Week 1, 2–3 pages).
3. Final report (max 8 pages, see [07-report-template.md](07-report-template.md)).
4. "Kill & Heal" live demo (15 min) on 2 computers, plus Q&A.
5. `CONTRIBUTIONS.md`: half a page per student.
