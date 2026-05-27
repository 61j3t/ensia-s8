# BDAV — Syllabus Skeleton (Phase 0)

> Course: Big Data Analytics & Visualization — ENSIA (F. Dahak, 2024/2025)
> Use this as a **mental scaffold**, not a study substitute. Active recall does the real work.

---

## Ch 1 — Understanding Big Data (8 pages)
- **Definitions** (IBM, Gartner, Techopedia) — common idea: beyond traditional RDBMS
- **7Vs**: **Volume**, **Velocity**, **Variety** (Laney 2001) → **Veracity** (IBM 2012) → **Value** (Oracle 2013) → **Variability** (2015) → **Visualization** (MS 2015+). Also: Validity.
- **Analytics types** (4): Descriptive → Diagnostic → Predictive → Prescriptive
- **Applications**: Social media, IoT, E-commerce, Healthcare, Finance, Agriculture
- "**Is it big data?**" test: ≥1 V + beyond RDBMS + needs special tools
- Annex: data size units (Bit → Byte → KB → … → Bronto → Geop → …)

## Ch 2 — Big Data Ecosystem & Technologies (5 pages)
- **RDBMS struggles**: vertical scaling only, structured-only, batch, costly at PB
- **ACID** (RDBMS): **A**tomicity, **C**onsistency, **I**solation, **D**urability
- **CAP theorem** (Brewer 2000): Consistency / Availability / Partition-tolerance — **pick 2**
- **BASE** (NoSQL): **B**asically Available, **S**oft state, **E**ventually consistent
- **ACID vs BASE** table (scale, flexibility, performance, sync)
- **Hadoop**: open-source, distributed, cluster, commodity HW. Java. Inspired by Google MR + GFS. Doug Cutting + Mike Cafarella, 2005. Named after son's elephant toy.
- **Architecture (4 modules)**: MapReduce, YARN, HDFS, Common Libraries
- **Ecosystem**: Ambari (mgmt), Sqoop (RDBMS), Oozie (workflow), Pig, Mahout (ML), Hive, HBase, Flume (logs), Zookeeper

## Ch 3 — HDFS (7 pages)
- **Goals**: throughput > latency, WORM (Write Once Read Many), very large files, cluster of commodity HW. Inspired by GFS.
- **Not good for**: many small files, low-latency, multi-writers, RDBMS replacement
- **Architecture** (master/slave):
  - **NameNode** — metadata (FsImage + EditLog), namespace, file→block mapping. 4 GB RAM ≈ huge cluster. **SPOF** (no auto failover!)
  - **DataNode** — block storage, Heartbeat + BlockReport to NN
  - **Secondary NameNode** — checkpointing (NOT a failover)
- **Block replication**: default **factor = 3**. Rack-aware placement: 1 local rack + 1 different node same rack + 1 different rack
- **Replica selection (read)**: closest replica (local node > local rack > remote)
- **Safemode**: NN startup, no replication, waits for enough block reports
- **Robustness**: heartbeat-based DN failure detection, re-replication, checksums on blocks
- **Read/Write flow** (client ↔ NN for metadata, then ↔ DN for data; write uses DN pipeline)

## Ch 4 — MapReduce (11 pages)
- **Programming model**: `map: (k1,v1) → list(k2,v2)`; `reduce: (k2, list(v2)) → list(v3)`
- **Origins**: Ghemawat & Dean, Google 2004
- **Phases**: split → map → shuffle/sort → reduce. Functional programming inspired.
- **9 execution components**: InputFormat → InputSplit → RecordReader → **Mapper** → Combiner → Partitioner → **Shuffle/Sort** → **Reducer** → OutputFormat
- **InputFormats**: Text (default), KeyValueText, NLine, SequenceFile, XML, DB
- **Key formulas**:
  - `splitSize = max(minSplit, min(maxSplit, blockSize))`
  - `#Mappers = totalDataSize / inputSplitSize` (e.g. 2.39 GB / 100 MB = 25)
  - `Partition = hash(key) % numReducers`
  - Default `#Reducers = 1`
- **Combiner** = "mini-reducer" — local aggregation, optional, reduces shuffle traffic
- **Job types**: Map-only, Map+Reduce, Map+Combiner+Reduce, Multiple-Mappers+Reduce
- **Design Patterns** (5 categories):
  1. **Summarization**: Numerical, Inverted Index, Counting
  2. **Filtering**: Filter, Bloom, Top-N, Distinct
  3. **Data Organization**: Hierarchical, Partition/Bin, Total-Order Sort, Shuffling
  4. **Join**: Reduce-Side, Replicated, Composite, Cartesian
  5. **Meta**: Job Chaining, Job Merging
- **Application building**: Driver (mandatory) + Mapper (mandatory) + Reducer/Combiner/Partitioner/RecordReader (optional). Compile → JAR → `hadoop jar`.

## Ch 5 — YARN (10 pages)
- **Yet Another Resource Negotiator** — Hadoop 2.0. **Decouples** resource mgmt from processing logic.
- **Hadoop 1 problems**: JobTracker bottleneck (≈4000 nodes max), MR-only, static slot allocation, poor multi-tenancy
- **Design goals**: scalability, programming-model support, high utilization, multi-tenancy, locality awareness
- **Architecture** (4 actors):
  - **ResourceManager** = global authority (Scheduler + ApplicationManager)
  - **NodeManager** = per-node agent (manages containers, heartbeats)
  - **ApplicationMaster** = per-app coordinator (negotiates containers, monitors tasks)
  - **Container** = resource bundle (RAM + CPU + disk on one node)
- **Workflow (8 steps)**: submit → RM allocates AM container → AM registers → AM requests containers → NM launches → code runs → client monitors → AM unregisters
- **Schedulers** (3): **FIFO** (simple, bad shared), **Capacity** (queues per org, min guarantee + elasticity), **Fair** (equal share via pools, supports preemption)
- **Failures**: RM (SPOF, manual restart), NM (timeout, AM redoes work), AM (RM restarts container), Container (AM handles)
- **Tools**: Web UI :8088, Hue Job Browser, CLI

## Ch 6 — Pig (11 pages)
- **Why?** MR is verbose / only 2 phases. **Pig Latin** = high-level dataflow → compiles to MR jobs. Yahoo 2006 → Apache 2007.
- **Use cases**: ETL, raw research, iterative
- **Modes**: Local, MapReduce, Spark, Tez (`pig -x <mode> script.pig`)
- **Architecture**: Script → **Parser** → **Optimizer** → **Compiler** → **Execution Engine** → MR/Hadoop
- **Data model**: Atom → Field → Tuple → Bag → Map → Relation (fully nested)
- **Data types**: int, long, float, double, chararray, bytearray, boolean, datetime, biginteger, bigdecimal + complex (Tuple, Bag, Map)
- **Operators**:
  - **Loading/Storing**: `LOAD`, `STORE`
  - **Filtering**: `FILTER`, `DISTINCT`, `FOREACH … GENERATE`, `STREAM`
  - **Grouping/Joining**: `JOIN`, `COGROUP`, `GROUP`, `CROSS`
  - **Sorting**: `ORDER`, `LIMIT`
  - **Combining/Splitting**: `UNION`, `SPLIT`
  - **Diagnostic**: `DUMP`, `DESCRIBE`, `EXPLAIN`, `ILLUSTRATE`
- **Syntax**: `R = LOAD 'path' USING PigStorage(',') AS (field:type, ...);`
- **Built-in fns**: AVG, COUNT, SUM, MAX, MIN, TOKENIZE, CONCAT, SIZE, …
- **UDFs** (in Java): Filter / Eval / Algebraic — extend `EvalFunc<T>`

## Ch 7 — Hive (9 pages)
- **Why over Pig?** SQL-familiar, BI-tool compatible (JDBC/ODBC), better for structured querying. Facebook → Apache.
- **OLAP, not OLTP**. Not a real-time engine. Not a substitute for RDBMS.
- **Architecture**: Client (JDBC/ODBC/Thrift) → **HiveServer2** → **Driver** (Compiler + Optimizer + Execution Engine) → MR/Tez/Spark + **Metastore** (relational DB for schema)
- **Query processing** (10 steps): execute → plan → metadata → execute plan → execute job → fetch result
- **Data model** (hierarchical):
  - **Database** → HDFS folder
  - **Table** → folder inside DB
  - **Partition** → subfolder by key (e.g. `year=2024/month=1/`)
  - **Bucket** → file inside partition (hash-distributed)
- **Bucketing tips**: size of dataset, query patterns, parallelism, join compatibility
- **Data types**:
  - Integral (INT, BIGINT, …), String (CHAR, VARCHAR), Timestamp, Decimal, Float/Double
  - Complex: **ARRAY**`<T>`, **MAP**`<K,V>`, **STRUCT**`<f1:T1,…>`, **UNIONTYPE**
- **Operators**: relational/arithmetic/logical + complex-access (`arr[n]`, `map[key]`, `struct.field`) + agg (count/sum/avg/min/max)
- **DDL**: `CREATE [EXTERNAL] TABLE … PARTITIONED BY … CLUSTERED BY … STORED AS ORC`; DROP, ALTER, SHOW, DESCRIBE
- **DML**: `LOAD DATA [LOCAL] INPATH … [OVERWRITE] INTO TABLE …`; SELECT (joins, GROUP BY, HAVING, LATERAL VIEW); INSERT, UPDATE, DELETE, MERGE
- **ACID-compliant table** needs 3: ORC + `transactional=true` + bucketing
- **Views** = virtual tables (no storage), good for abstraction / hiding columns

## Ch 8 — Spark (10 pages)
- **What?** Open-source distributed engine, in-memory caching, optimized DAG execution. ~100x MR for in-memory iterative work. Zaharia @ UC Berkeley AMPLab 2009 → Apache 2014.
- **Uses Hadoop only for storage**; has its own cluster computation
- **Spark vs MR** (10 dims): in-memory > disk; RDD/DF > KV pairs; built-in MLlib/Streaming; multi-language; YARN/Mesos/K8s/Standalone
- **Two core abstractions**:
  - **RDD** (Resilient Distributed Dataset): immutable, partitioned, fault-tolerant, lazy
  - **DAG**: lineage of transformations, used for fault recovery
- **Architecture**:
  - **Driver** (master) — SparkContext, **DAGScheduler**, **TaskScheduler**, BlockManager
  - **Executor** (slave) — runs tasks, holds cached data
  - **Cluster Manager** — YARN / Mesos / Standalone / K8s
- **Components** (on top of Spark Core): **Spark SQL**, **Spark Streaming**, **MLlib**, **GraphX**
- **App launch (7 steps)**: spark-submit → CM starts Driver → Driver requests executors → CM assigns → NM launches executors → executors register with Driver → Driver schedules tasks
- **RDD ops**:
  - **Transformations (lazy)**:
    - **Narrow** (no shuffle): `map`, `filter`, `flatMap`, `mapPartitions`, `union`, `sample`
    - **Wide** (shuffle): `groupByKey`, `reduceByKey`, `join`, `distinct`, `repartition`, `sortBy`, `cogroup`
  - **Actions** (trigger compute): `collect`, `count`, `take`, `reduce`, `foreach`, `saveAsTextFile`
- **DataFrames** = RDD + Schema (SQL-style, optimized via Catalyst)
  - Narrow: `select`, `withColumn`, `filter`, `drop`, `limit`
  - Wide: `groupBy`, `agg`, `join`, `orderBy`, `union`, `repartition`
  - Actions: `show`, `collect`, `count`, `write`

---

## Cross-cutting themes (likely exam targets)
- **Scaling**: horizontal (Hadoop/Spark) vs vertical (RDBMS)
- **Consistency models**: ACID vs BASE; CAP trade-offs
- **Fault tolerance**: HDFS replication, MR re-execution, YARN AM restart, Spark RDD lineage
- **Performance**: data locality (move compute to data), in-memory (Spark), partitioning + bucketing (Hive), Combiner (MR)
- **Layers stack**: Storage (HDFS) → Resource mgmt (YARN) → Processing (MR/Spark) → High-level (Pig/Hive)

---

## Study plan from here

1. ✅ **Phase 0 — Syllabus map** (this file)
2. ⏳ **Phase 1 — Per chapter**: flashcards + Feynman + exam traps (one chapter at a time)
3. ⏳ **Phase 2 — Interleaved review** (mix questions across chapters)
4. ⏳ **Phase 3 — Mock exam**
