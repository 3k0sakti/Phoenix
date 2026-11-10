# 06 — Grading Rubric

**Total: 100 points.** The project grade is a team grade. An individual adjustment of up to ±15% can be applied based on the Q&A and `CONTRIBUTIONS.md`.

## 1. Breakdown

| Component | Points |
|---|---|
| Milestones M1–M3 (on time, complete) | 10 |
| Core: ring, placement, quorums, versioning, membership (F1–F5, F10–F11) | 15 |
| Core: self-healing mechanisms (F6–F9) | 15 |
| 2-computer Docker deployment, dashboard, tooling (F12–F14) | 10 |
| Extension(s) | 10 |
| Testing and correctness evidence | 15 |
| Experimental evaluation | 15 |
| Report | 5 |
| "Kill & Heal" demo and Q&A | 5 |

## 2. Criteria

### Core: data path (15)
| Level | Description |
|---|---|
| 14–15 | Correct zone-aware ring. Vector clocks with correct dominance and sibling handling. Gossip membership with hysteresis and incarnations. Tombstones. Clean code with no data races. |
| 10–13 | Works, with minor issues (e.g., vector-clock growth not handled, occasional false `DEAD`). |
| 6–9 | Basic replication works. Versioning is LWW-like or concurrent writes are lost. |
| < 6 | Not reliably replicated. |

### Core: self-healing (15)
| Level | Description |
|---|---|
| 14–15 | All four mechanisms work and interact correctly. Hints are durable. Merkle sync is proportional to the differences. Re-replication and a safe handoff GC. Resurrection is demonstrated and prevented. Phoenix rebuild from an empty disk works. |
| 10–13 | All mechanisms present, with one flawed or unsafe in edge cases. |
| 6–9 | Only some mechanisms, e.g., no Merkle trees or no re-replication. |
| < 6 | No real healing. |

### Deployment and tooling (10)
| Level | Description |
|---|---|
| 9–10 | Reproducible 2-computer setup from `.env` and documented commands. Correct advertise addresses. The dashboard shows the health meter and node states per zone. Seeded `chaos.sh` across hosts. |
| 6–8 | Runs on 2 computers, but the setup is manual or fragile, or the dashboard is minimal. |
| 3–5 | Only partly distributed. |
| 0–2 | Single machine only. |

### Extension(s) (10)
| Level | Description |
|---|---|
| 9–10 | Two extensions, or one ⭐ (E1 Raft config service, E4 erasure coding), fully tested and evaluated. |
| 6–8 | One extension, fully working and evaluated. |
| 3–5 | Partly working or not evaluated. |
| 0–2 | Missing. |

### Testing and correctness evidence (15)
| Level | Description |
|---|---|
| 13–15 | Full suite. Checker C1–C8. Chaos pass rate ≥ 99% over ≥ 200 runs. ≥ 2 mutation tests show that the checker catches real bugs. Reproducible seeds. |
| 9–12 | Most tests present. Fewer runs, or failures not analyzed. |
| 5–8 | Mostly happy-path tests. |
| 0–4 | Little automated testing. |

### Experimental evaluation (15)
| Level | Description |
|---|---|
| 13–15 | X1–X7 on the real 2-computer setup, with hypotheses, repetitions, CIs/percentiles, health-meter time series, and interpretation tied to theory (quorum overlap, gossip convergence O(log M), Merkle complexity, bandwidth limits). Reproducible. |
| 9–12 | All experiments done. Shallow interpretation or weak statistics. |
| 5–8 | Experiments missing, or numbers without analysis. |
| 0–4 | No meaningful evaluation. |

### Report (5)
Precise terminology from the textbook, justified design decisions with alternatives, an explicit CAP/PACELC discussion, honest limitations, and within the page limit.

### Demo and Q&A (5)
The kill, cable-pull, and wipe scenarios all heal live, with clear explanations. Each member answers technical questions.

## 3. Sample Q&A questions

1. Why must replica placement be zone-aware with 2 computers? What exactly happens to reads and writes when computer B loses power?
2. Your partition demo produced siblings. Why can't you just keep the version with the latest timestamp? Give a concrete lost-update scenario.
3. With N=3, R=2, W=2 and sloppy quorums, can a read miss an acknowledged write? Construct the case.
4. Why do you separate `SUSPECT` from `DEAD`? What would happen during a 3-second Wi-Fi hiccup if you didn't?
5. Hinted handoff, read repair, and anti-entropy all repair replicas. Why do you need all three? What would break if you removed each one?
6. Walk through a Merkle sync between two replicas that differ in 5 keys out of 1 million. How many hashes are exchanged?
7. A node returns with its old disk after being offline longer than the tombstone GC period. What can go wrong, and how does your system prevent it?
8. When is it safe to delete the temporary replicas created during re-replication? What happens if you delete them too early?
9. Compare Phoenix with a Raft-based store on the same 2 computers during a cable pull. Which do you prefer for a shopping cart, and which for a bank balance?
10. Swarm/Kubernetes restart failed containers. Why is that not enough to call a storage system "self-healing"?

## 4. Penalties

| Issue | Penalty |
|---|---|
| Late submission | −10% per day (max 3 days) |
| Report over the page limit | −2 points |
| Results not reproducible from the repo | −5 points |
| Using an existing distributed store (Cassandra, Riak, …) as the replication layer | Core sections scored 0 |
| Copied implementation | Grade 0, and referral under academic integrity rules |
