# 05 — Testing and Evaluation

A self-healing system is judged on two questions. Did it stay reachable while broken? Did it actually heal, without losing or resurrecting data? This document defines the healing checker, the automated test suite, and the required experiments.

| Level | Where | Network | Purpose |
|---|---|---|---|
| **Automated suite** | One machine, `make test` | In-process simulated network (deterministic, seeded) | Fast, reproducible correctness testing. Hundreds of runs. |
| **Deployment chaos** | 2 computers, Docker | Real LAN plus `docker kill`, volume wipes, `iptables`, `tc`, cable pulls | Shows that it works for real. All experiments run here. |

Language note: Go is the easiest choice (goroutines, gRPC, a good standard library for hashing). With Python, lower N5 to 500 ops/s and discuss why.

---

## 1. The healing checker

Every chaos run has three phases:

```
|---- CHAOS (faults + workload) ----|---- HEAL (faults stopped, workload optional) ----|-- CHECK --|
                                     t_stop                                  t_converged
```

**Time-to-heal** = `t_converged − t_stop`, where `t_converged` is the first moment at which every key is fully replicated and identical on all its replicas (from `/health` on all nodes, or Merkle roots equal for every range).

The checker reads **every replica directly** through a debug endpoint that bypasses quorums, and verifies:

| Check | Pass condition |
|---|---|
| C1 Durability (P1) | Every acknowledged `Put` is either present as a sibling in the final state, **or** causally superseded: some later version's vector clock dominates it, and that version was written with a context that included it. |
| C2 No silent drop (P2) | No concurrent acknowledged write disappeared without being in a merge context |
| C3 Convergence (P3) | All replicas of every key hold an identical version set |
| C4 Replication factor (P4) | Every key has exactly `N` replicas on the current preference list. Temporary copies are gone after the GC grace period. |
| C5 Zone diversity (P6) | Every key has replicas on both computers (when both are alive) |
| C6 No resurrection (P5) | No key whose last acknowledged operation was a `Delete` is live, unless a later `Put` exists |
| C7 Availability stats | Success rate of reads and writes during the chaos phase, per zone (reported, not pass/fail) |
| C8 Time-to-heal | Reported, and must be under the configured bound (e.g., 60 s in tests) |

Output example:

```
run=chaos-seed-4471  ops=182340  acked_puts=36120  deletes=1804
availability: zoneA reads 99.97% writes 99.95% | zoneB reads 99.91% writes 99.88%
time_to_heal=11.4s  repair_bytes: hints=4.1MB readrepair=0.6MB merkle=2.3MB rereplication=0B
C1 lost_writes=0  C2 silent_drops=0  C3 diverged_keys=0  C4 underreplicated=0 orphan_copies=0
C5 single_zone_keys=0  C6 resurrected=0
RESULT: PASS
```

---

## 2. Automated test suite (`make test`)

All randomized tests log their seed. Failed seeds must be replayable.

### 2.1 Ring and placement
| Test | Pass condition |
|---|---|
| `TestPreferenceListDistinct` | `N` distinct physical nodes per key |
| `TestZoneAwarePlacement` | Every key spans both zones |
| `TestLoadBalance` | The stddev of keys per node decreases as `V` grows (reported) |
| `TestRingDeterminism` | Identical membership views produce identical preference lists on all nodes |

### 2.2 Versioning
| Test | Pass condition |
|---|---|
| `TestDominanceAndConcurrency` | Unit tests of the vector-clock compare/merge laws (associative, commutative, idempotent merge) |
| `TestConcurrentWritesSiblings` | Two writes with the same context produce 2 siblings, never 1 |
| `TestMergeSupersedes` | A write with the merged context replaces all siblings |
| `TestTombstoneDominance` | A delete dominates the versions in its context. A concurrent put survives as a sibling. |

### 2.3 Failure detection
| Test | Pass condition |
|---|---|
| `TestDetectCrash` | A crashed node becomes `SUSPECT` within `T_suspect + ε` and `DEAD` within `T_dead + ε` on every node |
| `TestNoFalseDeadUnderJitter` | 200 ms jitter and 5% loss do not cause `DEAD` declarations |
| `TestRejoinIncarnation` | A returning node is `ALIVE` everywhere despite old `DEAD` rumors |

### 2.4 Healing mechanisms
| Test | Pass condition |
|---|---|
| `TestHintedHandoff` | A node down for 10 s receives all missed writes from hints. The hints are deleted afterwards. |
| `TestHintsSurviveFallbackRestart` | The fallback node restarts before delivering, and the hints are still delivered |
| `TestReadRepair` | A stale replica is fixed after one read of the key |
| `TestMerkleFindsColdDivergence` | With hints disabled, unread keys still converge through anti-entropy. The number of exchanged keys ≈ the number of divergent keys. |
| `TestReReplication` | A node is `DEAD` past `T_dead`, and every key has `N` replicas again |
| `TestPhoenixEmptyDisk` | A node restarts with an empty volume and has its full data set after healing |
| `TestHandoffGCSafe` | Temporary replicas are deleted only after the returning node's ranges match |
| `TestResurrection` | (a) Demonstrates resurrection with naive tombstone GC. (b) Your fix prevents it. |
| `TestPartitionBothSidesWrite` | During a 2-zone partition, both sides accept writes. After healing, C1–C6 pass. |

### 2.5 End-to-end chaos
| Test | Pass condition |
|---|---|
| `TestChaosHeal` | 6 nodes, 2 zones, mixed workload, random crashes, restarts, wipes, partitions, and loss for 60 s, then heal, then **checker PASS**. Run ≥ 200 times and report the pass rate. |

### 2.6 Extension tests
At least two targeted tests per extension. Examples: an OR-Set cart converges after a partition without client merge (E2); a SWIM false-positive rate below the heartbeat detector's under jitter (E3); one lost EC fragment rebuilt from `k` others (E4); a flipped byte detected and repaired by the scrubber (E6).

---

## 3. Required experiments (on the 2-computer deployment)

For each experiment, give: **hypothesis → setup → result (figure with CI/percentiles) → interpretation tied to theory**. Use ≥ 5 repetitions per configuration. Report the measured RTT and bandwidth.

| # | Experiment | Vary | Measure | Example hypothesis |
|---|---|---|---|---|
| X1 | **Failure detection** | `T_gossip`, `T_suspect`, injected jitter | Detection latency; false `SUSPECT`/`DEAD` per hour | "Halving `T_suspect` halves detection time, but false suspicions grow sharply once `T_suspect` < ~10× the RTT jitter." |
| X2 | **Reachability under faults** | 0/1/2 nodes down, a full computer down, A⟷B partition; strict vs. sloppy quorum | Read/write success rate per zone over time | "Sloppy quorum keeps ≥ 99% write availability on both sides of a partition. Strict quorum loses all writes whose quorum spans both zones." |
| X3 | **Healing after a transient failure** | Outage duration (5 s, 30 s, 120 s); hinted handoff on/off | Time-to-heal, stale-read rate during healing, repair bytes per mechanism | "Hints make healing time proportional to the writes missed. Without hints, anti-entropy dominates and healing is slower but still bounded." |
| X4 | **Anti-entropy efficiency** | Number of divergent keys (10 … 100k), Merkle tree depth | Bytes exchanged, sync time; compared with naive full-range exchange | "Merkle sync bytes grow with the number of differences × log(leaves), not with the total data size." |
| X5 | **Re-replication / Phoenix rebuild** | Data volume (10k–500k records); repair throttle on/off | Time to restore `N` replicas; foreground p99 latency during repair | "Rebuild time ≈ data / min(disk, LAN bandwidth). Unthrottled repair raises client p99 a lot." |
| X6 | **Staleness (PBS)** | `R`, `W`, strict/sloppy, time since write | Probability that a read returns a stale version | "With R+W>N and a strict quorum, stale reads are ~0 without faults. Sloppy quorums during failures produce measurable staleness that decays as hints are delivered." |
| X7 | **Extension-specific** | E.g., CRDT vs. sibling merge; SWIM vs. heartbeat; EC vs. replication; throttle rate | Your choice | Your own |

Plot rules:
- Labeled axes with units, and legends.
- Time-series plots mark fault events from the `chaos.sh` log, and plot the **health meter** (% fully replicated) over time. This is the key figure of the report.
- Every figure can be regenerated from `eval/` by one script.

---

## 4. Practical notes on checking

- Give each workload write a **unique value**, e.g., `(clientId, seq)` added to a cart, so the checker can trace every acknowledged write to the final state.
- A **timed-out** write has an unknown outcome. It may or may not appear, and the checker must accept both. It must never appear *twice* as different versions. Explain why it can't.
- **Mutation tests (required).** Break the system on purpose and show that the checker catches it. Examples: replace vector clocks with wall-clock LWW (C1/C2 fail); GC tombstones immediately (C6 fails); GC temporary replicas before confirmation (C1 fails after a wipe); turn off zone-awareness (C5 fails, availability drops when a computer is lost). Put at least two of these in the report.

---

## 5. Definition of done (minimum to pass)

- All tests in §2.1–2.5 pass.
- `TestChaosHeal` passes ≥ 99% of 200 runs, and every failure is explained.
- The 2-computer "Kill & Heal" demo works: node kill, cable pull, and volume wipe all heal to 100%.
- X1–X6 are complete. At least one extension is implemented and evaluated (X7).
