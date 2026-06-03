# Big Data Analytics & Visualization — Exam Plan (Tomorrow, 09:00)

> 8 chapters total, ~286 PDF pages, ~4,300 lines of notes.
> Tight ~10-hour budget. **All 8 chapters in scope.** Mostly conceptual/architectural — not heavy math, but be ready to draw architectures, recall formulas (MR splits, Spark transformations), and write small code snippets (HiveQL, Pig Latin, Spark Scala).

---

## Time inventory

| Block | Window | Hours |
|---|---|---|
| Today (rest of day) | now → bed | **6-7 h** |
| Tomorrow morning | wake → 08:30 | **2 h** review |
| **Total** | | **8-9 h** |

---

## Triage — what gets time

### Tier 1 — HIGH PRIORITY (deep coverage)
The architecture-heavy chapters with very examinable mechanics (workflows, formulas, code).

- **Ch 3 HDFS** (50-60 min) — architecture, replication, read/write flow
- **Ch 4 MapReduce** (60-80 min) — programming model, 9 components, design patterns, key formulas
- **Ch 7 Hive** (50 min) — HiveQL syntax, data model, ACID tables
- **Ch 8 Spark** (60-70 min) — RDD/DataFrame, narrow vs wide, lazy evaluation, architecture

**Tier 1 total: ~4 hours.**

### Tier 2 — MEDIUM (focused coverage)

- **Ch 2 Ecosystem & Technologies** (35-40 min) — ACID/BASE/CAP, Hadoop overview, ecosystem tools
- **Ch 5 YARN** (35-45 min) — 4-actor architecture, 8-step workflow, 3 schedulers
- **Ch 6 Pig** (30-40 min) — Pig Latin operators, data model, basic scripts

**Tier 2 total: ~2 hours.**

### Tier 3 — LOW (conceptual skim)

- **Ch 1 Big Data Intro** (20 min) — 7Vs, 4 analytics types, "is it big data?" test

**Tier 3 total: ~20 min.**

---

## Day-by-day schedule

### Today (6-7 h)

| Block | Time | Activity |
|---|---|---|
| **1 — 90 min** | Tier 1 start: **Ch 3 HDFS (60 min)** + **Ch 2 Ecosystem (30 min)** |
| 15 min break | |
| **2 — 90 min** | **Ch 4 MapReduce — part 1**: model + 9 components + key formulas |
| 15 min break | |
| **3 — 75 min** | **Ch 4 MapReduce — part 2**: design patterns + Word Count + Driver/Mapper/Reducer code |
| 15 min break | |
| **4 — 60-75 min** | **Ch 8 Spark — part 1**: RDD vs DataFrame, narrow vs wide, architecture |

**Sleep by 23:30.** Sleep > extra cramming.

### Tomorrow morning (2 h)

Wake 06:30. Light breakfast.

| Time | Activity |
|---|---|
| 06:45-07:30 | **Ch 8 Spark part 2** (transformations + actions, code patterns) + quick **Ch 7 Hive recap** |
| 07:30-08:00 | **Ch 5 YARN** + **Ch 6 Pig** (30 min total, skim notes + recall operators) |
| 08:00-08:25 | **Ch 1 bird's eye** (5 min) + **cross-chapter cheatsheet** below |
| 08:25-08:50 | Travel + settle |
| 09:00 | Exam |

---

## Per-chapter — what to focus on

### Ch 1 — Understanding Big Data (20 min, Tier 3)

**Concepts only — don't waste time here.**

- **7 Vs** + who/when added each:
  - **Volume, Velocity, Variety** — Laney 2001 (the original 3)
  - **Veracity** — IBM 2012
  - **Value** — Oracle 2013
  - **Variability** — 2015
  - **Visualization** — MS 2015+
- **4 analytics types** (in order): **Descriptive → Diagnostic → Predictive → Prescriptive** (what / why / what-will / what-to-do).
- **"Is it big data?" 3-question test**: (1) exhibits one+ V? (2) beyond RDBMS capacity? (3) needs specialized tools?
- Common applications: social media, IoT, e-commerce, healthcare, finance.

### Ch 2 — Ecosystem & Technologies (35-40 min, Tier 2)

**Two big examinable concepts: ACID vs BASE, CAP theorem.**

- **Why RDBMS struggles for big data** (4 reasons):
  1. Vertical-scaling only (big data needs horizontal)
  2. Schema-rigid (no semi/unstructured)
  3. Batch-oriented (no real-time)
  4. Cost prohibitive at PB scale
- **ACID** (Atomicity, Consistency, Isolation, Durability) — strict, RDBMS, scales vertically
- **CAP theorem** (Brewer 2000): a distributed system can guarantee only **2 of 3** — Consistency, Availability, Partition tolerance. **Partition always happens → real choice is CP vs AP.**
- **BASE** (Basically Available, Soft state, Eventually consistent) — NoSQL, scales horizontally
- **ACID vs BASE comparison table** — be ready to draw:
  - Scale: vertical vs horizontal
  - Flexibility: locks vs concurrent
  - Performance: degrades vs scales
  - Sync: yes vs no
- **Hadoop = 4 core modules**: MapReduce, YARN, HDFS, Common.
- Origin: Doug Cutting + Mike Cafarella, 2005; named after his son's elephant toy.
- **Ecosystem tools to know by 1-liner**: Ambari (mgmt), Sqoop (RDBMS↔Hadoop), Flume (log collector), Oozie (workflow), Pig (scripting), Hive (SQL), HBase (NoSQL columnar), Mahout (ML), Zookeeper (coordination).

### Ch 3 — HDFS (50-60 min, Tier 1) ★

**The architecture chapter. Must be able to sketch + explain read/write flows.**

#### Architecture (draw this from memory)
- **Master/slave**:
  - **NameNode (master)** — metadata only (in RAM); namespace tree + file→block mapping; FsImage + EditLog on disk
  - **DataNode (slaves)** — block storage; one per node typically
  - **Secondary NameNode** — periodic FsImage checkpointing (**NOT a failover** — common trap!)
  - **Client** — talks to NameNode for metadata, then directly to DataNodes for I/O

#### Replication & placement
- Default **replication factor = 3**
- **Rack-aware placement** (memorize the exact rule):
  - 1 replica on the writer's local rack
  - 1 replica on a different node in the same rack
  - 1 replica on a node in a different rack
- **Read selection**: closest replica (local node > local rack > remote)

#### Liveness mechanisms
- **Heartbeat** — DataNode → NameNode (alive signal)
- **BlockReport** — list of all blocks a DataNode holds, sent to NameNode
- **Safemode** — startup state, NameNode waits for BlockReports; no replication during this
- DataNode failure detected via missed heartbeats → triggers re-replication

#### Robustness
- Re-replication triggers: DataNode down, replica corrupt, disk fails, factor increased
- Data integrity via **client-side checksum** (verified on read)
- Metadata protection: multiple FsImage+EditLog copies, synchronously updated
- **NameNode is SPOF** — no auto-failover in classic HDFS (only manual recovery)

#### WORM + design
- **Write Once, Read Many** — files not modified after creation (only append/truncate)
- Optimized for: large files, throughput, batch processing
- **NOT good for**: many small files (NameNode RAM), low-latency, multi-writers, RDBMS replacement

#### Read / write flow (be ready to sketch)
- **Read**: client → NameNode (get block locations) → client → DataNodes (read blocks) → close
- **Write**: client → NameNode (create) → client writes to **pipeline of DataNodes** (D1→D2→D3) → ack returns reverse → close, NameNode marks complete

### Ch 4 — MapReduce (60-80 min, Tier 1) ★

**Densest chapter. Many formulas, design patterns, code skeletons.**

#### Programming model
- Two pure functions:
  - `map: (k1, v1) → list(k2, v2)`
  - `reduce: (k2, list(v2)) → list(v3)`
- **Shuffle & sort** between: group all values with the same intermediate key, route to one reducer
- Origin: **Ghemawat & Dean (Google, 2004)**

#### 9-component pipeline (memorize the order!)
```
Input → InputFormat → InputSplit → RecordReader → Mapper → [Combiner] → Partitioner → Shuffle/Sort → Reducer → OutputFormat → Output
```
- **InputFormat**: how to split + which RecordReader. Most common: `TextInputFormat` (default).
- **InputSplit**: logical chunk (1 per Mapper).
- **RecordReader**: bytes → KV. `LineRecordReader` (default): key=offset, value=line.
- **Mapper**: 1 instance per split.
- **Combiner**: optional "mini-reducer", runs locally after Mapper, reduces shuffle traffic.
- **Partitioner**: routes KV to reducer.
- **Shuffle/Sort**: sorts by key, transfers to reducers via RPC.
- **Reducer**: 1 invocation per unique key.
- **OutputFormat**: serialization. Default: `TextOutputFormat` (tab-separated).

#### Key formulas (must know cold)
- $\text{splitSize} = \max(\text{minSplit}, \min(\text{maxSplit}, \text{blockSize}))$
- $\text{Number of Mappers} = \dfrac{\text{total data size}}{\text{split size}}$ — e.g., 2.39 GB / 100 MB = 25
- $\text{Partition} = \text{hash(key) mod numReducers}$
- Default **#Reducers = 1**
- Mapper output stored on **local disk**, not HDFS!

#### Design patterns (5 families)
1. **Summarization** — Numerical sums, Inverted Index, Counting w/ counters
2. **Filtering** — Filter, Bloom, Top-N, Distinct
3. **Data Organization** — Hierarchical (flat→nested), Partition/Binning, Total-Order Sort, Shuffling
4. **Join** — **Reduce-Side** (general, slow), **Replicated/Map-Side** (small dataset broadcast), **Composite** (pre-partitioned + sorted), **Cartesian** (expensive)
5. **Meta** — Job Chaining (output→input), Job Merging (same input, combine jobs)

#### Job types
- Map-only (filter, conversion)
- Map + Reduce (classic)
- Map + Combiner + Reduce (local pre-aggregation)
- Multiple Mappers + Reducer (multi-source)

#### Application coding
- **Driver** (mandatory) — configure Job, set Mapper/Reducer/Partitioner classes, submit
- **Mapper** (mandatory) — `extends Mapper<K1,V1,K2,V2>`, override `map()`
- **Reducer** (optional) — `extends Reducer<K2,V2,K3,V3>`, override `reduce()`
- **Combiner** (optional) — usually same as Reducer
- For **map-only jobs**: `job.setNumReduceTasks(0)`

**Word Count** — be ready to write Mapper + Reducer pseudocode from memory.

### Ch 5 — YARN (35-45 min, Tier 2) ★

**Hadoop 2.0 architecture chapter. Sketchable.**

#### Why YARN (vs Hadoop 1)
- Hadoop 1's **JobTracker** was a bottleneck (~4000 nodes max), MR-only, static slot allocation, poor multi-tenancy
- YARN **decouples resource management from processing logic** → multiple frameworks (Spark, Tez, HBase) share the cluster

#### 4-actor architecture (memorize roles)
- **ResourceManager (RM)** — global resource arbiter
  - **Scheduler** sub-component (allocates resources)
  - **Application Manager** sub-component (accepts submissions, launches AMs)
- **NodeManager (NM)** — per-node agent, manages containers
- **ApplicationMaster (AM)** — per-application coordinator, negotiates containers
- **Container** — bundle of resources (RAM + CPU + disk) on a node

⚠️ **Pitfall**: **ApplicationMaster** (per-app, lives in a container) ≠ **Application Manager** (sub-component of RM). Students confuse these.

#### 8-step application lifecycle (memorize)
1. Client submits app to RM (`hadoop jar`)
2. RM allocates a container for the **ApplicationMaster**
3. AM registers with RM
4. AM requests containers from RM
5. AM tells NMs to launch those containers
6. Application code runs in containers
7. Client contacts RM to monitor status
8. AM unregisters when done; containers released

#### 3 schedulers
| Scheduler | Behavior | Use |
|---|---|---|
| **FIFO** | Queue, one at a time | Simple, single-tenant |
| **Capacity** | Queues per org with min guarantee + elasticity | Multi-tenant shared cluster |
| **Fair** | Pools, equal share over time, **preemption** supported | Equal-priority multi-user |

#### 4 failure types
- **RM failure** — cluster down, manual restart, jobs lose progress (RM is SPOF in classic YARN)
- **NM failure** — RM detects via missed heartbeats, kills its containers, AMs redo work
- **AM failure** — RM starts a new AM container; new AM must recover prior state (if persisted)
- **Container failure** — AM handles via application framework

#### Tools
- **YARN Web UI**: port **8088** (monitor only, can't configure)
- **Hue Job Browser**: monitor, kill, view logs
- **YARN CLI**: admin-oriented

### Ch 6 — Pig (30-40 min, Tier 2)

**Why? MR is verbose + only 2 phases. Pig Latin = dataflow scripting → compiles to MR jobs.**

#### Architecture
```
Pig Latin script → Parser → Optimizer → Compiler → Execution Engine → MR/Hadoop
```

#### Data model (nested)
**Atom → Field → Tuple → Bag → Map → Relation**
- Atom: single value (e.g., `22`, `"Ahmad"`)
- Tuple: ordered fields (`(Ahmad, 22)`) — like a row
- Bag: unordered set of tuples — like a table
- Relation: a bag (the dataset)

#### Modes
- Local · MapReduce · Spark · Tez (`pig -x local|mapreduce|spark|tez script.pig`)

#### Operator categories (must know)
| Category | Operators |
|---|---|
| **Load/Store** | `LOAD`, `STORE` |
| **Filter/Project** | `FILTER`, `DISTINCT`, `FOREACH … GENERATE`, `STREAM` |
| **Group/Join** | `GROUP`, `COGROUP`, `JOIN`, `CROSS` |
| **Sort/Limit** | `ORDER`, `LIMIT` |
| **Combine/Split** | `UNION`, `SPLIT` |
| **Diagnostic** | `DUMP`, `DESCRIBE`, `EXPLAIN`, `ILLUSTRATE` |

#### Syntax (must remember)
```pig
R = LOAD 'path' USING PigStorage(',') AS (field:type, ...);
F = FILTER R BY condition;
G = GROUP R BY column;
T = FOREACH G GENERATE group AS key, SUM(R.amount) AS total;
DUMP T;
```

#### Built-in functions
`AVG`, `COUNT`, `SUM`, `MAX`, `MIN`, `TOKENIZE`, `CONCAT`, `SIZE`, `IsEmpty`

#### Statements end with `;`
**Pig is lazy** — nothing executes until `DUMP` or `STORE`.

#### UDFs (Java)
- 3 types: **Filter** (return boolean), **Eval** (return any), **Algebraic** (full MR on inner bag)
- `REGISTER 'myudfs.jar';` + `DEFINE Alias myudfs.ClassName();`

### Ch 7 — Hive (50 min, Tier 1) ★

**SQL-like on Hadoop. Heavy emphasis on syntax + data model.**

#### Why over Pig?
- SQL-familiar for analysts
- JDBC/ODBC → BI tool compatibility (Tableau, Power BI)
- Better for structured querying (joins, aggregations)

#### What Hive is NOT
- Not OLTP, not RDBMS, not real-time, no row-level updates by default
- For OLAP / batch analytics only

#### Architecture
```
Client → HiveServer2 → Driver (Compiler + Optimizer + Engine) → MR/Tez/Spark + MetaStore
```
- **Metastore** = relational DB storing schema/table-metadata
- Execution engine compiles HiveQL → MR/Tez/Spark DAG

#### Data model (hierarchical — HDFS layout)
```
Database  (folder)
  └── Table  (subfolder)
        └── Partition  (subfolder per key value, e.g., year=2024/)
              └── Bucket  (file inside partition, hash-distributed)
```

#### HiveQL data types
- **Integral**: `INT`, `BIGINT`, `SMALLINT`, `TINYINT`
- **String**: `STRING`, `CHAR`, `VARCHAR`
- **Decimal**: `DECIMAL(p, s)`, `FLOAT`, `DOUBLE`
- **Date**: `DATE`, `TIMESTAMP`
- **Complex** (know these well):
  - `ARRAY<T>` — access via `arr[n]`
  - `MAP<K, V>` — access via `m[key]`
  - `STRUCT<f1:T1, f2:T2>` — access via `s.f1`
  - `UNIONTYPE<T1, T2, ...>` — multi-type single column

#### DDL (key statements)
- `CREATE [EXTERNAL] TABLE name (cols) [PARTITIONED BY (...)] [CLUSTERED BY (col) INTO N BUCKETS] [STORED AS ORC] [LOCATION 'path'];`
- `DROP TABLE`, `ALTER TABLE name RENAME TO new`
- `SHOW TABLES`, `DESCRIBE [EXTENDED|FORMATTED] table`
- **External table** = data NOT deleted on DROP (only metadata removed); managed (internal) table = data also deleted

#### DML (SELECT skeleton)
```sql
SELECT cols
FROM t1 [JOIN t2 ON cond]
WHERE cond
GROUP BY cols
HAVING cond
ORDER BY cols
LIMIT n;
```
- `LATERAL VIEW explode(map_col) AS k, v` — flatten complex types

#### LOAD DATA
```sql
LOAD DATA [LOCAL] INPATH 'path' [OVERWRITE] INTO TABLE name [PARTITION (col=val)];
```
- **Managed table**: Hive MOVES the file (not copies)
- **External table**: data stays where it is

#### ACID-compliant tables (3 requirements)
1. `STORED AS ORC`
2. `TBLPROPERTIES ('transactional' = 'true')`
3. `CLUSTERED BY (key) INTO N BUCKETS`

Without all 3, UPDATE/DELETE/MERGE don't work.

#### Views
- Virtual tables (no storage)
- `CREATE [OR REPLACE] VIEW view_name AS SELECT ...;`
- Use cases: simplify joins, hide sensitive columns, abstraction layer

### Ch 8 — Spark (60-70 min, Tier 1) ★

**The "modern Hadoop" chapter. Critical for the exam.**

#### Origin + scope
- Open-source distributed engine, **in-memory caching** (~100× MR for iterative)
- Zaharia, UC Berkeley AMPLab, 2009 → Apache 2014
- Uses Hadoop only for storage; has its own compute engine
- Polyglot: Scala (native), Java, Python (PySpark), R, SQL

#### Spark vs MR (10-axis table) — be ready to recall the key differences:
| | MR | Spark |
|---|---|---|
| Processing | Batch only | Batch + streaming |
| Execution | Disk-heavy | In-memory |
| Speed | 1× | ~100× (iterative) |
| Iterative | Inefficient (new job per iteration) | Efficient (data in RAM) |
| Streaming | External | Built-in (Spark Streaming) |
| ML | External (Mahout) | Built-in (MLlib) |
| Engines | YARN only | YARN/Mesos/Standalone/K8s |
| Fault tolerance | HDFS replication | RDD lineage |

#### Two core abstractions
- **RDD (Resilient Distributed Dataset)** — immutable, partitioned, fault-tolerant collection. **Lazy** (no compute until action). Fault-tolerant via lineage (recompute lost partitions).
- **DAG (Directed Acyclic Graph)** — lineage of transformations. Used for fault recovery + optimization.

#### Architecture (must sketch)
- **Driver (master)** — runs `main()`, holds **SparkContext**, contains DAGScheduler / TaskScheduler / BlockManager
- **Executor (slave)** — JVM on worker, runs tasks, stores cached data
- **Cluster Manager** — YARN / Mesos / Standalone / Kubernetes

#### Spark Components (layered on Spark Core)
| Layer | Role |
|---|---|
| **Spark Core** | RDD API, scheduling, fault recovery |
| **Spark SQL** | DataFrames + SQL queries, Hive integration |
| **Spark Streaming** | Real-time stream processing |
| **MLlib** | Machine learning algorithms |
| **GraphX** | Graph processing |

#### 7-step application launch
1. User runs `spark-submit`
2. Cluster Manager launches **Driver**
3. Driver requests executors
4. CM assigns executors to nodes
5. NMs launch executor processes
6. Executors register with Driver
7. Driver assigns tasks to executors

#### RDD operations — Narrow vs Wide

**Transformations (lazy — build the DAG, don't run yet)**:
| Type | Examples | Shuffle? | Cost |
|---|---|---|---|
| **Narrow** | `map`, `filter`, `flatMap`, `mapPartitions`, `union`, `sample` | No — 1-to-1 partition mapping | Fast, pipelined |
| **Wide** | `groupByKey`, `reduceByKey`, `join`, `distinct`, `repartition`, `sortBy`, `cogroup` | **Yes** — many-to-many | Slow, barrier (new stage) |

**Actions (eager — trigger compute)**:
`collect`, `count`, `first`, `take(n)`, `reduce`, `foreach`, `saveAsTextFile`, `countByValue`

#### Code patterns to know
```scala
// Word count
val rdd = sc.textFile("input.txt")
val counts = rdd.flatMap(_.split(" "))
                .map(w => (w, 1))
                .reduceByKey(_ + _)
counts.saveAsTextFile("output")
```

#### DataFrames
- Built on top of RDDs: **DataFrame = RDD + schema**
- Higher-level, SQL-friendly, **Catalyst optimizer** makes them faster than raw RDDs
- Transformations: `select`, `filter`, `withColumn`, `groupBy`, `agg`, `join`, `orderBy`
- Actions: `show()`, `collect()`, `count()`, `write`
- Load: `spark.read.csv("file.csv", header=true, inferSchema=true)`

#### Why prefer DataFrame over RDD
- Catalyst query planner / Tungsten execution → faster
- Schema awareness → optimizations
- Easier SQL-like API
- RDD is still useful for low-level control and unstructured data

---

## High-yield cheatsheet (memorize cold)

### Comparison tables likely on exam
1. **ACID vs BASE** — Scale / Flexibility / Performance / Sync (4 rows)
2. **CAP** — C, A, P; can pick only 2; banking = CP, social = AP
3. **HDFS rack-aware placement** — 1 local, 1 same-rack different node, 1 different rack
4. **Hadoop 1 vs Hadoop 2 (YARN)** — JobTracker bottleneck vs RM + AM decoupling
5. **MR vs Spark** — disk vs memory, batch vs streaming, key-value only vs RDD/DF
6. **Narrow vs Wide transformations** — examples + shuffle implication

### Key formulas
- `splitSize = max(minSplit, min(maxSplit, blockSize))`
- `#Mappers = totalData / splitSize`
- `partition = hash(key) % numReducers`
- Default #Reducers = 1
- HDFS default replication factor = **3**
- HDFS default block size = **128 MB**
- Spark speed: **~100×** MR (iterative)
- JobTracker (MR1) max nodes ≈ **4000**
- YARN UI port = **8088**

### Architectures to be able to sketch from memory
1. **HDFS** — NameNode + DataNodes (across racks) + Secondary NN + Client; arrows for metadata ops vs block ops
2. **YARN** — RM (Scheduler + AppMgr) + NMs + AM in container + worker containers
3. **Hive** — Client → HiveServer2 → Driver (Compiler/Optimizer/Engine) → Hadoop + Metastore
4. **Spark** — Driver (SparkContext) + Cluster Manager + Worker Nodes (Executors)

### Key flows to draw
- **HDFS read** (4 steps): open → get block locations → read from DataNodes → close
- **HDFS write** (5 steps): create → write to DataNode pipeline → ack back → complete
- **YARN 8-step app lifecycle**
- **MR data flow**: split → map → shuffle/sort → reduce → output
- **Spark 7-step launch**

---

## Pitfalls (the trap questions)

- **Secondary NameNode ≠ failover backup** — it does checkpointing only.
- **ApplicationMaster** (per-app) ≠ **Application Manager** (sub-component of RM).
- **HDFS bad for small files** — each file = metadata in NameNode RAM.
- **Mapper output = local disk** (not HDFS). Only Reducer output goes to HDFS.
- **Default #Reducers = 1** — for parallelism set explicitly.
- **Combiner not guaranteed to run** — Hadoop may skip; don't rely on it for correctness.
- **Map-only jobs**: `setNumReduceTasks(0)` (not 1).
- **Pig is lazy** — `LOAD` doesn't validate at parse time, only on `DUMP`/`STORE`.
- **Hive external table** — `DROP TABLE` removes only metadata; data stays in HDFS.
- **Hive `ORDER BY` uses single reducer** — expensive. Use `SORT BY` / `DISTRIBUTE BY` for parallel.
- **ACID Hive table** needs all 3: ORC + bucketing + `transactional=true`.
- **Spark `collect()`** pulls all data to driver — OOM risk on large RDDs.
- **`groupByKey` vs `reduceByKey`** — prefer `reduceByKey` (does local aggregation before shuffle).
- **Spark transformations are lazy** — nothing runs until an action.
- **`cache()` is also lazy** — only caches after the first action triggers compute.
- **Driver is SPOF** — if it dies, the whole Spark app dies.

---

## Tactical advice

1. **Don't re-read the PDFs cover-to-cover** — the `.md` notes are denser. Look at PDFs only for specific diagrams.
2. **Active recall**: after each Tier 1 chapter, close the notes and try to:
   - Draw the architecture from memory
   - Write the flow steps without peeking
   - Recall the key formulas
3. **Code muscle memory** — practice writing (by hand if possible):
   - Word Count Mapper + Reducer (Java pseudo)
   - A 3-step Pig script (LOAD → GROUP → FOREACH → DUMP)
   - A HiveQL `CREATE TABLE` + `SELECT … JOIN … GROUP BY`
   - A Spark `reduceByKey` word count (Scala)
4. **YouTube** — if a topic isn't clicking, **one 15-min video max**:
   - "HDFS architecture explained"
   - "MapReduce word count example"
   - "Spark RDD vs DataFrame"
5. **Sleep by 23:30** — non-negotiable. Tired exam ≠ knowledge.
6. **Morning**: don't try new material — only review. The brain locks in during sleep, not at 06:30.

**Good luck. You can do this.**
