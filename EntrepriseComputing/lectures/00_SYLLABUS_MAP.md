# Enterprise Computing — Syllabus Skeleton (Phase 0)

> Course: Enterprise Computing — ENSIA (Dr. KamelEddine HERAGMI, 2025/2026)
> Use this as a **mental scaffold** + quick-scan map. Each week has its own detailed notes file (`0X_weekX_topic.md`).

## The arc of the course

This course walks the **full enterprise stack**, from architectural philosophy down to runtime operations:

- **Weeks 1–3 — Foundations**: what enterprise computing is, the inherent difficulty of distribution, and SOA as the dominant integration paradigm.
- **Weeks 4–5 — Build & Run**: containers (Docker) and their orchestration (Kubernetes + Minikube hands-on).
- **Weeks 6–9 — Run a real business**: data management, asynchronous messaging, security & identity, observability.
- **Recurring threads**:
  - Trade-offs as engineering: **CAP** (W2), **ACID vs BASE** (W6), **SOA vs Microservices** (W3 vs W4-5), **at-least-once vs exactly-once** (W7).
  - Decoupling: **services** (W3), **containers** (W4), **messages** (W7), **identity tokens** (W8).
  - Observability everywhere: monitoring, logs, tracing, metrics (W9) tie back to reliability (W2) and security audit (W8).

---

## Week 1 — Introduction to the Enterprise Landscape (61 pp)
- **Enterprise computing** = large-scale, distributed, mission-critical IT systems.
- **5 core characteristics**: Scalability · Reliability · Security · Integration · Maintainability.
- **System categories**: ERP (SAP/Oracle), CRM (Salesforce), SCM, Financial/Banking.
- **5 evolutionary eras**: Mainframes (1950s) → PCs (1980s) → Client-Server/Internet (1990s) → Cloud (2010s, IaaS/PaaS/SaaS) → Edge/Quantum.
- **Architecture theory**: 3 structure categories — static/module (decomposition, dependency, data model), dynamic/C&C (service, concurrency), allocation (deployment, work assignment). Distinction: **SA vs SysA vs EA** scope.
- **6 architecture patterns**: Layered (Presentation/BL/DAL), MVC, Microservices (API Gateway), Event-Driven (broker-based), Hexagonal (ports & adapters), Clean Architecture (concentric inward-only dependencies).

## Week 2 — Distributed System Design (54 pp)
- **High Availability**: 99.9% HA standard ≈ 8.76 h downtime/year. Strategies: **redundancy, load balancing, failover, monitoring**.
- **Availability vs Reliability vs Fault Tolerance** — different but interrelated guarantees.
- **Consistency spectrum**: Strong/Linearizability → Causal → Eventual → Weak. Client-centric variants: Read-Your-Writes, Monotonic Reads, Session.
- **Mechanisms**: gossip, read repair, anti-entropy, quorum, 2PL, CRDTs, vector clocks.
- **CAP theorem**: P is mandatory; real choice is **CP vs AP** (banking = CP, social feed = AP, e-commerce = hybrid).
- **Three NFR pillars**: Maintainability (modularity, testability), Reliability (MTBF, MTTR, uptime %, no SPOF), Fault Tolerance (RAID, microservice isolation, multi-zone).
- **The iron triangle of resilience**: scalability ↔ performance ↔ cost — pick two trade-offs.

## Week 3 — SOA Enterprise Architecture (14 pp)
- **3-actor ecosystem**: Provider / Registry / Consumer, lifecycle **Publish → Find → Bind**.
- **ESB** (Enterprise Service Bus): hub-and-spoke topology; internals = Translation + Routing + Orchestration. **SPOF risk.**
- **5 guiding principles**: Loose Coupling · Abstraction · Interoperability · Granularity · Statelessness — plus the classical 4: Reusability, Discoverability, Composability, Standard Contract.
- **Web-services stack**: **SOAP** envelope (header + body, XML); **WSDL** (interface description); **UDDI** White/Yellow/Green pages. Compared with **REST**.
- **Coordination patterns**: **Orchestration** (centralized BPEL conductor) vs **Choreography** (peer-to-peer event mesh).
- **SOA vs Microservices** — 4-axis comparison (scope, communication, data, deployment); microservices = SOA without the smart bus.

## Week 4 — Virtualization and Containerization (Docker) (31 pp)
- **VMs vs containers**: VMs need a full hypervisor (Type 1/2); containers share the host kernel via Linux **namespaces** (PID/network/mount/UTS/IPC/user) + **cgroups** for resource limits → containers win on CPU efficiency and density.
- **Docker architecture**: client / daemon / registry triad. Images = stack of read-only layers (UnionFS) + 1 ephemeral RW layer. **Copy-on-Write**.
- **Lifecycle**: Build → Ship → Run.
- **Dockerfile**: `FROM RUN COPY ADD WORKDIR EXPOSE CMD ENTRYPOINT ENV VOLUME`; instruction order matters for **build-cache**; **multi-stage builds** for minimal production images.
- **Networking**: bridge, host, overlay drivers; port mapping.
- **Persistence**: named volumes vs bind mounts (containers are ephemeral).
- **Security**: shared-kernel threat model (e.g., CVE-2019-5736); defenses = non-root, `--cap-drop=all`, seccomp, user namespaces.
- **Beyond single-host**: Docker Compose (multi-container), Swarm, Kubernetes; **OCI** standard.

## Week 5 — Kubernetes Orchestration + Minikube (29 + 5 pp)
- **Why K8s?** Docker packages, but doesn't *manage a fleet* at scale (scheduling, lifecycle, scaling).
- **Cluster architecture**:
  - **Control Plane** — API Server, etcd, Scheduler, Controller Manager
  - **Worker Nodes** — Kubelet, Container Runtime, Kube-proxy
- **Workloads**: **Pod** (atomic, shared IP/volumes, sidecar/init patterns); **Deployment + ReplicaSet** (desired-state reconciliation loop) → rolling updates, rollbacks, canary, HPA; DaemonSet, StatefulSet, Job/CronJob.
- **Networking — Service types**: ClusterIP, NodePort, LoadBalancer, Ingress.
- **Storage**: PV / PVC / StorageClass model.
- **Configuration**: ConfigMaps vs Secrets (Secrets are *not* encrypted by default).
- **Minikube**: local single-node cluster; full hands-on flow — namespace → PV → PVC → PostgreSQL Deployment → Service → NGINX web app → persistence verification.
- **kubectl** as the universal CLI.

## Week 6 — Enterprise Data Management (16 + 6 pp)
- **ACID vs BASE**: relational serializability vs NoSQL eventual consistency. **Object-Relational Mismatch** is the root motivation for NoSQL; **Twitter Fan-Out** case study.
- **NoSQL taxonomy** (4 families): **Key-Value** O(1) hash (Redis); **Document** BSON single seek (MongoDB); **Wide-Column** LSM-tree/SSTable (Cassandra); **Graph** index-free adjacency O(1) hop (Neo4j).
- **Sharding**: `Shard_ID = Hash(Key) mod n`; strategies — **Hash / Range / Directory**; the **Resharding Penalty** → why **Consistent Hashing** exists.
- **OLTP vs OLAP**: row-oriented B-tree vs columnar MPP; **ETL vs ELT** (compute on staging vs in-warehouse).
- **Analytical storage evolution**: **Data Warehouse** (Schema-on-Write, Star/Snowflake) → **Data Lake** (Schema-on-Read, swamp risk) → **Lakehouse** (Iceberg/Delta on S3, ACID + cheap storage, compute/storage separation).
- **Polyglot Persistence**: one app, many stores per access pattern; complexity moves to the integration layer.

## Week 7 — Message-Oriented Middleware (MOM) (18 + 6 pp)
- **Why?** RPC couples in time + space; MOM **decouples in time, space, and synchronization**.
- **Transient vs persistent communication** (sockets/MPI vs Kafka/RabbitMQ/SQS).
- **Two core patterns**: **Message Queue** (1-to-1, `put/get/poll/notify`) vs **Publish/Subscribe** (1-to-many, event dispatcher + routing).
- **Delivery semantics**: at-most-once · **at-least-once** (default) · exactly-once (transactional) · timeout/visibility-timeout. Consumers must be **idempotent**.
- **Broker** = format transformer + dynamic router; **Queue Managers** form a logical overlay.
- **Framework comparison**:
  - **Amazon SQS** — Standard vs FIFO; visibility timeout lifecycle; 256 KB msg limit.
  - **RabbitMQ** — Exchange / Binding / Routing-Key push model (AMQP, MQTT for IoT).
  - **Apache Kafka** — append-only log; topics/partitions/offsets/consumer groups; replay; ZooKeeper or KRaft.
- **Selection rule**: data volume × routing complexity × replayability.

## Week 8 — Enterprise Security & Identity (51 pp)
- **Principles**: Least Privilege, Defense in Depth, prevent/detect/respond triad.
- **Credentials & secrets lifecycle**: creation → distribution → storage → monitoring → rotation/revocation. **HashiCorp Vault**.
- **Data protection**: **TLS/HTTPS** (in transit, confidentiality + integrity + server identity), **mTLS** (microservice-to-microservice client identity), **bcrypt / argon2 / scrypt** for passwords at rest, GDPR data minimization.
- **AuthN vs AuthZ**; **SSO** via Identity Provider; **JWT** = Header.Payload.Signature; standard claims; `Authorization: Bearer`.
- **OAuth 2.0 grants**: Authorization Code (+ **PKCE**) · Implicit (deprecated) · Resource Owner Password Credentials · Client Credentials · Refresh Tokens. `state` parameter for CSRF protection.
- **OIDC** = identity layer on OAuth 2.0; `scope=openid` → ID token (who you are) ≠ access token (what you can do).
- **Zero Trust** — verify per-request regardless of network origin. IAM types: Centralized / Federated / Decentralized.

## Week 9 — Observability (75 pp)
- **Observability ⊃ Monitoring**. Monitoring detects ("is it up?"); observability *explains* ("why is it behaving like this?").
- **4 pillars**: **Monitoring**, **Logging**, **Tracing**, **Metrics**.
- **Logs**: severity levels DEBUG / INFO / WARN / ERROR / FATAL; structured JSON; aggregation via **ELK / Loki / Splunk**; rotation + retention; alerting via ElastAlert / Grafana Alerts.
- **Distributed tracing**: span/trace anatomy; context propagation across services; trace waterfall view; sampling. Tools: **OpenTelemetry** (SDK + Collector), Jaeger, Zipkin.
- **Metrics**: **Prometheus** pull model + **PromQL**; **Alertmanager** routing/grouping/silencing; **Grafana** dashboards. Mitigate **alert fatigue**.
- **Infrastructure**: run observability in a dedicated environment, not on prod servers.
- **Load testing as a complement**: Gatling / JMeter / Locust / k6; gradual ramp-up vs sudden spike pitfall; observe under load.

---

## Cross-cutting themes (likely exam targets)
- **Distribution trade-offs** — CAP (W2), ACID vs BASE (W6), strong vs eventual consistency, at-least-once vs exactly-once (W7).
- **Decoupling mechanisms** — services (W3), containers (W4), messages (W7), tokens (W8).
- **The "build it" vs "run it" stack** — Docker (W4) → Kubernetes (W5) → Observability (W9).
- **Architecture comparisons** — SOA vs Microservices (W3 vs W4-5), DW vs Data Lake vs Lakehouse (W6), SQS vs RabbitMQ vs Kafka (W7), Centralized vs Federated vs Decentralized IAM (W8).
- **Failure modes & defenses** — SPOF (W2 + W3 ESB), shared-kernel breakout (W4), Secrets not encrypted (W5), alert fatigue (W9).

---

## Study plan from here
1. ✅ **Phase 0 — Syllabus map** (this file)
2. **Phase 1 — Per week**: read each `0X_*.md` (bird's eye for a quick pass; detailed notes for depth)
3. **Phase 2 — Practice**: be able to sketch architecture diagrams from memory (ESB, K8s control plane, OAuth Authorization Code flow, Prometheus topology); rehearse comparison tables
4. **Phase 3 — Cross-links**: revise the recurring themes above
