# Chapter 2 — Big Data Ecosystem & Technologies

## Bird's eye view

- **RDBMS can't handle big data** because it scales vertically, is schema-rigid, batch-oriented, and expensive at PB scale.
- Two consistency models govern data systems: **ACID** (strict, RDBMS) vs **BASE** (eventual, NoSQL).
- **CAP theorem** (Brewer, 2000): a distributed system can guarantee only **2 of 3** — Consistency, Availability, Partition tolerance.
- The big data solution is **distributed storage + distributed processing** on clusters of commodity hardware.
- **Hadoop** = open-source framework for distributed storage (HDFS) + processing (MapReduce, later YARN), inspired by Google's GFS + MapReduce papers (Doug Cutting, 2005).
- Hadoop has **4 core modules**: MapReduce, YARN, HDFS, Common.
- The **Hadoop ecosystem** = a constellation of tools layered on these cores: Pig, Hive, HBase, Sqoop, Flume, Oozie, Mahout, Zookeeper, Ambari, etc.

---

## 1. Big Data with Traditional Data Storage

### 1.1. Classical RDBMS recap

RDBMS = software that stores and manages structured data in tables (rows + columns).

Core properties:
- **Schema**: predefined structures and relationships
- Enforces the **ACID** transaction properties (below)
- Strengths: structured data with clear relationships, reliability, integrity, widespread use

### 1.2. Why RDBMS struggles with big data

| Issue | Detail |
|---|---|
| **Scalability** | Relies on vertical scaling (more power per server); big data needs horizontal scaling (more servers). |
| **Variety limitations** | Optimized for structured data with predefined schemas — can't handle semi/unstructured well. |
| **Velocity** | Batch-oriented; struggles with high-velocity real-time streams (IoT, social). |
| **Cost** | Managing petabytes is prohibitively expensive and operationally complex. |

### 1.3. ACID properties

| Letter | Meaning |
|---|---|
| **A — Atomicity** | A transaction is all-or-nothing: either all changes commit or none do. |
| **C — Consistency** | Database moves from one valid state to another, enforcing constraints/integrity. |
| **I — Isolation** | Concurrent transactions don't interfere — no anomalies. |
| **D — Durability** | Once committed, changes survive system failures permanently. |

ACID trade-offs:
- ✅ Reliable, consistent data; data integrity; complex transactions; strong protection
- ❌ Performance overhead; limited scalability

### 1.4. The big data challenges (the "3 Too's")

| Problem axis | Challenges |
|---|---|
| **Too Big — Scalability** | Exponential data growth; seamless horizontal scaling |
| **Too Big — Infrastructure** | High infra costs; HA + fault tolerance |
| **Too Complex — Privacy** | Compliance with regulations; protecting personal data |
| **Too Complex — Quality** | Noisy/incomplete/duplicate data; consistency across distributed sets |
| **Too Fast** | Real-time velocity overwhelms RDBMS |

Grace Hopper quote (slide motif): *"In pioneer days they used oxen for heavy pulling. When one ox couldn't budge a log, they didn't grow a larger ox — they used more oxen. We shouldn't try for bigger computers, but for more systems of computers."* → distributed > vertical.

---

## 2. Distributed Data Storage as the Solution

### 2.1. CAP Theorem (Brewer, 2000)

A distributed system cannot simultaneously provide all three of:

| Property | Meaning |
|---|---|
| **C — Consistency** | All nodes see the same data at all times. |
| **A — Availability** | System is operational; handles read/write even if some nodes are down. |
| **P — Partition tolerance** | System continues despite network partitions (lost messages between parts of the system). |

→ **Can only guarantee two of three** at any given time. In practice, partitions happen, so the trade-off is usually CP vs AP.

### 2.2. BASE Properties (the NoSQL counterpart)

| Letter | Meaning |
|---|---|
| **BA — Basically Available** | DB is concurrently accessible to users; no waiting for others to finish. |
| **S — Soft state** | Data can be in transient states that change over time without external triggers. |
| **E — Eventually consistent** | Once all concurrent updates complete, all queries see the same value. |

### 2.3. ACID vs BASE — comparison

| Dimension | ACID | BASE |
|---|---|---|
| **Scale** | Vertically | Horizontally |
| **Flexibility** | Less (locks records) | More (concurrent updates allowed) |
| **Performance** | Degrades on large volumes | High throughput on large unstructured data |
| **Synchronization** | Yes (adds delay) | No DB-level sync |

---

## 3. The Big Data Ecosystem

**Big Data Ecosystem** = software specifically designed to **analyze, process, and extract** information from complex datasets.

### 3.1. Apache Hadoop

- An **open-source framework** allowing distributed processing of large datasets across clusters of **commodity hardware**.
- Written in **Java**.
- Inspired by Google's **MapReduce** programming model + **GFS** (Google File System).
- Originally developed by **Doug Cutting** (with Mike Cafarella) for the **Nutch** search engine project (2005); named after his son's stuffed yellow elephant toy. Cutting became Chief Architect of Cloudera.

Four key terms in its description:
1. **Open Source** — source freely available, redistributable, modifiable
2. **Distributed Processing** — data processed independently across nodes
3. **Cluster** — multiple machines connected via LAN
4. **Commodity Hardware** — affordable, low-performance servers

### 3.2. Hadoop history (timeline)

| Year | Event |
|---|---|
| 2002 | Nutch Project started (Cutting + Cafarella) |
| 2003 | Google releases GFS white paper |
| 2004 | Google releases MapReduce paper |
| Mid-2004 | Nutch implements NDFS + MapReduce |
| 2006 | Hadoop development starts (sub-project of Nutch) |
| 2007 | Yahoo starts using Hadoop on 1000-node cluster |
| 2008 | Hadoop becomes top-level Apache project |
| Mid-2008 | Hadoop defeats supercomputers (sorts 1TB faster) |
| 2011 | First stable release Hadoop 1.0 |
| 2012 | Hadoop 2.0 with YARN |
| 2017 | Hadoop 3.0 released |
| 2020 | Hadoop 3.1.13 |

### 3.3. Hadoop characteristics

- **Open Source**
- **Distributed Processing**
- **Fault Tolerance** — failures of nodes/tasks recovered automatically
- **Reliability** — data reliably stored despite machine failures
- **High Availability** — data accessible despite hardware failure
- **Scalability** — vertical (HW) + horizontal (new nodes)
- **Economic** — no costly licenses or hardware
- **Easy to Use** — framework hides distributed-computing complexity

### 3.4. Hadoop architecture — 4 core modules

| Module | Role |
|---|---|
| **MapReduce** | Programming model for large-scale data processing |
| **YARN** | Resource-management platform (compute resources across cluster) |
| **HDFS** | Distributed file system providing high-throughput data access |
| **Common Libraries / Utilities** | Shared libraries and utilities used by all modules |

Underlying assumption: **hardware failures are common** and must be handled automatically in software.

### 3.5. Hadoop ecosystem (tools layered on the core)

The full ecosystem stack typically looks like:

| Layer | Tools |
|---|---|
| **Management** | **Ambari** — provisioning, managing, monitoring clusters |
| **Coordination** | **Zookeeper** |
| **Workflow** | **Oozie** |
| **Data exchange** | **Sqoop** (RDBMS ↔ Hadoop), **Flume** (log collector) |
| **High-level processing** | **Pig** (scripting), **Hive** (SQL on Hadoop) |
| **Machine learning** | **Mahout** |
| **Statistics** | **R Connectors** |
| **NoSQL store** | **HBase** (columnar) |
| **Processing engines** | **YARN + MapReduce v2** |
| **Storage** | **HDFS** (foundation) |

---

## Key terms (glossary)

- **Commodity hardware** — affordable, off-the-shelf servers (not specialized).
- **Horizontal scaling** — adding more machines to the cluster.
- **Vertical scaling** — adding more power to a single machine.
- **Fault tolerance** — ability to keep working despite component failures.
- **Cluster** — group of machines working together.
- **NoSQL** — non-relational databases that typically use BASE rather than ACID (e.g., MongoDB, Cassandra).

---

## Exam targets

1. **Why RDBMS doesn't work for big data** — list 4 reasons (scalability, variety, velocity, cost).
2. **State and explain ACID** — 4 properties with definitions.
3. **State and explain CAP theorem** — 3 properties and the trade-off; *give an example* of a system that picks each pair (CP, AP, etc.).
4. **Compare ACID vs BASE** — produce the comparison table.
5. **Describe Hadoop's 4 core modules** and their roles.
6. **Why is fault tolerance a fundamental design assumption** in Hadoop? (Because at cluster scale, hardware failure is the norm.)
7. **Sketch the Hadoop ecosystem** — name 5+ tools and their roles.

### Pitfalls
- **CAP is about distributed systems**, not RDBMS. RDBMS is typically CA but single-node.
- BASE ≠ "no consistency" — it's *eventual* consistency.
- Hadoop is not a database — it's a framework. HDFS is its storage layer.
- "Commodity hardware" doesn't mean *bad* hardware — it means *standard*.
