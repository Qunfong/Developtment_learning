# Learning Path Guide

This guide tells you where to start, how to navigate the course, and how to get to any advanced topic quickly.

See `GOL_ARCHITECTURE.md` for the capstone project you will build throughout the course.
See `COURSE_DESIGN.md` for the full module listing by PART.

---

## What You Will Build

Every GOL Challenge in this course adds one component to the **Global Order Ledger** — a distributed financial ledger processing 50,000 transactions/second across three geographic regions.

By the end of Chapter 19, you will have:
- Designed the domain model (sealed interfaces, records)
- Bootstrapped it as a production Spring Boot service with JWT security
- Tuned the database layer (MVCC, connection pooling, N+1)
- Added Kafka for the immutable audit log
- Protected the payment gateway with a circuit breaker
- Cached account balances in Redis with stampede prevention
- Refactored the ledger to event-sourced architecture
- Diagnosed a production memory leak
- Designed the entire system from scratch in 45 minutes

---

## 5-Part Structure

| PART | Theme | Goal |
|---|---|---|
| **I — Foundations** | Java + Spring + Database | Build and test a Spring service backed by PostgreSQL |
| **II — Design** | System Design + Distributed | Design systems that scale and survive failures |
| **III — Production** | SRE + Caching + Messaging | Run services reliably under real load |
| **IV — Scale** | Architecture Patterns + Data | Handle growth without rewriting everything |
| **V — Excellence** | Performance + Security + Leadership + Interview | Engineer at Staff/Principal level |

---

## Start Here: First 5 Modules

If you are starting fresh:

1. `Chapter_01_Core_Java/00_Functional_Foundations.md` — type algebra, Result monad pattern; establishes vocabulary for all later modules
2. `Chapter_01_Core_Java/08_Generics_Physics.md` — type erasure, PECS, bounded wildcards; prerequisite for understanding Spring internals
3. `Chapter_01_Core_Java/09_Streams_Collectors_Deep_Dive.md` — pipeline internals, parallel pitfalls, custom Collectors
4. `Chapter_01_Core_Java/02_Domain_Modeling.md` — sealed classes, records, pattern matching; **first GOL Challenge (v0)**
5. `Chapter_02_Spring/2.0_Spring_Bootstrap_Internals.md` — Spring startup, `ApplicationContext`; **GOL v1**

These five modules unlock the rest of Part I and give you enough vocabulary to read any chapter.

---

## Fast Track (Already Know Ch 1–3 Basics)

If you are comfortable with Java 17+, Spring Boot, and basic SQL:

| Skip To | Requires | First GOL Version Here |
|---|---|---|
| Ch 1.5 JMM → Ch 1.6 Lock-free → Ch 1.7 CF | Java 8+ familiarity | v0.5 (Ch 1.7) |
| Ch 2.5 Spring Security | Ch 2.0 read | v1.5 |
| Ch 3.5 Transaction Isolation + Ch 3.6 ORM | Ch 3.3 JDBC or equivalent | v2, v2.5 |
| Ch 16 Testing Mastery | Ch 1–3 | v3 |
| Ch 7 Messaging → Ch 5 Distributed | Ch 3 complete | v5 |
| Ch 20 Resilience Patterns | See prerequisite map below | v6 |

---

## Recommended Reading Order

This is the order `COURSE_DESIGN.md` encodes as the PARTS model. Read sequentially to guarantee no forward dependencies.

### PART I — Foundations

```
Ch 1.0 Functional Foundations
Ch 1.8 Generics Physics
Ch 1.9 Streams & Collectors
Ch 1.2 Domain Modeling          ← GOL v0
Ch 1.1 Advanced Data Structures
Ch 1.4 JVM & Cloud Efficiency
Ch 1.5 Java Memory Model
Ch 1.3 Project Loom (Virtual Threads)
Ch 1.6 Lock-Free Concurrency
Ch 1.7 CompletableFuture        ← GOL v0.5

Ch 2.0 Spring Bootstrap         ← GOL v1
Ch 2.1 ApplicationContext
Ch 2.2 Architectural Blueprint
Ch 2.3 Proxy Engine & AOP
Ch 2.4 Adaptive Infrastructure
Ch 2.5 Spring Security          ← GOL v1.5
Ch 2.6 Spring WebFlux

Ch 3.1 Zero-Copy Persistence
Ch 3.2 Off-Heap Mastery
Ch 3.3 JDBC Internals
Ch 3.4 Index Internals
Ch 3.5 Transaction Isolation     ← GOL v2
Ch 3.6 ORM Physics               ← GOL v2.5

Ch 16.0 JUnit5 + Mockito
Ch 16.1 Test Pyramid             ← GOL v3
Ch 16.2 Contract Testing (Pact)
Ch 16.3 Property-Based Testing
Ch 16.4 Mutation Testing
```

### PART II — Design

```
Ch 4.1–4.6 System Design Foundations  ← GOL v4 (in 4.1 or nearest foundations module)
Ch 5.1 Clock Skew & Causality
Ch 5.2 Distributed Consensus
Ch 5.3 Distributed Sagas
Ch 5.4 Network Partitions
Ch 4.7–4.16 Case Studies (after Ch 5)
```

### PART III — Production

```
Ch 6.1 Cell-Based Architecture
Ch 6.2 Sidecar Physics
Ch 6.3 Adaptive Load Shedding
Ch 6.4 SLO Math
Ch 6.5 Kubernetes Scheduling
Ch 6.6 Pod Networking
Ch 7.1 Messaging Battlefield     ← GOL v5
Ch 7.2 Database Combat
Ch 7.3 Idempotency & Conflict
Ch 7.4 Technical ADRs
Ch 8.1a Caffeine L1 Caching
Ch 8.1b Redis Internals          ← GOL v6.5
Ch 8.2 Distributed Caching
Ch 8.3 Cache Hierarchies
Ch 8.4 Cache Stampedes
Ch 11.1 CDC & Debezium
Ch 11.4 Schema Registry
Ch 20.1 Circuit Breaker          ← GOL v6
Ch 20.2 Bulkhead
Ch 20.3 Retry Engineering
Ch 20.4 Timeout Propagation
Ch 20.5 Graceful Degradation
```

### PART IV — Scale

```
Ch 9.1a–9.4 Object Storage & Media
Ch 10.1–10.4 Monitoring & Incidents
Ch 11.2 OpenTelemetry
Ch 11.3 Testcontainers
Ch 14.1 Strategic DDD
Ch 14.2 Aggregate Consistency
Ch 14.3 Event Sourcing            ← GOL v7
Ch 14.4 CQRS
Ch 15.1–15.4 Legacy Migration
Ch 13.1–13.4 Security Architecture
```

### PART V — Excellence

```
Ch 17.1 JFR & AsyncProfiler
Ch 17.2 JMH Benchmarks
Ch 17.3 Memory Profiling          ← GOL v8
Ch 17.4 Capacity Planning Math
Ch 12.1 Technical ROI Math
Ch 12.2 ADR Mastery
Ch 12.3 Boardroom Negotiation
Ch 12.4 Architectural Audit
Ch 18.1 RFC Writing
Ch 18.2 Design Docs
Ch 18.3 Technical Proposals
Ch 18.4 Code Review as Knowledge Transfer
Ch 19.1 System Design Interview   ← GOL v9 (CAPSTONE)
Ch 19.2–19.4 Interview Practice
```

---

## Prerequisite Map: Chapter 20 (Resilience Patterns)

Chapter 20 is an advanced chapter. These are its hard prerequisites:

```
Ch 1.7 CompletableFuture (async execution model)
    ↓
Ch 1.3 Project Loom (virtual threads)
    ↓
Ch 2.0 Spring Bootstrap (Spring app structure)
    ↓
Ch 20.1 Circuit Breaker   ← start here
Ch 20.2 Bulkhead          ← requires 20.1
Ch 20.3 Retry Engineering ← requires 20.1
Ch 20.4 Timeout Propagation ← requires 20.1, 1.7, 1.3
Ch 20.5 Graceful Degradation ← requires 20.1, 20.4, 2.0
```

---

## Prerequisite Map: Chapter 14 (DDD + Event Sourcing)

```
Ch 3.5 Transaction Isolation (ACID understanding)
    ↓
Ch 7.1 Messaging (event-driven thinking)
    ↓
Ch 14.1 Strategic DDD
    ↓
Ch 14.2 Aggregate Consistency
    ↓
Ch 14.3 Event Sourcing  ← GOL v7
    ↓
Ch 14.4 CQRS
```

---

## Prerequisite Map: Chapter 19 (Interview Mastery)

Chapter 19 is the capstone. To get the most from it:

```
PART I complete (all Ch 1–3 + Ch 16)
PART II complete (all Ch 4–5)
Ch 6.1–6.4 SRE basics
Ch 7.1–7.3 Architecture patterns
Ch 8 Caching
Ch 20 Resilience Patterns
Ch 14.3 Event Sourcing
    ↓
Ch 19.1 System Design Framework  ← GOL v9
Ch 19.2 System Design Practice
Ch 19.3 Behavioural Interviews
Ch 19.4 Salary Negotiation
```

---

## For L&D / Facilitators

- **Spaced repetition**: GOL Challenges are spaced by design — v0 and v1 are weeks apart so learners revisit the domain model after internalizing Spring
- **Each module is standalone**: assessments, exercises, and GOL challenges are self-contained; you can use individual modules in workshops
- **Cognitive Load**: modules follow the 9-section template with Critical Dialogue first, Theory second — this matches Cognitive Load Theory (problem before explanation)
- **Progressive disclosure**: flashcard decks at the end of each module are designed for Anki export
