# 🗳️ Consensus Algorithms — Complete Cheatsheet

---

## 📌 What is a Consensus Algorithm?

In a distributed system, multiple nodes (servers/processes) need to **agree on a single value or a sequence of values** — even when some nodes crash, messages are delayed, or the network is unreliable.

A **consensus algorithm** is the protocol that allows this agreement to happen **safely and reliably**.

### Why is it hard?
- Nodes can crash at any time
- Messages can be delayed, duplicated, or dropped
- Network can partition (nodes can't talk to each other temporarily)
- You can't tell the difference between a slow node and a dead node

### The CAP Theorem (context)
A distributed system can only guarantee **2 of 3**:
- **C**onsistency — every read gets the latest write
- **A**vailability — every request gets a response
- **P**artition Tolerance — system works despite network partitions

Consensus algorithms typically choose **CP** (Consistency + Partition Tolerance).

---

## 🧩 Key Properties Every Consensus Algorithm Must Satisfy

| Property | Meaning |
|---|---|
| **Safety** | Only a value that was proposed can be chosen. Two nodes never decide on different values. |
| **Liveness** | Eventually *some* value will be decided (progress is made). |
| **Fault Tolerance** | The system works as long as a majority (quorum) of nodes are alive. |
| **Termination** | Every correct process eventually decides. |

> ⚠️ **FLP Impossibility (1985):** It's *theoretically impossible* to have a consensus algorithm that is both safe AND live in a purely asynchronous system with even one faulty node. Real algorithms solve this by adding timeouts/randomization.

---

## 📐 Core Concepts (Used Across All Algorithms)

### Quorum
A **majority** of nodes must agree. For `n` nodes, quorum = `⌊n/2⌋ + 1`.

- 3 nodes → quorum = 2
- 5 nodes → quorum = 3
- 7 nodes → quorum = 4

This means the system can tolerate `⌊n/2⌋` failures.

### Leader vs. Leaderless
- **Leader-based:** One node coordinates decisions (Raft, Multi-Paxos)
- **Leaderless:** Any node can propose (Basic Paxos, Zab)

### Log Replication
Most modern systems use consensus to replicate a **sequence of commands** (a log) across nodes, not just a single value.

---

## 1️⃣ PAXOS

Invented by **Leslie Lamport** (1989, published 1998). The foundational consensus algorithm. Notoriously difficult to understand.

### Roles
| Role | Description |
|---|---|
| **Proposer** | Proposes a value to be agreed upon |
| **Acceptor** | Votes on proposals. Must be a quorum. |
| **Learner** | Learns the final decided value |

> One node can play multiple roles simultaneously.

### Basic Paxos — Two Phase Protocol

#### Phase 1: Prepare / Promise
1. A **Proposer** picks a unique, monotonically increasing proposal number `n`.
2. Proposer sends `PREPARE(n)` to all Acceptors.
3. Each **Acceptor** responds with `PROMISE(n)` if:
   - It hasn't seen a higher proposal number before
   - It also sends back the **highest-numbered proposal it has already accepted** (if any)
4. Acceptor promises to **ignore any future proposals with a number < n**

#### Phase 2: Accept / Accepted
1. If the Proposer receives promises from a **quorum** of acceptors:
   - If any acceptor sent back a previously accepted value → Proposer **must** use that value (not its own)
   - Otherwise → Proposer can use its own value
2. Proposer sends `ACCEPT(n, value)` to quorum of acceptors
3. Each Acceptor **accepts** the proposal unless it has promised a higher number
4. If a quorum accepts → **consensus is reached** (value is "chosen")
5. Proposer notifies Learners of the decided value

```
Proposer          Acceptor 1    Acceptor 2    Acceptor 3
   |                  |             |             |
   |--- PREPARE(5) -->|             |             |
   |--- PREPARE(5) ---|------------>|             |
   |--- PREPARE(5) ---|-------------|------------>|
   |                  |             |             |
   |<-- PROMISE(5) ---|             |             |
   |<-- PROMISE(5) ---|-------------|             |
   |<-- PROMISE(5) ---|-------------|-------------|
   |                  |             |             |
   |--- ACCEPT(5,v) ->|             |             |
   |--- ACCEPT(5,v) --|------------>|             |
   |--- ACCEPT(5,v) --|-------------|------------>|
   |                  |             |             |
   |<-- ACCEPTED(5) --|             |             |
   |<-- ACCEPTED(5) --|-------------|             |
   |<-- ACCEPTED(5) --|-------------|-------------|
   |                  |             |             |
   ✅ Consensus on v
```

### Problems with Basic Paxos
- **Dueling Proposers (Livelock):** Two proposers keep interrupting each other with higher proposal numbers and neither reaches consensus. Solved by randomized backoff or electing a single leader.
- **Only decides one value:** For a replicated log you need to run Paxos repeatedly.
- **Complex to implement:** Many edge cases.

### Multi-Paxos
Extends Basic Paxos for a **replicated log** (sequence of values):
- Elect a **single stable leader** (a distinguished proposer)
- Leader skips Phase 1 for subsequent log entries (only does it once to establish leadership)
- Each log slot runs Phase 2 independently
- Used as the foundation for systems like **Google Chubby** and **Google Spanner**

---

## 2️⃣ RAFT

Designed by **Diego Ongaro and John Ousterhout** (2013). The explicit goal was to be **more understandable than Paxos**. Used in etcd, CockroachDB, TiKV, Consul.

### Key Design Principle
Raft decomposes consensus into 3 relatively independent sub-problems:
1. **Leader Election** — choosing one leader
2. **Log Replication** — leader accepts entries, replicates to followers
3. **Safety** — only up-to-date nodes can become leader

### Roles
| Role | Description |
|---|---|
| **Leader** | Exactly one at a time. Handles all client requests. Replicates log entries. |
| **Follower** | Passive. Responds to RPCs from leader and candidates. |
| **Candidate** | Temporary state when trying to become leader. |

### Terms
- Time is divided into **terms** (monotonically increasing integers).
- Each term starts with an election.
- Terms act as a **logical clock** — stale messages from old terms are rejected.

```
Term 1       Term 2      Term 3     Term 4
|---election--|--leader---|---split--|---leader---|
```

### Leader Election
1. Followers start with an **election timeout** (150–300ms randomly chosen).
2. If a follower doesn't hear from a leader → it **times out**, increments term, becomes **Candidate**.
3. Candidate votes for itself and sends `RequestVote` RPCs to all other nodes.
4. A node grants a vote if:
   - It hasn't voted in this term yet
   - The candidate's log is **at least as up-to-date** as its own
5. If candidate gets votes from **quorum → becomes Leader**
6. New leader immediately sends heartbeats (`AppendEntries` with no entries) to prevent new elections.

**Split vote:** Multiple candidates → nobody wins. Wait for timeout, try again with higher term. Randomized timeouts make this rare.

### Log Replication
1. Client sends command to Leader.
2. Leader appends entry to its local log (marked **uncommitted**).
3. Leader sends `AppendEntries` RPC to all followers.
4. Followers append to their log and respond with success.
5. Once a **quorum** acknowledges → leader **commits** the entry (applies to state machine).
6. Leader notifies followers of commit in next `AppendEntries`.
7. Followers apply committed entry to their state machines.

```
Leader Log:   [1:set x=1] [2:set y=2] [3:set z=3] ← committed up to index 2
Follower 1:   [1:set x=1] [2:set y=2]
Follower 2:   [1:set x=1] [2:set y=2] [3:set z=3]
Follower 3:   [1:set x=1]   ← lagging, will be caught up
```

### Log Consistency Guarantee
`AppendEntries` includes `prevLogIndex` and `prevLogTerm`. Follower rejects if it doesn't have a matching previous entry. Leader walks back and finds the conflict point, then overwrites follower log. **Follower logs always converge to match the leader's log.**

### Safety: Election Restriction
A candidate **cannot** win an election unless its log is at least as up-to-date as the majority. This ensures the new leader has all committed entries.

Log comparison:
- Higher term in last entry = more up-to-date
- Same last term, longer log = more up-to-date

### Leader Completeness Property
If a log entry is committed in a term, it will be present in the logs of all future leaders.

### Log Compaction (Snapshots)
Log grows forever → periodically take a **snapshot** of the current state machine and discard log up to that point. New nodes are bootstrapped with a snapshot.

### Raft Summary Flow

```
[Client] --write--> [Leader]
                       |
              AppendEntries (parallel)
               /        |        \
        [F1] ✅     [F2] ✅    [F3] ✅  ← quorum = commit
                       |
              notify commit in next heartbeat
               /        |        \
        [F1] apply  [F2] apply  [F3] apply
```

---

## 3️⃣ ZAB (ZooKeeper Atomic Broadcast)

Developed at **Yahoo** for **Apache ZooKeeper**. Very similar to Raft but invented independently (~2008). Powers ZooKeeper which is used by Kafka, Hadoop, HBase.

### Key Difference from Raft/Paxos
- ZAB is an **atomic broadcast** protocol, not just consensus
- Guarantees **total order** of all updates
- Primary-backup model

### Two Modes
1. **Recovery Mode (Leader Election):** When leader crashes or on startup. Elect a new leader and sync all nodes.
2. **Broadcast Mode (Normal operation):** Leader broadcasts `PROPOSAL`, followers ACK, leader sends `COMMIT`.

### ZAB vs Raft
| | Raft | ZAB |
|---|---|---|
| Origin | Stanford 2013 | Yahoo ~2008 |
| Used in | etcd, CockroachDB | ZooKeeper |
| Primary Use | Key-value store replication | Coordination service |
| Epoch/Term | Term | Epoch |
| Sync on leader change | Log matching | Synchronization phase |

---

## 4️⃣ VIEWSTAMPED REPLICATION (VSR)

Created by **Liskov and Cowling** (1988, predates Paxos!). Very similar to Paxos but described from a replication perspective rather than consensus.

- Uses **view numbers** (like terms in Raft)
- **View change** = leader election
- Basis for many practical systems
- Raft is essentially a rediscovery/refinement of VSR

---

## 5️⃣ PBFT (Practical Byzantine Fault Tolerance)

By **Castro and Liskov** (1999). Handles **Byzantine faults** — nodes that don't just crash but actively **lie, send wrong data, collude**.

### Byzantine vs Crash Failures
| | Crash Fault Tolerant (CFT) | Byzantine Fault Tolerant (BFT) |
|---|---|---|
| Node behavior | Crash and stop | Can send wrong/malicious data |
| Examples | Raft, Paxos, ZAB | PBFT, Tendermint, HotStuff |
| Nodes needed | 2f+1 for f failures | **3f+1** for f failures |
| Use case | Internal cluster | Untrusted environments (blockchain) |

### PBFT Phases
1. **Pre-prepare:** Primary (leader) assigns sequence number to request
2. **Prepare:** Nodes broadcast that they received the pre-prepare
3. **Commit:** After receiving 2f+1 prepares, nodes broadcast commit
4. **Reply:** After receiving 2f+1 commits, execute and reply to client

### PBFT Limitation
O(n²) message complexity — doesn't scale beyond ~20-30 nodes.

---

## 6️⃣ TENDERMINT / HOTSTUFF (Modern BFT)

Used in **blockchain systems** (Cosmos, Ethereum's Casper).

### HotStuff (2018 — used in Facebook's Diem/Libra)
- Reduces PBFT's O(n²) to **O(n)** message complexity using a **leader + threshold signatures**
- Three-phase commit: Prepare → Pre-commit → Commit → Decide
- **Linear communication** — nodes only communicate with leader, not each other

### Tendermint
- Round-based BFT consensus
- Used in Cosmos blockchain
- Steps: Propose → Prevote → Precommit → Commit
- Requires 2/3+ of validators to agree

---

## 7️⃣ EPAXOS (Egalitarian Paxos)

By **Moraru et al., CMU (2013)**. Leaderless variant of Paxos.

- **No single leader** — any replica can commit commands directly
- Commands that don't conflict with each other are committed in **parallel**
- Achieves better **geo-distributed latency** (no single bottleneck leader)
- Used in experimental systems; more complex to implement

---

## 8️⃣ RAFT vs PAXOS — Direct Comparison

| Aspect | Paxos (Multi-Paxos) | Raft |
|---|---|---|
| **Understandability** | Very complex | Designed to be simple |
| **Leader** | Implicit | Explicit, strong leader |
| **Log holes** | Can have gaps | No gaps — strictly sequential |
| **Membership change** | Not specified | Joint consensus / single-server changes |
| **Practical use** | Google Chubby, Spanner | etcd, CockroachDB, Consul, TiKV |
| **Formally proven** | Yes | Yes |
| **Performance** | Similar | Similar |
| **Cluster config changes** | Ad-hoc | Built-in |

---

## 🔧 Real-World Usage

| System | Algorithm | Notes |
|---|---|---|
| **etcd** | Raft | Kubernetes uses etcd for all cluster state |
| **CockroachDB** | Raft | One Raft group per range of data |
| **TiKV** (TiDB) | Raft | Multi-Raft for sharding |
| **Consul** | Raft | Service discovery |
| **Apache ZooKeeper** | ZAB | Kafka uses ZooKeeper (or KRaft now) |
| **Apache Kafka (KRaft)** | Raft | Replacing ZooKeeper dependency |
| **Google Chubby** | Multi-Paxos | Lock service |
| **Google Spanner** | Multi-Paxos | Global distributed database |
| **MongoDB** | Raft-like | Replica set election protocol |
| **Cosmos** | Tendermint | BFT blockchain |

---

## 🧠 Interview Questions & Answers

---

### Q1: What is consensus in distributed systems and why do we need it?

**Answer:**
Consensus is the process by which multiple nodes in a distributed system agree on a single value or sequence of values, even in the presence of failures. We need it whenever multiple nodes maintain replicated state that must stay consistent — for example, a distributed database where all replicas must have the same data, or a distributed lock service where only one client can hold a lock at a time. Without consensus, nodes could diverge and you'd get split-brain scenarios where different parts of the cluster believe different things are true.

---

### Q2: Explain the Raft election process. What happens if two nodes both time out at the same time?

**Answer:**
Raft uses randomized election timeouts (typically 150–300ms) to reduce the chance of split votes. When a follower's timeout fires, it increments its term, transitions to Candidate, votes for itself, and sends `RequestVote` RPCs to all other nodes.

If two nodes time out at nearly the same time, both become candidates and neither may get a majority (split vote). In this case, both wait out a new randomized timeout and try again with a higher term. Because the timeouts are randomized, one will usually fire first and win the next election. This is eventually liveness — it will resolve, just not instantly.

---

### Q3: How does Raft ensure that a newly elected leader has all committed entries?

**Answer:**
Through the **Election Restriction**. A candidate can only win if its log is at least as up-to-date as the majority. Log freshness is compared by:
1. The **term** of the last log entry — higher term wins
2. If terms are equal, **longer log** wins

Since an entry is only committed when a majority of nodes have it, and a new leader must get votes from a majority, there must be at least one overlap node that has all committed entries. That node will only vote for a candidate whose log is as current as its own, guaranteeing the new leader already has all committed entries.

---

### Q4: What is the difference between Paxos and Raft?

**Answer:**
Both are crash fault-tolerant consensus algorithms that use a single leader and quorum-based voting. The key differences are:

Paxos is more flexible but underspecified — it doesn't tell you how to handle leader election, log compaction, or membership changes. Implementers must figure these out, which leads to many different and complex implementations. Raft was explicitly designed for understandability and provides complete specification for all these aspects. Raft also maintains a strict sequential log with no gaps, while Multi-Paxos can have holes in the log that need to be filled. In practice, their performance is similar.

---

### Q5: What is a quorum and why does it matter?

**Answer:**
A quorum is the minimum number of nodes that must agree for an operation to succeed, typically `⌊n/2⌋ + 1` (a majority). It matters because it prevents split-brain: since any two quorums must overlap by at least one node, you can never have two different values simultaneously accepted. If you have 5 nodes and quorum is 3, you can't have two different groups of 3 nodes agree on different values — they'd have to share at least one node, and that node can't have voted for two different things.

Quorum also defines fault tolerance: with 5 nodes and quorum 3, you can lose 2 nodes and still make progress.

---

### Q6: What's the difference between crash fault tolerance and Byzantine fault tolerance?

**Answer:**
In **crash fault tolerance (CFT)** — used by Raft, Paxos, ZAB — nodes either work correctly or stop working entirely (crash). You can't have a node that sends wrong or malicious data.

In **Byzantine fault tolerance (BFT)** — used by PBFT, Tendermint, HotStuff — nodes can behave arbitrarily: they can send incorrect data, lie about their state, selectively respond, or even collude. This is the threat model for blockchains and adversarial environments.

The cost is higher: CFT needs `2f+1` nodes to tolerate `f` failures. BFT needs `3f+1` nodes because you need to be able to tell the difference between a crashed node and one actively lying to you.

---

### Q7: What happens in Raft when there is a network partition?

**Answer:**
Suppose you have 5 nodes and the network splits into a group of 2 and a group of 3.

The **minority partition (2 nodes)** cannot elect a leader (needs 3 votes) or commit any new entries. If the old leader is in this partition, it can still accept client requests, but it can't commit them. It will eventually realize it's no longer the leader when the partition heals and it sees a higher term.

The **majority partition (3 nodes)** will elect a new leader (can get 3 votes = quorum) and continue making progress.

When the partition heals, the old leader/minority nodes see the higher term from the majority partition and step down. Their logs are overwritten to match the new leader. This ensures consistency — only entries committed by the majority are preserved.

---

### Q8: What is the FLP Impossibility result and how do real systems work around it?

**Answer:**
The **Fischer, Lynch, and Paterson (1985)** result proves that in a purely asynchronous distributed system (no bounds on message delay), no consensus algorithm can guarantee both **safety and liveness** if even one process can fail.

Real systems work around it by:
1. **Using timeouts** (partially synchronous model) — assuming messages eventually arrive within some unknown bound. Raft's election timeouts are an example.
2. **Randomization** — randomized timeouts break symmetry and ensure eventual progress with high probability.
3. **Making weak assumptions about timing** — accepting that the system is "usually" synchronous and handling rare cases of delay gracefully.

Essentially, real systems accept that they might stall temporarily during truly asynchronous periods but will make progress once the network stabilizes.

---

### Q9: Explain log compaction / snapshotting in Raft. Why is it needed?

**Answer:**
Without compaction, the Raft log grows forever. Snapshotting solves this: each node periodically takes a snapshot of its **entire current state machine state** (e.g., all key-value pairs) and writes it to stable storage along with the log index and term it reflects. It then discards all log entries up to that index.

When a new node joins or a lagging follower needs log entries that have been discarded, the leader sends it an `InstallSnapshot` RPC with the full snapshot. The follower replaces its log and state machine with the snapshot.

This keeps disk usage bounded and allows new nodes to catch up quickly without replaying the entire history of log entries.

---

### Q10: How does Raft handle membership changes (adding/removing nodes)?

**Answer:**
Naively switching from one configuration to another is dangerous — you could briefly have two leaders if nodes disagree on the current membership. Raft handles this with two approaches:

**Joint Consensus (original paper):** Transition through a joint configuration `C_old,new` where decisions require majorities from both old AND new configurations. No unsafe state is possible.

**Single-Server Changes (simpler, preferred in practice):** Only add or remove one node at a time. It can be proven that this never creates two simultaneous majorities. This is what most implementations (like etcd) use.

---

### Q11: What is leader lease and how does it improve read performance?

**Answer:**
In standard Raft, even reads must go through the leader and be committed to the log to ensure linearizability (you might be reading from a stale leader that doesn't know it's been replaced).

A **leader lease** is a time-based optimization: once a leader gets heartbeat acknowledgments from a quorum, it assumes it's the only valid leader for some bounded time period (the lease duration). During this window it can serve reads locally without any log round-trip, since no new leader can be elected during that window (elections require the lease to expire first).

This assumes bounded clock skew and is used in practice by systems like CockroachDB to reduce read latency.

---

### Q12: What is Viewstamped Replication and how does it relate to Raft?

**Answer:**
Viewstamped Replication (VSR) was developed by Liskov in 1988, predating Paxos. It's a replication protocol that uses "views" (equivalent to Raft's terms) and has a primary (leader) that replicates operations to backups (followers). A "view change" is triggered when the primary fails, equivalent to a leader election in Raft.

Raft independently rediscovered many of the same ideas as VSR. They are functionally very similar. The main contribution of Raft was a clear, complete specification and the explicit focus on understandability, whereas VSR's original paper was more theoretical. Ongaro acknowledged the similarity when Raft was published.

---

## 🔑 Quick Reference Cheat Card

```
Paxos:
  Phase 1 → Prepare/Promise (leader election for a slot)
  Phase 2 → Accept/Accepted (value agreement)
  Problem: Underspecified, hard to implement

Raft:
  Election: randomized timeouts → RequestVote → quorum → leader
  Replication: AppendEntries → quorum ACK → commit → apply
  Safety: only fresh logs can win election
  Key: no log gaps, strong leader, complete spec

ZAB:
  Like Raft but for ZooKeeper
  Broadcast mode + Recovery mode

BFT Family:
  PBFT:       3f+1 nodes, O(n²) messages
  HotStuff:   3f+1 nodes, O(n) messages (linear)
  Tendermint: 3f+1 nodes, blockchain use

Fault Tolerance:
  n nodes → tolerate ⌊n/2⌋ CFT failures (need 2f+1 nodes)
  n nodes → tolerate ⌊n/3⌋ BFT failures (need 3f+1 nodes)
```

---

That's everything you need to understand consensus algorithms deeply — from the theory all the way to how real systems like etcd and ZooKeeper implement them. Let me know if you want me to dive deeper into any specific algorithm or concept!