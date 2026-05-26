# Chapter 5 — YARN (Yet Another Resource Negotiator)

## Bird's eye view

- **YARN** was introduced in **Hadoop 2** to fix Hadoop 1's MapReduce-only limitation and JobTracker scaling bottleneck.
- It **decouples resource management from processing logic** — one cluster can now run MR, Spark, Storm, Tez, HBase batch jobs, etc.
- Architecture has **4 actors**: ResourceManager (global), NodeManager (per node), ApplicationMaster (per app), Container (resource bundle).
- Application lifecycle = 8 well-defined steps from `hadoop jar` submission to AM unregistration.
- 3 scheduling policies: **FIFO** (simple, single-tenant), **Capacity** (queues + minimum guarantees, multi-tenant), **Fair** (equal share via pools, supports preemption).
- 4 failure types to know: **ResourceManager** (still effectively SPOF), **NodeManager**, **ApplicationMaster**, **Container**.
- Some characterize YARN as a "large-scale distributed OS for big data."
- Admin tools: YARN Web UI on port 8088, Hue Job Browser, YARN CLI.

---

## 1. Introduction & Motivation

### 1.1. What YARN is

- **YARN (Yet Another Resource Negotiator)** = cluster resource management system for Hadoop.
- Introduced in **Hadoop 2** to overcome MR limitations and JobTracker scheduling problems.
- Acts as a **connecting link** between high-level applications (Spark, HBase, etc.) and the low-level Hadoop environment.
- With YARN, Hadoop evolves from a "MapReduce framework" into a **general big data processing core**.

### 1.2. Why we need YARN — the Hadoop 1 problems

Hadoop 1 architecture had:
- **Job Tracker** (in master node): sends user tasks to task trackers, applies MR, tracks tasks, redirects on failure.
- **Task Tracker** (in slave nodes): runs tasks given by Job Tracker; co-located with DataNode.

#### Limitations of MapReduce 1

| Issue | Detail |
|---|---|
| **Scalability bottleneck of JobTracker** | Could manage up to ~4,000 nodes and ~40,000 concurrent tasks. Beyond this, latency rises and failures multiply. |
| **Single application type** | 100% of cluster compute dedicated to MR jobs — other apps (Spark, HBase) couldn't share resources. |
| **Static resource allocation** | Each task got a fixed memory amount (often 1 GB). A task needing only 300 MB wastes 700 MB × thousands of tasks. |
| **Job failures & recovery** | ~2% node failure per day on a 2,000-node cluster. MRv1 re-runs failed tasks but lacks efficient fault-tolerant scheduling. |
| **Poor multi-tenancy** | No queue-level isolation or priority management. One user could monopolize the entire cluster. |

### 1.3. YARN's design goals

| Goal | Detail |
|---|---|
| **Scalability** | Efficient on networks of thousands of nodes |
| **Support for programming models** | Beyond MR — Spark, Tez, MPI, etc. |
| **High cluster utilization** | Fine-grained resources (RAM, CPU, disk) via containers |
| **Multitenancy** | Pluggable schedulers (e.g., Capacity Scheduler) allow secure resource sharing among orgs |
| **Locality awareness** | Move computation to data (cheaper than moving data) — minimizes network congestion |

---

## 2. YARN Architecture

### 2.1. The four actors

| Actor | Role | Lifetime |
|---|---|---|
| **ResourceManager (RM)** | Global authority for resource allocation in the cluster | Cluster lifetime |
| **NodeManager (NM)** | Per-node agent; manages containers and reports node health | Cluster lifetime |
| **ApplicationMaster (AM)** | Per-application coordinator; framework-specific; negotiates containers, monitors tasks | Application lifetime |
| **Container** | A resource bundle (RAM + CPU + disk) on a single node | Task lifetime |

### 2.2. ResourceManager (RM) — global authority

- The **ultimate authority** in resource allocation.
- Two major components: **Scheduler** + **Application Manager**.
- Optimizes cluster utilization (keeps resources busy subject to constraints).

#### ResourceManager internal sub-components:
- **Scheduler** — allocates resources; uses pluggable policy (Capacity, Fair, FIFO).
- **ApplicationMasterService** — interacts with every AM regarding resource/container negotiation.
- **ApplicationMasterLauncher** — responsible for launching the AM container on job submission.
- **Admin and Client services** — handle job submissions, restarts.
- **Security component** — generates ApplicationToken and ContainerToken.
- **ResourceTrackerService** — coordinates with NodeManager.

#### ResourceManager: Scheduler
- Allocates resources to running applications based on their resource requirements.
- Pluggable policy plug-in partitions cluster resources across applications.
- Two main pluggable scheduler implementations: **Capacity Scheduler** and **Fair Scheduler**.

#### ResourceManager: Application Manager
- Accepts job submissions.
- Negotiates the **first container** from RM for executing the application-specific AM.
- Restarts AM container on failure.

### 2.3. NodeManager (NM) — per-node agent

- Manages user jobs and workflow on its node.
- Registers with the RM and sends **heartbeats** with node health.
- Manages **application containers** assigned to it by the RM.
- Creates the requested container process; monitors resource usage (memory, CPU).

#### NodeManager internal sub-components:
- **NodeStatusUpdater** — communication with the RM.
- **ContainerManager** — core component; manages all containers on the node.
- **ContainerExecutor** — interacts with native hardware/software to start/stop containers.
- **NodeHealthCheckerService** — monitors node health; sends heartbeats to RM via NodeStatusUpdater.
- **Security** — manages Access Control Lists (ACLs) and access tokens.

### 2.4. ApplicationMaster (AM) — per-application coordinator

- Each app has its own AM (framework-specific).
- Coordinates the app's execution; manages faults.
- Negotiates resources from RM; works with NM to execute/monitor tasks.
- Sends periodic heartbeats to RM with health + resource demands.
- **An "application" = a single job submitted to the framework.**

### 2.5. Container — resource bundle

- A collection of physical resources (RAM, CPU cores, disk) on a single node.
- Managed by a **Container Launch Context (CLC)** describing:
  - Environment variables
  - Dependencies (in remote storage)
- Grants the application rights to use a specific amount of resources on a specific host.

---

## 3. Application Workflow in Hadoop YARN

### 3.1. The 8-step lifecycle (you should memorise this)

1. **Client submits** an application to the ResourceManager (`hadoop jar` command).
2. **ResourceManager allocates a container** to start the **Application Master**.
3. **Application Master registers** with the ResourceManager.
4. **AM asks for containers** from RM (resource requests).
5. **AM notifies NodeManagers** to launch those containers.
6. **Application code runs** in the containers (the actual work).
7. **Client contacts RM** to monitor application status.
8. **AM unregisters** with the RM when work is done; containers are released.

### 3.2. The same lifecycle described differently (slide phrasing)

- **Step 1 — Client submits.** Submission via `hadoop jar`. RM maintains the list of apps + available NM resources.
- **Step 2 — RM allocates container.** Scheduler picks a container; AM is started and is responsible for that app's lifecycle. AM then sends resource requests to RM.
- **Step 3 — AM contacts NM.** Asks NM to use the allocated container to launch app-specific tasks.
- **Step 4 — RM launches containers.** AM negotiates containers throughout the app's life; monitors progress; restarts failed tasks; reports to client.
- **Step 5 — Container executes the AM-managed task.** Once app is complete, AM shuts down and releases its own container.

---

## 4. Scheduling and Resource Allocation

The RM acts as a **pluggable global scheduler**. Three policies:

### 4.1. FIFO Scheduler

- Apps queued in order of submission.
- First app's requests satisfied; then the next; and so on.
- Simple, no configuration needed.
- **Not suitable for shared clusters** — large apps monopolize the cluster while others wait.
- On a shared cluster, prefer **Capacity** or **Fair**.

### 4.2. Capacity Scheduler

- Designed for **shared multi-tenant clusters** in an operator-friendly way.
- Maximizes throughput and utilization.
- Cluster resources are **partitioned into queues**, each with a **minimum capacity guarantee** for the funding org.
- An org can **access excess capacity not used by others** → elasticity, cost-effective.

### 4.3. Fair Scheduler

- Assigns resources so that all jobs get, **on average, an equal share over time**.
- Single job → uses entire cluster.
- New jobs → as slots free up, they're assigned to newcomers so each job gets roughly equal CPU time.
- Organizes jobs into **pools** — by default, one pool per user.
- Within each pool, jobs scheduled either fair-share or FIFO.
- Supports **preemption** — if a pool's minimum share is not met for some time, the scheduler can kill tasks from other pools to make room.

### 4.4. Quick comparison

| Feature | FIFO | Capacity | Fair |
|---|---|---|---|
| Multi-tenant | ❌ | ✅ queues + min guarantee | ✅ pools |
| Elasticity | ❌ | ✅ | ✅ |
| Preemption | ❌ | Limited | ✅ |
| Config complexity | None | Medium | Medium-high |

---

## 5. YARN Administration — Failures

| Failure type | What happens |
|---|---|
| **ResourceManager failure** | RM stores cluster state (app metadata, container info, configs). HW failure means manual debug + restart. **Cluster unavailable** during downtime; all jobs need restart on recovery → half-completed jobs lose data. |
| **NodeManager failure** | RM detects via missed heartbeats (timeout). NM is removed from the pool; all containers on that node are killed; the failure is reported to all running AMs. AMs are responsible for redoing the work. A new NM joining triggers AM notification of new resources. |
| **ApplicationMaster failure** | RM starts another container with a new AM for another **application attempt**. The new AM must recover the old AM's state (possible only if AMs persist state externally). Otherwise the app runs from scratch. |
| **Container failure** | The AM handles container failures (application framework manages them; YARN supplies the info). When a container finishes, RM notifies the AM, which interprets the exit status. AM tracks job-container failures and decides retries. Allocate API call returns no containers until ready. |

---

## 6. Development and Administration Tools

| Tool | Use |
|---|---|
| **YARN Web UI** (port **8088** by default) | View jobs; cannot control/configure from here. |
| **Hue Job Browser** | Monitor status; kill jobs; view logs. |
| **YARN Command Line** | Most YARN commands are admin-oriented (not developer). |

---

## 7. The Big Picture — YARN in the Hadoop stack

A typical Hadoop 2.0 ETL stack:

```
Source DB → Extract → HADOOP 2.0 ( MapReduce/Spark + YARN + HDFS ) → Transform → Load → Data Mart → BI Reports
```

YARN sits between storage (HDFS) and processing frameworks (MR, Spark, etc.) — it allocates and orchestrates resources for whichever processing engine the user chose.

---

## Key terms (glossary)

- **ResourceManager (RM)** — global cluster-level resource arbitrator.
- **NodeManager (NM)** — per-node container manager.
- **ApplicationMaster (AM)** — per-application coordinator.
- **Container** — bundle of resources on a node (RAM, CPU, disk).
- **Scheduler** — RM sub-component that allocates resources; FIFO/Capacity/Fair.
- **Application Manager** (capital A — distinct from AM) — RM sub-component that accepts submissions.
- **Heartbeat** — periodic liveness signal (NM ↔ RM, AM ↔ RM).
- **Preemption** — Fair Scheduler ability to kill tasks to free resources.

---

## Exam targets

1. **Why was YARN introduced?** — list 3-4 Hadoop 1 problems it solves (scalability bottleneck, MR-only, static allocation, poor multi-tenancy).
2. **Sketch the YARN architecture** — RM (with Scheduler + AppMgr), multiple NMs, AM per app, containers.
3. **Describe the 8-step application workflow.**
4. **Compare the 3 schedulers** — when to use which.
5. **Describe how YARN handles each of the 4 failure types.**
6. **Explain the difference between Application Master (per app) and Application Manager (sub-component of RM)** — students confuse these.
7. **Explain how YARN enables multiple frameworks on Hadoop** — separation of resource management from processing logic.

### Pitfalls
- **ApplicationMaster ≠ Application Manager**. AM is per-app; AppMgr is inside the RM.
- **YARN replaces JobTracker** but Hadoop 2.0 still runs MR — MR now runs *on top of* YARN.
- RM is essentially a **SPOF** (manual recovery). HA setups use Active+Standby RM with shared state, but classic YARN is single RM.
- The **Application Manager** (inside RM) handles **first container** for the AM — easy to confuse with the AM's role of requesting containers for its tasks.
- Default scheduler is **Capacity** in modern Hadoop distributions; older docs may say FIFO.
