# Week 1 — Introduction to the Enterprise Landscape

## Bird's eye view

- **Enterprise Computing** = the IT infrastructure, software, and systems organisations use to support business operations at scale; characterised by large-scale, distributed, mission-critical applications.
- Five non-negotiable characteristics: **Scalability, Reliability, Security, Integration, Maintainability**.
- Historical arc: Mainframes (1950s) → PCs (1980s) → Client-Server & Internet (1990s) → Cloud (2010s) → Edge & Quantum (emerging).
- **Architecture** = the set of structures needed to reason about a system; comes in three flavours — static (modules), dynamic (runtime), allocation (deployment).
- Six common enterprise architecture patterns: Layered, MVC, Microservices, Event-Driven (EDA), Hexagonal, Clean Architecture — each targeting a different quality attribute.
- Key enterprise system categories: ERP (SAP, Oracle), CRM (Salesforce), SCM, and Financial/Banking Systems.
- Modern answers to enterprise challenges: Cloud (AWS/Azure/GCP), microservices, API-first development, DevOps/CI-CD, AI and Big Data analytics.

---

## Detailed notes

### 1. What is Enterprise Computing?

**Definition:** Enterprise Computing refers to the IT infrastructure, software, and systems used by organisations to support business operations at scale. It involves large-scale, distributed, and mission-critical applications.

The scope is fundamentally different from consumer or desktop computing: the focus shifts from a single user's experience to supporting thousands of concurrent users, cross-departmental workflows, regulatory obligations, and multi-year system lifecycles.

#### 1.1 Key Characteristics

| Characteristic | Meaning |
|---|---|
| **Scalability** | Must handle increasing load efficiently (more users, more data, more transactions). |
| **Reliability** | Mission-critical systems must ensure high uptime; failures have direct business/financial consequences. |
| **Security** | Enterprise data (customer records, financials, IP) must be protected from threats — internal and external. |
| **Integration** | Systems must communicate seamlessly; enterprises rarely run a single application. |
| **Maintainability** | Long lifecycle — systems are updated and supported for years, not months. |

#### 1.2 Examples of Enterprise Systems

- **ERP (Enterprise Resource Planning):** SAP, Oracle — unify finance, HR, supply chain, manufacturing under one system.
- **CRM (Customer Relationship Management):** Salesforce — manages customer interactions and sales pipelines.
- **SCM (Supply Chain Management):** Coordinates procurement, logistics, and inventory across partners.
- **Financial and Banking Systems:** Transaction processing, core banking, trading platforms.

#### 1.3 Common Challenges

- Managing large data volumes
- Ensuring high availability and disaster recovery
- Handling security threats
- Dealing with legacy systems (often decades old)
- Cost of infrastructure and ongoing maintenance

#### 1.4 Modern Solutions

| Solution | What it addresses |
|---|---|
| Cloud Computing (AWS, Azure, GCP) | On-demand scalability and elastic resources |
| Microservices Architecture | Decoupled, independently deployable services |
| API-First Development | Standardised integration across systems |
| DevOps & CI/CD Pipelines | Faster, reliable software delivery |
| AI & Big Data Analytics | Data-driven decisions at enterprise scale |

---

### 2. Evolution: From Mainframes to Cloud Computing

The history of enterprise computing is a story of progressively **decentralising** computing power, then **abstracting** it again at a higher level.

#### 2.1 The Era of Mainframes (1950–1970)

- **Centralised Power:** Massive units (e.g., IBM 700 series) housed in dedicated rooms.
- **Access:** Limited to large corporations and government agencies — computing was expensive and rare.
- **Processing:** Batch processing via punch cards; jobs were queued and processed sequentially.
- No interactivity; results returned hours later.

#### 2.2 The Rise of the PC (1980–1990)

- **Decentralisation:** Computing power moved to the desktop.
- **Key Players:** Apple II, IBM PC, and the Windows revolution.
- **Impact:** Transformed individual productivity and office work; computing became accessible to individuals.
- Enterprise implication: employees could now have local compute, but data was fragmented.

#### 2.3 Client-Server and The Internet (1990–2010)

- **Connectivity:** Local Area Networks (LAN) and the World Wide Web linked machines.
- **Architecture:** Distributed tasks between "clients" (requesting services) and "servers" (providing services).
- **Shift:** Transition from isolated hardware to networked ecosystems.
- Enterprise systems began centralising shared data on servers while keeping UI on client machines — the classic two-tier or three-tier model.

#### 2.4 The Cloud Era (2010–Present)

- **Abstraction:** Hardware becomes invisible; developers consume resources as services on demand.
- **Scalability:** Elastic resources — scale up for peak load, scale down to save cost (AWS, Azure, Google Cloud).
- **Delivery Models:** IaaS (Infrastructure as a Service), PaaS (Platform as a Service), SaaS (Software as a Service).
- Enterprises can now avoid owning physical data centres entirely.

#### 2.5 The Edge Computing Era (2020–Present)

- **Decentralisation 2.0:** Computation moves closer to the data source (IoT devices, sensors).
- **Low Latency:** Essential for autonomous vehicles, smart cities, and real-time AI inference.
- **Bandwidth Efficiency:** Reduces the volume of raw data that must travel to central clouds.
- Complements cloud rather than replacing it — a hybrid model of edge + cloud.

#### 2.6 The Quantum Frontier (Emerging)

- **Beyond Classical Bits:** Uses *qubits* that exist in superposition (0 and 1 simultaneously).
- **Entanglement:** Linked particles enable massive parallel processing paths.
- **Enterprise Impact:** Revolutionising cryptography (breaking and creating secure protocols), drug discovery, and complex optimisation problems (e.g., logistics).
- Still pre-commercial for most enterprise use cases but increasingly relevant strategically.

---

### 3. Architectures for Enterprise Apps and Patterns

#### 3.1 What is an Architecture?

**Structural definition:** The set of structures necessary for reasoning about a system. It includes:
- The components of the system
- The visible properties of those components
- The dependencies between components

**Process-oriented definition:** The set of "major" and "early" design decisions — the ones that are hardest to change later.

Architecture is an **abstraction**: it hides unnecessary detail and surfaces only what matters for reasoning about system properties (functionality, availability, modifiability, performance, security, etc.).

An architecture can be good or bad. A good architecture allows reasoning about system properties; a bad one prevents it or obscures it.

#### 3.2 Three Types of Structures

**1. Static Structures (Module view)**

Describe how the system is decomposed into units of code.

| Static Structure | Description | Targeted Quality |
|---|---|---|
| **Decomposition** | "Is a submodule of" relationships; modules are the starting point for design and development. Each has interface specs, code, test plans. | Modifiability (changes localised to few modules) |
| **Dependency (Usage)** | "Uses" relationships between modules — which module depends on which. | Extensibility |
| **Data Model** | Business domain entities, their properties, and relationships. | Changeability and performance |

**2. Dynamic Structures (Component-and-Connector view)**

Describe how the system behaves at runtime.

| Dynamic Structure | Description | Targeted Quality |
|---|---|---|
| **Service Structure** | Units are services interacting via precise protocols. | Interoperability and performance |
| **Concurrency Structure** | Components and connectors organised into logical threads/processes; shows what can be parallelised. | Performance and availability |

**3. Allocation Structures**

Describe how software maps onto hardware and team organisation.

| Allocation Structure | Description | Targeted Quality |
|---|---|---|
| **Deployment Structure** | How software is allocated to hardware and communication elements. | Performance, availability, security |
| **Implementation Structure** | How modules map to the file system / version control (IDE or Git repos). | Development efficiency |
| **Work Assignment Structure** | How modules are assigned to team members; critical in multi-site projects. | Development efficiency |

#### 3.3 Views vs. Structures

A **structure** is the actual set of elements (without representation). A **view** is its graphical or textual representation, made for a specific stakeholder. This mirrors the UML distinction between a Model (structure) and a Diagram (view).

Architects design structures; they document views of those structures. Different stakeholders need different views — a developer cares about decomposition; an ops engineer cares about deployment.

#### 3.4 SA vs. SysA vs. EA

| Type | Scope |
|---|---|
| **Software Architecture (SA)** | Software components and their relationships. |
| **System Architecture (SysA)** | Broader — includes hardware, software, and their interactions with humans. |
| **Enterprise Architecture (EA)** | Broadest — covers structure and behaviour of an organisation's processes, information flows, personnel, objectives, and strategy. Software is one aspect of EA. |

All three share common traits: they represent important elements and their interdependencies, they can be designed/evaluated/documented, and they respond to stakeholder needs. This course focuses on **Software Architecture**.

#### 3.5 Why Architecture Matters

- Highlights where a system can be modified; promotes development through reuse.
- Improves stakeholder communication when well-documented.
- Conveys the first (and most important) design decisions — the hardest to reverse.
- Constrains implementation and can dictate development team organisation.
- Channels creativity by limiting design alternatives, thereby reducing complexity.
- Serves as a basis for onboarding new team members.

#### 3.6 Best Practices for Defining an Architecture

**Process recommendations:**
- Should be the work of a single architect or a small group with a clear technical lead (conceptual integrity).
- Base decisions on a hierarchical list of quality attribute priorities (informs trade-offs).
- Document using multiple views tailored to different stakeholders; start minimalist.
- Evaluate the architecture early for its ability to deliver the required quality attributes.
- Lend itself to incremental implementation — avoid big-bang integration.

**Structural recommendations:**
- Modules should have well-defined functional responsibilities following information encapsulation and separation of concerns.
- Satisfy quality needs using known and confirmed architectural styles.
- Never depend on a specific commercial product version (support change).
- Separate data-producing modules from data-consuming modules (ease of change).
- Limit the number of distinct communication means between components (uniformity).
- Do not confuse static module descriptors with dynamic runtime instances — multiple instances of the same descriptor can coexist and be deployed differently.

#### 3.7 Common Architecture Patterns/Styles

Six patterns are highlighted in this course:

**1. Layered Architecture**

The system is divided into horizontal layers with strict direction of dependency. The canonical three-layer enterprise model:

```
+---------------------------+
|    Presentation Layer     |  ← UI, handles user interaction
+---------------------------+
|   Business Logic Layer    |  ← Domain rules, workflows
+---------------------------+
|    Data Access Layer      |  ← Queries, persistence
+---------------------------+
```

- Request flows top-down; response flows bottom-up.
- Separates concerns; each layer can be changed or replaced independently.
- Very common in large enterprise applications (Spring MVC, .NET, Django all follow this).
- Target quality: modularity and maintainability.

**2. Model-View-Controller (MVC)**

Separates application logic into three collaborating components:

```
   User Input
       |
  [Controller] --updates--> [Model]
                                |
                           notifies
                                |
                            [View] --displays--> User
```

- **Model:** Manages data and business rules.
- **View:** Displays data to the user.
- **Controller:** Handles user input and updates the model.
- Used in web applications and desktop GUI frameworks.
- Target quality: separation of presentation from business logic.

**3. Microservices Architecture**

Decomposes a monolithic application into small, independently deployable services. Each service owns its own data store and exposes an API.

```
         [API Gateway]
        /      |       \
  [Service A] [Service B] [Service C]
      |            |           |
   (DB-A)       (DB-B)      (DB-C)
```

- Services communicate through APIs (REST or messaging).
- Each service can be scaled, deployed, and updated independently.
- Improves scalability and flexibility; introduces operational complexity (distributed tracing, service discovery).
- Target quality: scalability and independent deployability.

**4. Event-Driven Architecture (EDA)**

Components communicate by publishing and consuming events through a central Event Bus/Broker (e.g., Apache Kafka).

```
[Event Producers] --> [Event Bus / Broker] --> [Consumers]
                        (publish/subscribe)
```

- Components are loosely coupled — producers do not know about consumers.
- Natural fit for real-time processing, IoT data pipelines, and financial trading systems.
- Target quality: scalability, loose coupling, and responsiveness.

**5. Hexagonal Architecture (Ports and Adapters)**

The application core (business logic) sits at the centre. External concerns (databases, UI, external APIs) connect via ports and adapters.

```
[Web UI Adapter]    [External API Adapter]
        \                  /
  [Port] --- [Application Core] --- [Port]
        /                  \
[Database Adapter]    [Other Adapters]
```

- Core business logic has no dependency on any specific framework, database, or UI technology.
- Adapters implement technology-specific bindings; they are interchangeable.
- Target quality: testability, maintainability, flexibility.

**6. Clean Architecture (Robert C. Martin / "Uncle Bob")**

Organises code into concentric rings with strict inward dependency rules:

```
(outermost) Frameworks & Drivers
            Interface Adapters
            Use Cases
(innermost) Entities (business rules)
```

- Dependencies point inward only — inner layers know nothing about outer layers.
- Entities (core business rules) are completely independent of UI, database, or frameworks.
- Ensures separation of concerns, maintainability, and testability.
- Target quality: independence from external tools and long-term evolvability.

#### 3.8 Considerations for Architecture Selection

Four factors drive the choice of pattern:

| Factor | Question to ask |
|---|---|
| **Scalability Requirements** | How much load growth is expected? Microservices or EDA for high growth. |
| **Complexity vs. Maintainability** | Is the team large enough to manage distributed complexity? Layered for simpler contexts. |
| **Performance and Security Needs** | Are there strict latency or data protection requirements? |
| **Cost of Implementation** | What is the budget for infrastructure, tooling, and team training? |

---

## Key terms (glossary)

- **Enterprise Computing** — IT infrastructure, software, and systems supporting business operations at organisational scale; large-scale, distributed, mission-critical.
- **ERP (Enterprise Resource Planning)** — Integrated software suite unifying finance, HR, supply chain (SAP, Oracle).
- **CRM (Customer Relationship Management)** — System for managing customer interactions (Salesforce).
- **SCM (Supply Chain Management)** — Coordination of procurement, logistics, and inventory.
- **Scalability** — Ability to handle increasing load without degradation.
- **Reliability** — Guarantee of high uptime and correct operation for mission-critical systems.
- **Maintainability** — Ease of modifying, extending, and supporting a system over its long lifecycle.
- **Mainframe** — Large centralised computer (e.g., IBM 700 series) processing batch jobs; dominant 1950–1970.
- **Client-Server** — Architecture dividing tasks between service requesters (clients) and providers (servers).
- **IaaS / PaaS / SaaS** — Cloud delivery layers: raw infrastructure / managed platform / full application.
- **Edge Computing** — Processing data at or near its source (IoT, sensors) rather than in a central cloud.
- **Qubit** — Quantum bit existing in superposition; enables massive parallelism.
- **Software Architecture (SA)** — Structures (components, properties, dependencies) needed to reason about a software system.
- **System Architecture (SysA)** — Broader than SA; includes hardware and human interactions.
- **Enterprise Architecture (EA)** — Broadest scope; covers organisation's processes, information flows, strategy.
- **Structure** — The actual set of architectural elements without representation.
- **View** — A graphical or textual representation of a structure, tailored for a stakeholder.
- **Decomposition Structure** — Static module hierarchy ("is a submodule of"); targets modifiability.
- **Dependency Structure** — Static "uses" relationships; targets extensibility.
- **Service Structure** — Dynamic; services interacting via protocols; targets interoperability.
- **Concurrency Structure** — Dynamic; threads/processes; targets performance and availability.
- **Deployment Structure** — Allocation of software to hardware; targets performance, availability, security.
- **Layered Architecture** — Horizontal tiers (Presentation / Business Logic / Data Access) with top-down dependency.
- **MVC (Model-View-Controller)** — Separates data/rules (Model), display (View), and input handling (Controller).
- **Microservices** — Application decomposed into small, independently deployable services communicating via APIs.
- **EDA (Event-Driven Architecture)** — Loosely coupled components communicating through events via a broker.
- **Hexagonal Architecture** — Core business logic isolated from external concerns via ports and adapters.
- **Clean Architecture** — Concentric rings with inward-only dependencies; innermost = pure business entities.
- **Separation of Concerns** — Design principle: each component/module handles one distinct responsibility.
- **Information Encapsulation** — Hiding internal implementation details behind well-defined interfaces.

---

## Exam targets

1. **Define Enterprise Computing** — give the definition, list the five key characteristics (Scalability, Reliability, Security, Integration, Maintainability), and name at least three system categories with real examples (SAP, Salesforce, etc.).
2. **Trace the historical evolution** — five eras with approximate dates, defining technology shift, and enterprise implication for each (Mainframes → PC → Client-Server → Cloud → Edge).
3. **Define architecture** precisely — both the structural definition and the process-oriented definition. Explain the difference between a structure and a view.
4. **Name and distinguish the three structure types** — static (module), dynamic (component-and-connector), allocation — and give one concrete sub-type per category.
5. **Compare SA, SysA, and EA** — scope, what each includes, what they share.
6. **Describe each of the six architecture patterns**, including: what problem it solves, how components interact, a simple diagram in words, and the quality attribute it targets.
7. **Explain the four selection factors** for choosing an architecture pattern: scalability requirements, complexity vs. maintainability, performance/security, cost.
8. **List best practices** for defining architecture — process side and structural side.

---

## Pitfalls

- **Confusing a view with a structure.** A view is a representation; the structure is the underlying thing. You document views; you design structures.
- **Treating architecture as purely structural.** Architecture must also capture inter-element behaviour when that behaviour affects a system property. Purely listing boxes and lines is not enough.
- **Assuming one pattern fits all.** Microservices introduce significant operational overhead — wrong choice for a small team or low-traffic system. Layered is simpler but can become a bottleneck at massive scale.
- **Confusing static modules with runtime instances.** A module is a descriptor; multiple runtime instances of the same module can coexist. Do not conflate deployment with code organisation.
- **Treating EA as just software.** Enterprise Architecture spans processes, personnel, information flows, and strategy. Software is only one of its aspects.
- **Thinking Edge Computing replaces Cloud.** Edge complements cloud; most architectures will use both in tandem.
- **Ignoring the "early decisions" nature of architecture.** Architectural decisions are the hardest to reverse — changing from a monolith to microservices mid-project is extremely costly. Early evaluation is essential.
- **SA scope creep.** An architectural element should only appear if its behaviour is relevant to reasoning about a system property. If it doesn't affect any quality attribute, it does not belong in the architecture document.
