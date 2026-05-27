# Chapter 3 — HDFS (Hadoop Distributed File System)

## Bird's eye view

- **HDFS** is Hadoop's distributed storage layer, modeled on Google's GFS, designed for huge files on clusters of commodity hardware.
- **Master/slave** architecture: a single **NameNode** (metadata authority) + many **DataNodes** (block storage).
- Files are split into **blocks** (default size ~128 MB), each **replicated 3 times** across nodes/racks for fault tolerance.
- **Write-Once-Read-Many (WORM)** model: optimized for throughput, not low latency or frequent updates.
- **Rack-aware placement**: 1 replica on local node's rack, 1 on a different node in the same rack, 1 on a different rack.
- Fault tolerance via **Heartbeats** (DataNodes ping NameNode), **BlockReports**, **re-replication** on failure, and **checksums** for data integrity.
- The **NameNode is a Single Point of Failure (SPOF)** — automatic failover is NOT supported (you can configure multiple copies of FsImage and EditLog).
- Use the **Secondary NameNode** for periodic checkpointing — it is NOT a failover backup.

---

## 1. Presentation of HDFS

### 1.1. What HDFS is

- A **distributed file system** that provides high-throughput access to application data.
- Built on the architecture of **Google File System (GFS)**.
- Similar architecture to other distributed engines like Amazon S3, Azure Blob Storage.
- A **stand-alone storage engine**, usable independently of any query engine.
- Designed to run on a **cluster of commodity hardware**.
- Pattern emphasis: **Write Once, Read Many (WORM)**.
- Supports **very large files** — one file could span the entire HDFS.
- All folders/files of thousands of machines appear as a **single transparent tree** containing PB of data — as if local.

### 1.2. What HDFS is NOT good for

| Anti-pattern | Why |
|---|---|
| **Many small files** | Scalability issue for the NameNode (each file is metadata in RAM). |
| **Low-latency access** | HDFS optimizes throughput, not first-byte time. |
| **Multiple writers / arbitrary updates** | Only one writer per file; create/rename/move/delete only. No in-place updates. |
| **Substitute for an RDBMS** | Data isn't indexed; finding something means scanning everything (usually via MapReduce). |

### 1.3. Architectural assumptions and goals

- **Hardware failure is the norm** — must be handled automatically.
- **Batch processing throughput** > low latency.
- **Large datasets** — supports hundreds of nodes and tens of millions of files.
- **Simple coherency** — WORM access model (no updates except append/truncate).
- **Master/slave** — decouples metadata from data operations:
  - One NameNode manages namespace + client access
  - DataNodes manage local storage
- **Replication** for reliability and HA.
- **Network traffic minimized** — move computation to data when possible.
- **Not layered** on or dependent on any other filesystem.

---

## 2. HDFS Architecture

### 2.1. The three roles

| Role | Responsibility |
|---|---|
| **NameNode (Master)** | Manages the file system namespace, regulates access by clients, holds all metadata (namespace tree + file→block mapping). |
| **DataNode (Slave)** | One per node typically. Manages locally attached storage. Serves read/write requests. Performs block create/delete/replicate on NameNode's instructions. |
| **Secondary NameNode** | Periodic checkpointing of FsImage + EditLog. **NOT a failover replacement.** |

Other entities:
- **Client Node** — runs the HDFS client, talks to NameNode for metadata ops, talks directly to DataNodes for block I/O.

### 2.2. File system namespace

- **Hierarchical** file system with directories and files (Unix-like tree).
- Supports create, remove, move, rename.
- **NameNode** maintains all namespace metadata.
- Applications can specify the **replication factor per file** — stored as part of metadata in the NameNode.

### 2.3. Data replication

- Each file = a sequence of **blocks**.
- All blocks except the last are the same size.
- Blocks are **replicated** for fault tolerance.
- Block size and replicas are **configurable per file**.
- The NameNode receives a **Heartbeat** and a **BlockReport** from each DataNode:
  - **Heartbeat** = "I'm alive" signal
  - **BlockReport** = list of all blocks held on that DataNode

### 2.4. Replica placement (the critical detail)

- Rack-aware placement improves reliability, availability, and bandwidth use.
- Communication within a rack > between racks (network bandwidth).
- NameNode determines the rack ID for each DataNode.

**Default policy with replication factor 3:**
1. One replica on a node in the **local rack** (where the writer is)
2. One replica on a **different node in the same rack**
3. One replica on a node in a **different rack**

This is a sweet spot:
- Tolerates rack-level failure (the cross-rack copy survives)
- Avoids needless cross-rack traffic for two of three copies
- Simpler "all replicas on unique racks" is suboptimal (writes become expensive)

### 2.5. Replica selection (for reads)

- HDFS tries to minimize **bandwidth consumption** and **latency**.
- Preference order:
  1. Replica on the reader's node
  2. Replica in the same data center
  3. Replica elsewhere

### 2.6. Safemode (startup)

- On startup, NameNode enters **Safemode**.
- **No replication occurs** in Safemode.
- Each DataNode checks in with Heartbeat + BlockReport.
- NameNode verifies each block has an acceptable number of replicas.
- After a configurable % of safely-replicated blocks check in, NameNode exits Safemode.
- It then computes which blocks need re-replication and proceeds.

### 2.7. Filesystem metadata structures

| Structure | Role |
|---|---|
| **FsImage** | Full snapshot of the namespace + block-to-file mapping + file system properties |
| **EditLog** | Transaction log recording every change since the last FsImage |

Startup flow:
1. NameNode reads FsImage + EditLog from local FS
2. Applies EditLog changes to bring FsImage current
3. Writes a fresh FsImage checkpoint
4. Periodic checkpointing continues thereafter

### 2.8. NameNode in detail

- Keeps the **entire namespace + Blockmap in memory**.
- ~**4 GB RAM** is sufficient for a huge number of files/directories.
- Periodic checkpointing means the system can recover to the last checkpoint after a crash.

### 2.9. DataNode in detail

- Stores HDFS blocks as **files in its local file system**.
- Has **no knowledge** of HDFS structure (no awareness of which file a block belongs to).
- Each block stored as a separate file.
- Doesn't place all blocks in one directory — uses heuristics for optimal files-per-directory.
- On startup, generates the full **BlockReport** for the NameNode.

---

## 3. Robustness (fault tolerance)

Primary goal: **store data reliably in the presence of failures**. Three failure types to handle:

| Failure type | What happens |
|---|---|
| **NameNode failure** | Catastrophic — no auto-failover. Cluster goes down. Recovery via FsImage + EditLog backup copies. |
| **DataNode failure** | NameNode detects via missing Heartbeats. Marks node unavailable. Triggers re-replication of affected blocks. |
| **Network partition** | Subset of DataNodes lose connectivity. Treated as failures. |

### 3.1. DataNode failure and heartbeat

- NameNode detects failure by **absence of Heartbeats** (timeout-based).
- Marks DataNode as down; stops sending IO requests to it.
- Any data registered to the failed DataNode is unavailable.
- The death may drop some blocks below their replication factor → triggers re-replication.

### 3.2. Re-replication

Triggered when:
- A DataNode becomes unavailable
- A replica is corrupted
- A disk fails on a DataNode
- The replication factor for a block is increased

### 3.3. Data integrity

- Corruption may occur from storage faults, network faults, buggy software.
- The HDFS **client** creates a **checksum** of every block when writing and stores it in hidden files in the namespace.
- On read, the client **verifies the checksum**. If mismatched, fetches the block from another replica.

### 3.4. Metadata disk failure

- **FsImage** and **EditLog** are critical; their corruption renders HDFS non-functional.
- NameNode can be configured to maintain **multiple copies** of FsImage and EditLog, updated **synchronously**.
- But: the NameNode is still a **Single Point of Failure** — **automatic failover is NOT supported** in classical HDFS. (Hadoop HA configurations use Quorum Journal Manager + Standby NameNode but that's an extension.)

---

## 4. Data Organization (read/write operations)

### 4.1. HDFS Read

1. Client calls `open()` via the DistributedFileSystem object.
2. DistributedFileSystem asks NameNode for **block locations** of the file.
3. NameNode returns the list of DataNodes (sorted by proximity).
4. Client receives an FSDataInputStream and calls `read()` on it.
5. Client reads blocks directly from DataNodes (no NameNode involvement for data).
6. Client calls `close()`.

### 4.2. HDFS Write

1. Client calls `create()` via the DistributedFileSystem.
2. DistributedFileSystem tells NameNode to create the file in the namespace.
3. NameNode performs checks (file doesn't exist, client has permission) and returns a handle.
4. Client writes via FSDataOutputStream.
5. Data is written in a **pipeline of DataNodes**:
   - Client → DataNode1 → DataNode2 → DataNode3
   - Each forwards packets to the next
6. Each DataNode in the pipeline returns an **ack packet** in reverse order.
7. Client calls `close()`, which flushes remaining packets and tells NameNode the write is **complete**.

The pipeline pattern is what gives HDFS its high write throughput.

---

## Key terms (glossary)

- **Block** — fixed-size chunk that files are split into (default ~128 MB).
- **Replication factor** — number of copies of each block (default 3).
- **Rack awareness** — knowledge of which DataNodes share network proximity.
- **Heartbeat** — periodic liveness signal from DataNode to NameNode.
- **BlockReport** — list of all blocks held on a DataNode, sent to NameNode.
- **FsImage** — namespace snapshot kept by NameNode.
- **EditLog** — log of namespace changes since the last FsImage.
- **Safemode** — initial read-only state during NameNode startup.
- **WORM (Write-Once-Read-Many)** — HDFS's access model.
- **SPOF (Single Point of Failure)** — the NameNode in classical HDFS.

---

## Exam targets

1. **Draw the HDFS architecture** — NameNode, DataNodes (across racks), Secondary NameNode, Client. Indicate which interactions are metadata ops vs block ops.
2. **Explain rack-aware replica placement** with default factor 3 — exact placement rule and why it's a good trade-off.
3. **Describe the read flow** and the **write flow** (pipeline of DataNodes + ack chain).
4. **Explain how HDFS handles DataNode failure** — heartbeat timeout → mark dead → re-replicate.
5. **Why is the NameNode a SPOF?** What mechanisms mitigate that (multiple FsImage/EditLog copies; Secondary NameNode for checkpointing; HA setup with Standby).
6. **Why is HDFS bad for small files?** (Each file = metadata in NameNode RAM → limits cluster size.)
7. **Define WORM and explain why** HDFS uses it.

### Pitfalls
- **Secondary NameNode is NOT a failover backup.** It does checkpointing only. Common student trap.
- Block size ≠ file size — files are *split into* blocks. Last block may be smaller.
- Data integrity is via **client-side checksum**, not server-side.
- Safemode is read-only, no replication occurs during it.
- The NameNode keeps metadata **in RAM** — not disk-bound for normal ops.
