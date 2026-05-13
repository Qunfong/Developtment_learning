# Course Blueprint: Java Principal Engineering Mastery

**Entry points:**
- `LEARNING_PATH.md` — where to start, fast-track table, prerequisite maps
- `GOL_ARCHITECTURE.md` — the capstone project (Global Order Ledger) you build throughout the course
- `Readme.md` — course overview

---

## Structure: 5 PARTS

This course is organized into 5 PARTS. Each PART has a single goal. Read in order to avoid forward dependencies.

| PART | Chapters | Goal |
|---|---|---|
| **I — Foundations** | Ch 1, 2, 3, 16 | Build and test a production Spring service backed by PostgreSQL |
| **II — Design** | Ch 4, 5 | Design systems that scale horizontally and survive network failures |
| **III — Production** | Ch 6, 7, 8, 11 (partial), 20 | Run services reliably under real load with full resilience stack |
| **IV — Scale** | Ch 9, 10, 11 (partial), 13, 14, 15 | Handle growth, observability, security, and architectural evolution |
| **V — Excellence** | Ch 17, 12, 18, 19 | Engineer at Staff/Principal level: performance, influence, interview |

---

## PART I — Foundations

**Goal:** After this part you can build, test, and deploy a Spring Boot service backed by PostgreSQL, with proper Java 21+ domain modeling and a full test suite.

### Chapter 1: Core Java & Performance Physics

Reading order within Ch 1 follows the dependency graph (Generics before Streams, JMM before Loom):

| Order | Module | Title | Core Concept | GOL |
|---|---|---|---|---|
| 1 | **1.0** | Functional Foundations | Type Algebra, Result Monads, Stack-Walker Tax | |
| 2 | **1.8** | Generics Physics | Type Erasure, PECS, Bounded Wildcards, Heap Pollution | |
| 3 | **1.9** | Streams & Collectors Deep Dive | Pipeline Fusion, Spliterator, Parallel Pitfalls, Custom Collectors | |
| 4 | **1.2** | Domain Modeling | DOP, Sealed Classes, Records, Pattern Matching | **v0** |
| 5 | **1.1** | Advanced Data Structures | Heap Physics, HashMap Internals, ArrayDeque | |
| 6 | **1.4** | JVM & Cloud Efficiency | GC Tuning, Native Images (GraalVM), Memory Footprint | |
| 7 | **1.5** | Java Memory Model | Happens-Before Rules, CPU Cache Visibility, volatile vs synchronized | |
| 8 | **1.3** | Project Loom Revolution | Virtual Threads, ForkJoinPool, Structured Concurrency | |
| 9 | **1.6** | Lock-Free Concurrency | CAS Mechanics, ABA Problem, StampedLock, Michael-Scott Queue | |
| 10 | **1.7** | CompletableFuture Internals | Completion Tree, Thread Execution Model, thenCompose vs thenCombine | **v0.5** |

### Chapter 2: Spring Framework Internals

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **2.0** | Spring Bootstrap | Demystifying startup process and configuration | **v1** |
| **2.1** | ApplicationContext Lifecycle | BeanFactory, Post-Processors, Context refresh | |
| **2.2** | Architectural Blueprint | Industrial orchestration of the Spring ecosystem | |
| **2.3** | Proxy Engine & AOP | JDK Proxies vs CGLIB, Bytecode manipulation | |
| **2.4** | Adaptive Infrastructure | Custom Registrars and Internal Power Hooks | |
| **2.5** | Spring Security Internals | SecurityFilterChain, JWT Validation Chain, MethodSecurityInterceptor | **v1.5** |
| **2.6** | Spring WebFlux & Reactor | Reactive Streams, Backpressure, Netty vs Tomcat, Virtual Threads Decision | |

### Chapter 3: Databases & Persistence Physics

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **3.1** | Zero-Copy Persistence | Memory-Mapped I/O (mmap), Page Cache | |
| **3.2** | Off-Heap Mastery | Foreign Memory API (Panama), Manual Memory Management | |
| **3.3** | JDBC Internals | Protocol Physics, Statement Batching, Connection Latency | |
| **3.4** | Index Internals & Query Plans | B-Tree Anatomy, Covering Indexes, EXPLAIN ANALYZE | |
| **3.5** | Transaction Isolation & MVCC | Four Isolation Levels, Anomalies, Row Versioning (xmin/xmax) | **v2** |
| **3.6** | ORM Physics & Hibernate | N+1 Problem, JOIN FETCH, L1/L2 Cache, LazyInitializationException | **v2.5** |

### Chapter 16: Testing Mastery

Placed in Part I because tests should be written alongside code, not after deployment.

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **16.0** | JUnit 5 + Mockito Fundamentals | @ParameterizedTest, @Nested, @Mock vs @Spy, ArgumentCaptor | |
| **16.1** | Test Pyramid Physics | Pyramid vs Ice Cream Cone, Test Double Taxonomy, CI Cost Numbers | **v3** |
| **16.2** | Contract Testing with Pact | Consumer-Driven Contracts, Provider Verification | |
| **16.3** | Property-Based Testing | jqwik Setup, Shrinking, Finding Minimal Failing Cases | |
| **16.4** | Mutation Testing with PITest | Surviving Mutants, 100% Coverage ≠ Good Tests | |

*Note: Module 11.3 (Testcontainers) belongs here conceptually — file is at `Chapter_11_Open_Source/11.3_Testcontainers_Physics.md`.*

---

## PART II — Design

**Goal:** After this part you can capacity-plan a system, reason about distributed consistency, and produce architectural recommendations backed by data.

### Chapter 4: System Design Foundations

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **4.1** | SQL vs. NoSQL | The Data Paradox, Consistency vs Scale | **v4** |
| **4.2** | The Caching Loop | Write Strategies, Expiry Physics (LRU/LFU) | |
| **4.3** | Selection Matrix | Trade-offs at Scale, Decision Logic | |
| **4.4** | Proxies & Load Balancing | LB Algorithms, L4 vs L7 Routing | |
| **4.5** | Content Delivery | CDN Push vs Pull, Edge Proximity | |
| **4.6** | API Architectural Combat | REST vs gRPC vs GraphQL, Binary Protocols | |

### Chapter 4 (continued): Case Studies

Read these **after Chapter 5** — they reference distributed concepts (vector clocks, Saga compensation) introduced there.

| Module | Case Study | Difficulty | Key Focus |
|---|---|---|---|
| **4.7** | Global Media Hub | **[Hard]** | Plane Separation, S3/CDN Handoff |
| **4.8** | Social Graph | **[Hard]** | Join Physics, Distributed Caches |
| **4.9** | Global Order Ledger | **[Hard]** | Multi-master, Vector Clocks, CDC |
| **4.10** | Stock Exchange | **[Hard]** | LMAX Disruptor, Ring Buffers |
| **4.11** | Ad-Tech Engine | **[Hard]** | 10ms RTT, Anycast, SIMD |
| **4.12** | Ridesharing Fleet | **[Hard]** | S2 Geo-sharding, Spatial Locality |
| **4.13** | Healthcare Registry | **[Medium]** | ACID Fortress, HSM Encryption |
| **4.14** | E-commerce Hybrid | **[Medium]** | CQRS, Debezium CDC, Search |
| **4.15** | Boutique NFT Gallery | **[Easy]** | Serverless Velocity, DynamoDB |
| **4.16** | Internal HR Portal | **[Easy]** | Saturated Monolith, Caffeine Cache |

### Chapter 5: Distributed Systems Physics

| Module | Title | Core Concept |
|---|---|---|
| **5.1** | Clock Skew & Causality | Vector Clocks, Logical Time, NTP drift |
| **5.2** | Distributed Consensus | Raft, Paxos, PACELC Law |
| **5.3** | Distributed Sagas | Compensation Physics, TCC (Try-Confirm-Cancel) |
| **5.4** | Network Partitions | Split-Brain Paradox, Quorum |

---

## PART III — Production

**Goal:** After this part you can operate services under real load: SLOs, traffic management, messaging, caching, and a full Resilience4j resilience stack.

### Chapter 6: SRE Operations & Fleet Management

| Module | Title | Core Concept |
|---|---|---|
| **6.1** | Cell-based Architecture | Blast Radius Mitigation, Shuffling |
| **6.2** | Sidecar Physics | eBPF, Envoy Latency, Service Mesh |
| **6.3** | Adaptive Load Shedding | TCP Vegas, Backpressure, Queue Management |
| **6.4** | SLO Math | Error Budgets, Windowing, Burn Rates |
| **6.5** | Kubernetes Scheduling Physics | Filter/Score/Bind Pipeline, QoS Classes, Taints/Tolerations |
| **6.6** | Pod Networking & CNI Internals | veth→bridge→overlay Packet Path, Service ClusterIP DNAT, DNS ndots:5 |

### Chapter 7: Architecture Patterns & Combat

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **7.1** | Messaging Battlefield | Zero-Copy vs Broker State (Kafka vs Rabbit vs Hub) | **v5** |
| **7.2** | Database Combat | Consistency vs Availability Trade-offs | |
| **7.3** | Idempotency & Conflict | Conflict Physics, Deduplication | |
| **7.4** | Technical ADRs | One-Way Door Matrix, Design Documentation | |

*Note: Module 11.1 (CDC/Debezium) belongs here conceptually — file at `Chapter_11_Open_Source/11.1_CDC_Debezium.md`. Module 11.4 (Schema Registry) also belongs here — file at `Chapter_11_Open_Source/11.4_Schema_Registry.md`.*

### Chapter 8: Advanced Caching

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **8.1a** | Caffeine L1 Caching | W-TinyLFU, JVM Heap Constraints | |
| **8.1b** | Redis Internals | Single-threaded physics, Memory optimization | **v6.5** |
| **8.2** | Distributed Caching | Valkey vs Redis vs Memcached | |
| **8.3** | Cache Hierarchies | MLC (Multi-Level Cache) Synchronization | |
| **8.4** | Cache Stampedes | PER (Probabilistic Early Recomputation) | |

### Chapter 20: Resilience Patterns

All five modules use Resilience4j 3.x (Spring Boot 3 / Java 21 compatible). Composition order: `Bulkhead → Retry → CircuitBreaker → TimeLimiter` (outer to inner).

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **20.1** | Circuit Breaker & Resilience4j | State Machine (CLOSED/OPEN/HALF-OPEN), Cascade Failure Prevention | **v6** |
| **20.2** | Bulkhead & Thread Isolation | Semaphore vs ThreadPool Bulkhead, Virtual Thread Interaction | |
| **20.3** | Retry Engineering | Retry Storms, Exponential Backoff, Jitter Strategies | |
| **20.4** | Timeout Propagation & Deadline Chaining | Connect vs Read vs Deadline, Budget Propagation | |
| **20.5** | Graceful Degradation & Shutdown | Fallback Taxonomy, Spring Graceful Shutdown, K8s SIGTERM Lifecycle | |

---

## PART IV — Scale

**Goal:** After this part you can handle growth: object storage at scale, production forensics, security architecture, event-sourced domain models, and legacy modernization.

### Chapter 9: Object Storage & Media Physics

| Module | Title | Core Concept |
|---|---|---|
| **9.1a** | BLOB Physics | Large-scale Media, Byte-range requests |
| **9.1b** | OSS Object Storage | MinIO, Erasure Coding Physics |
| **9.2** | Edge Delivery | Envoy and Nginx Physics |
| **9.3** | Plane Orchestration | Control vs Data Plane separation |
| **9.4** | Adaptive Streaming | HLS, DASH, Video Transcoding |

### Chapter 10: Monitoring & Incidents

| Module | Title | Core Concept |
|---|---|---|
| **10.1** | Connection Pool Crisis | M/M/1 Queueing Theory, Pool Exhaustion |
| **10.2** | Heap Archaeology | MAT (Memory Analyzer Tool), Dominator Trees |
| **10.3** | Thread Jitter | Safepoints, Flamegraph Forensics |
| **10.4** | Network Forensics | TCP Storms, SocketRead0 Analysis |

*Note: Module 11.2 (OpenTelemetry) belongs here — file at `Chapter_11_Open_Source/11.2_OpenTelemetry.md`.*

### Chapter 13: Security Architecture

| Module | Title | Core Concept |
|---|---|---|
| **13.1** | JWT Physics | Signature Tax (RSA vs EdDSA), Zero-Trust |
| **13.2** | mTLS & Certificates | SPIFFE/SPIRE, Identity Propagation |
| **13.3** | Secret Management | Entropy Tax, Dynamic Secrets |
| **13.4** | Hardware Security | HSM, TPM, Field-Level Encryption |

### Chapter 14: Domain-Driven Design (DDD)

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **14.1** | Strategic DDD | Bounded Contexts, Semantic Physics | |
| **14.2** | Aggregate Consistency | Transactional Boundaries | |
| **14.3** | Event Sourcing | Immutable Logs, Snapshotting | **v7** |
| **14.4** | CQRS | Read-Model Projections, Eventual Parity | |

### Chapter 15: Legacy Migration & Evolution

| Module | Title | Core Concept |
|---|---|---|
| **15.1** | Strangler Fig Pattern | Interception, Propagation, Elimination |
| **15.2** | Anti-Corruption Layer | Sovereign Border, Legacy Wash |
| **15.3** | Data Migration Physics | Double-Write Pattern, Backfill |
| **15.4** | Dark Launches | Shadow Traffic, Feature Flags |

---

## PART V — Excellence

**Goal:** After this part you can operate at Staff/Principal level: profiling production JVMs, quantifying technical ROI, writing documents that influence org-wide decisions, and performing in Staff/Principal interviews.

### Chapter 17: Performance Engineering

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **17.1** | JFR & AsyncProfiler Mastery | Continuous Recording, Flamegraph Reading, Safepoint Bias | |
| **17.2** | JMH Benchmarks | Dead Code Elimination, Blackhole, JIT Warmup Effects | |
| **17.3** | Memory Profiling Workflow | Heap Dump (jcmd), MAT Dominator Tree, Leak Patterns | **v8** |
| **17.4** | Capacity Planning Math | Little's Law, Amdahl's Law, M/M/1 Latency Model | |

### Engineering Leadership & Technical Influence

Ch 12 and Ch 18 form a single unit. Ch 12 is applied leadership (ROI, ADRs, board negotiation); Ch 18 is written influence (RFCs, design docs, code review).

*Note on ADRs: Ch 7.4 (Technical ADRs) is the foundation; Ch 12.2 (ADR Mastery) is the deep dive. Read 7.4 before 12.2.*

**Chapter 12: Engineering Leadership**

| Module | Title | Core Concept |
|---|---|---|
| **12.1** | Technical ROI Math | Unit Cost of an Order, TCO Calculation |
| **12.2** | ADR Mastery | One-Way Door Matrix, Paper Trails *(prerequisite: 7.4)* |
| **12.3** | Boardroom Negotiation | Strategic Pitching, Conflict Resolution |
| **12.4** | Architectural Audit | Simulation of a High-Stakes Audit |

**Chapter 18: Technical Influence**

| Module | Title | Core Concept |
|---|---|---|
| **18.1** | RFC Writing & Adoption | RFC Structure Template, Alternatives Section, Organisational Memory |
| **18.2** | Design Docs Communication | Design Doc vs RFC vs ADR, Audience-Layered Structure |
| **18.3** | Technical Proposals & Trade-offs | Quantified Trade-off Tables, One-Way vs Two-Way Door |
| **18.4** | Code Review as Knowledge Transfer | Blocking vs Teaching Comments, Establishing Standards |

### Chapter 19: Interview Mastery

| Module | Title | Core Concept | GOL |
|---|---|---|---|
| **19.1** | System Design Framework | 45-Minute Framework, Scoping, Estimation | **v9 (CAPSTONE)** |
| **19.2** | System Design Practice | Worked Examples, Common Mistakes | |
| **19.3** | Behavioural Interviews | STAR Variants, Failure Framing, Leadership Stories | |
| **19.4** | Salary Negotiation | BATNA, Anchoring, Competing Offers | |

---

## Ch 11 Module Placement Guide

Chapter 11 modules are placed in the chapter where they have the most impact:

| Module | File | Canonical PART | Placed With |
|---|---|---|---|
| 11.1 CDC & Debezium | Chapter_11_Open_Source/11.1_CDC_Debezium.md | PART III | Ch 7 (Event-Driven) |
| 11.2 OpenTelemetry | Chapter_11_Open_Source/11.2_OpenTelemetry.md | PART IV | Ch 10 (Monitoring) |
| 11.3 Testcontainers | Chapter_11_Open_Source/11.3_Testcontainers_Physics.md | PART I | Ch 16 (Testing) |
| 11.4 Schema Registry | Chapter_11_Open_Source/11.4_Schema_Registry.md | PART III | Ch 7 (Messaging) |

---

## GOL Version Summary

| Version | PART | Module | What It Builds |
|---|---|---|---|
| v0 | I | 1.2 | Domain model: sealed `LedgerEvent`, records |
| v0.5 | I | 1.7 | Async account validation, virtual thread executor |
| v1 | I | 2.0 | Spring Boot app structure, `ApplicationContext` |
| v1.5 | I | 2.5 | JWT security, `SecurityFilterChain` |
| v2 | I | 3.5 | Transaction isolation level choice |
| v2.5 | I | 3.6 | N+1 fix, HikariCP pool sizing |
| v3 | I | 16.1 | Full test suite structure |
| v4 | II | 4.1 | Capacity plan for 50k writes/sec |
| v5 | III | 7.1 | Kafka audit log, partition strategy |
| v6 | III | 20.1 | Payment gateway circuit breaker |
| v6.5 | III | 8.1b | Redis balance cache, stampede prevention |
| v7 | IV | 14.3 | Event-sourced ledger model |
| v8 | V | 17.3 | Memory leak diagnosis (p99 regression) |
| v9 | V | 19.1 | Full system design from scratch (capstone) |
