# 08 — Reading List

## Required

1. **M. van Steen & A.S. Tanenbaum.** *Distributed Systems*, 4th ed., 2023. <https://www.distributed-systems.net/>
   Ch. 2 Architectures (decentralized) · Ch. 4 Communication (gossip-based / epidemic protocols) · Ch. 5 Coordination (vector clocks) · Ch. 6 Naming (DHTs) · Ch. 7 Consistency and Replication · Ch. 8 Fault Tolerance
2. **G. DeCandia et al.** *Dynamo: Amazon's Highly Available Key-Value Store.* SOSP 2007. **Your main design reference.**
3. **A. Demers et al.** *Epidemic Algorithms for Replicated Database Maintenance.* PODC 1987. The origin of anti-entropy and rumor mongering.

## Strongly recommended

4. **R. Merkle.** *A Digital Signature Based on a Conventional Encryption Function.* CRYPTO 1987. Merkle trees.
5. **D. Karger et al.** *Consistent Hashing and Random Trees.* STOC 1997.
6. **C. Fidge (1988) / F. Mattern (1989).** The original vector-clock papers. Also: Lamport, *Time, Clocks, and the Ordering of Events*, CACM 1978.
7. **A. Lakshman & P. Malik.** *Cassandra: A Decentralized Structured Storage System.* LADIS 2009.
8. **P. Bailis et al.** *Probabilistically Bounded Staleness for Practical Partial Quorums.* VLDB 2012. Basis for experiment X6.
9. **E. Brewer.** *CAP Twelve Years Later.* IEEE Computer 2012 · **D. Abadi.** *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC), IEEE Computer 2012.

## Extension-specific

| Extension | Reading |
|---|---|
| E1 Raft config service | Ongaro & Ousterhout, *In Search of an Understandable Consensus Algorithm*, USENIX ATC 2014 <https://raft.github.io/raft.pdf> |
| E2 CRDTs | Shapiro et al., *Conflict-Free Replicated Data Types*, SSS 2011 · Shapiro et al., *A Comprehensive Study of CRDTs*, INRIA TR 2011 |
| E3 Failure detection | Das, Gupta, Motivala, *SWIM*, DSN 2002 · Hayashibara et al., *The φ Accrual Failure Detector*, SRDS 2004 |
| E4 Erasure coding | Rashmi et al., *A Solution to the Network Challenges of Data Recovery in Erasure-coded Distributed Storage Systems* (Facebook warehouse study), HotStorage 2013 · Huang et al., *Erasure Coding in Windows Azure Storage*, USENIX ATC 2012 |
| E5 Session guarantees | Terry et al., *Session Guarantees for Weakly Consistent Replicated Data*, PDIS 1994 · Textbook Ch. 7.3 |
| E6 Scrubbing | Bairavasundaram et al., *An Analysis of Data Corruption in the Storage Stack*, FAST 2008 |
| E7 Rebalancing | Stoica et al., *Chord*, SIGCOMM 2001 · Weil et al., *Ceph* / *CRUSH*, OSDI 2006 / SC 2006 |
| E8 Throttling | Dynamo §6 (background tasks and admission control) |

## Tools

- **Docker Compose** / **Docker Swarm** docs: <https://docs.docker.com/>
- **tc netem** (latency/loss) · **iptables** (partitions) · **iperf3** (bandwidth) · **Toxiproxy** (TCP fault proxy)
- **Jepsen** analyses: <https://jepsen.io/analyses>. Read the Riak and Cassandra analyses to see real self-healing systems lose data.
- Embedded storage engines you may use locally: bbolt, Pebble, BadgerDB (Go) · RocksDB · SQLite
