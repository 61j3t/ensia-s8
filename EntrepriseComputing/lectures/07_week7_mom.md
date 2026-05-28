# Week 7 — Message-Oriented Communication in Distributed Systems

## Bird's eye view

- **MOM (Message-Oriented Middleware)** is the infrastructure layer that lets distributed components exchange messages **asynchronously** through a persistent intermediary, breaking the synchronous lock of RPC.
- RPC couples caller and receiver in **time** (both must be active simultaneously) and in **space** (both must share direct addressing). MOM eliminates both constraints, adding a third decoupling: **synchronization** — the sender fires-and-forgets.
- Two fundamental primitives: **Message Queue** (point-to-point, 1-to-1) and **Publish/Subscribe** (broadcast, 1-to-many via an Event Dispatcher). Modern systems hybridize both.
- The **Message Broker** sits in the middle, holds queuing and routing logic, converts between heterogeneous formats, and can route messages dynamically using a rules repository.
- **Delivery semantics** are not free: at-most-once, at-least-once, exactly-once, and transaction-based each carry distinct trade-offs; idempotent consumers are required under at-least-once.
- Key products: **Amazon SQS** (managed polling queue), **RabbitMQ** (broker with Exchange/Queue push model, AMQP/MQTT), **Apache Kafka** (distributed partitioned append-only log, pull model, replay). Choice depends on data volume, routing complexity, and replayability requirements.

---

## Detailed notes

### 1. Why MOM? — The bottleneck of synchronous systems

**RPC and sockets** improve distribution transparency but preserve two critical couplings:

| Coupling type | Description |
|---|---|
| **Coupled in Time** | The caller halts and waits for a reply; both sender and receiver must be active simultaneously. |
| **Coupled in Space** | Communication requires direct, shared addressing — each side must know the other's identity and location. |

The consequence in microservice architectures: functionality and communication are inextricably linked, limiting scalability and resilience. A single slow or offline downstream service blocks the entire call chain.

**MOM breaks all three couplings:**

| Decoupling dimension | What it means |
|---|---|
| **Space decoupling** | Senders do not need to know the location or identity of receivers. |
| **Time decoupling** | Senders and receivers operate on their own independent schedules; neither needs to be online simultaneously. |
| **Synchronization decoupling** | Senders can resume execution immediately after `put`, rather than waiting for an acknowledgement. |

MOM diagram (described): Multiple heterogeneous producers (Java monolith, Python serverless function, SQL database) all push messages into a central MOM layer (cylinders and envelopes icon). From there, solid arrows deliver to an always-on Real-time Analytics Engine, dashed arrows queue for a Batch Processing Service that is currently waiting, and dotted arrows are held for an Archival System that is idle — each consumer runs at its own pace.

MOM is defined formally as **communication middleware that supports sending and receiving messages in a persistent manner** through intermediate-term storage. This contrasts with **transient communication** (Berkeley sockets, MPI) where a message is discarded if the receiver is not actively executing at transmission time.

---

### 2. The two core primitives

#### 2.1 Message Queue (Point-to-Point, 1-to-1)

Diagram described: multiple producers label messages 1, 2, 3 and insert them into an orange cylindrical queue. On the right, three separate consumers each receive exactly one message. Each message is consumed by **exactly one** consumer and is then removed.

Queue API:

```text
put    : Non-blocking send. Inserts message into the queue; sender immediately releases resources.
get    : Blocking receive. Halts the receiver until the queue is non-empty, then extracts a message.
poll   : Non-blocking receive. Checks the queue and returns a message if available, never blocks.
notify : Event-driven receive. Installs a callback handler triggered automatically when a new message arrives.
```

**Message Queue API Lifecycle** (four-step diagram described):
1. **put** — Sender inserts a message; sender's resources are freed immediately; receiver is not yet involved.
2. **get** — Receiver blocks waiting on the queue until a message appears.
3. **poll** — Receiver checks the queue without blocking; returns immediately whether or not a message is present.
4. **notify** — A dormant receiver registers a callback; the queue fires the callback upon message arrival (event-driven).

**Key properties:**
- Messages are stored until retrieved; a successfully processed message is removed.
- Multiple producers and consumers can share one queue, but each specific message goes to exactly one consumer.
- **Selection logic:** FIFO by default, or consumer can search for specific message types.
- **Primary use cases:** task scheduling, load balancing, distributing worker workloads across a pool.
- Managed at scale by **Queue Managers (QMs)**.

#### 2.2 Publish/Subscribe (1-to-Many)

Diagram described: three Publishers each emit events labelled a, b, c into a central **PubSubService (Event Dispatcher)**. The dispatcher holds a routing table:

| Topic | Subscribers |
|---|---|
| a | S1, S3 |
| b | S2 |
| c | S2, S3 |

The dispatcher fans out: S1 receives `a`; S2 receives `b` and `c`; S3 receives `a` and `c`. Publishers have no knowledge of who, if anyone, is subscribed.

Pub/Sub API:

```text
publish(event)                        : Broadcasts event with metadata to the dispatcher.
subscribe(filter_expr, notify_cb, expiry) : Registers interest; returns a subscription handle.
notify_cb(sub_handle, event)          : Callback executed by the system upon a match.
```

**Key properties:**
- **One-to-many** broadcast; a single event reaches all matching subscribers.
- Publishers emit without specific destinations — high dynamic decoupling.
- Subscriptions are filter-based (by topic name or content expressions).
- The Event Dispatcher is typically distributed for scalability.
- **Primary use cases:** event notifications, logging/tracing, dynamic component integration, real-time fan-out.

#### 2.3 Queue vs. Pub/Sub — side-by-side

| Feature | Message Queue (1-to-1) | Publish/Subscribe (1-to-many) |
|---|---|---|
| Communication model | Point-to-point | Broadcast |
| Primary entities | Producers and Consumers | Publishers and Subscribers |
| Storage/Routing | Queue stores until a single consumer retrieves | Event Dispatcher matches events to subscription filters |
| Selection logic | FIFO or message-type search | Topic name or subscription filter expressions |
| Scalability | Queue Managers (QMs) | Distributed Event Dispatcher |
| Typical use case | Task scheduling, load balancing | Event notifications, logging, dynamic integration |
| Message fate | Removed after successful consumption | Delivered to all matching subscribers simultaneously |

---

### 3. Delivery semantics

Delivery guarantees are not free; choosing one shapes the entire consumer design.

#### 3.1 At-Least-Once Delivery

- The queue **resends** the message if it does not receive an ACK within a timeout.
- **Impact:** the same message may arrive multiple times. Consumers **must be idempotent** — processing the same message twice must produce the same outcome as processing it once.
- Default mode in Kafka and SQS Standard queues.

#### 3.2 Exactly-Once Delivery

- The system automatically **deduplicates** using unique Message IDs mapped from sender to receiver.
- **Impact:** no duplicates, but higher overhead. Used by SQS FIFO queues and optionally by Kafka with idempotent producers and atomic writes (at the cost of higher latency and lower throughput).

#### 3.3 Transaction-Based Delivery

- The MOM and receiver participate in an **ACID transaction**: the "read" and "delete" operations are atomically coupled.
- **Impact:** the message is removed from the middleware only if the receiver successfully completes its task. If the transaction rolls back, the message remains available for redelivery.

#### 3.4 Timeout-Based Delivery (Visibility Timeout)

- A message is marked **invisible** to other consumers the moment one consumer reads it. A clock starts.
- If an ACK arrives before the timer expires, the consumer explicitly deletes the message.
- If the timer expires (consumer crashed or is hung), the message **reappears** for another consumer to claim.
- This is the mechanism used by **Amazon SQS** (default visibility timeout: 30 seconds).

At-Most-Once delivery (fire-and-forget): the broker does not wait for an ACK and does not retry. Suitable for non-critical telemetry where some loss is acceptable.

---

### 4. The Message Broker and overlay networks

#### 4.1 The Message Broker

Diagram described: a Source Application connects through an Interface into a Message Broker box. Inside the broker: a **Queuing Layer** (receives and buffers messages) feeds into a **Broker Plugins/Rules** engine backed by a **Rules Repository**. The rules engine transforms the message (square to circle icon) and forwards it to the Destination Application via its Interface.

**Problem:** distributed applications rarely share a common data format. A Java monolith, a Python function, and a Go service all speak different serialization dialects.

**Solution:** the broker acts as an **application gateway and transformation engine**:
- Converts incoming messages to target formats (access transparency).
- Manages a repository of conversion rules.
- Provides **subject-based dynamic routing** — routes change at runtime without restarting producers or consumers.

#### 4.2 Queue Managers and Overlay Networks

Diagram described: two sender applications (A and B) each have a local Send or Receive Queue managed by a **Source Queue Manager (QM)**. The QM queries an **Address Lookup Database** to find the physical contact address of the Destination QM. Messages hop through a mesh of **Router/Queue Managers (R1, R2, R3)** that form a logical **overlay network** on top of the physical infrastructure. Receiver applications C and D each read from their local Receive Queue via a Destination QM.

Key concepts:
- **Applications interact only with their local QM** — they address queues by logical name, never by physical IP.
- QMs maintain **routing tables** that dynamically map logical queue names to physical contact addresses.
- QMs relay messages to each other across the overlay, so the physical path may differ from the logical path.
- Manual routing tables work for small, static setups; dynamic routing is required for large, evolving environments.

---

### 5. Framework landscape

Modern frameworks rarely stick to a pure queue or pure pub/sub model; most hybridize. Two broad categories:

| Category | Products |
|---|---|
| Self-hosted / Open Source | Apache Kafka, RabbitMQ, ActiveMQ, ZeroMQ |
| Fully Managed Cloud | Amazon SQS / SNS, Google Cloud Pub/Sub, Microsoft Azure Service Bus |

---

### 6. Framework deep dive: Amazon SQS

Amazon SQS is a **fully managed, polling-based** (pull model) queue service.

**Architecture:** SQS replicates message copies across multiple servers within an AWS Region for high availability. A logical "queue" is actually a distributed set of redundant storage nodes — the diagram shows messages A, B, C, D, E scattered across many cylindrical server icons, all belonging to one logical queue.

**Message lifecycle (Visibility Timeout mechanism):**
1. Consumer pulls Message A; SQS immediately marks A invisible (clock starts, default 30 s).
2. Consumer processes Message A while the clock counts down.
3a. **Success path:** consumer explicitly deletes A — message is gone permanently.
3b. **Failure/timeout path:** clock reaches zero; A reappears in the queue for another consumer to claim.

This implements **at-least-once delivery** automatically — no consumer code changes needed for crash recovery.

**Queue types:**

| | Standard Queue | FIFO Queue |
|---|---|---|
| Ordering | Best-effort (messages may arrive out of order) | Strict in-order (FIFO) processing |
| Delivery | At-least-once (duplicates possible) | Exactly-once (no duplicates) |
| Throughput | Nearly unlimited | Reduced (throughput cap applies) |
| Use case | Maximum scale, tolerate duplicates | Ordered workflows, financial transactions |

**Technical constraints:**
- Max payload: **256 KB** per message. Larger payloads must store the data in S3 and pass a reference in the message.
- Max retention: **14 days**.

**Amazon SNS** (Simple Notification Service) complements SQS for the pub/sub pattern within the AWS ecosystem: SNS topics fan out to SQS queues, Lambda functions, HTTP endpoints, etc.

---

### 7. Framework deep dive: RabbitMQ

RabbitMQ implements a traditional **push-based broker** model using the AMQP protocol (also supports MQTT for IoT).

**Architecture (diagram described):** Producers never publish directly to a queue. Instead, they send to an **Exchange** (diamond shape). The Exchange acts as an intelligent router, applying **Bindings** and **Routing Keys** to push the message into one or more destination **Queues**. Consumers receive messages pushed from their queue.

```
Producer --> Exchange --[binding/routing key]--> Queue --> Consumer
                    \--[binding/routing key]--> Queue
```

**Exchange types** (routing algorithms):
- **Direct:** route by exact routing key match.
- **Fanout:** route to all bound queues regardless of key (broadcast).
- **Topic:** route by wildcard pattern matching on routing key.
- **Headers:** route by message header attributes.

**Key properties:**
- **Push model** — broker pushes messages to consumers; consumers do not poll.
- FIFO ordering guarantees **at the queue level**.
- Messages are **ephemeral by default** (deleted on ACK); durable queues and persistent messages require explicit configuration.
- Supports complex routing out of the box through Exchange/Binding logic.
- **Primary strength:** rich, flexible routing algorithms; strong support for legacy enterprise protocols (AMQP, MQTT, STOMP).

---

### 8. Framework deep dive: Apache Kafka

Kafka is a **distributed, partitioned, append-only log** — a fundamentally different paradigm from traditional queues.

#### 8.1 The log paradigm

Diagram described: a horizontal tube representing a single partition, with numbered slots 0 through 8 from left to right. An arrow points down at slot 4 labelled "Offset." New messages append to the right end. The log is immutable — messages are never modified or deleted on read.

- Messages are **not deleted upon read**; they are retained for a configurable **retention period** (e.g., one week) or until a size limit is reached.
- Multiple independent consumers can read the same log at different positions — enabling **message replay**.
- Each consumer (or consumer group) maintains its own **offset** — a sequence number indicating the last message it has read.

#### 8.2 Topics, partitions, and consumer groups

- **Topic:** a named logical feed of messages, analogous to a subject or category.
- **Partition:** each topic is split into N partitions distributed across a cluster of **brokers**. Partitioning enables **massive parallel reads and writes**.
- Ordering guarantee: **total order within a single partition**; no cross-partition ordering.
- **Consumer group:** a set of consumers that collectively consume a topic. Each partition is assigned to exactly one consumer in the group at a time — this implements **load balancing** (queue-like, 1-to-1 per partition). Multiple independent groups can each consume the same topic simultaneously — this implements **pub/sub** (1-to-many across groups).

**Kafka lifecycle:**
1. **Producer** publishes a message to the leader broker of a specific topic partition.
2. Message is appended to the immutable log and assigned a unique **offset**.
3. **Consumers** retrieve messages by specifying their current offset. Kafka does not remove the message.

#### 8.3 Kafka delivery semantics

| Semantic | Mechanism | Trade-off |
|---|---|---|
| At-least-once (default) | Wait for ACK from partition leader; retry on timeout | No message loss; duplicates possible |
| Exactly-once | Idempotent producers + atomic writes | No loss, no duplicates; higher latency, lower throughput |
| At-most-once | No retry on failure | Possible message loss; lowest latency |

**Partition-level ordering** is a total order guarantee. Cross-partition ordering is not guaranteed; design partition keys carefully to co-locate related events in the same partition.

#### 8.4 Kafka infrastructure

- **Brokers:** the server nodes that store partitions and serve producer/consumer requests.
- **ZooKeeper (legacy) / KRaft (modern):** cluster coordination and metadata management. KRaft removes the ZooKeeper dependency in newer Kafka versions.
- Performance is **effectively constant** regardless of stored data size — Kafka reads from disk sequentially, which is I/O-efficient.

---

### 9. Transient vs. persistent communication

| | Transient | Persistent (MOM) |
|---|---|---|
| Intermediate storage | None | Yes — middleware buffers messages |
| Receiver availability | Must be active during transmission | Does not need to be active |
| Message on receiver down | Discarded | Held by middleware until receiver is ready |
| Coupling | High (time and space) | Low (time, space, synchronization) |
| Examples | Berkeley Sockets, MPI, traditional RPC | Kafka, RabbitMQ, IBM MQ, Amazon SQS |

**Extended persistence (Kafka):** messages survive delivery and remain on disk for the full retention period, allowing consumers to re-read ("replay") historical data. This goes beyond standard persistence, where messages are discarded immediately after a successful consumer ACK.

---

### 10. Architect's matrix: choosing a framework

| | Amazon SQS | RabbitMQ | Apache Kafka |
|---|---|---|---|
| **Architecture** | Cloud-native managed queue | Broker with Exchanges | Distributed partitioned log |
| **Delivery model** | Polling (pull) | Push-based | Pull-based (sequential read) |
| **Persistence** | Ephemeral (deleted on ACK) | Ephemeral (deleted on ACK) | Persistent (retained by time/size; replayable) |
| **Ordering** | Best-effort (Standard) / strict FIFO (FIFO queue) | FIFO at queue level | Total order per partition |
| **Primary strength** | Infinite scale, zero maintenance | Complex routing, flexible protocols | Massive data streams, event sourcing, parallel I/O |
| **Delivery semantic** | At-least-once (Standard) / exactly-once (FIFO) | Configurable | At-least-once default; exactly-once optional |
| **Choose when** | Simple task queues, AWS-native workloads | Rich routing logic, legacy protocol support, MQTT/IoT | High-throughput streaming, event sourcing, replay needed |

**Design rule:** choice is dictated by three axes — **data volume**, **routing complexity**, and **requirement for message replayability**. Do not use Kafka for simple task queues; do not use SQS for replayable event streams.

---

### 11. Core architectural takeaways

1. **Decoupling is the goal.** MOM breaks synchronous bottlenecks, decoupling systems in time, space, and synchronization.
2. **Semantics define design.** There is no magic-bullet delivery mode. Choose at-least-once / exactly-once / timeout-based deliberately, and always design consumers to be idempotent.
3. **Match tool to topology.** Align the middleware's internal architecture (broker vs. log) with the system's exact routing and throughput requirements.

---

## Key terms

- **MOM (Message-Oriented Middleware)** — infrastructure supporting asynchronous, persistent message exchange to enable loose coupling.
- **Message Queue** — a persistent FIFO buffer; each message is consumed by exactly one consumer and then removed.
- **Publish/Subscribe** — a 1-to-many pattern where an Event Dispatcher routes events to all matching subscribers.
- **Event Dispatcher** — the component in a pub/sub system that matches published events to subscription filters; typically distributed for scalability.
- **Exchange (RabbitMQ)** — the routing component that producers publish to; it applies binding/routing-key rules to forward messages to queues.
- **Topic (Kafka)** — a named, partitioned, append-only log of messages.
- **Partition (Kafka)** — a subdivision of a Kafka topic stored on a single broker; enables parallel I/O.
- **Offset (Kafka)** — a consumer-maintained sequence number indicating the last message read from a partition.
- **Consumer Group (Kafka)** — a set of consumers that together consume a topic, each partition assigned to one member; enables both load-balancing and pub/sub semantics.
- **Queue Manager (QM)** — a service that manages local queues and acts as a relay in an overlay network for routing messages.
- **Overlay Network** — a logical network of QMs built on top of the physical infrastructure to manage message routing.
- **Message Broker** — an application-level gateway that transforms data formats and provides dynamic routing to resolve heterogeneity between systems.
- **At-least-once** — delivery guarantee where the broker retries until it receives an ACK; duplicates are possible.
- **Exactly-once** — delivery guarantee where the system deduplicates using unique message IDs; higher overhead.
- **Idempotency** — the property that processing the same message multiple times yields the same result as processing it once; required under at-least-once.
- **Visibility Timeout** — the period during which a message is hidden from other consumers while one consumer processes it (SQS mechanism).
- **Dead-Letter Queue (DLQ)** — a holding queue for messages that have exceeded the maximum delivery attempts and cannot be processed successfully.
- **Transient communication** — communication with no intermediate storage; message is lost if receiver is unavailable (e.g., sockets, MPI).
- **Persistent communication** — communication where the middleware stores the message until the receiver is ready (e.g., all MOM systems).
- **AMQP** — Advanced Message Queuing Protocol; open wire protocol used by RabbitMQ.
- **MQTT** — lightweight pub/sub protocol optimized for IoT devices with constrained bandwidth; supported by RabbitMQ.

---

## Exam targets

1. **Explain the three decoupling dimensions** of MOM (time, space, synchronization) and contrast with RPC's two couplings. Give a concrete example for each.
2. **Distinguish Message Queue from Pub/Sub**: communication model, cardinality, message fate, routing logic, and typical use cases.
3. **Describe the four queue API operations** (`put`, `get`, `poll`, `notify`) — which are blocking, which are non-blocking, and what each does.
4. **Compare the four delivery semantics** (at-least-once, exactly-once, transaction-based, timeout-based) — mechanism, impact on consumer design, and which systems use each.
5. **Explain idempotency**: why it is required under at-least-once, and give a design strategy to achieve it (e.g., message ID deduplication table).
6. **Draw the SQS Visibility Timeout lifecycle**: message pulled → hidden → ACK deletes / timeout reappears. Identify what queue type this gives (at-least-once).
7. **Explain RabbitMQ's Exchange mechanism** — why producers never publish directly to a queue, how bindings and routing keys work, and name the four exchange types.
8. **Describe Kafka's log paradigm**: append-only, immutable, offset-tracked, consumer-group model, partition-level ordering, and replay capability.
9. **Fill in the Architect's Matrix**: compare SQS, RabbitMQ, and Kafka across architecture, delivery model, persistence, ordering, and primary strength.
10. **Explain the message broker's role** in heterogeneous environments: format transformation, rules repository, dynamic subject-based routing.
11. **Contrast transient vs. persistent communication** with examples. Explain extended persistence in Kafka.

---

## Pitfalls

- **"MOM gives exactly-once by default"** — false. At-least-once is the default in both SQS Standard and Kafka. Exactly-once requires explicit configuration and carries overhead.
- **"Kafka is a message queue"** — Kafka is a **distributed log**, not a traditional queue. Messages are not deleted on read; consumers track offsets themselves. Calling it a queue will cost marks.
- **"Pub/Sub means Kafka; Queue means RabbitMQ"** — both Kafka and RabbitMQ support both patterns. Kafka uses consumer groups to emulate queues; RabbitMQ uses fanout exchanges for pub/sub.
- **"The Event Dispatcher in Pub/Sub is always a single node"** — the dispatcher is typically **distributed** for scalability.
- **"The Secondary NameNode"-style trap for QMs** — QMs are not just backup brokers; they are routing relays forming the overlay network. An application can only interact with its **local** QM.
- **"SQS preserves strict message order"** — only **FIFO queues** guarantee ordering. Standard queues offer best-effort ordering and allow duplicates.
- **"Exactly-once in Kafka is free"** — it requires idempotent producers + atomic writes, which reduces throughput. Do not claim it is the default.
- **"Visibility Timeout = message deleted"** — the message is only **hidden**, not deleted. It reappears if the ACK is not received before the timer expires.
- **"Back-pressure is automatic in all MOM systems"** — in pull-based systems (Kafka, SQS) consumers naturally apply back-pressure by controlling their read rate. In push-based systems (RabbitMQ) explicit flow control or prefetch settings must be configured.
- **"Message TTL and DLQ are the same"** — TTL controls how long a message lives before expiry; a DLQ is where expired or undeliverable messages are sent for inspection. They are complementary, not synonymous.
