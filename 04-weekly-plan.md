# 04 — Four-Week Plan

Each week ends with a **10-minute milestone check-in** (demo plus a short written update). M1–M3 earn points for being on time and complete. They are formative, not graded in depth.

---

## Week 1 — Design, 2-computer setup, ring and replication

**Reading:** Textbook Ch. 2 (decentralized architectures), Ch. 6 (DHTs, consistent hashing), Ch. 7.4 (replica management) · DeCandia et al., *Dynamo* §1–4.3

### Tasks
- [ ] Repository, build, CI, node `Dockerfile`.
- [ ] **Deployment checklist from [03-deployment-docker.md](03-deployment-docker.md) §6.** Do this on day 1–2. Networking is the biggest schedule risk.
- [ ] Local storage layer on a volume (embedded engine or your own log) with fsync.
- [ ] Consistent hashing ring with virtual nodes, **zone-aware** preference lists.
- [ ] Coordinator `Put`/`Get` with N/R/W, *static* membership (from config), no failures yet.
- [ ] Peer RPC layer with a **fault-injection hook** (drop/delay/partition), for the automated tests.
- [ ] **Design document** (2–3 pages): architecture, placement over the two zones and what survives what, versioning plan, failure-detector parameters, the four healing mechanisms and how they interact, guarantees P1–P7, and the chosen extension(s).

### M1 deliverable
- Design document.
- Demo **on 2 computers**: write 10k records through any node. Show that every key's 3 replicas span both computers (a `/debug/placement?key=` endpoint). Read every record back from a node on the other computer.
- Test: load distribution (keys per node) for V ∈ {1, 16, 64, 256}.

### Pitfalls
- Collecting `N` *virtual* nodes that belong to the same physical node.
- Hardcoded container IPs in the configuration.

---

## Week 2 — Versioning, failure detection, sloppy quorum, hinted handoff

**Reading:** Textbook Ch. 5 (vector clocks), Ch. 4 (gossip/epidemic protocols), Ch. 8.1–8.2 (failure models, failure detection) · Dynamo §4.4–4.8 · Demers et al., *Epidemic Algorithms*

### Tasks
- [ ] Vector clocks, contexts, sibling storage and merging. Tombstones for deletes.
- [ ] Gossip membership: heartbeat counters, `ALIVE/SUSPECT/DEAD`, incarnations, seeds.
- [ ] Sloppy quorum: fallback nodes from the extended preference list.
- [ ] Hinted handoff: durable hints, delivery on return, cleanup.
- [ ] Demo app: shopping-cart client that merges siblings (union of items).
- [ ] Tests: concurrent writes produce siblings, never a silent overwrite. Hints are delivered after a node returns.

### M2 deliverable
- Demo on 2 computers: `docker kill` a zone-B node and keep writing. Writes succeed and hints accumulate (shown as a counter). Restart the node: the hints drain and the node has every write.
- Partition demo: pull the cable and add different items to the same cart on both sides. Reconnect and read: two siblings. The client merges them, and the result contains every item.

### Pitfalls
- Using wall-clock time to order versions. That is LWW in disguise, and it loses writes.
- Treating `SUSPECT` as `DEAD`. A short Wi-Fi hiccup should not trigger a data reshuffle.
- Hints kept only in memory and lost when the fallback node restarts.

---

## Week 3 — Read repair, Merkle anti-entropy, re-replication, dashboard, checker

**Reading:** Textbook Ch. 7.3 (client-centric consistency), Ch. 8.6 (recovery) · Merkle (1987) · Dynamo §4.7 · extension papers ([08-reading-list.md](08-reading-list.md))

### Tasks
- [ ] Read repair (asynchronous, after the merge).
- [ ] Merkle trees per replicated range. Periodic top-down comparison with peer replicas. Exchange only differing keys.
- [ ] Re-replication when a node becomes `DEAD`: the ring is recomputed and anti-entropy fills the replacement replicas.
- [ ] Rejoin: a node with old or empty data catches up. Handoff GC of temporary replicas after confirmation.
- [ ] Tombstone GC plus a test that demonstrates the resurrection problem and your fix.
- [ ] `/health` and `/metrics`. **Dashboard** with the ring and the replication-health meter.
- [ ] **Workload generator** with history recording, and the **healing checker** ([05](05-testing-and-evaluation.md) §1).
- [ ] `scripts/chaos.sh` for seeded faults across both computers.
- [ ] Implement **one extension** end to end, with tests.

### M3 deliverable
- Demo: the workload runs while `chaos.sh` kills, partitions, and wipes nodes. Then a quiet period follows, the health meter returns to 100%, and the checker prints `PASS`.
- Extension demo.
- Draft evaluation plan (hypotheses, experiments, metrics).

### Pitfalls
- Merkle trees that compare *ranges with different boundaries* on two nodes. The trees must cover identical ranges.
- Deleting temporary replicas before the returning node is confirmed up to date, which loses data.
- Repair storms: every node repairing everything at once after a rejoin.

---

## Week 4 — Evaluation, hardening, "Kill & Heal"

### Tasks
- [ ] Run experiments X1–X7 on the **2-computer deployment**, with ≥ 5 repetitions each.
- [ ] Run the automated chaos suite ≥ 200 times. Fix and analyze every failure.
- [ ] *(Top grade)* a second extension, or polish of a ⭐ extension.
- [ ] Final report (max 8 pages).
- [ ] Clean up the repository: 2-computer reproduction steps, and scripts that regenerate every figure.
- [ ] Rehearse **on the classroom network**. Bring your own switch or hotspot.

### M4 — "Kill & Heal" (15 min per team)

| Minute | What |
|---|---|
| 0–3 | Architecture, zone-aware placement, the four healing mechanisms |
| 3–4 | Load 50k records. The health meter shows 100%. The workload starts. |
| 4–6 | **Kill a node.** Reads keep succeeding and the meter drops. Restart it: hints drain and the meter recovers. |
| 6–9 | **Pull the cable A ⟷ B.** Both sides keep writing to the same carts. Reconnect: siblings appear and merge, and the Merkle repair is visible. |
| 9–11 | **Phoenix:** wipe a node's volume and restart it. The node is rebuilt from its peers. (Or keep it dead past `T_dead` and watch re-replication.) |
| 11–12 | Checker output live: 0 lost writes, converged, N replicas restored, 0 resurrections |
| 12–15 | Key evaluation results plus extension highlight |

Then Q&A, in which every member answers.

---

## Suggested split for a team of 3

| Member | Primary | Secondary |
|---|---|---|
| A | Ring, placement, coordinator, quorums, vector clocks | Review of B |
| B | Membership/gossip, hinted handoff, Merkle anti-entropy, re-replication | Extension |
| C | Docker 2-computer deployment, chaos scripts, workload generator, checker, dashboard, experiments | Report figures |

Everyone must understand vector clocks, the quorum analysis, and how the four healing mechanisms interact. Any of these can be asked of anyone in the Q&A.
