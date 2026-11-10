# 03 — Deployment: Docker on 2 Computers

The final system **must** run on two physical computers. Each computer is one **zone** (failure domain). This document covers hardware, placement, networking, fault injection, and common pitfalls.

---

## 1. Hardware and network

| Item | Recommendation |
|---|---|
| Computers | 2 laptops/PCs. Linux is easiest. macOS/Windows with Docker Desktop work with Option A below. |
| Network | **Wired Ethernet through a small switch/router** (best), or a dedicated hotspot |
| ⚠ Campus Wi-Fi | Often has *client isolation*, which blocks laptop-to-laptop traffic. Test this in Week 1. |
| IP addresses | Static or DHCP-reserved, set in `.env` (`HOST_A=…`, `HOST_B=…`) |
| Disk | SSD recommended. Re-replication is I/O-heavy. |

Measure the network first, using `ping` (RTT mean/p99) and `iperf3` (bandwidth). These numbers drive your timeout settings, and you need them to interpret repair times. For example, the minimum re-replication time is roughly data size divided by bandwidth.

---

## 2. Placement: nodes, zones, and what survives what

Default: **6 nodes, 3 per computer, N = 3, zone-aware placement** (each key has replicas on both computers, 2+1 or 1+2).

| Event | Reads | Writes | Healing afterwards |
|---|---|---|---|
| 1 node crashes | ✅ (R=2 from remaining replicas) | ✅ (fallback + hint if needed) | Hinted handoff and anti-entropy when it returns, or re-replication after `T_dead` |
| Computer B loses power | ✅ every key still has ≥ 1 replica on A | ✅ sloppy quorum writes to fallbacks on A | When B returns: hints + Merkle repair. If B stays dead, keys can only be re-replicated *inside A*, so zone diversity is lost (P6 violated) until B is back. Detect and report this. |
| **Cable pulled A ⟷ B** | ✅ on both sides | ✅ on both sides (sloppy) | On reconnect: concurrent writes become **siblings**, and anti-entropy converges both sides |
| Disk (volume) of a node wiped | ✅ | ✅ | The node rejoins empty and is rebuilt by anti-entropy ("Phoenix") |

**Compare with what you know from consensus systems:** a Raft cluster on 2 computers *stops* on the minority side during a partition. Phoenix keeps both sides available and pays with temporary inconsistency (siblings) that must be repaired. This is the AP choice in CAP. You must discuss it explicitly in the report.

Things to analyze in the report:
- Why is zone-aware placement essential with 2 computers? Measure how many keys would be unreachable after losing a computer **without** the zone constraint.
- With `R = 2` and only one replica on the surviving side, what does a read return? It may get one real response and one empty response from a fallback. Is the response correct? Is it complete?
- What does "restore N replicas" mean when only one zone is left?

---

## 3. Networking options

Pick **one** and document it.

### Option A — Docker Compose per computer, published ports (recommended, works everywhere)

`deploy/compose.zone-a.yml` (sketch):

```yaml
x-node: &node
  image: phoenix/node:latest
  cap_add: ["NET_ADMIN"]      # tc / iptables fault injection inside the container
  restart: "no"               # crashes must stay crashed during experiments
  environment:
    SEEDS: "${HOST_A}:7001,${HOST_B}:7004"
    N: "3"
    R: "2"
    W: "2"

services:
  n1:
    <<: *node
    command: ["--id=n1", "--zone=A", "--listen=0.0.0.0:7001", "--advertise=${HOST_A}:7001", "--data=/data"]
    ports: ["7001:7001"]
    volumes: ["n1-data:/data"]
  n2: { <<: *node, command: [...port 7002...], ports: ["7002:7002"], volumes: ["n2-data:/data"] }
  n3: { <<: *node, command: [...port 7003...], ports: ["7003:7003"], volumes: ["n3-data:/data"] }
  dashboard:
    image: phoenix/dashboard:latest
    ports: ["8080:8080"]
    environment: ["SEEDS=${HOST_A}:7001,${HOST_B}:7004"]

volumes: { n1-data: {}, n2-data: {}, n3-data: {} }
```

`deploy/compose.zone-b.yml` defines `n4`–`n6` (`--zone=B`), `workload-gen`, and `healing-checker`.

```bash
# computer A
docker compose --env-file .env -f deploy/compose.zone-a.yml up -d
# computer B
docker compose --env-file .env -f deploy/compose.zone-b.yml up -d
```

⚠ **Advertise address:** a node must gossip the address that is reachable *from the other computer* (host IP plus published port), never its container IP (`172.x.x.x`).

### Option B — Docker Swarm with an overlay network (Linux recommended)

```bash
docker swarm init --advertise-addr $HOST_A             # on A
docker swarm join --token <token> $HOST_A:2377          # on B
docker node update --label-add zone=A <node-A>          # on A (manager)
docker node update --label-add zone=B <node-B>
docker stack deploy -c deploy/stack.yml phoenix
```

- Put services on a computer with `deploy.placement.constraints: [node.labels.zone == A]`.
- Open ports `2377/tcp`, `7946/tcp+udp`, `4789/udp`.
- Set `restart_policy: condition: none`. Swarm's own "self-healing" (restarting containers) must not hide *your* self-healing during experiments. Discuss in the report how container-level healing (Swarm/Kubernetes) and data-level healing (Phoenix) differ and complement each other.

---

## 4. Fault injection on real hardware

The automated suite uses an in-process simulated network ([05](05-testing-and-evaluation.md)). On the real deployment:

| Fault | How |
|---|---|
| Crash a node | `docker kill phoenix-n2-1` |
| Return a node (old disk) | `docker start phoenix-n2-1` |
| **Return a node with an empty disk** | `docker rm -f phoenix-n2-1 && docker volume rm phoenix_n2-data && docker compose ... up -d n2` |
| Crash a whole computer | `docker compose -f deploy/compose.zone-b.yml kill` |
| **Partition A ⟷ B** | Pull the cable / disable Wi-Fi on B, or `iptables -A INPUT -s $HOST_A -j DROP` on B |
| Isolate one node | Inside the container: `iptables -A INPUT -p tcp --dport 7002 -j DROP` |
| Slow / lossy network | Inside a container: `tc qdisc add dev eth0 root netem delay 80ms 30ms loss 3%` |
| Flapping node | Script: kill/start every 5 s, which tests `SUSPECT` vs. `DEAD` hysteresis |
| Corrupt a replica (E6) | Flip bytes in a file in the node's volume while it is stopped |

Provide `scripts/chaos.sh --seed S --duration D`, which runs a random fault schedule across both computers (via `ssh` or a small agent per host) and logs every action with a timestamp. The plots of experiments X2–X5 use this log.

---

## 5. Measuring across two machines

- Run the workload generator and the history recorder on **one** machine (default: B), so that all timestamps come from one monotonic clock.
- If you run generators on both computers, check durability per generator, or widen the time intervals by the measured clock offset ε. Explain why.
- During an A⟷B partition, a generator on B can only reach zone-B nodes, and that is the point. Plot the success rates per side.

---

## 6. Week-1 deployment checklist

- [ ] Docker runs on both computers. The image builds on both, or is moved with `docker save`/`load` or a registry.
- [ ] `ping` and `iperf3` between the hosts work. RTT and bandwidth are recorded.
- [ ] A container on A can reach a container on B through the chosen option.
- [ ] All addresses and parameters come from `.env`, with no hardcoded IPs.
- [ ] `docker kill` + `docker start` keeps the data. Deleting the volume really empties the node.
- [ ] `deploy/README.md` lets another team reproduce the setup.

## 7. Common problems

| Symptom | Likely cause |
|---|---|
| Nodes on A and B never see each other | Wi-Fi client isolation, host firewall, wrong advertise address |
| Nodes constantly flip to `SUSPECT` | `T_suspect` too small for Wi-Fi jitter, or gossip threads starved under load |
| Huge repair traffic after a short blip | A `SUSPECT` node treated as `DEAD` (ring changed too early), or no hysteresis |
| Deleted records come back | Tombstones GC'd before all replicas had them, or a stale node rejoined after `T_gc` |
| Re-replication saturates the LAN, and client p99 explodes | No repair throttling (see E8) |
| Data gone after restart | `docker compose down -v`, or the volume is not mounted at `/data` |
| Killed node comes back by itself | `restart: always`, or a Swarm restart policy |
