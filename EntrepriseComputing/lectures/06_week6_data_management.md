# Week 6 — Enterprise Data Management

## Bird's eye view

- The **Crisis of Scale** (Volume, Velocity, Variety) broke the single-node relational model; the response was **Polyglot Persistence** — using specialized storage engines rather than one monolithic database.
- The fundamental tension is **ACID vs. BASE**: relational systems give strict serializability; NoSQL systems trade consistency for horizontal elasticity and availability.
- Four NoSQL families serve distinct access patterns: **Key-Value** (O(1) lookup), **Document** (hierarchical object locality), **Wide-Column** (extreme write velocity via LSM-Trees), **Graph** (index-free adjacency for multi-hop traversals).
- **Sharding** distributes data horizontally: Key-Based (hash) prevents hot spots but destroys locality; Range-Based preserves order but risks skew; Directory-Based is flexible but introduces a SPOF routing service.
- Analytical architectures split into **OLTP** (row-oriented, B-Tree, low latency) and **OLAP** (column-oriented, MPP, massive aggregations), bridged by **ETL/ELT** pipelines; the evolution is Data Warehouse → Data Lake → **Data Lakehouse** (Apache Iceberg/Delta on cheap object storage).
- **Polyglot Persistence** means system complexity migrates entirely from the database engine to the **application integration layer**.

---

## Detailed notes

### 1. The Crisis of Scale: Why Relational Systems Hit a Wall

The Web 2.0 era introduced a "Crisis of Scale" across three dimensions:

| Dimension | Problem |
|---|---|
| **Volume** | Petabyte-scale data exceeds single-machine SAN/NAS shared-memory architectures. |
| **Velocity** | Millions of IOPS saturate CPU and disk I/O bottlenecks. |
| **Variety** | Unstructured payloads (JSON, graphs) expose the rigidity of normalized relational schemas. |

The core architectural tension: RDBMS enforce strict ACID compliance (required for deterministic consistency) but cannot scale horizontally. When hardware physics bound a single node, architectures must transition to **shared-nothing distributed networks**.

The **Object-Relational Mismatch** (impedance mismatch — a term borrowed from electronics) means developer productivity suffers when the application's data structures do not match the storage model.

---

### 2. The Relational Model and the ACID/BASE Dichotomy

#### 2.1 ACID (Relational/RDBMS)

The physical storage engine uses:
- **Schema-on-Write**: strict normalization eliminates duplication via Foreign Keys.
- **B-Tree Storage**: on-disk structures optimized for high-concurrency read/writes and range scans — O(log n) lookup.

The ACID guarantees:

| Property | Meaning |
|---|---|
| **Atomicity** | All-or-nothing. A transaction either fully commits or fully rolls back. |
| **Consistency** | The database moves from one valid state to another, enforcing all constraints. |
| **Isolation** | Concurrent transactions appear serial. Implemented via Two-Phase Locking (2PL) and MVCC. |
| **Durability** | Committed data survives crashes via write-ahead logging (WAL). |

Example atomicity (bank transfer):

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
COMMIT;
```

Key takeaway: **The relational model sacrifices horizontal elasticity to mathematically guarantee strict serializability and data integrity.**

#### 2.2 BASE (NoSQL)

NoSQL systems adopt the BASE philosophy to achieve global scale:
- **B**asically **A**vailable — the system guarantees availability.
- **S**oft State — the state of the system may change over time even without input.
- **E**ventual Consistency — the system will eventually become consistent.

BASE accepts transient inconsistencies in exchange for the high uptime required by distributed web services. Concurrent conflicts are resolved via **vector clocks** or **last-write-wins (LWW)** rather than blocking/locking.

#### 2.3 Schema-on-Write vs. Schema-on-Read

Analogous to static vs. dynamic type checking in programming:

| Approach | Behavior | Trade-off |
|---|---|---|
| **Schema-on-Write** (Relational) | Structure enforced before commit. | Safe, but slow migrations. |
| **Schema-on-Read** (Document) | Structure implicit; interpreted at query time. | Flexible for heterogeneous data; integrity responsibility shifts to the application. |

#### 2.4 CAP Theorem: NoSQL's Design Space

The ACID vs. BASE Diagnostic Matrix maps to CAP: NoSQL focuses on **Availability + Partition Tolerance**, sacrificing strict consistency. This is an engineering concession — trading absolute mathematical consistency for limitless horizontal scalability and high availability.

---

### 3. Case Study — X (Twitter) Fan-Out Problem

The Twitter case illustrates the ACID/BASE trade-off in practice.

#### Approach 1: Join-on-Read (Relational)

Home timelines assembled at query time by joining the `follows` and `tweets` tables:

```sql
SELECT * FROM tweets
JOIN follows ON tweets.userid = follows.followeeid
WHERE follows.followerid = [CurrentUser]
ORDER BY createdat DESC LIMIT 20
```

- **Pro**: highly data-efficient; deleting a tweet instantly removes it from all feeds.
- **Breaking Point**: celebrity accounts (Obama, Ellen DeGeneres) with millions of followers made this join millions of times per second — computationally impossible. Read latency spiked → the "Fail Whale."

#### Approach 2: Fan-Out-on-Write (Pre-computation)

Twitter moved the heavy lifting from the **Read** phase to the **Write** phase:
- A background Fan-out daemon looks up all followers of a posting user.
- Inserts the Tweet ID into each follower's personal "Timeline" cache (in-memory stores like Redis via `LRANGE`).
- On login: zero joins — just a direct read from the personal cache.

**Hybrid in production**: standard users get fan-out-on-write; celebrities with millions of followers revert to join-on-read to avoid write-multiplying a single tweet to 30 million caches.

---

### 4. NoSQL Taxonomy: The Four Families

#### 4.1 Key-Value Stores (Redis, DynamoDB)

- Backed by a **hash table** — an input string (e.g., `session:12345`) is passed through a hash function (e.g., MD5), returning a memory address pointer.
- **O(1) access** for reads and writes.
- Ideal for: high-speed caching, ephemeral session state, leaderboards.
- Values are opaque blobs — the store cannot query inside the value.

Diagram summary: Input String → MD5 Hash Function → Memory Address Pointer → Value in Memory Blocks.

#### 4.2 Document Stores (MongoDB)

- Stores semi-structured BSON/JSON documents — a single physical object containing nested arrays and sub-documents.
- Retrieves the entire data tree in **one disk seek**, avoiding expensive relational joins.
- Schema-on-Read: structure is interpreted at query time.
- Ideal for: polymorphic objects, e-commerce catalogs with heterogeneous attributes.

Example document structure: `User {"_id": 1, "name": "Alice"}` with embedded `Orders: [{order_id: 101, total: 50}, ...]` and `Preferences: [{theme: dark, ...}]` all in one object.

Both KV and Document models **eliminate complex relational joins** by prioritizing lookup speed and structural data locality.

#### 4.3 Wide-Column Stores (Cassandra, HBase)

- Optimized for **extreme write-velocity** and time-series data.
- Physical storage uses **Log-Structured Merge-Trees (LSM-Trees)**:
  1. Incoming writes go to an **in-memory Memtable**.
  2. When the Memtable fills, it is **flushed** to disk as an immutable **SSTable** (Sorted String Table).
  3. Sequential disk writes avoid B-Tree fragmentation entirely.
- Null columns consume zero space — rows can have completely different columns.
- Ideal for: IoT sensor streams, time-series, write-heavy workloads.

Choose Wide-Column for **infinite write scalability**.

#### 4.4 Graph Databases (Neo4j)

- The **Property Graph Model**: Vertices (entities) + Edges (relationships, which are first-class citizens with properties).
- **Index-Free Adjacency**: physical edges store direct memory pointers to adjacent nodes — traversal is O(1) per hop regardless of dataset size.
- Contrast with the legacy **CODASYL / Network Model** (1970s), which forced the "Programmer as Navigator" to manually track Access Paths defined at insert time. Modern graph DBs use declarative query languages (Cypher) and let the optimizer determine paths at query time.

Why relational struggles with graph data — traversing variable-length paths (e.g., "people who emigrated from US to Europe via City → State → Country → Continent") requires `WITH RECURSIVE` SQL (often 25+ lines, difficult for the optimizer to handle).

Choose Graph when relationships are **many-to-many** and access paths are **unpredictable**.

---

### 5. Scaling — Partitioning and Sharding

When vertical scaling (adding RAM/CPU to a single node) hits its physical or economic limit, the architecture must adopt a **Shared-Nothing** approach: distributing data across a cluster of independent nodes.

#### 5.1 Core Formula: Algorithmic (Key-Based / Hash) Sharding

```
Shard_ID = Hash(Shard_Key) mod n
```

- **Shard Key**: the attribute used to determine distribution (e.g., `User_ID`).
- **Hash Function**: maps the Shard Key to a large integer, ensuring uniform distribution and preventing data skew.
- **n**: total number of logical shards/partitions available in the system.
- **Modulo**: ensures the Shard_ID is always an integer between 0 and n-1.

Practical example (n = 4 shards): Shard Key = `105` → `Hash(105)` = 8294 → `8294 mod 4` = 2 → route to Shard 2.

Scalability note: if n changes (adding a server), almost every mapping changes → **massive resharding** event. Solution: **Consistent Hashing** to minimize data migration.

#### 5.2 Sharding Strategies Compared

| Strategy | Logic | Pros | Cons |
|---|---|---|---|
| **Key-Based (Hash)** | `Hash(key) mod n` | Uniform distribution; no hot spots. | Adding/removing nodes requires moving almost all data. |
| **Range-Based** | Assign ranges (e.g., A-M, N-Z or date ranges) | Efficient for range queries (e.g., all records from March). | Hot spots if activity concentrates (e.g., current date range). |
| **Directory-Based** | Centralized metadata router maps keys to shard addresses. | Maximum flexibility; move partitions without code changes. | The directory is a SPOF and a latency bottleneck. |

Diagram — Directory-Based: Application Tier → sends `GET User_105` → Metadata Router (ZooKeeper/Gossip) holds [Logical Keys ↔ Physical Address] table → routes to Shard 4. The router has a SPOF / Latency Bottleneck warning.

Diagram — Sharded Cluster (4 nodes, replication factor 3): each partition has one Leader and two Followers spread across nodes. Replication streams connect Leaders to Followers across the cluster. Writing to partition 4 hits Node 4's Partition 4 Leader, which replicates to followers on other nodes.

**Architectural trade-off: The Resharding Penalty.** Key-Based sharding prevents hot spots but introduces a significant maintenance burden when n changes. Directory-Based allows live rebalancing but demands highly available, consensus-driven routing infrastructure.

---

### 6. Analytical Architectures: OLTP vs. OLAP

Enterprises must physically separate **operational** systems from **analytical** systems because their storage layouts are fundamentally opposed.

| Dimension | OLTP | OLAP |
|---|---|---|
| **Full name** | Online Transaction Processing | Online Analytical Processing |
| **Purpose** | High concurrency, low latency, point lookups | Massive dataset scans, complex aggregations, high throughput |
| **Storage layout** | Row-oriented (optimizes writing complete records sequentially) | Column-oriented (optimizes CPU cache lines and massive aggregations via run-length compression) |
| **Storage engine** | B-Tree optimized | Massively Parallel Processing (MPP) |
| **Typical query** | `SELECT * FROM users WHERE id = 1024` | `SELECT SUM(revenue) FROM sales GROUP BY region` |

Key rule: **Never run complex analytical aggregations on a production OLTP database** — the physical storage layouts are fundamentally opposed.

---

### 7. The Analytical Storage Evolution: Warehouse → Lake → Lakehouse

#### 7.1 Data Warehouse ("The Library")

- **Schema-on-Write**: data is cleaned, transformed, and structured before entry.
- Uses **Star Schema** or **Snowflake Schema**:
  - **Fact Table**: central table holding quantitative measurements (e.g., sales amounts).
  - **Dimension Tables**: descriptive context surrounding the fact (e.g., time, product, geography).
- Answers known, pre-defined business questions reliably.
- Guarantees data quality at the expense of agility.

Diagram — Star Schema: a central Fact Table (e.g., sales) is surrounded by Dimension Tables (time, product, customer, geography). Each dimension table connects to the fact table via a foreign key.

#### 7.2 Data Lake ("The Reservoir")

- **Schema-on-Read**: raw data ingested as-is (CSV, JSON, Parquet, JPG, logs) into cheap object storage (S3/blob).
- No structure enforced at ingest — the schema is applied only when a query engine reads the data.
- Stores data for "questions not yet asked" — ideal for Data Science and ML workloads.
- Risk: without governance, a Data Lake degrades into an **unqueryable data swamp**.

#### 7.3 Data Lakehouse ("The Synthesis") — e.g., Databricks

The Lakehouse unifies both models:

```
Cheap Object Storage (S3/Schema-on-Read)
  + Metadata Layer (Apache Iceberg / Delta Lake)
  = Data Lakehouse
```

- Provides ACID transactions and Warehouse-level performance on top of cheap Lake storage.
- **Apache Iceberg / Delta Lake**: open table formats that bring strict snapshot isolation to raw Parquet files, tracking file changes with ACID guarantees.
- **Separates compute from storage** entirely — multiple query engines can access the same data.
- Avoids duplicating data across two separate systems.

Key takeaway: **The Lakehouse is the modern standard**, stitching isolated analytical pipelines into a single, high-performance platform.

---

### 8. ETL vs. ELT: Data Integration Paradigms

The shift from on-premise to cloud changed where transformations happen.

#### ETL (Extract, Transform, Load) — Traditional

Flow: `Source DB → Staging Server (Compute Engine) → Data Warehouse`

- Heavy transformations execute in an **intermediary network tier** before loading.
- The staging server becomes a bottleneck.
- Moving the **data to the code**.

#### ELT (Extract, Load, Transform) — Cloud Era

Flow: `Source DB → Cloud Warehouse MPP (Compute Engine inside)`

- Raw data is loaded instantly into a massive distributed system (e.g., BigQuery, Snowflake, Redshift).
- Transformations run **in-situ** using the warehouse's own MPP compute power, leveraging Schema-on-Read.
- Analysts use SQL to transform data using the database's own compute rather than a separate staging area.
- Moving the **code to the data**.

Key takeaway: ELT is dominant in cloud architectures because it moves the compute burden to where the data already lives.

---

### 9. Polyglot Persistence: The Architectural Paradigm

Modern enterprise systems do not use one database for everything — they combine specialized engines:

Diagram — E-Commerce Polyglot Architecture:
```
               API Gateway (E-Commerce Gateway)
              /              |               \
{financial_txn,      {session_token,    (User)-[:BOUGHT]->(Product)
 ACID_strict}          ttl: 3600}
      |                     |                    |
  PostgreSQL              Redis                Neo4j
(Relational DB:       (Key-Value Store:    (Graph DB: Multi-hop
 strict financial      ephemeral state,     relational state and
 state/normalization)  ultra-low latency)   recommendation engines)
```

**Complexity Shift**: system complexity moves from the database engine entirely to the **application integration layer**. Engineers must now maintain consistency across multiple heterogeneous stores in application code.

Design heuristics for engine selection:
- **RDBMS**: financial integrity and complex, multi-object transactions.
- **Document/Key-Value**: e-commerce catalogs with heterogeneous attributes or high-speed session states.
- **Graph**: when access paths are unpredictable and relationships are many-to-many.
- **Wide-Column**: time-series, IoT, extremely high write velocity.
- **Analytical Dual-Track**: Data Warehouses for known structured questions; Data Lakes to preserve raw data for unknown future questions.

---

### 10. System Design Heuristics (Practical Application)

Three dimensions for enterprise system design decisions:

| Dimension | Heuristic |
|---|---|
| **Performance** | Scale horizontally (NoSQL/Sharding) when throughput and volume exceed single-node physical limits. Optimize storage layout for read/write ratios. |
| **Integrity** | Retain RDBMS/ACID models when strict serializability and financial state are absolute non-negotiables. |
| **Synthesis** | Modern backends require Polyglot Persistence for operations, and ELT Lakehouses for analytics. Separate compute from storage. |

**Fan-out Load Analysis**: if one write triggers 10,000+ updates (e.g., timeline caches), a hybrid read-time/write-time strategy is required.

**Sharding Thresholds**: only implement sharding when load parameters (requests per second or total volume) exceed the capacity of high-end vertical nodes. Factor in the Resharding Penalty before choosing a Hash-based strategy.

---

## Key terms

- **ACID** — Atomicity, Consistency, Isolation, Durability: the transaction guarantees of relational databases.
- **BASE** — Basically Available, Soft State, Eventual Consistency: the philosophy of NoSQL systems.
- **B-Tree** — on-disk index structure used by RDBMS; O(log n) lookup; good for range scans and high-concurrency reads/writes.
- **LSM-Tree (Log-Structured Merge-Tree)** — write-optimized storage structure used by Wide-Column stores; writes go to in-memory Memtable, flushed as immutable SSTables.
- **CAP Theorem** — a distributed system can guarantee at most two of: Consistency, Availability, Partition Tolerance. NoSQL systems favor AP.
- **Polyglot Persistence** — the practice of using multiple specialized storage engines within one application, each matched to its access pattern.
- **Sharding** — horizontal partitioning of a database across multiple independent nodes (shared-nothing architecture).
- **Shard Key** — the attribute used to determine which shard a record belongs to.
- **Consistent Hashing** — a sharding technique that minimizes data movement when the number of nodes changes.
- **Hot Spot** — a shard that receives a disproportionate share of read/write load, typically from range-based partitioning.
- **Resharding** — the process of redistributing data across shards after the cluster topology changes; expensive with hash-based sharding.
- **Index-Free Adjacency** — a Graph DB property (e.g., Neo4j) where edges store direct memory pointers to adjacent nodes, enabling O(1) hop traversal.
- **CODASYL / Network Model** — 1970s pre-relational model that required the programmer to manually navigate pre-defined access paths.
- **Object-Relational Mismatch (Impedance Mismatch)** — the friction between an application's object model and a relational storage model.
- **OLTP** — Online Transaction Processing: high-concurrency, low-latency, row-oriented, B-Tree storage.
- **OLAP** — Online Analytical Processing: massive aggregations, column-oriented storage, MPP engines.
- **ETL** — Extract, Transform, Load: classic pipeline where transformation happens in a staging server before loading.
- **ELT** — Extract, Load, Transform: cloud-era pipeline where raw data is loaded first, then transformed in-situ using MPP compute.
- **Star Schema** — data warehouse schema with a central Fact Table surrounded by Dimension Tables.
- **Snowflake Schema** — a normalized extension of star schema where dimension tables are further normalized.
- **Fact Table** — holds quantitative measurements (metrics/events) in a data warehouse.
- **Dimension Table** — holds descriptive context (time, product, geography) surrounding facts.
- **Data Warehouse** — Schema-on-Write analytical store; answers known business questions reliably.
- **Data Lake** — Schema-on-Read raw object storage; preserves data for future unknown questions.
- **Data Lakehouse** — unifies Lake storage (cheap S3) with Warehouse reliability via metadata layers (Apache Iceberg, Delta Lake).
- **Apache Iceberg / Delta Lake** — open table formats that bring ACID transactions and snapshot isolation to raw Parquet files on object storage.
- **MPP (Massively Parallel Processing)** — distributed compute model used by cloud warehouses (BigQuery, Snowflake) to parallelize analytical queries across many nodes.
- **Fan-out** — the write multiplication problem: one write triggering N downstream updates (e.g., celebrity tweet to millions of follower caches).
- **Schema-on-Write** — structure enforced at data entry time; analogous to static typing.
- **Schema-on-Read** — structure applied at query time; analogous to dynamic typing.
- **WAL (Write-Ahead Log)** — the mechanism that ensures durability in RDBMS; changes are logged before being applied.
- **Two-Phase Locking (2PL)** — the concurrency control protocol that guarantees isolation in RDBMS by acquiring all locks before releasing any.
- **MVCC (Multi-Version Concurrency Control)** — allows readers and writers to proceed without blocking each other by maintaining multiple versions of data.
- **LWW (Last-Write-Wins)** — a conflict resolution strategy in NoSQL where the most recent write takes precedence.
- **Vector Clocks** — a mechanism for tracking causality and resolving conflicts in distributed systems without a central coordinator.

---

## Exam targets

1. **Explain the ACID/BASE dichotomy** and when each is appropriate. Be able to write the four ACID properties with their implementation mechanisms (2PL, MVCC, WAL).
2. **Compare the four NoSQL families** (KV, Document, Wide-Column, Graph): storage mechanism, asymptotic access complexity, ideal use case, and a real-world example system for each.
3. **Derive the sharding formula** `Shard_ID = Hash(Key) mod n`. Given a worked example, trace the request to the correct shard. Explain what happens when n changes.
4. **Compare all three sharding strategies** (Hash, Range, Directory): pros, cons, hot spot risk, and SPOF risk.
5. **Explain the Fan-Out problem** using the Twitter case study. Describe both approaches (Join-on-Read vs. Fan-out-on-Write) and why a hybrid is used for celebrity accounts.
6. **Contrast OLTP and OLAP** along storage layout, engine type, and query type. Explain why running OLAP queries on OLTP systems is harmful.
7. **Trace data through ETL and ELT pipelines**, labeling where compute happens in each and why ELT dominates in cloud environments.
8. **Compare Data Warehouse, Data Lake, and Data Lakehouse**: schema philosophy, strengths, failure modes, and which metadata formats enable the Lakehouse.
9. **Design a polyglot persistence architecture** for a given system (e.g., e-commerce, ride-sharing): identify which component needs RDBMS, which needs KV, which needs Graph, and why.
10. **Explain the Graph Model vs. CODASYL**: why is index-free adjacency superior to pre-defined access paths for unpredictable traversals?

---

## Pitfalls

- **Confusing Schema-on-Write with Schema-on-Read**: Write = relational (structure enforced before storage); Read = NoSQL document (structure applied at query time). Wrong direction is a common error.
- **Assuming NoSQL is always faster than SQL**: KV/Document are faster for their specific access patterns; OLAP column stores are faster for analytics. For complex multi-table joins, a well-indexed RDBMS often wins.
- **Hash sharding destroys data locality**: unlike range sharding, Hash sharding scatters consecutive keys across different shards. This means range queries (`WHERE date BETWEEN ...`) require a scatter-gather to all shards — extremely expensive.
- **The Resharding Penalty is catastrophic**: changing n in `Hash(key) mod n` invalidates almost every existing mapping. This is not a minor rebalancing — it is a full data migration. This is why Consistent Hashing exists.
- **Data Lakes become data swamps**: without governance and schema enforcement at query time, a Data Lake becomes unqueryable. The antidote is the Lakehouse metadata layer (Iceberg/Delta).
- **ETL vs. ELT confusion**: in ETL, the "T" happens in the staging server (before loading). In ELT, the "T" happens inside the cloud warehouse after loading. The key shift is where compute lives.
- **The CODASYL trap**: in the Network Model, if you didn't pre-define an access path at insert time, you couldn't query that relationship. Modern graph databases solve this with declarative Cypher queries — the optimizer finds the path at query time.
- **Directory-Based Sharding SPOF**: the metadata router (ZooKeeper/Gossip) is both a SPOF and a latency bottleneck. Every request must pass through it. This must be made highly available with consensus protocols.
- **Fan-out arithmetic**: 1 celebrity tweet × 30 million followers = 30 million cache writes. Never fan-out for users whose follower count causes write-multiplication beyond system capacity.
- **OLTP/OLAP mixing**: running a `GROUP BY` aggregation over millions of rows on a production OLTP database will lock rows and kill concurrency for transactional users. Always route analytics to a dedicated OLAP system.
- **BASE is not weaker than ACID — it is a different contract**: BASE systems are engineered to be highly available and partition-tolerant; they do not "fail" at consistency, they deliberately deprioritize it in exchange for uptime guarantees that ACID systems cannot provide at global scale.
