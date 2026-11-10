# 02 — System Specification

## 1. Architecture overview

```
        COMPUTER A  (zone A, e.g. 192.168.1.10)          COMPUTER B  (zone B, e.g. 192.168.1.20)
  +---------------------------------------------+   +---------------------------------------------+
  |  +-------+   +-------+   +-------+          |   |          +-------+   +-------+   +-------+  |
  |  | n1    |   | n2    |   | n3    |<---------+---+--------->| n4    |   | n5    |   | n6    |  |
  |  +---+---+   +---+---+   +---+---+  gossip, |   | replicate +---+---+   +---+---+   +---+---+  |
  |   [vol]       [vol]       [vol]     repair  |   |            [vol]       [vol]       [vol]    |
  |                                             |   |                                             |
  |  +-----------+                              |   |  +-------------+   +------------------+     |
  |  | dashboard |  :8080                       |   |  | workload-gen|   | healing-checker  |     |
  |  +-----------+                              |   |  +-------------+   +------------------+     |
  +---------------------------------------------+   +---------------------------------------------+
                          \______________ LAN (wired preferred) ______________/
```

All nodes are **equal peers**. There is no master. Any node can coordinate any request.

Components inside each node:

```
+-------------------------------------------------------------+
| Client API (HTTP/gRPC)   |   Internal peer API              |
+-------------------------------------------------------------+
| Coordinator: quorum Get/Put/Delete, sibling merge,          |
|              read repair, sloppy quorum + hint creation     |
+-------------------------------------------------------------+
| Ring & placement: consistent hashing, vnodes, zone-aware    |
|                   preference lists                          |
+-------------------------------------------------------------+
| Membership: gossip, heartbeat counters, ALIVE/SUSPECT/DEAD  |
+-------------------------------------------------------------+
| Healing workers: hint delivery | anti-entropy (Merkle) |    |
|                  re-replication | handoff GC | tombstone GC |
+-------------------------------------------------------------+
| Local storage: records + vector clocks + hints              |
|                (embedded engine or own log, Docker volume)  |
+-------------------------------------------------------------+
```

## 2. Data model and client API

A **record** is `key → { siblings: [ {value: JSON, vclock: {...}, deleted: bool} ] }`.

| Operation | Input | Output |
|---|---|---|
| `Get(key)` | key | `siblings[]`, opaque `context` (merged vector clock) \| `NOT_FOUND` |
| `Put(key, value, context)` | key, value, context from the last `Get` (empty for a blind write) | `ok`, new `context` |
| `Delete(key, context)` | key, context | `ok` (writes a tombstone) |

Optional per-request overrides: `R`, `W`, `sloppy=true|false`.

**Client contract:** if `Get` returns more than one sibling, the application merges them and writes the result back with the returned `context`, which supersedes all siblings. The demo application is a **shopping cart** (merge = union of items) and a **user profile** (merge = field-wise, application-defined).

## 3. Ring and placement

- Hash space: 64- or 128-bit (e.g., the first bytes of SHA-1/xxHash). Each physical node owns `V` virtual nodes (default `V = 64`).
- **Preference list for a key:** walk the ring clockwise from `hash(key)` and collect distinct *physical* nodes until you have `N`, **under the zone constraint that both zones are represented**. With N=3 and 2 zones, you get 2 replicas in one zone and 1 in the other.
- **Extended preference list:** continue the walk past the first `N` nodes to get fallback nodes, which sloppy quorums use.
- The ring is derived from the current **membership view**, so all nodes compute identical preference lists once their views agree.

Analyze in the report: the load distribution (keys per node) as a function of `V`, and what happens to placement when your zone constraint is turned off (measure how many keys would have all replicas on one computer).

## 4. Versioning with vector clocks

- Each replica node that coordinates a write increments its own entry: `vclock[coordinatorId]++`, starting from the client-supplied context.
- On receiving a version, a replica keeps every version that is **not dominated** by another one: `v1 ≤ v2` (entrywise) means v1 is obsolete. Incomparable clocks mean the versions are **concurrent**, and both are kept as siblings.
- A tombstone is a version with `deleted = true`. It takes part in dominance like any other version.
- Vector-clock growth: entries can be pruned when they exceed a size limit. If you prune, explain the risk (a sibling may be dropped wrongly) in the report.

## 5. Quorum operations

**Put / Delete** (coordinator = any node; ideally the first node in the preference list):
1. Create the new version (bump the vector clock) and write it locally if the coordinator is a replica.
2. Send it to all `N` nodes in the preference list.
3. If a target is not `ALIVE` or does not answer within `T_rpc`, then with `sloppy=true` send the write to the **next fallback node**, tagged with a *hint* `{intendedFor: nX}`.
4. Reply `ok` after **`W` durable acks** (fsync'd). Otherwise reply `UNAVAILABLE`.

**Get:**
1. Ask the `N` preference-list nodes (or fallbacks if they are down) for their versions.
2. Wait for `R` responses, then merge: drop dominated versions and keep concurrent ones.
3. **Read repair** (asynchronous): any responding replica that is missing the merged result, or holds an older version, gets the merged versions pushed to it.
4. Return the siblings plus the merged context.

`R + W > N` with a **strict** quorum gives "read sees the latest acknowledged write" in the absence of failures. With **sloppy** quorums that guarantee is gone, because the fallback nodes may not overlap. Quantify this staleness (experiment X6).

## 6. Membership and failure detection

- Each node keeps `{nodeId → (heartbeatCounter, lastUpdatedLocalTime, state, zone, addr, incarnation)}`.
- Every `T_gossip` (default 500 ms) a node increments its own counter and gossips its table to `k` random peers (default 2). Peers merge the tables, keeping the higher counter.
- A node is `SUSPECT` if its counter hasn't increased for `T_suspect` (default 3 s), and `DEAD` after `T_dead` (default 60 s; use a small value in tests).
- `SUSPECT` nodes are skipped for new writes, which then use fallbacks. **Only `DEAD` changes the ring** and triggers re-replication. This split keeps a short hiccup from causing an expensive data shuffle.
- A returning node increases its `incarnation`, so that old `DEAD` rumors cannot override its fresh `ALIVE` state.
- Seed nodes (at least one per computer) are configured for bootstrapping.

## 7. Self-healing mechanisms

Phoenix heals at four levels. They differ in **when** they run and **what** they fix:

| Mechanism | Triggered by | Fixes | Cost |
|---|---|---|---|
| **Hinted handoff** | Node returns (`SUSPECT/DEAD → ALIVE`) | Writes the node missed during a *short* outage | Cheap: only the hinted writes |
| **Read repair** | A client read finds divergence | Stale replicas of *frequently read* keys | Free on hot keys, nothing for cold keys |
| **Anti-entropy (Merkle)** | Periodic timer (`T_ae`, e.g., 10 s per range pair) | *Any* divergence, including cold keys and lost hints | Bandwidth ∝ differences, not data size |
| **Re-replication** | Node becomes `DEAD` (> `T_dead`) | Missing replicas after a *permanent* loss | Copies the whole range, so throttling is needed |

### 7.1 Hinted handoff
- Hints are stored durably on the fallback node: `(intendedFor, key, version)`.
- When gossip shows the intended node `ALIVE` again, the fallback streams its hints to it, deletes each hint after an ack, and drops its own copy if it is not itself a replica of the key.
- Hints for a node that becomes `DEAD` are not needed: re-replication takes over. Document what you do with them.

### 7.2 Merkle-tree anti-entropy
- Each node keeps, **per key range it replicates**, a Merkle tree (fixed depth, e.g., 2^10 leaves). A leaf is the hash of the `(key, sorted versions)` pairs in a hash-bucket.
- Periodically, for each range, a node picks a peer replica of that range and compares trees top-down. Only the subtrees whose hashes differ are explored, and only keys in differing leaves are exchanged. Both sides merge the versions (the vector-clock rules from §4).
- Trees are updated incrementally on writes, or rebuilt lazily with a dirty flag. Document which you chose.

### 7.3 Re-replication and rejoin ("Phoenix")
1. Node `nX` becomes `DEAD`. Every node recomputes the ring without `nX`, so preference lists now include replacement nodes that respect the zone constraint.
2. Anti-entropy on the new preference lists fills the replacement replicas automatically. The **under-replicated** count on the dashboard drops to 0.
3. `nX` returns, with its old disk or an **empty** one. It rejoins as `ALIVE` with a new incarnation, the ring includes it again, and anti-entropy brings it up to date.
4. Once `nX`'s Merkle roots match its peers for a range, the temporary replacement replicas for that range are **garbage-collected** (handoff GC). Never delete a temporary replica before the returning node is confirmed up to date. Explain why.

### 7.4 Tombstones and the resurrection problem
- A delete writes a tombstone, which is replicated like any other write.
- Tombstones are removed after `T_gc` (e.g., 10 min in production, 30 s in tests), but **only if** anti-entropy has confirmed that all `N` replicas hold the tombstone.
- If a node that was offline longer than `T_gc` returns with the old live value, the record can *resurrect*. Demonstrate the problem with a test, then prevent it. Possible approaches: refuse to rejoin with old data after `T_gc` (wipe and rebuild), or keep tombstones until all replicas acknowledge them.

## 8. Health metrics (for dashboard and checker)

Every node exposes `/health` and `/metrics`:

| Metric | Meaning |
|---|---|
| `keys_fully_replicated` | Keys whose `N` preference-list replicas all hold the same version set |
| `keys_under_replicated` | Keys with fewer than `N` live replicas |
| `keys_diverged` | Keys whose live replicas disagree |
| `hints_pending{target}` | Queued hints per target node |
| `repair_bytes_total{mechanism}` | Bytes moved by handoff / read repair / anti-entropy / re-replication |
| `membership_view` | The node's view of every peer's state |

Dashboard (any framework; plain HTML and polling is fine):
- A **ring visualization** with nodes colored by state and grouped by computer.
- The **replication-health meter** (% fully replicated), plus under-replicated and diverged counts over time.
- Repair traffic per mechanism, pending hints, and a timeline of fault events.

## 9. Guarantees to document (and check)

| # | Guarantee | Condition |
|---|---|---|
| P1 | No acknowledged write is lost | Fewer than all `W` acknowledging nodes lose their disk permanently |
| P2 | Siblings are never silently dropped | Vector-clock rules, no pruning bug |
| P3 | Convergence | Faults stop, every node is `ALIVE` or `DEAD`, enough time passes |
| P4 | Replication factor restored | ≥ `N` nodes alive across both zones, `T_dead` elapsed |
| P5 | No resurrection | Within your tombstone policy |
| P6 | Zone diversity | After healing, every key has replicas on both computers |
| P7 | Read-your-writes (strict quorum, no failures) | `R + W > N`, `sloppy=false` |

## 10. Extension notes

- **E1 Raft config service:** 3 Raft nodes (2 on A, 1 on B; analyze this placement!) store the ring and the `DEAD` decisions. Data nodes read the ring from it. Compare split-brain behavior of the ring (gossip views can disagree during a partition) with the Raft approach. Which side may declare nodes dead during a partition?
- **E2 CRDTs:** replace client-side sibling merge with state-based CRDTs. Show that convergence happens automatically after a partition, without application code.
- **E3 SWIM / φ-accrual:** compare false-positive rate and detection latency with the simple heartbeat detector, under `tc netem` jitter.
- **E4 Erasure coding:** `k + m` fragments placed across zones. Repairing one lost fragment needs `k` reads. Compare with replication.
- **E5 Session guarantees:** the client keeps a session vector. Coordinators wait or redirect until they can serve a version ≥ the session vector.
- **E6 Scrubbing:** CRC/SHA per version stored on disk. A scrubber walks the data and treats a checksum mismatch as a missing replica, which triggers repair.
- **E7 Rebalancing:** a joining node streams the ranges it now owns, and the old owners drop them after confirmation. Report the fraction of keys moved (should be ≈ 1/M).
- **E8 Throttling:** a token bucket per repair mechanism, with priority by "fewest live replicas first."
