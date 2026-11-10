# Course Project — **Phoenix**: A Self-Healing Distributed Record Store

**Course:** Distributed Systems (Master's level)
**Textbook:** M. van Steen & A.S. Tanenbaum, *Distributed Systems*, 4th ed. — <https://www.distributed-systems.net/>
**Duration:** 4 weeks · **Team size:** 2–3 students
**Deployment:** Docker, on **2 physical computers** connected over a LAN
**Recommended language:** Go (Rust, Java, Kotlin, or Python are allowed; see [05-testing-and-evaluation.md](05-testing-and-evaluation.md))

---

## The story

Nodes crash, disks die, cables get pulled, and laptops go to sleep. A *self-healing* system treats this as normal operation:

1. **Stay reachable.** Every record can be read and written while nodes are down, and even while the two computers are cut off from each other.
2. **Detect damage.** The system notices which replicas are missing, stale, or diverged.
3. **Repair automatically.** When a node returns, it catches up on what it missed. When a node never returns, the system creates new replicas elsewhere until every record again has its full replication factor. Nobody has to intervene.

Your team builds **Phoenix**, a replicated record store that does all three. It is named after the bird that rises from its ashes. In the final demo, you wipe a node's disk entirely and watch it rebuild itself from its peers.

## The final demo: "Kill & Heal"

The dashboard shows a **replication-health meter**: the percentage of records that have all N replicas up to date. It starts at 100% green. Then the instructor:

1. Kills a node. Records turn yellow (under-replicated) but **every read still succeeds**.
2. Pulls the cable between the two computers. **Both sides keep accepting writes**, and conflicting updates are created on purpose.
3. Reconnects the cable. Conflicts are detected and merged, Merkle trees find the differences, and the meter climbs back to 100%.
4. Deletes a node's Docker volume and restarts it. The node **rebuilds itself from its peers**.

An automated checker then confirms: *0 acknowledged writes lost · all replicas converged · replication factor restored · 0 deleted records resurrected*.

## Real-world systems that work like Phoenix

Every mechanism in this project is used in production today:

| System | Who uses it / for what | Self-healing mechanisms it shares with Phoenix |
|---|---|---|
| **Amazon Dynamo** (2007 paper) | Amazon's shopping cart, which had to stay writable even during data-center failures | The original design: consistent hashing, sloppy quorum, hinted handoff, vector clocks + siblings, Merkle anti-entropy, gossip membership |
| **Apache Cassandra** | Netflix (viewing history, user data), Apple, Instagram, Uber, and many others | Hinted handoff, read repair, Merkle-tree repair (`nodetool repair`), gossip + φ-accrual failure detector, tombstones with `gc_grace_seconds`, rack/DC-aware placement |
| **ScyllaDB** | Discord (trillions of chat messages, migrated from Cassandra) | Cassandra-compatible healing (hints, read repair, row-level repair) |
| **Riak KV** | The UK National Health Service's *Spine* (national patient-record infrastructure), gaming and betting platforms | Dynamo design: vector clocks/siblings, hinted handoff, active anti-entropy with Merkle trees, CRDT data types |
| **Project Voldemort** | LinkedIn (historically) | Dynamo-style quorums, hinted handoff, read repair |
| **Ceph** | CERN, OpenStack clouds, many universities and telecoms | Detects dead OSDs and *re-replicates* ("recovery/backfill"), scrubbing for bit-rot, failure-domain-aware placement (CRUSH) |
| **HDFS** | Hadoop clusters (Yahoo!, Meta historically, banks) | The NameNode detects missing block replicas and re-replicates them. Rack-aware placement. Block checksums. |
| **Azure Storage / Amazon S3** | Cloud object storage for millions of customers | Background integrity scanning, automatic re-replication or erasure-code repair across failure domains (zones) |

How Phoenix's goals map to these systems:
- *"Keep records reachable"*: Dynamo, Cassandra, and Riak take the AP choice so that shopping carts, chat, and patient lookups never refuse a request.
- *"Repair replicas when nodes return"*: hinted handoff, read repair, and Merkle anti-entropy (Cassandra/Riak).
- *"Restore redundancy when nodes don't return"*: re-replication (Ceph/HDFS).

Further reading: the Dynamo paper, the Cassandra docs on repair, and the Jepsen analyses of Riak and Cassandra (see [08-reading-list.md](08-reading-list.md)).

## Textbook coverage

| Chapter | Where it shows up in Phoenix |
|---|---|
| Ch. 1 Introduction | Failure transparency. The *fallacies of distributed computing* show up on a real LAN. |
| Ch. 2 Architectures | Decentralized peer-to-peer architecture with no master; structured overlay (ring) |
| Ch. 3 Processes | Containers, background repair workers, throttling repair against foreground traffic |
| Ch. 4 Communication | RPC between nodes on different hosts; **gossip/epidemic protocols** for membership and anti-entropy |
| Ch. 5 Coordination | **Vector clocks** for versioning and conflict detection; failure detection |
| Ch. 6 Naming | **Consistent hashing / DHT** with virtual nodes; zone-aware placement |
| Ch. 7 Consistency & Replication | Eventual consistency, quorums (N, R, W), client-centric consistency, replica placement and management |
| Ch. 8 Fault Tolerance | Failure detectors, crash-recovery, hinted handoff, re-replication, durability |
| Ch. 9 Security | (extension) mTLS between hosts, integrity checking of records |

## Documents

| File | Contents |
|---|---|
| [01-project-brief.md](01-project-brief.md) | Learning objectives, requirements, extensions, scope |
| [02-system-specification.md](02-system-specification.md) | Architecture, API, ring and placement, versioning, all self-healing mechanisms, guarantees |
| [03-deployment-docker.md](03-deployment-docker.md) | Running on 2 computers with Docker: topology, zone-aware placement, fault injection |
| [04-weekly-plan.md](04-weekly-plan.md) | Week-by-week milestones and the demo script |
| [05-testing-and-evaluation.md](05-testing-and-evaluation.md) | Healing checker, test suite, required experiments |
| [06-grading-rubric.md](06-grading-rubric.md) | Assessment criteria and sample Q&A questions |
| [07-report-template.md](07-report-template.md) | Structure of the final report (paper style, max 8 pages) |
| [08-reading-list.md](08-reading-list.md) | Papers and tools |

## Milestones

| When | Milestone |
|---|---|
| End of Week 1 | **M1** — Design doc, 2-computer Docker setup, ring plus replicated put/get |
| End of Week 2 | **M2** — Versioning (vector clocks), gossip failure detection, sloppy quorum plus hinted handoff |
| End of Week 3 | **M3** — Read repair, Merkle anti-entropy, re-replication, dashboard, checker, one extension |
| End of Week 4 | **M4** — "Kill & Heal" live demo on 2 computers, plus report and code |

## Academic integrity

You may read papers, the textbook, and documentation, and you may discuss ideas with other teams. You **may not** copy code from existing distributed stores (Cassandra, Riak, Dynomite, Voldemort, etc.), and you may not use them as your replication layer. You **may** use an embedded *local* storage engine on each node (e.g., bbolt, Pebble, SQLite, RocksDB). **Ring, replication, versioning, failure detection, and every repair mechanism must be your own code**, including the Merkle trees. Every member must be able to explain any part of the system. Disclose AI-assistant use in the report.
