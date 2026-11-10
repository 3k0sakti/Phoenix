# 07 — Final Report Template

**Format:** PDF, IEEE or ACM two-column template, **max 8 pages** excluding references and appendix.
**File name:** `Phoenix-report-<team-name>.pdf`

---

## Title
*Phoenix: [subtitle, e.g., "Self-Healing Replicated Storage Across Two Failure Domains"]*
Authors, student IDs, emails.

## Abstract (≤ 200 words)
Problem, approach, key results with numbers (availability during faults, time-to-heal, rebuild time, 0 lost writes in N chaos runs), and the main conclusion.

## 1. Introduction (~0.5 page)
- Why storage systems must heal themselves.
- What you built, and your contributions as bullets.
- The main findings.

## 2. Background and system model (~0.75 page)
- Failure model: crash-recovery, disk loss, partitions, asynchronous network (Ch. 8).
- Consistency: eventual consistency, quorums, client-centric models (Ch. 7). CAP/PACELC, and the position Phoenix takes.
- Vector clocks and consistent hashing in brief (Ch. 5, Ch. 6).

## 3. Design (~1.5 pages)
- Architecture, and zone-aware placement over the 2 computers: **what survives what**.
- Quorum operations, sloppy quorum, versioning, sibling merge.
- Membership and failure detection: parameters and hysteresis.
- **The four healing mechanisms**: when each triggers, what it fixes, and how they interact. Add a figure.
- Deletes, tombstones, and the resurrection problem.
- Deployment: networking option, advertise addresses, volumes.
- **Design decisions table:** decision | alternatives | rationale.

## 4. Extension(s) (~1 page)

## 5. Correctness (~1 page)
- Guarantees P1–P7: how each is ensured and checked (C1–C8).
- Fault-injection methodology (simulated network and real 2-host chaos).
- Chaos results: number of runs, pass rate, analysis of every failure (bug "war stories" are welcome).
- Mutation tests: the bugs you introduced and how the checker caught them.

## 6. Evaluation (~2 pages)
- Setup: both computers' hardware and OS, Docker version, network type, **measured RTT and bandwidth**, parameters, repetitions.
- X1–X7: **hypothesis → figure → interpretation**.
- Must include: a **health-meter time series** through a kill → partition → wipe scenario, with the fault events marked.

## 7. Discussion and limitations (~0.5 page)
- What two failure domains can and cannot guarantee (zone diversity when one computer is permanently gone).
- Threats to validity: two hosts, LAN only, synthetic workload, small data sizes.
- Vector-clock growth, client merge burden, Byzantine faults, and what you would do differently.

## 8. Related work (~0.25 page)
Dynamo, Cassandra, Riak (hinted handoff, read repair, Merkle trees); Ceph/HDFS (re-replication); CRDT-based systems; and consensus-based stores (etcd, Spanner) as the CP contrast.

## 9. Conclusion (~0.25 page)

## References

## Appendix (not counted toward the page limit)
- A. Reproduction: build, `make test`, 2-computer deployment commands, `chaos.sh`, figure scripts.
- B. **AI-assistance disclosure:** tools, what for, and how you checked the output.
- C. Individual contributions.
