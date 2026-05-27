# Chapter 4 — MapReduce Programming Model

## Bird's eye view

- **MapReduce** = programming model + execution framework for processing huge datasets *in parallel* on a cluster (Google paper by Ghemawat & Dean, 2004).
- Two pure functions: **map** transforms input records into intermediate key-value pairs; **reduce** combines all values for the same key.
- Between them: **shuffle & sort** — Hadoop groups intermediate pairs by key and routes them to reducers (the magic that makes MR work).
- Built-in characteristics: **parallel**, **scalable**, **fault-tolerant**, **data-locality aware**, **batch** (not real-time), **simple programming model** (KV pairs).
- The full execution pipeline has **9 components**: InputFormat → InputSplit → RecordReader → Mapper → Combiner → Partitioner → Shuffle/Sort → Reducer → OutputFormat.
- Key formulas to memorise: `splitSize = max(minSplit, min(maxSplit, blockSize))`, `#Mappers = totalDataSize / splitSize`, `Partition = hash(key) % numReducers`, default `#Reducers = 1`.
- **Combiner** = optional "mini-reducer" running locally after the mapper — reduces shuffle traffic.
- Classic MR **design patterns**: Summarization, Filtering, Data Organization, Join, Meta.
- Application = a **Driver** (mandatory) + a **Mapper** class (mandatory) + optional Reducer/Combiner/Partitioner.

---

## 1. Hadoop MapReduce — Presentation

- A **programming model** AND a **distributed computing framework** for processing large datasets in parallel across a cluster of computers.
- Originated in the Google paper *"MapReduce: Simplified Data Processing on Large Clusters"* by **Sanjay Ghemawat** and **Jeffrey Dean** (2004), which described the methods used at Google for search engine indexing.
- **Hadoop MapReduce** is the open-source implementation released within the Apache Hadoop project.

### 1.1. Characteristics

| Characteristic | Detail |
|---|---|
| **Parallel processing** | Distributes tasks across machines to improve efficiency |
| **Scalability** | Handles PB by adding nodes; works in environments like Hadoop |
| **Fault tolerance** | Auto-reassigns tasks on failed nodes; data is replicated for reliability |
| **Data locality** | Processes data close to where it's stored to reduce network traffic |
| **Simplified programming model** | Abstracts distributed complexity; uses key-value pairs |
| **Batch processing** | Suited for large-scale batch jobs (logs, indexing, aggregation) — *not* real-time |

### 1.2. The Map and Reduce signatures

```
map    : (inKey, inValue)        → list(outKey, interValue)
reduce : (outKey, list(interValue)) → list(outValue)
```

- `map` processes input KV pairs; one map call per input pair; produces intermediate pairs.
- `reduce` combines all intermediate values for a given key; one reduce call per unique key; produces output values (often just one).

### 1.3. The full MapReduce paradigm (the visual map)

1. **Input files split** across machines.
2. Each split is **fed to a Mapper**.
3. Each mapper processes its input line by line, outputting KV pairs to its local disk.
4. **Shuffle phase**: Hadoop sorts and routes KV pairs by key — same-key pairs go to the same Reducer.
5. Each **Reducer** receives a group of KV pairs sharing a key and produces a final single KV pair (typically).

The user **only writes** the map and reduce functions — Hadoop handles split/shuffle/sort/scheduling/recovery.

### 1.4. Functional programming background

- Map and Reduce come from functional programming (FP).
- FP treats computation as evaluation of mathematical functions; avoids mutable state.
- A trivial Java parallel `.map(n -> n*n)` over a stream is the basic idea.

### 1.5. Worked examples

**Word Count** (the canonical example):
- Input: `Deer Bear River / Car Car River / Deer Car Bear`
- Split → Map (each word → (word, 1)) → Shuffle (group by word) → Reduce (sum counts) → Output: `Bear=2, Car=3, Deer=2, River=2`

**Movies viewed per user** (exercise from the slides):
- Input rows like `(user_id, movie_id, rating, timestamp)`
- Map: emit `(user_id, 1)` per row
- Shuffle: group by user_id
- Reduce: sum counts
- Output: `{166: 1, 186: 3, 196: 2, 244: 1}`

---

## 2. MapReduce Data Flow — the 9 components

The execution pipeline is:

```
Input Data (HDFS)
   ↓
[InputFormat] → [InputSplit] → [RecordReader] → [Mapper] → [Combiner?] → [Partitioner]
   ↓
[Shuffle and Sort]
   ↓
[Reducer] → [OutputFormat]
   ↓
Output Data (HDFS)
```

### 2.1. InputFormat (1)

- Checks the **input specification** of the job.
- **Splits** the input file into InputSplits, one per Mapper.
- Defines the data splits (size of individual map tasks).
- Defines the **RecordReader** that reads actual records.

Common InputFormats:
1. **TextInputFormat** (default) — files broken into lines, key = byte offset, value = line content.
2. **KeyValueTextInputFormat** — like Text but splits each line on a tab character into K and V.
3. **NLineInputFormat** — each mapper gets a fixed *number of lines* (instead of bytes).
4. **SequenceFileInputFormat** — reads binary sequence files (KV pairs, block-compressed, supports user types).
5. **SequenceFileAsTextInputFormat** — converts sequence file KV to Text.
6. **SequenceFileAsBinaryInputFormat** — reads sequence file as raw bytes.
7. **XML (XmlInputFormat)** — for XML records with custom start/end tags.
8. **DBInputFormat** — reads from relational DBs via JDBC (no partitioning — be careful with large tables).

### 2.2. InputSplit (2)

- The **logical representation** of a chunk of data — what a single Mapper will process.
- Splits convert physical blocks → logical units for the Mapper.
- **#Map tasks = #InputSplits**.
- Users don't usually deal with InputSplit directly — FileInputFormat generates it.
- FileInputFormat default: splits into **128 MB chunks** (matching HDFS block size).
- Configurable via `mapred.min.split.size` in mapred-site.xml.

Key formula:
```
splitSize = max(minSplitSize, min(maxSplitSize, blockSize))
```

Each split holds the first byte's offset and length.

### 2.3. RecordReader (3)

- Communicates with InputSplit; **converts data into KV pairs** suitable for the Mapper.
- Two main types:
  - **LineRecordReader** (default) — considers each line as a record; key = byte offset, value = line.
  - **SequenceFileRecordReader** — reads sequence files.
- Reads from the start to end of the split.
- Max line size configurable via `mapred.linerecordreader.maxlength`.

### 2.4. Mapper (4)

- Processes each input record and emits new KV pairs.
- The framework generates **one Mapper per InputSplit**.
- Mapper output = intermediate output, **written to local disk** (NOT HDFS — saves replication overhead).
- Then passed to the Combiner (if any) or directly to shuffle.

Key formula:
```
#Mappers = total data size / input split size
```
Example: 2.39 GB / 100 MB = **25 Mappers**.

### 2.5. Combiner (5) — the "Mini-Reducer"

- **Optional**. Performs **local aggregation** on each Mapper's output.
- **Runs after the Mapper, before the Reducer**, on the same node.
- Greatly **reduces shuffle network traffic** (key benefit on large jobs).
- Same input/output types as Reducer (often you can reuse the Reducer class).
- The Combiner's role: take intermediate KV pairs and pre-aggregate them locally before they're sent across the network.

Example: For Word Count, the Combiner sums word counts per mapper before shuffle, so instead of sending `(the, 1)` 1000 times, it sends `(the, 1000)` once.

### 2.6. Partitioner (6)

- Decides which Reducer each intermediate KV pair goes to.
- Default: hash partitioning → `partition = hash(key) % numReducers`.
- **#Partitions = #Reducers**.
- The Partitioner creates partitions only if multiple Reducers exist.

### 2.7. Shuffle and Sort (7)

- The data transfer phase from Mappers to Reducers.
- Hadoop **automatically sorts** KV pairs by key before sending them to Reducers — sorting helps Reducers easily distinguish when a new key's group starts.
- Each Reducer pulls (via HTTP-based RPC) the partitions assigned to it from every Mapper.
- Shuffle can start before all mappers finish.

Step-by-step:
1. Mapper output is stored locally in partitions.
2. Mapper sorts its KV pairs by key.
3. Partitioner assigns each pair to a Reducer.
4. Reducers PULL their partitions from all Mappers via RPC.
5. Each Reducer merges and sorts all incoming data.

### 2.8. Reducer (8)

- Receives all values for a given key (sorted).
- Runs the user-defined `reduce()` function.
- Produces output, stored to HDFS.
- Default `#Reducers = 1` (configurable via `Job.setNumReduceTasks(int)`).
- **#Reducers** can be tuned: 0.95 or 1.75 × `<#nodes> × <max containers/node>`:
  - 0.95: all reducers launch immediately.
  - 1.75: faster nodes run two waves → better load balancing.

### 2.9. OutputFormat (9)

- Defines how output is written.
- Uses a **RecordWriter** internally to write output to files.

Common OutputFormats:
1. **TextOutputFormat** (default) — writes KV pairs as text (key TAB value), tab-separated.
2. **SequenceFileOutputFormat** — writes binary sequence files (compact, compressible).
3. **SequenceFileAsBinaryOutputFormat** — sequence file in raw binary.
4. **MapFileOutputFormat** — sorted MapFile output (sort order required from reducer).
5. **MultipleOutputs** — multiple output files with different names.
6. **LazyOutputFormat** — only creates output if needed.
7. **DBOutputFormat** — writes to relational DBs.

### 2.10. Understanding the KV pairs at each stage

| Stage | Input | Output |
|---|---|---|
| **Map** | Line offset (key) + line content (value), unless custom InputFormat | Depends on logic |
| **Reduce** | Same as Map output | Depends on requirements |

---

## 3. Job Types and Design Patterns

### 3.1. Job types

| Type | Description |
|---|---|
| **Single Mapper Job** | Mapper does everything (data filter/conversion/upload). |
| **Single Mapper + Reducer** | Classic MR. |
| **Single Mapper + Combiner + Reducer** | With local aggregation. |
| **Multiple Mappers + Reducer** | Read multiple data sources in parallel. |

### 3.2. MapReduce design patterns

Categorized into 5 families:

#### A. Summarization Patterns
- **Numerical summarization** — group records by a key, compute aggregate (sum, count, min, max, average, median, stddev).
- **Inverted Index** — generate an index from data → enables faster search / enrichment (this is how search engines work).
- **Counting with counters** — global counts emitted via the framework's counters utility (no output records produced).

When to use:
- Numerical data or counting
- Data can be grouped by specific fields

#### B. Filtering Patterns
The patterns don't change records; they keep/discard them.
- **Filtering** — the most basic pattern. Each record evaluated separately by a function deciding keep/discard.
- **Bloom Filtering** — same goal but uses a **Bloom filter** for evaluation (efficient for very large hot-value sets).
- **Top Ten** — find the top N records. Output is fixed-size irrespective of input.
- **Distinct** — deduplication. Output size depends on data.

When to use:
- Want a smaller subset, search-like operations

#### C. Data Organization Patterns
Reorganize the data — value of records is in *how they're arranged*.
- **Structured to Hierarchical** — flat data → nested/hierarchical (e.g., JSON, XML).
- **Partitioning and Binning** — moves records into category bins (no sorting within bins).
- **Total Order Sorting** — globally sorted output across all output files.
- **Shuffling** — randomize record order (opposite of sorting).

When to use:
- Restructure data layout for other systems (e.g., row-based → hierarchical)

#### D. Join Patterns
Combine records from two or more data sources.
- **Reduce-Side Join** — most general, but takes the longest (shuffles all data).
- **Replicated Join (map-side join)** — when one dataset is small (fits in memory), broadcast it to all mappers.
- **Composite Join** — both datasets pre-partitioned and sorted on the join key; both large.
- **Cartesian Product** — pair every record with every other (expensive).

Note: joins are network-heavy — optimization matters.

#### E. Meta Patterns
- **Job Chaining** — when a problem needs multiple MR jobs; output of one feeds the next. (Hadoop doesn't natively support multi-job orchestration well — Oozie or scripting fills this gap.)
- **Job Merging** — combine two unrelated jobs reading the same input into one to save I/O. The data is loaded and parsed only once.

---

## 4. MapReduce Applications (writing one)

### 4.1. Job components

- **Driver (mandatory)** — application shell that configures the Job, submits it to the Resource Manager.
- **Mapper class (mandatory)** — defines KV input/output formats; implements `map()`.
- **Reducer class (optional)** — for map-only jobs no Reducer is needed.
- **Combiner class (optional)** — usually same as Reducer; for special tasks may differ (e.g., secondary sort).
- **Partitioner (optional)** — for rare cases needing sparse keys or special routing.
- **RecordReader / RecordWriter (optional)** — custom serialization.

The API is primarily Java-based, but **Hadoop Streaming** lets you use other languages (commonly C, Python, Perl) for mapper/reducer.

### 4.2. Skeleton — Driver

```java
public class FlightsByCarrier {
  public static void main(String[] args) throws Exception {
    Job job = new Job();
    job.setJarByClass(FlightsByCarrier.class);
    job.setJobName("FlightsByCarrier");

    TextInputFormat.addInputPath(job, new Path(args[0]));
    job.setInputFormatClass(TextInputFormat.class);

    job.setMapperClass(FlightsByCarrierMapper.class);
    job.setReducerClass(FlightsByCarrierReducer.class);

    TextOutputFormat.setOutputPath(job, new Path(args[1]));
    job.setOutputFormatClass(TextOutputFormat.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    job.waitForCompletion(true);
  }
}
```

Anatomy:
- Catch input path from args
- Define overall structure (Mapper + Reducer classes)
- If map-only: `job.setNumReduceTasks(0)`
- Indicate HDFS path for output + format
- `job.waitForCompletion(true)` — submit and wait until finished

### 4.3. Skeleton — Mapper

```java
public class FlightsByCarrierMapper
    extends Mapper<LongWritable, Text, Text, IntWritable> {
  @Override
  protected void map(LongWritable key, Text value, Context context)
      throws IOException, InterruptedException {
    if (key.get() > 0) {  // skip header row
      String[] lines = new CSVParser().parseLine(value.toString());
      context.write(new Text(lines[8]), new IntWritable(1));
    }
  }
}
```

Anatomy:
- Class declares the input/output KV types
- The `map` method gets called for each line
- Uses an `if` to skip the header (offset 0)
- Parses CSV (the 9th value is the carrier)
- Writes `(carrier, 1)` per flight

### 4.4. Skeleton — Reducer

```java
public class FlightsByCarrierReducer
    extends Reducer<Text, IntWritable, Text, IntWritable> {
  @Override
  protected void reduce(Text token, Iterable<IntWritable> counts, Context context)
      throws IOException, InterruptedException {
    int sum = 0;
    for (IntWritable count : counts) sum += count.get();
    context.write(token, new IntWritable(sum));
  }
}
```

Anatomy:
- Reducer runs once per key (here per carrier).
- The `for` loop sums all the 1's → total flights.
- Output: `(carrier, total)`.

### 4.5. Running the job (4 commands)

```bash
javac -classpath $CLASSPATH MapRed/FlightsByCarrier/*.java
jar cvf FlightsByCarrier.jar *.class
hadoop jar FlightsByCarrier.jar FlightsByCarrier \
       /user/root/airlinedata/2008.csv /user/root/output/flightsCount
hadoop fs -cat /user/root/output/flightsCount/part-r-00000
```

---

## Key terms (glossary)

- **InputSplit** — logical chunk of input data assigned to one Mapper.
- **RecordReader** — converts split bytes into KV records.
- **Mapper** — function called once per input record; emits intermediate KVs.
- **Combiner** — local pre-aggregation between Mapper and shuffle.
- **Partitioner** — routes intermediate KVs to Reducers.
- **Shuffle** — phase that transfers Mapper output to Reducers, sorted by key.
- **Reducer** — function called once per unique key; receives all its values, produces output.
- **OutputFormat** — defines how output is serialized.
- **Job** — a complete MR application instance.
- **Driver** — the main class that wires up and submits the Job.

---

## Exam targets

1. **Sketch the MR data flow** (9 components) and explain each.
2. **Define map and reduce signatures** with KV types.
3. **Word count** — write the pseudo-code for Mapper and Reducer (very common written exam question).
4. **Compute #Mappers** given dataset size and split size.
5. **Explain Combiner** — what it does, where it runs, why it matters, when it's safe to use (commutative + associative ops).
6. **Explain Partitioner** — what it does and the default formula.
7. **Explain Shuffle and Sort** — why sorting helps the Reducer.
8. **Compare design patterns** — given a problem (e.g., "find top 10 users by activity"), pick the right pattern (Top Ten filtering pattern).
9. **Compare join types** — when to use Reduce-Side vs Replicated vs Composite.
10. **Limitations of MR** that motivate YARN and Spark (only two phases, batch only, disk-heavy, verbose code).

### Pitfalls
- Mapper output is on **local disk**, not HDFS.
- **Default Reducer count is 1** — for parallelism you must increase it.
- The Combiner is **not guaranteed to run** — Hadoop may skip it. Don't rely on it for correctness, only optimization.
- The Partitioner doesn't change the *order* within a partition; sorting happens during shuffle.
- Map-only jobs: set `numReduceTasks=0`, NOT to 1.
- Hadoop Streaming uses **stdin/stdout** — different style from Java API.
