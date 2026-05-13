# 🎓 Java Principal Engineering Mastery (V4)

A self-study reference for Java engineers targeting Staff/Principal Engineer level. Covers the "physics of software" — the underlying constraints of memory, network, and economics that drive high-scale architecture decisions.

**20 chapters · 95+ modules** — all following a uniform 9-section template:
Learning Objectives → Prerequisites → Critical Dialogue → Theory + Diagrams → Source Archaeology → Code Lab → Production Lens → Exercises → Summary/Flashcard

## Entry Points

| Document | Purpose |
|---|---|
| **[LEARNING_PATH.md](./LEARNING_PATH.md)** | Where to start, fast-track table, prerequisite maps for advanced chapters |
| **[GOL_ARCHITECTURE.md](./GOL_ARCHITECTURE.md)** | The capstone project (Global Order Ledger) — what you build throughout the course |
| **[COURSE_DESIGN.md](./COURSE_DESIGN.md)** | Full module listing by PART with reading order |

---

## 📚 Chapter Overview

| Range | Topic Area |
| :--- | :--- |
| **Ch 1** (10 modules) | Core Java Physics: JMM, Lock-Free, Virtual Threads, CompletableFuture, Generics, Streams |
| **Ch 2** (7 modules) | Spring Internals: AOP, Security, WebFlux, Lifecycle |
| **Ch 3** (6 modules) | Database Physics: mmap, Indexes, MVCC, Hibernate N+1 |
| **Ch 4** (16 modules) | System Design: Foundations + 10 Case Studies (Easy → Hard) |
| **Ch 5** (4 modules) | Distributed Systems: Consensus, Sagas, Vector Clocks |
| **Ch 6** (6 modules) | SRE & Kubernetes: SLOs, Load Shedding, Pod Networking |
| **Ch 7** (4 modules) | Architecture Patterns: Messaging, Database Trade-offs, ADRs |
| **Ch 8** (4 modules) | Caching: Caffeine, Redis, Cache Stampedes |
| **Ch 9** (5 modules) | Object Storage & Media: MinIO, HLS, Edge Delivery |
| **Ch 10** (4 modules) | Monitoring & Incidents: Heap, Threads, Network Forensics |
| **Ch 11** (4 modules) | OSS Ecosystem: Debezium CDC, OTel, Testcontainers, Schema Registry |
| **Ch 12** (4 modules) | Engineering Leadership: ROI Math, ADR Mastery, Boardroom |
| **Ch 13** (4 modules) | Security: JWT, mTLS, Secrets, HSM |
| **Ch 14** (4 modules) | DDD: Strategic, Aggregates, Event Sourcing, CQRS |
| **Ch 15** (4 modules) | Legacy Migration: Strangler Fig, ACL, Dark Launches |
| **Ch 16** (5 modules) | Testing Mastery: JUnit 5/Mockito, Contract, Mutation |
| **Ch 17** (4 modules) | Performance Engineering: JFR, JMH, Memory Profiling, Capacity Math |
| **Ch 18** (4 modules) | Technical Influence: RFC Writing, Design Docs, Trade-off Tables |
| **Ch 19** (4 modules) | Interview Mastery: System Design, Behavioral STAR, Staff Narrative |
| **Ch 20** (5 modules) | Resilience Patterns: Circuit Breaker, Bulkhead, Retry, Timeouts, Graceful Shutdown |

---

## 🛠️ Viewing Diagrams (PlantUML)

This course uses **PlantUML** for all architectural diagrams. To view them in Obsidian or your IDE:

### Obsidian:
1. Install the **PlantUML** community plugin.
2. Set **Server URL** to: `https://www.plantuml.com/plantuml`
3. Diagrams render automatically inside ` ```plantuml ... ``` ` blocks.

---

## 🗺️ Course Navigation
Full syllabus and module mapping: [COURSE_DESIGN.md](./COURSE_DESIGN.md)

New to the course? Start at [LEARNING_PATH.md](./LEARNING_PATH.md).
