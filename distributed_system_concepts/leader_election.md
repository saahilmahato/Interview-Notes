# 🗳️ Leader Election & Coordination — Complete Notes

---

## 🧠 What Is Leader Election?

In a distributed system, you often have **multiple nodes** (servers/processes) running the same service. But some operations must be done by **exactly one node at a time** — like writing to a database, processing a job queue, or managing cluster state.

**Leader Election** is the process by which nodes in a distributed system **agree on a single "leader"** who is responsible for coordinating work. All other nodes become **followers**.

> Think of it like an office — everyone can do work, but only one person is the "manager" who makes final decisions. If that manager quits, everyone votes for a new one.

---

## 🤔 Why Do We Need Coordination?

Distributed systems face these hard problems:

- **Split-brain**: Two nodes both think they're the leader → data corruption
- **Race conditions**: Multiple nodes try to do the same thing simultaneously
- **Node failures**: A leader dies — who takes over?
- **Network partitions**: Nodes can't talk to each other — who's the real leader?

Coordination services solve all of this.

---

## 📐 Core Concepts to Know First

### Consensus
The process where multiple nodes **agree on a single value** even if some nodes fail or messages are delayed.

### CAP Theorem (refresher)
- **Consistency** — all nodes see the same data
- **Availability** — every request gets a response
- **Partition Tolerance** — system works despite network failures

Coordination systems like ZooKeeper and etcd are **CP systems** — they choose Consistency + Partition Tolerance over Availability.

### Quorum
A **majority of nodes** (N/2 + 1) must agree before any decision is committed. If you have 5 nodes, at least 3 must agree. This prevents split-brain.

---

## 🦁 Apache ZooKeeper

### What is it?
ZooKeeper is a **centralized coordination service** built by Yahoo (donated to Apache). It provides distributed synchronization, configuration management, naming, and group services.

Think of ZooKeeper as a **distributed filesystem** where every "file" (called a **znode**) can have data and watchers attached to it.

### Architecture
```
Clients
   ↓
ZooKeeper Ensemble (cluster of ZK servers)
├── Leader (handles writes)
├── Follower 1 (handles reads, forwards writes to leader)
└── Follower 2 (handles reads, forwards writes to leader)
```

- An **ensemble** is a group of ZooKeeper servers (always odd: 3, 5, 7...)
- One server is elected **leader** internally using the **ZAB protocol** (ZooKeeper Atomic Broadcast)
- All **writes** go through the leader
- **Reads** can be served by any follower

### ZNodes — The Core Abstraction
ZooKeeper stores data in a **hierarchical tree** of znodes (like a filesystem):

```
/
├── /config
│   ├── /config/db-host  → "10.0.0.1"
│   └── /config/timeout  → "5000"
├── /election
│   ├── /election/candidate-001
│   └── /election/candidate-002
└── /locks
    └── /locks/resource-A
```

**Types of Znodes:**

| Type | Description |
|------|-------------|
| **Persistent** | Stays until explicitly deleted |
| **Ephemeral** | Auto-deleted when the client session ends (critical for leader election!) |
| **Persistent Sequential** | Persistent + auto-numbered suffix (e.g., `node-0001`) |
| **Ephemeral Sequential** | Ephemeral + auto-numbered suffix |

### Watches
ZooKeeper lets you **watch a znode** — you get notified when it changes or is deleted. This is how nodes detect when a leader dies.

> Watches are one-time triggers. After firing, you must re-register.

---

### 🏆 How Leader Election Works in ZooKeeper

**Algorithm:**

1. Every node that wants to be leader creates an **Ephemeral Sequential** znode under `/election/`
   - Node A → `/election/candidate-0000000001`
   - Node B → `/election/candidate-0000000002`
   - Node C → `/election/candidate-0000000003`

2. Each node **lists all children** of `/election/` and finds the one with the **lowest sequence number**

3. The node with the lowest number **declares itself leader**

4. All other nodes **watch the node just before them** in sequence (not the leader directly — this avoids herd effect)
   - Node B watches Node A
   - Node C watches Node B

5. If Node A (leader) **dies** → its ephemeral znode is deleted → Node B gets notified → Node B becomes the new leader

6. Node C now watches Node B (new leader)

**Why watch the node just before you (not the leader)?**
- If everyone watched the leader, when the leader dies, **all nodes fire simultaneously** → thundering herd problem
- Watching the previous node distributes the notification load

```
[A-0001] ← leader
[B-0002] watches [A-0001]
[C-0003] watches [B-0002]

A dies → B gets notified → B becomes leader
C now watches B
```

---

### ZAB Protocol (ZooKeeper Atomic Broadcast)
This is ZooKeeper's internal consensus protocol:

1. **Leader election phase**: Nodes vote for a leader using epoch numbers (like term numbers)
2. **Synchronization phase**: New leader syncs its state with followers
3. **Broadcast phase**: Leader broadcasts write transactions. Followers ACK. Once quorum ACKs, transaction is committed.

ZAB guarantees **total order** — all nodes see writes in the same order.

---

### ZooKeeper Use Cases
- **Leader election** (Kafka uses ZK for broker leadership, though moving away)
- **Distributed locks** (create an ephemeral znode = acquire lock)
- **Configuration management** — store shared config, watch for changes
- **Service discovery** — services register themselves as znodes
- **Distributed barriers** — all nodes wait at a point until a condition is met

---

## 🗝️ etcd

### What is it?
**etcd** is a distributed key-value store built by CoreOS (now part of CNCF). It's the backbone of **Kubernetes** — all cluster state, configs, and secrets are stored in etcd.

> etcd = "etc distributed" (inspired by Linux's `/etc` directory where config lives)

### Architecture
```
etcd cluster (always odd number of nodes: 3, 5, 7)
├── Leader (handles all writes)
├── Follower 1
└── Follower 2
```

- Uses **Raft consensus** (simpler and more understandable than ZAB/Paxos)
- **All writes go to the leader**, who replicates to followers
- Reads can be **linearizable** (from leader) or **serializable** (from follower, slightly stale)

---

### Raft Consensus Protocol
Raft was designed to be **easy to understand** compared to Paxos. It breaks consensus into 3 sub-problems:

**1. Leader Election**
- Time is divided into **terms** (monotonically increasing integers)
- Each term has at most one leader
- If a follower doesn't hear from the leader within the **election timeout** (e.g., 150-300ms), it starts an election
- It increments its term, votes for itself, and requests votes from others
- First node to get majority votes becomes leader
- Leader sends **heartbeats** to all followers to prevent new elections

```
Term 1: [Node A is leader] → A crashes
Term 2: [B starts election, gets majority votes, B is leader]
Term 3: [B crashes, C starts election...]
```

**2. Log Replication**
- Leader receives write → appends to its **log** → sends `AppendEntries` RPC to followers
- Once majority acknowledge → entry is **committed** → applied to state machine
- Committed entries are **never overwritten**

**3. Safety**
- Only a node with the **most up-to-date log** can win an election
- This prevents a stale node from becoming leader and losing data

---

### 🏆 How Leader Election Works in etcd

etcd exposes a high-level **Election API** built on top of its leases:

1. **Lease**: A time-to-live (TTL) object. If the holder doesn't renew it within TTL, it expires.

2. **Campaign**: Node creates a key `/election/leader` with a **lease attached**. Only one node can hold this key.

3. **Observe**: Other nodes watch `/election/leader`. When it expires (leader died), they race to `Campaign`.

```go
// Pseudocode
lease, _ := client.Grant(ctx, 10) // 10-second TTL
election := concurrency.NewElection(session, "/election/")
election.Campaign(ctx, "node-1") // blocks until we're leader

// In background: keep renewing lease (KeepAlive)
// If process dies → lease expires → another node wins
```

**Key difference from ZooKeeper**: etcd uses **leases with TTL** instead of **ephemeral znodes**. Both achieve the same goal — detecting node death.

---

### etcd vs ZooKeeper — Key Differences

| Feature | ZooKeeper | etcd |
|---|---|---|
| **Consensus Protocol** | ZAB | Raft |
| **API** | Znode tree (like filesystem) | Flat key-value |
| **Watch mechanism** | One-time watchers | Continuous watch streams |
| **TTL/Leases** | Session-based TTL | First-class Lease objects |
| **Language** | Java | Go |
| **Primary user** | Kafka, HBase, old Hadoop | Kubernetes, CoreDNS |
| **Complexity** | Higher operational overhead | Simpler to operate |
| **Read performance** | Higher (followers serve reads) | Good (linearizable reads from leader by default) |

---

## 🔩 Other Coordination Systems

### Consul (by HashiCorp)
- Built for **service mesh and service discovery**, but also does leader election and KV store
- Uses **Raft consensus** internally
- Has a built-in **health checking** system — nodes register services and Consul checks if they're alive
- Supports **sessions** (like leases in etcd) for leader election
- Has a nice **UI dashboard** and DNS interface

**Leader election in Consul:**
1. Node acquires a **session** with a TTL
2. Creates a key with `acquire` using that session
3. If session expires → key is released → another node can acquire it

---

### Chubby (Google — closed source)
- Designed by Google, described in the famous **"Chubby" paper by Mike Burrows (2006)**
- Inspired both ZooKeeper and etcd
- Uses **Paxos** for consensus
- Provides distributed locks and small file storage
- Designed for **coarse-grained locking** (hold locks for minutes/hours, not milliseconds)
- Not open source, but it influenced everything else

---

### Paxos (Algorithm, not a system)
The **foundational consensus algorithm** by Leslie Lamport. Most production systems use a variant of Paxos or something inspired by it.

**Key phases:**
1. **Prepare phase**: A proposer sends `Prepare(n)` to a majority of acceptors. If they haven't seen a higher proposal number, they promise not to accept lower ones.
2. **Accept phase**: Proposer sends `Accept(n, value)`. Acceptors accept if they haven't promised to ignore it.
3. **Learn phase**: Once majority accepts, the value is chosen.

> Paxos is notoriously hard to implement correctly. Raft was created specifically to be more understandable.

---

### Raft (Algorithm)
Already covered in etcd section. Systems using Raft: etcd, Consul, CockroachDB, TiKV, RethinkDB.

---

### SWIM Protocol (Gossip-based — different approach)
**Scalable Weakly-consistent Infection-style Membership** protocol.

Used when you need to track **which nodes are alive in a large cluster** without a central coordinator.

- Instead of a central ZooKeeper/etcd, every node **gossips** with a few random peers
- If node A suspects node B is dead, it asks other nodes to verify (indirect probing)
- Avoids single point of failure
- Used in: **Cassandra, Consul (for membership), HashiCorp Serf**

> SWIM is not for strong leader election — it's for **membership detection** in gossip-based systems.

---

## 🔐 Distributed Locks (related concept)

Leader election is essentially acquiring a **distributed lock** and holding it. The patterns are nearly identical.

**ZooKeeper Lock:**
1. Create ephemeral sequential znode under `/locks/resource`
2. If your znode has the lowest sequence number → you hold the lock
3. Otherwise, watch the node just before yours
4. When previous node deletes its znode → you acquire the lock
5. When done → delete your znode (or let session expire = auto-release)

**etcd Lock:**
1. Create a lease with TTL
2. Try to `txn` (transaction): "if key doesn't exist, create it with my lease"
3. If successful → you hold the lock
4. Continuously renew lease while holding it
5. Delete key when done (or let lease expire)

**Redlock (Redis-based distributed lock):**
- Acquire lock on majority of N Redis nodes (N=5 recommended)
- Lock is valid only if acquired on majority within a time window
- Controversial — Martin Kleppmann argued it has subtle safety issues under clock skew

---

## ⚡ Fencing Tokens — Critical Concept

Even with distributed locks, there's a dangerous scenario:

```
1. Node A acquires the lock (becomes leader)
2. Node A pauses (GC pause, network hiccup) for 2 minutes
3. Lock expires → Node B acquires the lock → Node B becomes leader
4. Node A wakes up — thinks it's STILL the leader → writes to DB → corruption!
```

**Solution: Fencing Tokens**
- Every time a lock/lease is acquired, the coordination service issues a **monotonically increasing token** (number)
- The resource being protected (e.g., database) **rejects any request with a token lower than what it has already seen**
- Node A's stale writes (with old token 33) are rejected because DB already saw token 34 from Node B

```
ZooKeeper → [A acquires, token=33] → A pauses → B acquires, token=34 → A wakes up
DB: "You sent token 33, I already saw 34 — REJECTED"
```

---

## 📊 Summary — When to Use What

| Need | Use |
|------|-----|
| Kubernetes cluster coordination | **etcd** (it's built in) |
| Kafka broker coordination | **ZooKeeper** (legacy) or **KRaft** (new Kafka-native Raft) |
| Service discovery + health checks | **Consul** |
| Simple key-value + leader election in Go | **etcd** |
| Legacy big data (HBase, Hadoop) | **ZooKeeper** |
| Large-scale gossip membership | **SWIM / Consul Serf** |
| You're at Google | **Chubby** 😄 |

---

## 🎤 Common Interview Questions & Answers

---

**Q1. What is leader election and why is it needed?**

**A:** Leader election is a process in distributed systems where multiple nodes collectively agree on exactly one node (the leader) to coordinate work. It's needed to avoid split-brain scenarios where multiple nodes perform conflicting operations simultaneously, to ensure consistency in writes, and to coordinate tasks that must be done by a single entity at a time. Examples include deciding which Kafka broker owns a partition, or which pod controls a shared resource in Kubernetes.

---

**Q2. How does ZooKeeper implement leader election?**

**A:** ZooKeeper uses **ephemeral sequential znodes**. Each candidate node creates a znode under a common path like `/election/candidate-` with a sequential suffix. The node with the **lowest sequence number** is the leader. Every other node watches the node just before it in sequence (not the leader directly, to avoid thundering herd). When the leader dies, its ephemeral znode is deleted (because the session ends), and the next node in line gets notified and becomes the new leader.

---

**Q3. What is the difference between ephemeral and persistent znodes?**

**A:** A **persistent znode** remains in ZooKeeper until explicitly deleted — it survives client disconnects and server restarts. An **ephemeral znode** is automatically deleted when the client session that created it ends (due to disconnect, crash, or timeout). Ephemeral znodes are critical for leader election and distributed locks because they ensure automatic cleanup when a node dies, preventing a dead leader from holding the lock forever.

---

**Q4. What is the Raft consensus algorithm?**

**A:** Raft is a consensus algorithm designed to be understandable. It elects a leader using **terms** (monotonically increasing integers). A node becomes a candidate if it doesn't hear from a leader within an election timeout. It requests votes from peers; if it gets a majority, it becomes leader. The leader handles all writes by appending to a log and replicating to followers. An entry is committed once the majority acknowledge it. Only nodes with the most up-to-date log can become leader, ensuring no data loss. etcd, Consul, and CockroachDB all use Raft.

---

**Q5. What is the difference between ZooKeeper and etcd?**

**A:** Both are CP distributed coordination stores, but they differ in several ways. ZooKeeper uses the **ZAB protocol** and exposes data as a **hierarchical znode tree** with one-time watchers, primarily used by Kafka and HBase. etcd uses **Raft** and exposes a **flat key-value API** with continuous watch streams, and is the storage backend for Kubernetes. etcd has first-class **Lease objects** for TTL, while ZooKeeper uses session-level TTLs. etcd is generally simpler to operate (written in Go, easier clustering), while ZooKeeper has higher read throughput since followers can serve reads.

---

**Q6. What is a split-brain problem and how do quorums prevent it?**

**A:** Split-brain occurs when a network partition divides a cluster into two groups, and both groups elect a leader — resulting in two leaders simultaneously making conflicting decisions. Quorums prevent this by requiring that any decision (including electing a leader) must be approved by **more than half the nodes (N/2 + 1)**. In a network partition, only one side can have a majority. The minority side cannot elect a leader and stops making progress. This ensures consistency at the cost of availability.

---

**Q7. What is a fencing token and why is it important?**

**A:** A fencing token is a monotonically increasing number issued each time a lock is granted. It protects against scenarios where a node holds a lock, pauses (due to GC or network issues), the lock expires, a new leader takes over, and then the old leader wakes up thinking it's still the leader. The resource being protected rejects any request with a token number lower than what it has already seen. So the old leader's stale write is rejected because the protected resource has already processed a higher-numbered token from the new leader.

---

**Q8. What happens in etcd if the leader crashes?**

**A:** When the leader crashes, it stops sending heartbeats. Followers have a configurable **election timeout** (typically 150-300ms). When a follower's timeout fires without receiving a heartbeat, it increments its term, transitions to candidate state, votes for itself, and sends `RequestVote` RPCs to other nodes. The first candidate to receive votes from a majority becomes the new leader for the new term. The new leader then sends heartbeats to all followers to establish its authority and prevent new elections.

---

**Q9. How does Consul differ from ZooKeeper and etcd?**

**A:** Consul is a more feature-rich service mesh tool compared to ZooKeeper and etcd which are focused coordination stores. Consul provides built-in **service discovery** with health checks (nodes register services and Consul actively checks their health), a **DNS interface** (you can resolve services as DNS names), a **KV store** for config, and support for **multiple datacenters**. ZooKeeper and etcd are lower-level building blocks — Consul is a higher-level solution that uses Raft internally like etcd. Choose Consul when you need service discovery + health checking in addition to coordination.

---

**Q10. What is the "thundering herd" problem in leader election, and how does ZooKeeper solve it?**

**A:** The thundering herd problem occurs when all watching nodes are notified simultaneously of the same event. In leader election, if all follower nodes watch the leader's znode, when the leader dies, all nodes receive the notification at the same time, flood the ZooKeeper ensemble with requests, and all try to become leader simultaneously — this creates a spike in load and slowdown. ZooKeeper's recipe solves this by having each node watch only the node with the **next lower sequence number** rather than the leader. Only one node (the next in line) is notified when the leader dies, creating an orderly chain of succession with no thundering herd.

---

**Q11. Can you explain how distributed locks work in etcd?**

**A:** In etcd, distributed locks are implemented using **leases** and **transactions**. A node first creates a lease with a TTL. Then it issues a transaction (compare-and-swap): "if key `/lock/resource` does not exist, create it with my lease ID." If the transaction succeeds, the node holds the lock. While holding the lock, it continuously renews the lease using `KeepAlive`. If the node crashes, the lease expires and the key is deleted, freeing the lock. Other nodes waiting for the lock watch the key and retry when it's deleted. This guarantees that the lock is automatically released even if the holder crashes.

---

**Q12. What is the CAP theorem implication for ZooKeeper and etcd?**

**A:** Both ZooKeeper and etcd are **CP systems** — they prioritize Consistency and Partition Tolerance over Availability. During a network partition, the minority partition stops accepting reads/writes rather than risk returning stale or inconsistent data. This is the correct choice for coordination services because the whole point is to have a single source of truth — a stale or split answer would defeat the purpose. The tradeoff is that if you lose quorum (majority of nodes unavailable), the system becomes unavailable entirely. This is acceptable because coordination services are usually deployed on a small, stable set of servers, making quorum loss rare.

---

**Q13. What is KRaft (Kafka's new approach) and why is Kafka moving away from ZooKeeper?**

**A:** **KRaft** (Kafka Raft Metadata) is Kafka's own built-in consensus protocol that replaces ZooKeeper. Kafka used ZooKeeper to store cluster metadata (broker registrations, topic configurations, partition leadership). The problems with this approach were: ZooKeeper is a separate cluster to manage (operational overhead), there's a scaling bottleneck because all metadata goes through ZooKeeper, and startup/failover is slower. With KRaft, Kafka manages its own metadata using Raft — one of the Kafka brokers acts as the metadata leader. This simplifies operations (one less system to manage), improves scalability (supports millions of partitions), and speeds up recovery.

---

## 🧩 Quick Cheat Sheet

```
ZooKeeper → ZAB protocol → Znode tree → Ephemeral+Sequential znodes → Leader election
etcd       → Raft         → Key-Value  → Leases (TTL)               → Kubernetes backbone
Consul     → Raft         → KV+Service → Sessions (TTL)             → Service discovery
Chubby     → Paxos        → Lock files → Session TTL                → Google internal

Raft Terms:    term, leader, follower, candidate, heartbeat, election timeout
ZAB  Phases:   leader election → sync → broadcast
Quorum:        majority = N/2 + 1
Fencing token: monotonically increasing, guards against stale leaders
Thundering herd solution: watch predecessor, not leader
Ephemeral znode: auto-deleted on session end = dead node detection
```

---

> 💡 **Golden rule**: Coordination systems are **CP** — they will sacrifice availability over correctness. When your leader election system goes down, it's safer to refuse requests than to give wrong answers.