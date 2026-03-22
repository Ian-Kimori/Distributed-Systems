# Distributed Systems
### COMP3XXX · Bachelor of Computer Science · Year 2–3

> The theory and practice of building correct, efficient, and fault-tolerant systems across multiple networked computers — the intellectual backbone of all modern infrastructure.

**Prerequisites:** Networks, Operating Systems, Algorithms  
**Level:** 2nd–3rd Year Undergraduate  
**Leads to:** Service Oriented Computing, Cloud Computing, Security

---

## Table of Contents

**Foundations**
- [Week 1 — Introduction & System Models](#week-1--introduction--system-models)
- [Week 2 — Processes & Threads](#week-2--processes--threads)
- [Week 3 — Communication](#week-3--communication)

**Time & Coordination**
- [Week 4 — Clocks & Time](#week-4--clocks--time)
- [Week 5 — Mutual Exclusion](#week-5--mutual-exclusion)
- [Week 6 — Election Algorithms](#week-6--election-algorithms)

**Consistency & Replication**
- [Week 7 — Replication](#week-7--replication)
- [Week 8 — Consistency Models](#week-8--consistency-models)
- [Week 9 — Transactions & Distributed Concurrency](#week-9--transactions--distributed-concurrency)

**Fault Tolerance**
- [Week 10 — Fault Tolerance](#week-10--fault-tolerance)
- [Week 11 — Consensus — Paxos & Raft](#week-11--consensus--paxos--raft)
- [Week 12 — Naming & Security](#week-12--naming--security)

---

## Week 1 — Introduction & System Models

### 1.1 What is a Distributed System?

> **Definition (Tanenbaum):** A **distributed system** is a collection of autonomous computing elements that appears to its users as a single coherent system. Two essential characteristics: (1) nodes are autonomous — they act independently; (2) users perceive a unified system.

Distributed systems are built to achieve goals that single machines cannot:

- **Resource sharing** — hardware (storage, compute), software (services, files).
- **Scalability** — grow capacity by adding nodes, not replacing hardware.
- **Fault tolerance** — individual node failures do not bring down the system.
- **Performance** — parallelism across nodes for data-intensive computation.
- **Geographic distribution** — place computation and data near users.

---

### 1.2 Challenges — The Fallacies of Distributed Computing

| # | Fallacy (False Assumption) | Reality |
|---|---------------------------|---------|
| 1 | The network is reliable | Packets are lost, reordered, delayed, or duplicated |
| 2 | Latency is zero | Network round-trips take microseconds to hundreds of milliseconds |
| 3 | Bandwidth is infinite | Networks saturate; data must be carefully sized and batched |
| 4 | The network is secure | Adversaries exist; encryption, authentication and authorisation are required |
| 5 | Topology doesn't change | Nodes join, leave, crash, and IP addresses change |
| 6 | There is one administrator | Multiple organisations own different parts of the system |
| 7 | Transport cost is zero | Serialisation, protocol overhead, and egress costs are real |
| 8 | The network is homogeneous | Nodes vary in OS, hardware, language, and protocol version |

---

### 1.3 System Models

A system model specifies the assumptions we make about the computing environment.

#### Interaction Model

- **Synchronous systems** — known upper bounds on process execution time, message delay, and clock drift. Timeouts are meaningful. Rare in practice.
- **Asynchronous systems** — no bounds on execution time, message delay, or clock skew. A message might never arrive. Most internet-scale systems must assume this.
- **Partially synchronous** — asynchronous most of the time, but eventually synchronous. Most realistic model.

#### Failure Model

| Failure Type | Description | Severity |
|-------------|-------------|----------|
| Crash failure | Process halts and stays halted. Others can detect via timeout. | Benign |
| Omission failure | Process fails to send or receive messages. | Moderate |
| Timing failure | Response arrives outside specified time bounds. | Moderate |
| Byzantine failure | Process behaves arbitrarily — may lie, corrupt messages, or collude. | Severe |
| Network partition | Network splits into two groups that cannot communicate. | Severe |

---

### 1.4 Transparency in Distributed Systems

| Transparency | Hides from User |
|-------------|----------------|
| Access | Differences in data representation and how a resource is accessed |
| Location | Where a resource is physically located |
| Migration | That a resource may move to another location |
| Relocation | That a resource moved while in use |
| Replication | That multiple copies of a resource exist |
| Concurrency | That a resource is shared with other users |
| Failure | That a resource has failed and been recovered |
| Persistence | Whether a software resource is in memory or on disk |

> **📝 Exam Tip:** The most commonly tested are **location transparency** (DNS — you don't know which server handles your request) and **replication transparency** (a load balancer hides the fact that multiple servers exist). Failure transparency is the hardest to achieve.

---

### 1.5 Scalability

A system can be scaled along three axes: **size** (users/resources), **geographic** (distance), and **administrative** (organisations).

**Scaling techniques:**
- **Vertical scaling (scale-up)** — bigger hardware; limited by physical bounds; single point of failure.
- **Horizontal scaling (scale-out)** — add more machines; requires distribution logic; preferred.
- **Hiding communication latency** — asynchronous communication, prefetching.
- **Partitioning (sharding)** — divide work and spread across nodes.
- **Replication** — copies of data/services near users (caches, CDNs, read replicas).

---

## Week 2 — Processes & Threads

### 2.1 Processes in Distributed Systems

> **Definition:** A **process** is an instance of a program in execution, with its own address space, file descriptors, and system resources. Processes are the fundamental units of distribution.

---

### 2.2 Threads

| Property | Process | Thread |
|----------|---------|--------|
| Address space | Own private address space | Shared with sibling threads |
| State | Heavy — full OS context | Light — PC, stack, registers |
| Creation cost | High (fork/exec) | Low (pthread_create) |
| Communication | IPC (pipes, sockets, shared mem) | Shared memory (no IPC needed) |
| Fault isolation | Crash does not affect others | Crash kills the whole process |
| Context switch | Expensive (full TLB flush) | Cheap (same address space) |

---

### 2.3 Thread Models for Servers

#### Thread-Per-Request
A new thread is spawned per incoming request. Simple programming model. Breaks down under high concurrency due to overhead and stack memory exhaustion.

#### Thread Pool
A fixed set of threads is created at startup. Incoming requests are queued and dispatched to idle workers. Bounded resource usage.

```
Incoming requests          Thread Pool (size = N)
       │
       ▼
┌─────────────┐       ┌──────┐ ┌──────┐ ┌──────┐
│  Work Queue │──────▶│  T1  │ │  T2  │ │  T3  │  ...up to N threads
│  (bounded)  │       └──────┘ └──────┘ └──────┘
└─────────────┘       Each thread picks next job from queue
```

#### Event-Driven / Reactor Pattern
A single thread handles all connections using non-blocking I/O and event multiplexing (`epoll`, `kqueue`, `io_uring`). Used by Nginx, Node.js, Redis.

---

### 2.4 Virtualisation

| Level | Technology | Overhead | Isolation |
|-------|-----------|----------|-----------|
| Full VM | VMware, KVM, Hyper-V | High (full OS per VM) | Strong (hardware isolation) |
| Para-virtualisation | Xen | Medium | Strong |
| Container | Docker, containerd | Very low (shared kernel) | Moderate (namespace/cgroup) |
| Unikernel | MirageOS, Unikraft | Minimal | Strong (single app per VM) |

---

### 2.5 Code Migration

Sometimes moving computation to data is more efficient than moving data to computation. Three components:
- **Code segment** — the program to execute.
- **Resource segment** — references to external resources.
- **Execution segment** — stack, heap, PC, registers. Full migration = **process migration / live migration** (used in cloud VM migration).

---

## Week 3 — Communication

### 3.1 Layered Protocols

| TCP/IP Layer | Protocol Examples | Unit |
|-------------|------------------|------|
| Application | HTTP, gRPC, DNS, SMTP, SSH | Message |
| Transport | TCP, UDP, QUIC, SCTP | Segment / Datagram |
| Internet (Network) | IP (v4/v6), ICMP, BGP, OSPF | Packet |
| Link (Data Link + Physical) | Ethernet, Wi-Fi, ARP | Frame |

---

### 3.2 Remote Procedure Call (RPC)

> **Definition:** **RPC** allows a process to call a procedure on a remote machine as if it were a local function call, achieving *location transparency*.

```
CLIENT SIDE                         SERVER SIDE
────────────                         ────────────
  Client                               Server
  Program                              Program
     │                                    ▲
     │ local call                         │ local call
     ▼                                    │
  Client stub   ─── network ──▶   Server stub
  (marshals args,                  (unmarshals args,
   sends request,                   dispatches to impl,
   receives reply,                  marshals result,
   unmarshals result)               sends reply)
```

#### RPC Semantics

| Semantic | Guarantee | Use Case |
|----------|-----------|----------|
| Maybe | Attempted once; no retries. Outcome unknown. | Unreliable notifications |
| At-least-once | Retried until acknowledged. May execute multiple times. | Idempotent operations (read, delete by ID) |
| At-most-once | Executed zero or one times. Server deduplicates via sequence numbers. | Non-idempotent operations (payment, create) |
| Exactly-once | Executed precisely once. Requires at-most-once + durability. | Ideal — very hard in practice |

---

### 3.3 gRPC

gRPC uses **Protocol Buffers** for interface definition and binary serialisation, running over HTTP/2.

```protobuf
syntax = "proto3";

service OrderService {
  rpc CreateOrder  (CreateOrderRequest)  returns (Order);
  rpc StreamOrders (StreamRequest)       returns (stream Order); // server streaming
  rpc BatchOrders  (stream OrderRequest) returns (BatchResult);  // client streaming
}

message Order {
  string  id          = 1;
  string  customer_id = 2;
  float   total       = 3;
  Status  status      = 4;
  repeated LineItem items = 5;
}

enum Status { PENDING = 0; PROCESSING = 1; SHIPPED = 2; DELIVERED = 3; }
```

**gRPC advantages over REST+JSON:** binary (smaller/faster), strongly typed, native streaming (4 patterns), code generation, built-in deadline propagation and cancellation.

---

### 3.4 Message-Oriented Communication

Unlike RPC (synchronous, coupled), message-oriented middleware (MOM) decouples sender and receiver in time and space.

```
Pub/Sub (Topic-based)

Publisher ──▶ [Topic: order.created] ──▶ Subscriber A (email service)
                                     ├──▶ Subscriber B (inventory service)
                                     └──▶ Subscriber C (analytics)

Message Queue (Point-to-Point)

Producer ──▶ [Queue] ──▶ Consumer A (picks up job 1)
                     ├──▶ Consumer B (picks up job 2)
                     └──▶ Consumer C (picks up job 3)
```

---

### 3.5 Stream Processing — Apache Kafka

Kafka is a distributed, partitioned, replicated **persistent event log**. Key properties:

- **Topics** — logical stream of records, divided into **partitions**.
- **Partitions** — ordered, immutable sequence. Messages within a partition are totally ordered.
- **Offsets** — each message has a monotonically increasing offset. Consumers track their offset.
- **Consumer groups** — multiple consumers in a group each get distinct partitions (horizontal scaling).
- **Retention** — messages retained for a configurable period (default 7 days); allows replay.

---

### 3.6 Multicast Ordering

| Ordering | Guarantee |
|----------|-----------|
| FIFO ordering | Messages from the same sender delivered in send order to all receivers |
| Causal ordering | If send(m1) causally precedes send(m2), then all receivers deliver m1 before m2 |
| Total ordering (Atomic broadcast) | All receivers deliver all messages in the same global order — equivalent to consensus |

---

## Week 4 — Clocks & Time

### 4.1 The Problem of Time

> **Fundamental Impossibility:** Two nodes cannot precisely agree on the current time — light speed and network propagation guarantee non-zero latency. The best we can do is bound the disagreement with synchronisation protocols, or use **logical clocks** that reason about ordering rather than absolute time.

---

### 4.2 Physical Clocks & NTP

Physical clocks drift at ~10–100 ppm. NTP uses a hierarchy of **stratum levels** (stratum 0 = atomic clocks/GPS; stratum 1 = directly connected to stratum 0, etc.). Accuracy: ~1–10 ms over internet; ~0.1 ms on LAN.

- **Cristian's Algorithm:** Client notes T0, sends request, receives reply at T1. Estimated server time = T_server + (T1 − T0)/2. Assumes symmetric round-trip delay.
- **Berkeley Algorithm:** Daemon polls all nodes, computes average, sends each node a delta to adjust by. Handles bad clocks by exclusion.

---

### 4.3 Logical Clocks — Lamport Timestamps

> **Definition (Lamport 1978):** Lamport timestamps provide a partial ordering using the *happens-before* relation (→). If a → b, then C(a) < C(b). The converse does not hold (concurrent events may have any timestamp relationship).

#### Rules

1. Before any event, process P increments: `C[P] = C[P] + 1`
2. When P sends message m: `ts(m) = C[P]`
3. When Q receives message m with timestamp T: `C[Q] = max(C[Q], T) + 1`

---

### 4.4 Vector Clocks

Vector clocks can determine whether events are causally related or concurrent — Lamport clocks cannot.

Each process P_i maintains vector **V[1..N]** (one entry per process).

#### Rules

1. Initially: `V[i] = [0, 0, ..., 0]`
2. Before each local event, P_i increments: `V[i][i] += 1`
3. When P_i sends message m: `ts(m) = V[i]` (piggybacked)
4. When P_j receives message with timestamp T:
   - `V[j][k] = max(V[j][k], T[k])` for all k
   - Then: `V[j][j] += 1`

#### Comparison

- **a → b** (a happens-before b): every component of VC(a) ≤ corresponding component of VC(b), and VC(a) ≠ VC(b)
- **a ∥ b** (concurrent): neither VC(a) ≤ VC(b) nor VC(b) ≤ VC(a)

> **📝 Exam Tip:** When tracing vector clock scenarios: (1) increment your own entry before each event, (2) on receive: component-wise max then increment your own entry. Never skip the increment after the merge on receive.

---

### 4.5 Global State & Chandy-Lamport Snapshot

The Chandy-Lamport algorithm records a consistent global state without pausing the system:

1. Initiator records its own local state; sends a **MARKER** on all outgoing channels.
2. When process Q receives a MARKER on channel C for the first time: record Q's local state; begin recording messages on all *other* channels.
3. When Q receives a MARKER on all channels: stop recording. Recorded messages = in-transit channel state.
4. Global state = all local states + all channel states.

**Uses:** deadlock detection, termination detection, checkpointing for fault tolerance.

---

## Week 5 — Mutual Exclusion

### 5.1 The Mutual Exclusion Problem

> **Definition:** Distributed mutual exclusion ensures at most one process can access a critical section (shared resource) at a time. All coordination must happen via message passing — no shared memory.

Requirements: **Safety** (at most one in CS), **Liveness** (requesting process eventually enters), **Fairness** (FIFO ordering — no starvation).

---

### 5.2 Centralised Algorithm

One coordinator manages access. Process requests → coordinator grants or queues → process releases.

- **Messages per entry/exit:** 3 (request + grant + release)
- **Delay:** 2 message delays
- **Problem:** single point of failure and performance bottleneck.

---

### 5.3 Ricart-Agrawala Algorithm

Fully distributed, based on Lamport timestamps. No coordinator.

1. **To enter CS:** multicast `REQUEST(ts, pid)` to all N−1 other processes.
2. **On receiving REQUEST from j:**
   - Not in CS and not wanting in: reply `OK` immediately.
   - Already in CS: queue the request.
   - Also wanting to enter: compare timestamps. Lower timestamp wins (tie-break on PID). If we lose: send `OK`; if we win: queue their request.
3. **Enter CS:** when `OK` received from all N−1 processes.
4. **Exit CS:** send `OK` to all queued processes.

- **Messages per entry/exit:** 2(N−1) — N−1 requests + N−1 replies
- **Problem:** crash of any process blocks all others.

---

### 5.4 Token Ring Algorithm

Processes arranged in a logical ring. A token circulates; hold it to enter CS, then pass it.

- **Messages per entry/exit:** 0 to N (1 on average)
- **Drawback:** lost token requires additional detection protocol.

---

### 5.5 Comparison

| Algorithm | Messages/entry | Delay | Problems |
|-----------|---------------|-------|----------|
| Centralised | 3 | 2 | Single point of failure |
| Ricart-Agrawala | 2(N−1) | 2(N−1) | N points of failure |
| Token Ring | 1 (avg) | 0 to N−1 | Lost token, ring maintenance |

---

### 5.6 Distributed Deadlock

A deadlock occurs when processes each hold a resource and wait for one held by another — forming a cycle in the **wait-for graph**.

- **Detection:** collect wait-for graphs from all nodes; search for cycles.
- **Probe-based algorithm:** deadlock detected when a probe initiated by process P returns to P.

---

## Week 6 — Election Algorithms

### 6.1 The Election Problem

> **Definition:** An **election algorithm** selects one process to act as coordinator. Triggered when the current coordinator crashes. Requirements: **Safety** (at most one leader at a time), **Liveness** (eventually all agree on a leader).

---

### 6.2 The Bully Algorithm

Each process has a unique numerical priority. Highest priority wins.

1. Process P notices coordinator is dead; sends `ELECTION` to all higher PIDs.
2. If no response: P declares itself leader; broadcasts `COORDINATOR(P)`.
3. If higher-priority process Q receives `ELECTION`: Q responds `OK` to P; Q starts its own election.
4. The process that receives all OKs without another election wins.

- **Best case:** O(1) messages — highest alive process wins immediately.
- **Worst case:** O(N²) messages — lowest process initiates.

---

### 6.3 Ring Election Algorithm

Processes arranged in a logical ring. Any process may initiate.

1. Initiator sends `ELECTION(pid)` around the ring.
2. Each process forwards, replacing PID with its own if larger.
3. When message returns to initiator: it contains the highest PID — that is the coordinator.
4. `COORDINATOR` message circulated to inform all.

- **Messages:** O(N) — 2N messages per election.

---

### 6.4 Raft Leader Election

States: **follower**, **candidate**, **leader**. Term-based.

- Followers expect periodic **heartbeats** from the leader.
- If election **timeout** expires: follower becomes candidate.
- Candidate increments **term**, votes for itself, sends `RequestVote` RPCs.
- Vote granted if: (1) server hasn't voted this term, and (2) candidate's log is at least as up-to-date.
- Candidate wins with **majority** (N/2 + 1) votes → becomes leader, sends heartbeats.
- Split vote → new term begins.

> **Key Property:** At most one leader per term — only one process can receive votes from a majority, since each server votes at most once per term.

---

## Week 7 — Replication

### 7.1 Why Replicate?

Two goals in tension:
- **Reliability** — surviving node failures.
- **Performance** — serve reads from nearby replicas; distribute write load.

More replicas = more coordination overhead on writes.

---

### 7.2 Primary-Backup (Primary-Replica)

All writes go to the **primary**; primary propagates to **backups**.

- **Synchronous replication** — primary waits for acknowledgement from all/quorum backups before acknowledging client. Strong consistency; lower write throughput.
- **Asynchronous replication** — primary acknowledges client immediately; replicates in background. Higher throughput; risk of data loss on primary failure.

---

### 7.3 Multi-Primary (Multi-Leader)

Multiple nodes accept writes. Conflicts must be detected and resolved.

**Conflict resolution strategies:**
- Last-write-wins (LWW) by timestamp
- First-write-wins
- Application-defined merge
- CRDTs (Conflict-free Replicated Data Types)

---

### 7.4 Leaderless (Quorum-based)

Used in Amazon Dynamo, Cassandra, Riak. With N replicas, write quorum W, read quorum R: **R + W > N** guarantees overlap.

```
N=3, W=2, R=2 (R+W=4 > 3 ✓)

Write: Client ──▶ Node 1 ✓
                ──▶ Node 2 ✓   (W=2 achieved → success)
                ──▶ Node 3 ✗   (tolerable failure)

Read:  Client ──▶ Node 1 ✓
                ──▶ Node 2 ✓   (R=2, at least 1 has latest write)
```

---

### 7.5 Anti-Entropy & Gossip Protocols

- **Merkle trees** — hash tree over data; compare trees to find diverging subtrees efficiently (used by Dynamo, Cassandra).
- **Gossip protocol** — each node periodically selects a random peer and exchanges state. Information spreads epidemically. Convergence: O(log N) rounds. Highly resilient to failures.

---

### 7.6 Sloppy Quorums & Hinted Handoff

During partitions/failures: write to any N reachable nodes (sloppy quorum). Nodes accepting writes on behalf of others store a **hint** and forward when the original node recovers (**hinted handoff**). Sacrifices strict consistency for availability.

---

## Week 8 — Consistency Models

### 8.1 The Consistency Spectrum

```
STRONGER (harder, more expensive)                    WEAKER (easier, cheaper)
────────────────────────────────────────────────────────────────────────────▶
Linearisability  →  Sequential  →  Causal  →  Monotonic Read  →  Eventual
```

---

### 8.2 Linearisability (Atomic Consistency)

> **Definition (Herlihy & Wing, 1990):** Every operation appears to take effect instantaneously at some point between its invocation and completion. The system behaves as if there were a single copy of data and all operations are atomic.

Strongest useful consistency model. If operation A completes before B begins, B will see A's effects. Used by: etcd, Google Spanner, ZooKeeper (writes).

---

### 8.3 Sequential Consistency

All operations appear to execute in some sequential order, and each process's operations appear in their program order within that sequence. Weaker than linearisability: does not require the sequence to respect real-time ordering across processes.

---

### 8.4 Causal Consistency

If A causally precedes B (A → B), all processes observe A before B. Concurrent writes may be observed in different orders by different processes. Used by: Facebook COPS, MongoDB causal sessions.

---

### 8.5 Eventual Consistency

> **Definition:** If no new updates are made, eventually all replicas will converge to the same value. No guarantee about *when* convergence occurs or what intermediate values reads may return. Used by: Amazon Dynamo, Cassandra (default), DNS.

---

### 8.6 The CAP Theorem

> **CAP Theorem (Brewer 2000, Gilbert & Lynch 2002):** A distributed data store cannot simultaneously provide all three of: **Consistency** (linearisability), **Availability** (every request receives a response), and **Partition Tolerance** (continues operating despite network partitions).

Since network partitions are unavoidable, designers choose:
- **CP systems** — refuse/error during partitions to preserve consistency. Examples: HBase, ZooKeeper, etcd.
- **AP systems** — return potentially stale data during partitions. Examples: Cassandra, DynamoDB, CouchDB.

> **⚠️ CAP Nuance:** "Consistency" in CAP means **linearisability** specifically — not the C in ACID. CAP only applies during a network partition; during normal operation most CP systems are also highly available.

---

### 8.7 PACELC

Extends CAP: *even when there is no partition, what is the trade-off between latency (L) and consistency (C)?*

**Formula: if P: choose A or C; else: choose L or C**

| System | Partition | Normal Operation |
|--------|-----------|-----------------|
| DynamoDB | PA (available) | EL (low latency) |
| Cassandra | PA (available) | EL (low latency) |
| Google Spanner | PC (consistent) | EC (higher latency) |
| etcd / ZooKeeper | PC (consistent) | EC (higher latency) |

---

### 8.8 CRDTs — Conflict-free Replicated Data Types

CRDTs can be updated concurrently without coordination and guarantee convergence. Operations must be **commutative**, **associative**, and **idempotent**.

| CRDT Type | Supported Operations | Example Use |
|-----------|---------------------|-------------|
| G-Counter (grow-only) | Increment only | View count, votes |
| PN-Counter | Increment + Decrement | Stock level |
| G-Set (grow-only set) | Add only | Unique visitors |
| 2P-Set | Add + Remove (no re-add) | Shopping cart removes |
| LWW-Element-Set | Add + Remove with timestamps | Presence detection |
| OR-Set (Observed-Remove Set) | Add + Remove (supports re-add) | Collaborative editing |

---

## Week 9 — Transactions & Distributed Concurrency

### 9.1 ACID Properties

| Property | Meaning |
|----------|---------|
| **Atomicity** | Transaction either commits fully or aborts completely. No partial execution. |
| **Consistency** | Transaction takes the database from one valid state to another, preserving all invariants. |
| **Isolation** | Concurrent transactions appear to execute serially. |
| **Durability** | Once committed, changes survive crashes (guaranteed by write-ahead logging). |

---

### 9.2 Isolation Levels & Anomalies

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|----------------|-----------|---------------------|-------------|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serialisable | Prevented | Prevented | Prevented |

- **Dirty read** — read uncommitted data from an aborted transaction.
- **Non-repeatable read** — same row yields different values within one transaction.
- **Phantom read** — re-executing a query returns different rows (another transaction inserted/deleted).

---

### 9.3 Two-Phase Locking (2PL)

- **Phase 1 (Growing)** — acquire locks; do not release any.
- **Phase 2 (Shrinking)** — release locks; do not acquire new ones.

Guarantees serialisability but can cause deadlock. **Strict 2PL** — release all locks at commit/abort; prevents cascading aborts.

---

### 9.4 Two-Phase Commit (2PC)

Atomic commitment protocol for distributed transactions.

```
Coordinator                 Participant 1        Participant 2

PHASE 1 (Voting):
  Send PREPARE ──────────▶  Log PREPARED         Log PREPARED
                             Reply YES ─────────▶
                                                  Reply YES ──▶
  Receive all YES

PHASE 2 (Commit):
  Log COMMIT
  Send COMMIT ───────────▶  Log COMMIT           Log COMMIT
                             Reply ACK ──────────▶
                                                  Reply ACK ───▶
```

**2PC Problems:**
- **Blocking problem** — if coordinator crashes after PREPARE but before COMMIT/ABORT, participants that voted YES are blocked indefinitely.
- **Single point of failure** — coordinator failure halts the protocol.

---

### 9.5 Three-Phase Commit (3PC)

Adds a **pre-commit phase** between PREPARE and COMMIT. Makes the protocol non-blocking under fail-stop failures. Breaks under network partitions. Rarely used; Paxos/Raft preferred.

---

### 9.6 Snapshot Isolation (MVCC)

MVCC maintains multiple versions of each data item. Readers see a consistent snapshot at transaction start; writers create new versions without blocking readers.

**Write skew anomaly:** two concurrent transactions each read data, make decisions, resulting in a combined outcome violating an invariant — even though each individual transaction is correct. MVCC does not prevent write skew; serialisable isolation required.

Used by: PostgreSQL, Oracle, MySQL InnoDB, Google Spanner.

---

## Week 10 — Fault Tolerance

### 10.1 Fault Tolerance Concepts

| Term | Definition |
|------|-----------|
| Fault | A defect in a component (hardware bug, packet loss, cosmic ray) |
| Error | Incorrect state caused by a fault being activated |
| Failure | Deviation of delivered service from specification — observable by users |
| Reliability | Probability of correct operation over time: R(t) = e^(−λt) |
| Availability | Fraction of time system is operational: A = MTTF / (MTTF + MTTR) |
| MTTF | Mean Time To Failure |
| MTTR | Mean Time To Repair |
| MTBF | Mean Time Between Failures = MTTF + MTTR |

---

### 10.2 Redundancy

- **Information redundancy** — extra bits for error detection/correction. Hamming codes, Reed-Solomon, CRC, parity.
- **Time redundancy** — repeat operations. Useful against transient faults. 2PC is an example.
- **Physical redundancy** — duplicate hardware. TMR (Triple Modular Redundancy), RAID disks, replicated servers.

---

### 10.3 k-Fault Tolerance

A system is *k-fault tolerant* if it handles k simultaneous component failures and still functions correctly.

- **Crash failures:** requires k+1 replicas.
- **Byzantine failures:** requires **3k+1** replicas for masking. Extra replicas needed because a Byzantine node may actively lie and collude.

---

### 10.4 RAID Levels

| Level | Description | Fault Tolerance | Overhead |
|-------|-------------|----------------|----------|
| RAID 0 | Striping — no redundancy | None (worse than single) | 0% |
| RAID 1 | Mirroring — exact copy | 1 disk failure | 50% |
| RAID 5 | Striping with distributed parity | 1 disk failure | 1/N |
| RAID 6 | Striping with double distributed parity | 2 disk failures | 2/N |
| RAID 10 | Mirroring + striping | Multiple (in different mirror sets) | 50% |

---

### 10.5 Checkpointing & Recovery

**Checkpointing** periodically saves system state to stable storage.

- **Rollback recovery** — restore from last checkpoint; redo lost work.
- **Message logging** — log all messages received since last checkpoint; replay after failure. Requires deterministic message processing.

**Coordinated vs Uncoordinated:**
- **Coordinated** — all processes checkpoint simultaneously (Chandy-Lamport style). Guaranteed consistent; coordination overhead.
- **Uncoordinated** — processes checkpoint independently. Risk of **domino effect** — recovery cascades back to the beginning.

---

### 10.6 Byzantine Fault Tolerance (BFT)

Byzantine failures are arbitrary — faulty nodes may send different values to different peers, selectively drop messages, or collude.

- Minimum **3f+1** total nodes to tolerate f Byzantine failures.
- **PBFT** (Castro & Liskov 1999) — O(N²) message complexity per consensus round.
- Modern alternatives: HotStuff (linear communication), Tendermint (blockchain consensus).

---

## Week 11 — Consensus — Paxos & Raft

### 11.1 The Consensus Problem

> **Definition:** A set of processes propose values and must agree on exactly one. Requirements: (1) **Agreement** — all non-faulty processes decide the same value; (2) **Validity** — the decided value was proposed by some process; (3) **Termination** — all non-faulty processes eventually decide.

---

### 11.2 The FLP Impossibility Result

> **FLP (Fischer, Lynch, Paterson 1985):** In an **asynchronous** system where even **one process may crash**, there is no deterministic algorithm that can guarantee consensus.

**Practical workarounds:**
- Use **partial synchrony** (timeouts) — allows non-deterministic termination.
- Use **randomised algorithms** — guarantee consensus with probability 1.
- Accept that termination may occasionally be delayed.

---

### 11.3 Paxos

Paxos operates with three roles: **Proposers**, **Acceptors**, **Learners**. A majority (quorum) of acceptors must agree.

```
PHASE 1 — PREPARE / PROMISE
  Proposer → all Acceptors: Prepare(n)     [n = ballot number]
  Acceptor → Proposer:      Promise(n, accepted_n, accepted_v)
    [Promise: won't accept proposals with ballot < n]
    [Returns: highest previously accepted (ballot, value) if any]

PHASE 2 — PROPOSE / ACCEPT
  Proposer → majority of Acceptors: Accept(n, v)
    [v = acceptor's previously accepted value (if any), else proposer's own]
  Acceptor → Proposer: Accepted(n, v)     [if ballot n ≥ promised ballot]

VALUE CHOSEN when majority send Accepted(n, v)
Proposer → Learners: Decided(v)
```

**Key insight:** if a value was previously chosen, phase 1 discovers it (via highest accepted value in promises), and phase 2 preserves it. Paxos never violates safety but may not terminate (FLP).

**Multi-Paxos:** for a replicated log, a stable leader can skip phase 1 for subsequent slots — reducing to 1 round-trip per decision.

---

### 11.4 Raft

Raft decomposes consensus into three subproblems:

#### a) Leader Election
Term-based; majority vote; election timeout. Covered in Week 6.

#### b) Log Replication

1. Client sends command to leader.
2. Leader appends command to its log with current term.
3. Leader sends `AppendEntries` RPC to all followers.
4. Once majority acknowledge: leader **commits** the entry, applies to state machine, responds to client.
5. Committed entries included in future `AppendEntries` (followers commit when they see leader committed).

```
Leader log:      [1:set x=1] [2:set y=2] [3:del x] ← committed up to index 3
                      │           │           │
AppendEntries ────────┼───────────┼───────────┼──────▶ Followers
                      ▼           ▼           ▼
Follower A log:  [1:set x=1] [2:set y=2] [3:del x]  ✓
Follower B log:  [1:set x=1] [2:set y=2]             (lagging — will catch up)
Follower C log:  [1:set x=1] [2:set y=2] [3:del x]  ✓
```

#### c) Log Matching & Safety

**Log Matching Property:** if two entries in different logs have the same index and term: (1) they store the same command; (2) the logs are identical in all preceding entries.

**Leader Completeness:** a leader is elected only if its log contains all committed entries — voters only grant votes to candidates whose log is at least as up-to-date.

> **📝 Exam Tip:** A leader in a minority partition continues appending entries but they won't be committed (no majority acknowledgement). When partition heals, the new leader's log overwrites the old leader's uncommitted entries on straggler nodes.

---

### 11.5 Paxos vs Raft

| Aspect | Paxos | Raft |
|--------|-------|------|
| Understandability | Notoriously difficult | Designed for clarity |
| Leader | Any proposer may act; optimised with stable leader | Strong leader — all writes via leader |
| Log gaps | Possible (holes in log) | No gaps — entries committed in order |
| Membership changes | Not specified in original paper | Joint consensus / single-server changes specified |
| Used in | Google Chubby, ZooKeeper (ZAB ≈ Paxos), Google Spanner | etcd, CockroachDB, TiKV, Consul |

---

## Week 12 — Naming & Security

### 12.1 Naming in Distributed Systems

> **Definition:** A **name** is a string used to refer to a resource. A **naming system** resolves names to the addresses needed to access those resources, providing *location transparency*.

#### Three Levels of Names

- **Human-friendly names** (symbolic) — domain names, file paths, URLs. Easy for humans; require resolution.
- **Addresses** — IP addresses, port numbers, MAC addresses. Machine-usable; may change.
- **Identifiers** — flat, unique, never reused (UUIDs, inodes, content hashes). Stable; no structure.

---

### 12.2 The Domain Name System (DNS)

DNS is the internet's globally distributed, hierarchical, delegated database mapping domain names to IP addresses.

```
DNS Hierarchy

              . (root)
             / \
           com  org  net  ke  ...
           /
         example
           /
          www   mail   api   ...

Resolution of "api.example.com":
1. Client → local resolver (recursive): resolve api.example.com
2. Local resolver → root server: who handles .com?
3. Local resolver → .com TLD server: who handles example.com?
4. Local resolver → example.com authoritative: what is api.example.com?
5. Authoritative → local resolver: 203.0.113.42
6. Local resolver → client: 203.0.113.42 (cached per TTL)
```

#### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | api.example.com → 203.0.113.42 |
| AAAA | IPv6 address | api.example.com → 2001:db8::1 |
| CNAME | Canonical name (alias) | www → api.example.com |
| MX | Mail exchange server | example.com → mail.example.com (prio 10) |
| NS | Authoritative nameserver | example.com → ns1.example.com |
| TXT | Arbitrary text (SPF, DKIM, verification) | v=spf1 include:_spf.google.com ~all |
| SOA | Start of Authority — zone metadata | Serial, refresh, retry, expire, TTL |
| SRV | Service location | _grpc._tcp.example.com → host:port |

---

### 12.3 Distributed Hash Tables — Chord

DHTs provide decentralised, peer-to-peer lookup without a central authority.

**Chord (Stoica et al., 2001):**
- N nodes on a logical ring of 2^m identifiers (m-bit hash space).
- Each node responsible for keys in range (predecessor, node].
- **Finger table:** each node stores O(log N) pointers enabling O(log N) lookup hops.

```
Chord ring (m=6, IDs 0-63):

         0
      56   8
    48       16
      40   24
         32

Node 8 finger table: [16, 24, 40, 8, 8, 8]
(finger[i] = successor of node 8 + 2^i)

Lookup for key 42 from node 8:
  8 → closest finger to 42 = 40 → 40 → successor(42) = 48
  Answer: key 42 is at node 48. Hops: O(log N)
```

---

### 12.4 Security Threat Model

- **Eavesdropping** — passive interception of messages.
- **Tampering** — active modification of messages in transit.
- **Replay attacks** — capturing and re-sending a valid message.
- **Masquerading (spoofing)** — impersonating another principal.
- **Denial-of-service** — overwhelming a node with requests.
- **Byzantine / insider attacks** — compromised nodes misbehaving.

---

### 12.5 Cryptographic Primitives

| Primitive | Provides | Algorithm Examples |
|-----------|---------|-------------------|
| Symmetric encryption | Confidentiality (same key) | AES-256-GCM, ChaCha20 |
| Asymmetric encryption | Confidentiality without shared secret | RSA-OAEP, ECIES |
| Digital signature | Authenticity + Integrity | RSA-PSS, ECDSA, EdDSA |
| Cryptographic hash | Integrity (one-way; collision-resistant) | SHA-256, SHA-3, BLAKE3 |
| MAC (HMAC) | Integrity + Authenticity (shared key) | HMAC-SHA256, Poly1305 |
| Key exchange | Establish shared secret over untrusted channel | ECDH, X25519 |
| Certificate (PKI) | Bind public key to identity | X.509, TLS certificates |

---

### 12.6 TLS 1.3 Handshake (1-RTT)

```
Client                                         Server
──────                                         ──────
ClientHello                ──────────────────▶
  supported_versions: TLS 1.3
  key_share: X25519 pub key
  cipher_suites: AES-128-GCM, ChaCha20

                           ◀──────────────────ServerHello
                                               key_share: X25519 pub key
                                               selected cipher
                           ◀──────────────────{Certificate}        [encrypted]
                           ◀──────────────────{CertificateVerify}  [signature]
                           ◀──────────────────{Finished}

{Finished}                 ──────────────────▶

Application Data           ◀─────────────────▶ Application Data
```

---

### 12.7 Kerberos Authentication

Kerberos provides mutual authentication using a trusted third party (Key Distribution Centre, KDC) and symmetric cryptography. Used in Active Directory, enterprise networks.

1. Client → KDC (Authentication Server): "I am Alice; give me a TGT."
2. KDC → Client: Ticket Granting Ticket (TGT) encrypted with Alice's key + session key.
3. Client → KDC (Ticket Granting Server): "I want to access FileServer; here's my TGT."
4. KDC → Client: service ticket encrypted with FileServer's key + session key.
5. Client → FileServer: service ticket + authenticator (timestamp encrypted with session key).
6. FileServer decrypts service ticket, verifies authenticator → mutual authentication confirmed.

---

### 12.8 Summary — Three Fundamental Tensions

Every distributed systems designer must navigate:

1. **Consistency vs. Availability** (CAP) — cannot have both during partitions; must choose.
2. **Safety vs. Liveness** (FLP) — in asynchronous systems, cannot guarantee both termination and safety under even one crash failure.
3. **Performance vs. Correctness** — weaker consistency models perform better but require complex application-level reasoning.

> **Final Observation:** Every major distributed systems problem — consensus, mutual exclusion, replication, fault tolerance — ultimately comes down to: how do independent, failure-prone nodes, communicating over an unreliable network, agree on anything? The answer is always: **quorums**, **message passing**, **careful reasoning about ordering**, and **graceful degradation under failure**.

---

*End of Course Notes — COMP3XXX Distributed Systems*
