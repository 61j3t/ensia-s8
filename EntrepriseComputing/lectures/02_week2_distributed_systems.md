# Week 2 — Distributed System Design

## Bird's eye view

- The lecture is organized into **three thematic decks**: System Availability, Data Consistency, and Architecting Resilience (Maintainability, Reliability, Fault Tolerance).
- A **distributed system** is a networked set of independent nodes that appear to users as a single coherent system — contrasted with the old single-server monolith.
- The core challenge is that distribution introduces **physics-level problems**: propagation delay, network partitions, and independent failures that make simultaneity an illusion.
- **Availability** is the proportion of time a system is operational; the High Availability (HA) standard targets 99.9%+ uptime, which allows only ~8.76 hours of downtime per year.
- **Consistency** is whether all nodes see the same data at the same moment; it exists on a spectrum from Strong (linearizability) to Eventual to Weak.
- The **CAP theorem** is the fundamental constraint: a distributed system can guarantee at most two of Consistency, Availability, and Partition Tolerance simultaneously. Since partitions are inevitable, the real choice is CP vs AP.
- The three non-functional pillars of resilient design are **Maintainability** (the foundation), **Reliability** (the promise), and **Fault Tolerance** (the safety net).

---

## Detailed notes

### 1. From Single Server to Distributed Systems

The past: a single server cube handles all requests. A single failure takes everything down.

The present: a mesh of interconnected nodes (visualized as a dense graph of cubes with edges between all pairs). Modern computing is inherently networked and shared-data. This distribution introduces complexity that the CAP theorem helps reason about.

**Why distribute at all?**
- Scale beyond what a single machine can handle
- Survive hardware failures
- Place computation closer to users geographically
- Enable concurrent processing

---

### 2. System Availability

#### 2.1. Definition and measurement

**System Availability** = the readiness and accessibility of a service at any given moment. Calculated as:

```
Availability (%) = (Uptime / (Uptime + Downtime)) * 100
```

**The High Availability (HA) standard**: remain operational for 99.9% or higher of time.

Example — cost of 99.9% over one year:
```
Total minutes in a year = 365 * 24 * 60 = 525,600
Allowable downtime = 0.001 * 525,600 = 525.6 minutes (~8.76 hours)
```
This 8.76 hours must cover all unplanned failures, maintenance windows, and recovery time combined.

#### 2.2. Business impact of downtime

Availability is not merely a technical metric:

| Dimension | Impact |
|---|---|
| **User Experience** | Frustration from outages leads to dissatisfaction and churn |
| **Business Continuity** | Even short outages cause significant financial losses and reputational harm |
| **SLAs & Compliance** | Failure to meet Service Level Agreements results in fines or legal consequences |
| **Competitive Advantage** | Superior uptime differentiates a product in dependability-critical markets |
| **Disaster Resilience** | Designing for availability protects against hardware failures, cyberattacks, and natural disasters |

#### 2.3. System Availability vs. Asset Reliability

- **Asset Reliability** (Component View): the ability of an *individual component* to perform without failure. Focus: reduce likelihood of a specific part going down.
- **System Availability** (Holistic View): how often the *entire system* is accessible, accounting for recovery time and redundancy.
- **Key distinction**: a system can be highly available even if individual assets are unreliable, provided sufficient redundancy exists. A failed component does not equal a failed system.

#### 2.4. Availability vs. Fault Tolerance

| Feature | Availability | Fault Tolerance |
|---|---|---|
| Goal | Maximize Uptime | Prevent Outages |
| Focus | Continuous Access | Handling & Recovering from Failure |
| Measurement | % of Uptime | MTBF & MTTR |
| User Experience | Minimal Disruption | Graceful Degradation (no complete failure) |
| Redundancy | Standard | High Degree |

Fault Tolerance often requires immediate backup for all components; Availability focuses on aggregate uptime percentage.

#### 2.5. The engineering toolkit for resilience

Three pillars of strategy:

**Pillar 1: Prevention & Architecture**
- **Redundancy**: Duplicate servers/components so that if one fails, another takes over seamlessly. Goal is elimination of single points of failure. Scope includes hardware (drives, power), networking (paths, switches), and data centers (geo-distribution).
  - Diagram — Active-Passive Redundancy: A load balancer sits on the left. Path A goes to the Primary Server (Active); Path B (shown as a dashed orange reroute) goes to the Secondary Server (Standby). When the primary fails (shown with an X), the load balancer redirects all traffic via Path B.
- **Scalability**: Designing the system to accept additional resources to accommodate increased demand without crashing.
  - Vertical Scaling (Scale Up): add more power (CPU/RAM) to an existing server stack.
  - Horizontal Scaling (Scale Out): add more server nodes alongside existing ones.
- **Performance Optimization**: tuning existing resources for maximum throughput.

**Pillar 2: Traffic Management**
- **Load Balancing**: Distributes incoming requests across multiple servers to prevent overload on any single component. Visualized as a funnel routing traffic to Server 1, 2, and 3. Balances workload to ensure optimal performance, reliability, and availability.

**Pillar 3: Recovery & Response**
- **Failover Mechanisms**: Automated processes that detect failures and switch operations to redundant systems without manual intervention.
  - Automatic Failover Sequence (diagram): Heartbeat Check → Failure Detected → Trigger Failover → Traffic Rerouted.
- **Disaster Recovery (DR)**: A comprehensive plan for catastrophic events (natural disasters, major infrastructure loss) to restore data integrity.
- **Monitoring and Alerting — The Vigilance Loop**: A continuous cycle: DETECT (real-time health tracking) → ALERT (immediate notification of anomalies) → ACT (prompt rectification before downtime) → back to DETECT. Continuous monitoring is the proactive layer that triggers reactive measures.

**The Cycle of Resilient Design** (four-arrow chevron diagram):
1. MEASURE — define uptime goals, calculate 99.9%
2. DESIGN — redundancy, load balancing, scalability
3. RECOVER — automated failover, disaster recovery
4. MONITOR — track compliance, detect anomalies

High Availability is not a feature; it is a continuous commitment to user trust and business continuity.

---

### 3. Data Consistency

#### 3.1. The problem: the illusion of simultaneity

In an ideal world, data updates appear instantly everywhere. In a distributed system, physics prevents this.

Diagram — The Ideal vs. The Reality:
- Ideal: Server A, B, and C each receive an update at t=0 simultaneously (clean horizontal arrows).
- Reality: Server A receives the update first; Servers B and C receive it later via propagation delay (t + latency). Network partitions (shown as dotted lines interrupting the arrows) can prevent or delay delivery entirely.

**Consistency** = the property ensuring all nodes in a distributed system have the same view of the data at any given point in time, despite concurrent operations.

The conflict: users expect instantaneous updates; the system reality is network delays and asynchronous propagation.

#### 3.2. Why consistency matters — The Pillars of Necessity

- **Data Integrity & Correctness**: preventing corruption or loss; essential for financial ledgers where the state cannot be wrong.
- **User Experience**: Read-Your-Writes — ensuring users see their own posts immediately to prevent confusion.
- **Reliability & Concurrency**: surviving node failure without data corruption; managing multi-user access to prevent conflicting modifications.
- **Scalability**: mechanisms to manage consistency without sacrificing performance as the system grows.

#### 3.3. The consistency spectrum

A gradient from left (strict, high latency, high accuracy) to right (permissive, low latency, high availability):

```
Strong Consistency  ---  Hybrid / Causal  ---  Weak / Eventual
(High Latency / High Accuracy)  (Context-Aware)  (Low Latency / High Availability)
<-- Strictness ------------------------------------------ Performance -->
```

System design is the art of choosing where your application lives on this line.

Real-world DB mapping:
```
SQL / ACID  -->  Google Spanner / MongoDB (Tunable)  -->  DynamoDB / Cassandra  -->  Redis / CDN
(Strong)         (Hybrid)                                 (Eventual)               (Weak)
```

#### 3.4. Strong Consistency (Linearizability)

Guarantees that every read receives the most recent write or an error. Updates appear instantaneous to all clients.

Architecture diagram: a Master Node at the top connects bidirectionally to Client 1. The Master Node fans out to Replica Node 1 and Replica Node 2. Client 2 connects to Replica Node 1 but its read is blocked (shown as a red bar) until the write propagates from the master.

**The cost**: Heavy coordination. Latency increases because the system must wait for all nodes to agree before acknowledging a write.

Use cases: financial transactions, inventory management, critical healthcare data.

**Mechanisms that enforce strong consistency:**

| Mechanism | How it works |
|---|---|
| **Synchronous Replication** | A write is not complete until it is propagated to all replicas. Zero data loss, maximum latency. |
| **Quorum Consistency** | A majority of replicas must agree on the value before commit. Rule: R + W > N (reads + writes > total nodes). |
| **Strict Two-Phase Locking (2PL)** | Phase 1 acquires all locks; Phase 2 releases all locks. Only one transaction accesses a data item at a time. |
| **Serializability** | Concurrent transactions execute as if they were serial (sequential), preserving system state. |

These patterns prioritize data consistency over partition tolerance.

#### 3.5. Eventual Consistency

Allows temporary inconsistencies between replicas but guarantees they will eventually converge to a consistent state without human intervention.

Diagram: many updates (shown as squares) enter a wide funnel (asynchronous propagation), narrowing down to a single Converged State output at the bottom.

Value proposition: high availability and low latency. Ideal for social media feeds. Example: Amazon DynamoDB.

**Mechanisms of eventual consistency (how systems heal):**

Three-node diagram with nodes A (Fresh), B (Stale initially, then updated), and C (Blue):
- **Read Repair**: when a client reads a stale value, the system automatically updates it to the most recent version in the background.
- **Anti-Entropy**: background processes periodically compare data between replicas and reconcile differences using Merkle Trees.
- **Gossip Protocols**: nodes randomly "whisper" updates to neighbors, ensuring information spreads like a virus through the network.

#### 3.6. Weak Consistency

Unlike Eventual Consistency, Weak Consistency offers the least assurance: it does not guarantee *when* replicas will converge.

Diagram: a central Origin Server fans out via blue lines to many Edge Servers distributed around a world map.

- Use case: distributed caching (Redis, Memcached); CDNs.
- Behavior: data is retrieved quickly from in-memory cache.
- Priority: low latency is the absolute priority over data accuracy.

#### 3.7. Causal Consistency

Preserves the causal relationship between related events. If Event A causes Event B, all nodes must see A before B. Unrelated events can be seen in any order.

Diagram: Box A ("Post: I love this!" at Time T1) → orange arrow → Box B ("Reply: Me too!" at Time T2). Everyone must see the post exist before seeing the reply.

Example: Google Docs collaborative editing — if User A writes a paragraph and User B corrects a typo in it, everyone must see the paragraph before they see the correction.

#### 3.8. Client-centric consistency patterns

These shift focus from the system's global state to an individual user's view:

| Pattern | Guarantee |
|---|---|
| **Read-Your-Writes** | A process always sees its own writes immediately, even if the rest of the world doesn't (e.g., social profiles). |
| **Monotonic Reads** | If a client reads a value, they never see an older value in the future. Time only moves forward. |
| **Session Consistency** | Sticky sessions ensure a consistent view for one user during their entire interaction with the system. |

#### 3.9. Conflict resolution when writes collide

When two concurrent writes reach the same data record (Write A at T1, Write B at T2) simultaneously:

| Strategy | Mechanism |
|---|---|
| **Last-Writer-Wins (LWW)** | Favors the update with the latest timestamp. Simple but risks data loss. |
| **Vector Clocks** | Associates updates with a logical clock vector [1, 0, 2] that tracks causality across replicas. Each node maintains its own vector timestamp. Detects conflicts rather than silently dropping. |
| **CRDTs (Conflict-free Replicated Data Types)** | Specialized data structures designed to merge concurrent updates mathematically without coordination. Guarantees eventual consistency by design. |

#### 3.10. Costs of strong consistency

Pursuing strict consistency incurs:
- **Performance Overhead**: synchronization kills throughput and increases latency.
- **Availability Trade-offs** (CAP Reality): strong consistency often means refusing requests during network partitions.
- **Operational Complexity**: configuring replication settings and conflict resolution logic is error-prone; debugging distributed state is notoriously difficult.
- **Scalability Limitations**: strict coordination becomes a bottleneck as the number of nodes increases.

#### 3.11. Hybrid strategies — real-world mapping

Design pattern: use a "Single Source of Truth" for critical data and "Asynchronous Updates" for scalable components.

| Use Case | Strategy | Rationale |
|---|---|---|
| Banking / Inventory | Strong Consistency (ACID) | Prevent financial errors |
| User Profiles | Read-Your-Writes | Session stability |
| Collaborative Apps | Causal Consistency | Logical ordering of edits |
| Social Media Timelines | Eventual Consistency | High volume, high availability |

---

### 4. The CAP Theorem

#### 4.1. The three desirables

The CAP theorem (Brewer's theorem) states that a distributed system can have at most two of the following three guarantees simultaneously:

| Property | Definition |
|---|---|
| **Consistency (C)** | All clients see the same data simultaneously, no matter which node they connect to. Any change on one node is immediately propagated to all others. |
| **Availability (A)** | All non-failing nodes return a response for all read and write requests in a bounded amount of time. Every request receives a response — whether successful or not. |
| **Partition Tolerance (P)** | The system continues to operate despite arbitrary message loss or network failure between parts of the system. It can gracefully recover once the partition heals. |

Triangle diagram: Consistency (top), Availability (bottom-left), Partition Tolerance (bottom-right). The region inside the triangle is labeled IMPOSSIBLE. Positioned on each edge: CA (e.g., RDBMS), CP (e.g., HBase), AP (e.g., Cassandra).

#### 4.2. Why P is mandatory — the hidden constraint

Network failures are inevitable: cables get cut, routers crash. If a system cannot handle a partition, it is not a distributed system. Therefore, P is a **requirement, not an option**. The practical choice becomes:

- **CP (Consistency + Partition Tolerance)**: during a partition, sacrifice availability to preserve correctness.
- **AP (Availability + Partition Tolerance)**: during a partition, sacrifice strict consistency to preserve uptime.

The Architect's Decision Tree:
```
Network Partition Occurs
         |
    +----+----+
    |         |
   CP         AP
(Prioritize  (Prioritize
 Data         System
 Consistency) Availability)
```

#### 4.3. CP Systems — "I need the data to be right, even if the system has to pause"

Mechanism: when a partition occurs between Node A and Node B, the system shuts down the non-available (non-updated) node until the partition is resolved. It refuses requests rather than risking inconsistent data.

Diagram: User requests go to Node A (Active). A lightning bolt partition separates Node A from Node B. Node B shows "SYSTEM PAUSED" with a lock icon and shuts down during partition to prevent data divergence.

Tech stack examples: MongoDB, Redis, HBase.
Case study — Banking: inconsistency could lead to double spending or incorrect balances. Better to show "Service Temporarily Unavailable" than to lie. "Better to fail than to lie."

#### 4.4. AP Systems — "I need the system to run, even if the data is slightly old"

Mechanism: when a partition occurs, all nodes remain available and accept requests. However, nodes on the wrong side of the partition may return an older version of data than others (nodes drift apart but stay online).

Diagram: Node A shows Balance: $100, Node B shows Balance: $50. A lightning bolt partition separates them. Both stay online; they drift apart but continue serving.

Tech stack examples: CouchDB, Cassandra, DynamoDB.
Case study — Social Media Newsfeed: a "Service Unavailable" error is worse than seeing a post 5 seconds late. The system prioritizes uptime over perfect synchronization.

#### 4.5. CA Systems — the monolith

Delivers Consistency and Availability but cannot handle a partition. This generally only exists in non-distributed (single-server) systems like a standalone SQL server. If a partition happens, a CA system fails entirely. This architecture does not support Partition Tolerance.

#### 4.6. System Design Matrix

| Property | CP | AP | CA |
|---|---|---|---|
| Primary Goal | Data Consistency | System Uptime | Monolithic Reliability |
| Behavior under Partition | Shuts down non-updated nodes | Returns potentially old data | System Failure |
| Ideal Use Case | Banking, Financial Ledgers | Social Feeds, Caching, DNS | Legacy / Single-site Apps |
| Key Technologies | MongoDB, HBase, Redis | Cassandra, DynamoDB | RDBMS (MySQL, PostgreSQL) |

#### 4.7. Key takeaways for the architect

1. **Partitions are inevitable** — design for P; you cannot wish away network failures.
2. **The choice is binary** — during a failure you must choose Data Accuracy (CP) or Uptime (AP); you cannot guarantee both simultaneously.
3. **Context is king** — there is no "best" database. A bank requires CP; a newsfeed requires AP. Advanced systems switch strategies dynamically (hybrid mode switching per operation type).

---

### 5. Architecting Resilience: The Three Pillars

Non-functional requirements determine a system's longevity and stability. A functioning feature is not a functioning system.

Architecture blueprint diagram: three interconnected data center modules labeled Reliability Core (top), Maintainability Module (bottom-left), and Fault Tolerance Matrix with Isolation Zones (bottom-right). Orange-highlighted Failover Paths connect them.

#### 5.1. Pillar I: Maintainability — The Foundation

Maintainability = the capacity for a system to be updated and repaired profitably. It determines how easy and cost-effective it is to maintain, update, and upgrade a software system.

**Four components of maintainability:**
- **Modularity**: organized into distinct modules; individual pieces can be modified without affecting the whole.
- **Readability**: codebase is clear, terse, and readable; prioritizes human understanding for future maintainers.
- **Error Handling**: problems handled with meaningful error messages, avoiding disastrous failures (fail gracefully).
- **Testability**: the system helps identify implicit issues quickly; effective testing structures are built-in.

**Measuring code health:**

| Metric | What it measures |
|---|---|
| **Maintainability Index** | A composite score combining complexity, duplication, and other factors. Higher = better. |
| **Code Churn** | Frequency of changes to a module over time (via VCS data). Excessive churn signals instability or need for refactoring. |
| **Cyclomatic Complexity** | Complexity of a code module. High complexity = difficult to understand, needs simplification. |
| **Code Duplication** | Percentage of repeated code segments. Duplication means one change requires changes elsewhere, increasing inconsistency risk. |

**Strategies for sustainable engineering:**
- **Design Patterns**: use MVC and SOLID principles to promote modularity and flexibility.
- **Comprehensive Documentation**: go beyond code comments — cover architectural decisions, system design, and API references to reduce onboarding cost.
- **Test-Driven Development (TDD)**: write test cases before implementation; ensures codebase is understood before written.
- **Code Consistency**: uniform style and meaningful variable naming.
- **Collaborative Culture**: peer Code Reviews and Knowledge Transfer (KT) programmes to prevent knowledge silos.

#### 5.2. Pillar II: Reliability — The Promise

Reliability = the probability that a system performs its intended function without failure over a specific period. It ensures stable performance and consistent availability.

**Factors affecting reliability:**
- **Design & Hardware**: poor planning or low-quality components lead to frequent breakdowns.
- **Software Bugs**: errors in code causing crashes or malfunctions.
- **Workload**: overloading a system beyond capacity triggers failure.
- **External Conditions**: heat, power surges, or network volatility.

**Reliability metrics (mathematics of uptime):**

| Metric | Formula | Meaning |
|---|---|---|
| **Uptime %** | ((TotalTime - Downtime) / TotalTime) * 100 | Ratio of operational time to total time |
| **MTBF** (Mean Time Between Failures) | Total Operational Time / Number of Failures | Long-term measure of performance over lifespan |
| **MTTR** (Mean Time To Repair) | Total Repair Time / Number of Failures | Measures the speed of recovery |
| **Error Rate** | (Number of Errors / Total Transactions) * 100 | Quantifies frequency of transaction failures |

**Single Point of Failure (SPOF)**: a component whose failure causes the entire system to stop working. Visualized as a hub-and-spoke network with one central orange node (SPOF) connecting all peripherals. After eliminating the SPOF: multiple dark-blue hub servers share the load with redundancy and load balancing.

The goal: consistent performance with minimal failures and smooth error handling.

#### 5.3. Pillar III: Fault Tolerance — The Safety Net

Fault Tolerance = a system's capacity to continue operating — or deteriorate gracefully — in the face of hardware or software failure.

Three-layer architecture diagram (isometric, stacked):
- **Top layer (Microservices Architecture)**: a mesh of many small service boxes; "FAIL: Service X" is shown failing in one corner while the rest of the mesh continues operating normally. Isolates faults so one failing service does not crash the entire application.
- **Middle layer (Distributed Cloud Architecture)**: a world map with two data center clusters (US-East marked as Zone Failure with an X; EU-Rest marked as Active Zone). Reduces impact of regional failures by distributing applications across providers or availability zones.
- **Bottom layer (RAID)**: multiple disk drives, with one showing "DISK FAILURE." RAID distributes data across multiple disks to survive drive failure.

**Replication strategies:**

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Full Replication** | Complete duplication of all data to another node (bidirectional sync) | Simple recovery | Resource-intensive |
| **Partial Replication** | Selective duplication of critical data only | Resource efficient | Complex selection logic |
| **Shadowing (Passive)** | Shadow copy sits idle; activates only on failure | Efficient during normal ops | Slower activation time |
| **Active Replication** | Both primary and replica process requests concurrently | Highest fault tolerance | High overhead |

#### 5.4. The cost of resilience — the Iron Triangle

Pursuing resilience involves unavoidable trade-offs across three vertices:
- **Scalability**: failover mechanisms must grow with data size.
- **Performance**: redundancy checks and active replication consume compute resources (overhead).
- **Cost**: expensive hardware, licenses, and monitoring systems.

You cannot optimize all three simultaneously. Engineering decisions must explicitly choose which vertices to prioritize.

---

## Key terms

- **System Availability** — the proportion of time a system is functional; measured as (Uptime / Total Time) * 100.
- **High Availability (HA)** — the goal of 99.9%+ uptime, leaving less than 8.76 hours of downtime per year.
- **SLA (Service Level Agreement)** — contractual availability commitment; failure results in fines or legal consequences.
- **Consistency** — all nodes see the same data view at the same time, despite concurrent operations.
- **Strong Consistency (Linearizability)** — every read returns the most recent write or an error; requires coordination.
- **Eventual Consistency** — replicas may temporarily diverge but are guaranteed to converge; high availability, low latency.
- **Weak Consistency** — no guarantee on when convergence occurs; used in CDNs and caches.
- **Causal Consistency** — causally related events are seen in causal order by all nodes; unrelated events may be reordered.
- **CAP Theorem** — a distributed system can satisfy at most two of: Consistency, Availability, Partition Tolerance.
- **Partition Tolerance (P)** — the system operates despite arbitrary network failures between nodes; mandatory for any true distributed system.
- **CP System** — chooses consistency over availability during partitions; refuses requests to prevent stale reads (e.g., HBase, MongoDB).
- **AP System** — chooses availability over consistency during partitions; may return stale data (e.g., Cassandra, DynamoDB).
- **CA System** — consistency + availability, no partition tolerance; only feasible in non-distributed single-node systems (e.g., RDBMS).
- **Read Repair** — on-read correction of stale replicas in eventually consistent systems.
- **Anti-Entropy** — background reconciliation of replicas using Merkle Trees.
- **Gossip Protocol** — nodes randomly propagate updates to neighbors; information spreads virally.
- **Quorum (R + W > N)** — majority-based agreement rule for reads and writes to enforce consistency.
- **Vector Clock** — logical timestamp vector tracking causality across replicas for conflict detection.
- **CRDT** — Conflict-free Replicated Data Type; merges concurrent writes mathematically without coordination.
- **SPOF (Single Point of Failure)** — a component whose failure takes down the entire system; to be eliminated via redundancy.
- **MTBF** — Mean Time Between Failures; measures how long a system runs without failing.
- **MTTR** — Mean Time To Repair; measures how quickly the system recovers after failure.
- **Redundancy** — duplicating components so failure of one does not bring down the system.
- **Load Balancing** — distributing incoming requests across servers to prevent overload.
- **Failover** — automated switching to a standby system when the primary fails.
- **Maintainability** — ease and cost-effectiveness of updating and repairing a system over its lifetime.
- **Reliability** — probability of failure-free operation over a specific timeframe.
- **Fault Tolerance** — ability to continue operating despite component failures (graceful degradation).
- **Cyclomatic Complexity** — metric counting the number of linearly independent paths through code.
- **Code Churn** — frequency of changes to a module; high churn may indicate instability.
- **Active Replication** — both primary and replica process requests concurrently; highest fault tolerance, highest overhead.
- **Shadowing (Passive Replication)** — replica stays idle and activates only on primary failure.
- **RAID** — Redundant Array of Independent Disks; distributes data across disks to survive drive failure.
- **Microservices Architecture** — decomposed services that isolate faults so one failing service cannot cascade.

---

## Exam targets

1. **Define availability and apply the formula**: given a downtime figure, calculate availability %; given a target %, calculate max allowable downtime in minutes.
2. **Distinguish System Availability from Asset Reliability**: a system can be available even when a component is unreliable, as long as redundancy masks the failure.
3. **Explain the three-pillar engineering toolkit**: Prevention & Architecture (redundancy, scalability), Traffic Management (load balancing), Recovery & Response (failover, DR, monitoring).
4. **Describe the Vigilance Loop**: DETECT → ALERT → ACT → repeat; explain why monitoring is the proactive layer.
5. **Explain the consistency spectrum**: place Strong, Causal, Eventual, and Weak on the spectrum with their latency/accuracy trade-offs and a real-world example for each.
6. **State the CAP theorem precisely**: Consistency, Availability, Partition Tolerance — pick any two. Explain why P is mandatory and therefore the real choice is CP vs AP.
7. **Compare CP and AP systems**: mechanism, behavior during partition, when to use each, and two technology examples each.
8. **Explain the CA system** and why it cannot exist in a truly distributed context.
9. **Describe mechanisms of strong consistency**: synchronous replication, quorum (R + W > N), strict 2PL, serializability — and their costs (latency, reduced availability).
10. **Describe mechanisms of eventual consistency**: read repair, anti-entropy, gossip protocols; draw the three-node healing diagram.
11. **Compare conflict resolution strategies**: Last-Writer-Wins vs. Vector Clocks vs. CRDTs — trade-offs in simplicity vs. safety.
12. **State the three non-functional pillars**: Maintainability (foundation), Reliability (promise), Fault Tolerance (safety net) — define each.
13. **Apply reliability metrics**: calculate MTBF, MTTR, uptime %, error rate given raw numbers; explain what each metric signals.
14. **Compare four replication strategies**: full, partial, shadowing, active — pros and cons of each.
15. **Explain SPOF and elimination**: what it is, why it is dangerous, and how redundancy plus load balancing eliminates it.
16. **Describe the iron triangle of resilience**: scalability, performance, and cost cannot all be optimized simultaneously.

---

## Pitfalls

- **CAP: CA does not exist in distributed systems.** CA systems are single-node monoliths. In any multi-node system you must design for P; the only real choice is CP vs AP. Students often misidentify RDBMS as CA in a distributed context.
- **Availability is not the same as Fault Tolerance.** Availability maximizes uptime (% metric, standard redundancy). Fault Tolerance prevents outages entirely (graceful degradation, high degree of redundancy, RAID/replication). A system can be highly available without being fault-tolerant.
- **Reliability is not the same as Availability.** Reliability is about a component's failure probability (MTBF); Availability is about the system's uptime percentage accounting for recovery (MTTR). A reliable component still fails sometimes; a highly available system can absorb those failures.
- **Eventual consistency does NOT mean "no consistency."** It guarantees convergence; it just does not guarantee when. Weak consistency is the one that makes no timing guarantee.
- **Strong consistency does not guarantee zero downtime.** CP systems sacrifice availability during partitions; they can legitimately refuse requests.
- **Read-Your-Writes is not global strong consistency.** It only guarantees a single user sees their own writes immediately; other users may still see stale data.
- **The CAP theorem applies specifically to network partitions.** The PACELC theorem (not in this lecture but useful context) extends CAP to cover latency vs. consistency trade-offs even when there is no partition.
- **Synchronous replication = zero data loss but maximum latency.** Do not confuse it with active replication (concurrent processing, different use case).
- **MTBF measures the interval between failures, not the frequency.** Higher MTBF is better (fewer failures over time). MTTR measures recovery speed; lower is better.
- **Code churn is measured via VCS history**, not static analysis. It is a dynamic metric that reveals which modules are unstable over time.
- **Shadowing (passive replication) has slower activation than active replication.** The shadow is idle and must warm up on failover; this delay can be significant.
