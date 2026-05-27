# Chapter 8 — Apache Spark

## Bird's eye view

- **Spark** = open-source **distributed processing engine** for big data — built around **in-memory caching** and an optimized DAG execution model.
- Originated at **UC Berkeley AMPLab (Zaharia, 2009)**, open-sourced 2010, Apache top-level 2014.
- **Up to ~100× faster than MapReduce** for iterative/in-memory workloads (key advantage over MR's disk-heavy model).
- Spark uses **Hadoop only for storage** — has its own compute engine. Runs on YARN, Mesos, Standalone, or Kubernetes.
- Two core abstractions: **RDD** (Resilient Distributed Dataset) and **DAG** (Directed Acyclic Graph of transformations).
- Architecture: **Driver** (master, with SparkContext) + **Executors** (slaves) + **Cluster Manager**.
- **Spark Components**: Spark Core, **Spark SQL**, **Spark Streaming**, **MLlib**, **GraphX**.
- Operations on RDDs/DataFrames are either **transformations** (lazy — narrow or wide) or **actions** (trigger execution).
- **Narrow** transformations stay within partitions (cheap); **wide** transformations require a **shuffle** across the network.
- **DataFrames** = RDD + schema → higher-level, SQL-friendly, faster (Catalyst optimizer).
- Supports **multi-language APIs**: Scala (native), Java, Python (PySpark), R.

---

## 1. Introduction to Apache Spark

### 1.1. Recap (from previous chapters)

- **MapReduce** = fault-tolerant programming model + framework on Hadoop. Schedules tasks, monitors, retries.
- **HDFS + MR** = compute and storage co-located (data locality).
- **YARN + MR** = MR running on top of YARN (Hadoop 2.0).

Spark fits into this picture by replacing/augmenting MR with a faster, more general engine.

### 1.2. What Spark is

- An **open-source distributed processing system** used for big-data analytics.
- Provides development APIs in **Java, Scala, Python, R**.
- Supports code reuse across nodes.
- Use cases: **batch processing, interactive queries, real-time analytics, ML, graph processing**.
- Started by **Matei Zaharia** at UC Berkeley's AMPLab in 2009.
- Switched license to Apache 2.0 in 2013.
- Became a top-level Apache project in **February 2014**.
- **Has its own cluster management for computation; uses Hadoop only for storage.**

Timeline of releases:
- 2009: Research project at UC Berkeley AMPLab
- 2010: Open-sourced
- 2013: Apache Foundation donation
- 2014: Spark becomes top-level project; Spark 1.0
- 2016: DataFrame and Dataset APIs unified (Spark 2.0)
- 2018: Project Hydrogen announced (distributed ML)
- 2022: Spark 3.3.0
- 2025: Spark 3.5.5 (latest)

### 1.3. Spark vs MapReduce — visualized

- **MapReduce**: each step writes intermediate data to HDFS → frequent serialization and disk I/O → slower.
- **Spark**: uses **RDDs** (immutable in-memory partitioned distributed collections) → keeps data in RAM between operations → faster.

### 1.4. Spark vs MapReduce — table

| Aspect | MapReduce | Spark |
|---|---|---|
| Processing model | Batch only | Batch + Real-time (Spark Streaming) |
| Execution | Disk-based I/O between map/reduce | In-memory via RDDs |
| Performance | Slower (frequent disk) | Up to 100× faster in-memory |
| Ease of use | Verbose Java code | Concise APIs in Scala, Java, Python, R, SQL |
| Iterative algos | Inefficient (new job per iteration) | Efficient (data in memory across iterations) |
| Data structures | Key-value pairs only | RDDs, DataFrames, Datasets |
| Streaming | Not native | Spark Streaming / Structured Streaming |
| Machine learning | External (Mahout) | Built-in MLlib |
| Execution engines | YARN only | YARN, Mesos, Kubernetes, Standalone |
| Fault tolerance | HDFS replication | RDD lineage + DAG recovery |

### 1.5. Features of Spark

| Feature | Detail |
|---|---|
| **Swift processing** | ~100× in memory, ~10× on disk vs MR |
| **Dynamic** | Parallel apps with 80+ high-level operators |
| **In-Memory** | Speed via RAM caching |
| **Reusability** | Code reuse: batch + stream + ad-hoc on stream state |
| **Fault tolerance** | RDD lineage allows recomputation |
| **Real-time** | Spark Streaming |
| **Lazy evaluation** | Transformations don't run until an action |
| **Polyglot** | Scala (native), Java, Python, R + SQL |
| **Sophisticated analysis** | Streaming, queries, ML libs |
| **Hadoop integration** | Can run standalone or on YARN; reads existing Hadoop data |
| **Cost efficient** | Avoid replication overhead of MR |
| **GraphX** | Graph and graph-parallel computation |

### 1.6. Spark use cases

- **Data integration / ETL** — fetch data from inconsistent sources, transform, load.
- **Stream processing** — handle real-time log files, detect fraud.
- **Machine learning** — in-memory makes iterative ML feasible.
- **Interactive analytics** — fast response; ad-hoc queries vs predefined.

Industry examples:
- **E-commerce** (Alibaba, eBay): real-time recommendations via streaming + ML.
- **Healthcare** (MyFitnessPal): data quality / food-item identification.
- **Gaming** (Tencent, Riot): pattern detection in real-time events.
- **Media** (Yahoo, Netflix): personalization, ad targeting.

---

## 2. Spark Architecture

### 2.1. Two abstractions

| Abstraction | Detail |
|---|---|
| **RDD (Resilient Distributed Dataset)** | The building block of any Spark app. **Resilient** = fault-tolerant; rebuilt on failure. **Distributed** = data across nodes. **Dataset** = partitioned collection. |
| **DAG (Directed Acyclic Graph)** | **Directed**: transformations move partition state from A to B. **Acyclic**: cannot return to older partition. A DAG is the lineage of operations on data — each node = an RDD partition, each edge = a transformation. |

### 2.2. Master/slave architecture

Spark has two daemons + a cluster manager:

| Daemon | Role |
|---|---|
| **Master Daemon** (Driver Process) | One per cluster. Master of the app. |
| **Worker Daemon** (Slave/Executor) | One+ per cluster. Where the actual work happens. |

A spark cluster has a single Master + any number of Workers.

### 2.3. Spark Driver (Master)

- **Central point** and **entry point** of the Spark shell (Scala, Python, R).
- Runs the `main()` function of the application.
- Where the **SparkContext** is created.
- Contains:
  - **DAGScheduler** — turns logical plan into stages.
  - **TaskScheduler** — sends tasks to executors.
  - **BackendScheduler** — interacts with cluster manager.
  - **BlockManager** — manages cached data.

### 2.4. Executor / Worker (Slave)

- **Distributed agent** responsible for task execution.
- Every Spark app has its own executor processes.
- Usually run for the **entire lifetime** of an app ("Static Allocation"). Can also be dynamic.
- **Performs all data processing.**
- Reads from and writes to external sources.
- **Stores computation results in-memory, cache, or disk.**
- Interacts with storage systems.

### 2.5. Cluster Manager

- External service for **acquiring resources** on the Spark cluster and allocating them to jobs.
- Manages memory, CPU, etc.
- Supported: **Hadoop YARN**, **Apache Mesos**, **Standalone**, **Kubernetes** — on-prem or in the cloud.

### 2.6. Spark Components (the stack)

Layered on **Apache Spark Core**:

```
[ Spark SQL ] [ Spark Streaming ] [ ML / MLlib ] [ GraphX ] [ 3rd-party libs ]
              ↑              ↑              ↑           ↑
              └────── Apache Spark Core (RDD API, scheduling, fault recovery) ──────┘
              ↑              ↑              ↑           ↑
              [ Standalone ] [ Mesos ] [ YARN ] [ Kubernetes ] [ EC2 ]
              ↑              ↑              ↑           ↑
              [ R ]    [ Java ]    [ Python ]    [ Scala ]
```

| Component | Role |
|---|---|
| **Spark Core** | Task scheduling, memory mgmt, fault recovery, storage interaction. Home of the RDD API. |
| **Spark SQL** | Structured data via SQL/HQL; supports Hive tables and JSON; can mix SQL with RDD ops. |
| **Spark Streaming** | Processes live data streams (e.g., web server logs) with the same fault tolerance/throughput as Core. |
| **MLlib** | ML library with classification, regression, clustering, collaborative filtering, model evaluation. |
| **GraphX** | Graph manipulation + graph-parallel computation (e.g., PageRank, subgraph, mapVertices). Extends RDD API. |
| **Cluster Managers** | YARN, Mesos, Standalone Scheduler, Kubernetes. |

### 2.7. Spark Execution Modes

| Mode | Description |
|---|---|
| **Standalone** | Spark runs on its own cluster, no Hadoop dependency for compute. |
| **Hadoop 2.x (YARN)** | Spark runs as a YARN application; uses Hadoop's resource manager + HDFS. |
| **Hadoop V1 (SIMR)** | Spark runs inside MapReduce as a wrapper (legacy). |
| **Kubernetes** | Spark on k8s, common in cloud-native deployments. |

### 2.8. Launching a Spark Application — 7 steps

1. **User runs `spark-submit`** specifying app JAR, main class, master URL.
2. **Cluster Manager launches the Driver** (allocates a container/process).
3. **Driver requests executors** (CPU + memory).
4. **CM schedules executors** — asks NodeManagers/pods to start them.
5. **Executors launch** as JVM processes on the worker nodes.
6. **Executors register with the Driver** — Driver now knows what compute it has.
7. **Driver assigns tasks** — schedules tasks from stages, sent to executors to operate on data partitions in parallel.

```
1. User runs spark-submit
2. Cluster Manager starts Driver
3. Driver requests executors
4. CM assigns nodes for Executors
5. Executors launched on Worker nodes
6. Executors register with Driver
7. Driver sends tasks to Executors → execution begins
```

---

## 3. Spark Programming Concepts

### 3.1. Resilient Distributed Datasets (RDDs)

- **Fundamental data structure** of Spark.
- Each RDD is divided into **logical partitions**, computed on different cluster nodes.
- A **read-only, partitioned collection of records**.
- **Immutable**.
- **Fault-tolerant** — can be operated on in parallel.
- Created by deterministic operations on stable storage data OR other RDDs.
- **Lazy** — doesn't compute anything until an action is called.

Two op types on RDDs:
- **Transformations** (lazy, produce new RDD)
- **Actions** (trigger computation, return value / write)

```scala
val rdd = spark.sparkContext.parallelize(Seq(1,2,3,4,5))
val squaredRDD = rdd.map(x => x * x)
squaredRDD.collect()  // Array(1, 4, 9, 16, 25)
```

### 3.2. Narrow vs Wide Transformations

The key performance distinction:

| Feature | Narrow | Wide |
|---|---|---|
| Data movement | Local — each input partition → one output partition | **Shuffle across network** |
| Execution stage | Same stage | New stage required |
| Performance | Fast, pipelined | Slower (network I/O, barriers) |
| Fault tolerance | Simple | Complex (lineage-based recovery) |
| RDD examples | `map`, `filter`, `flatMap` | `reduceByKey`, `join`, `groupBy` |
| DF examples | `select`, `filter`, `withColumn` | `join`, `groupBy`, `orderBy` |

Visual: narrow = each parent partition feeds one child partition; wide = many-to-many across partitions.

### 3.3. RDD Operations — Transformations

#### Narrow transformations
| Op | Description | Example |
|---|---|---|
| `map(func)` | Apply func to each element | `rdd.map(x => x * 2)` |
| `flatMap(func)` | Like map, but func returns multiple | `rdd.flatMap(line => line.split(" "))` |
| `filter(func)` | Keep elements matching condition | `rdd.filter(x => x > 10)` |
| `mapPartitions(func)` | Apply func to each partition | `rdd.mapPartitions(iter => iter.map(_ * 2))` |
| `sample(...)` | Sampled subset | `rdd.sample(false, 0.1)` |
| `union(other)` | Combine two RDDs (same partitioning) | `rdd1.union(rdd2)` |
| `glom()` | Each partition → array | `rdd.glom()` |

#### Wide transformations
| Op | Description | Example |
|---|---|---|
| `groupByKey()` | Group values with same key | `rdd.groupByKey()` |
| `reduceByKey()` | Aggregate values per key | `rdd.reduceByKey(_ + _)` |
| `join(other)` | Inner join by key | `rdd1.join(rdd2)` |
| `leftOuterJoin()` / `rightOuterJoin()` | Outer joins | `rdd1.leftOuterJoin(rdd2)` |
| `distinct()` | Remove duplicates | `rdd.distinct()` |
| `repartition(n)` | Reshuffle into n partitions | `rdd.repartition(4)` |
| `coalesce(n)` | Reduce partition count (no shuffle if narrow possible) | `rdd.coalesce(2)` |
| `sortBy()` | Global sort | `rdd.sortBy(x => x)` |
| `cogroup()` | Group RDDs with same keys | `rdd1.cogroup(rdd2)` |

### 3.4. RDD Operations — Actions (trigger compute)

| Action | Description | Example |
|---|---|---|
| `collect()` | Return all elements to driver (be cautious!) | `val data = rdd.collect()` |
| `count()` | Number of elements | `val total = rdd.count()` |
| `first()` | First element | `val item = rdd.first()` |
| `take(n)` | First n elements | `val top5 = rdd.take(5)` |
| `takeOrdered(n)` | First n sorted ascending | `rdd.takeOrdered(3)` |
| `top(n)` | Top n descending | `rdd.top(3)` |
| `reduce(func)` | Aggregate via binary func | `val sum = rdd.reduce(_ + _)` |
| `fold(zero)(func)` | Like reduce with starting value | `val sum = rdd.fold(0)(_ + _)` |
| `aggregate(z)(s, c)` | Combine within + across partitions | (advanced) |
| `foreach(func)` | Side-effect per element | `rdd.foreach(println)` |
| `countByValue()` | Count occurrences | `rdd.countByValue()` |
| `saveAsTextFile()` | Write to HDFS/local | `rdd.saveAsTextFile("output/")` |
| `saveAsSequenceFile()` | Write as SequenceFile | |
| `saveAsObjectFile()` | Java serialization | |
| `lookup(key)` | All values for a PairRDD key | `rdd.lookup("k")` |

### 3.5. DataFrames (DF)

- Built on top of RDDs: **DataFrames = RDDs + Schema**.
- Higher-level, SQL-friendly data structure.
- Supports SQL queries and high-level transformations.
- Implemented as immutable distributed tables with rows, columns, and a schema.
- Divided into partitions, distributed across executors.
- Analogous to:
  - A table in a DB (but: no indexes, primary keys, constraints)
  - A DataFrame in Python/R
- Offers **better performance and readability** than RDDs (Catalyst optimizer, vectorized execution).

```scala
val df = spark.read.csv("employees.csv", header=true, inferSchema=true)
df.select("id","name").filter("gender" === "MALE").show()
```

Like RDDs: lazy, with transformations + actions.

### 3.6. DataFrame Operations — Transformations

#### Narrow
| Op | Description |
|---|---|
| `select(...)` | Pick columns |
| `withColumn(...)` | Add/replace column with expression |
| `drop(...)` | Remove column |
| `filter(...)` / `where(...)` | Row filter |
| `limit(n)` | First n rows |
| `na.fill(...)` / `na.drop()` | Null handling |
| `dropDuplicates(...)` | Without shuffle (one partition) |
| `cache()` / `persist()` | Mark for caching |
| `coalesce(n)` | Reduce partitions without shuffle |

#### Wide
| Op | Description |
|---|---|
| `groupBy(...)` | Group by keys |
| `agg(...)` | Aggregate grouped data |
| `orderBy(...)` / `sort(...)` | Global sort (shuffle) |
| `join(...)` | Combine two DFs |
| `union(...)` | Combine (same schema) |
| `intersect(...)` | Common rows |
| `except(...)` | Rows in df1 not df2 |
| `dropDuplicates(...)` | Wide if needs cross-partition dedup |
| `repartition(n)` | Reshuffle |

### 3.7. DataFrame Operations — Actions

| Action | Description |
|---|---|
| `show(n)` | Display first n rows (default 20) |
| `collect()` | All rows to driver |
| `take(n)` | First n rows |
| `head()` / `first()` | First row(s) |
| `count()` | Row count |
| `reduce(func)` | Binary aggregation |
| `foreach(func)` / `foreachPartition(func)` | Side-effects |
| `toLocalIterator()` | Iterator pulling rows to driver |
| `write` | Save to CSV/Parquet/etc. |
| `rdd` | Convert to underlying RDD |

---

## 4. Spark Programming — worked examples

### 4.1. Load data
```scala
val employees = spark.read
    .option("header", "true").option("inferSchema", "true")
    .csv("/shared/employees.csv")

val contributions = spark.read
    .option("header", "true").option("inferSchema", "true")
    .csv("/shared/contributions.csv")

val reimbursements = spark.read
    .option("header", "true").option("inferSchema", "true")
    .csv("/shared/reimbursements.csv")
```

### 4.2. Filter females
```scala
val females = employees.filter($"gender" === "FEMALE")
females.show()
```

### 4.3. Left-join employees with contributions
```scala
val empWithContrib = employees
    .join(contributions, employees("id") === contributions("employee_id"), "left")
    .select("id", "name", "amount")
empWithContrib.show()
```

### 4.4. Total contributions per employee
```scala
val totalContrib = contributions
    .groupBy("employee_id")
    .agg(sum("amount").alias("total_contrib"))
totalContrib.show()
```

### 4.5. Females WITHOUT any reimbursement (left anti-join)
```scala
val empWithReimb = employees
    .filter($"gender" === "FEMALE")
    .join(reimbursements, employees("id") === reimbursements("employee_id"), "left_anti")
empWithReimb.show()
```

### 4.6. Full summary — total contribution + total reimbursement per employee
```scala
val totalReimb = reimbursements
    .groupBy("employee_id")
    .agg(sum("amount").alias("total_reimb"))

val summary = employees
    .join(totalContrib, employees("id") === totalContrib("employee_id"), "left")
    .join(totalReimb, employees("id") === totalReimb("employee_id"), "left")
    .select($"id", $"name", $"gender", $"total_contrib", $"total_reimb")
summary.show()
```

---

## Key terms (glossary)

- **RDD** — Resilient Distributed Dataset: immutable, partitioned, fault-tolerant collection.
- **DAG** — Directed Acyclic Graph; the lineage of transformations.
- **Driver** — master process; entry point; runs SparkContext.
- **Executor** — worker process; runs tasks, stores cached data.
- **Cluster Manager** — YARN/Mesos/Standalone/K8s.
- **SparkContext** — main entry point in code (Scala/Python/Java/R).
- **DataFrame** — RDD with a schema; SQL-friendly.
- **Dataset** — typed version of DataFrame (Scala/Java).
- **Transformation** — lazy op producing a new RDD/DF.
- **Action** — eager op that triggers execution.
- **Narrow transformation** — no shuffle.
- **Wide transformation** — requires shuffle.
- **Shuffle** — redistribution of data across partitions (expensive).
- **Lineage** — chain of transformations used for fault recovery.
- **Stage** — sequence of narrow transformations between two wide ones.
- **Task** — unit of work assigned to an executor.
- **Catalyst** — Spark SQL's query optimizer.

---

## Exam targets

1. **Why Spark over MapReduce?** — list 4-5 reasons (in-memory speed, iterative algos, streaming, language support, unified API).
2. **Define RDD** with its properties (resilient, distributed, immutable, lazy).
3. **Sketch Spark's architecture** — Driver + Executors + Cluster Manager.
4. **Describe Spark Components** (Core, SQL, Streaming, MLlib, GraphX).
5. **Explain Narrow vs Wide transformations** — give examples, performance implications.
6. **Describe the 7-step app launch process** for a Spark application.
7. **Write a small Spark snippet** (Scala or PySpark): load CSV, filter, join, groupBy, aggregate.
8. **DataFrame vs RDD** — when to use which.
9. **Why is lazy evaluation an optimization?** (Allows query plan optimization, avoids recomputation.)
10. **How does Spark achieve fault tolerance?** (RDD lineage allows recomputing lost partitions.)

### Pitfalls
- **Spark doesn't replace HDFS** — it replaces (or runs alongside) MR for compute. Storage is still HDFS/S3/etc.
- **`collect()` returns to the driver** — pulls all data to one machine. Use `take(n)` or write to file for large outputs.
- **`groupByKey` vs `reduceByKey`**: prefer `reduceByKey` — it does local aggregation before shuffle (less network).
- **Transformations are lazy** — nothing runs until an action.
- **Each stage's boundary = a shuffle** = a wide transformation.
- **`cache()` is also lazy** — only caches after an action triggers computation.
- **Driver is a SPOF** — if it dies, the whole app dies.
- **Executor JVM heap size** matters — out-of-memory errors are common in production.
- **In code, `$"col"` is Scala-specific** — equivalent to `col("col")` or `df("col")`.
