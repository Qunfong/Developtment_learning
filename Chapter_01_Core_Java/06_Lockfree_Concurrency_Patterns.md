# 06: Lock-Free Algorithms & Advanced Concurrency Patterns

## Learning Objectives

After this module you can:
- Implement a CAS retry loop and explain why it is lock-free but not wait-free
- Identify the ABA problem in code and fix it with `AtomicStampedReference`
- Implement a simplified Michael-Scott lock-free queue from scratch
- Choose between `StampedLock` optimistic reads and `ReadWriteLock` for a given access pattern
- Benchmark lock-free vs lock-based structures under contention using JMH

## Prerequisites

- Ch 1.5 Java Memory Model (happens-before, volatile, CAS semantics are required)
- Basic `AtomicInteger` usage

---

## The Critical Dialogue

**Student:** I replaced all my `synchronized` blocks with `AtomicInteger` and `ConcurrentHashMap` — is my code now "lock-free"? Also, what exactly does "lock-free" mean, and when should I bother?

**Principal:** Using `AtomicInteger` is a step toward lock-free, but "lock-free" is a precise algorithm property, not a library choice. Lock-free means: at least one thread always makes progress, even if others are delayed or suspended. Contrast with `synchronized`: if the thread holding the lock is paused by the OS scheduler for 100ms, every other thread blocks for 100ms too. Lock-free algorithms eliminate this "lock holder tax." But they are harder to implement correctly — so you should reach for them only when profiling shows lock contention is your bottleneck.

---

## 1. Compare-and-Swap (CAS): The Foundation

### 1.1 What CAS Does

CAS is a single atomic CPU instruction: **compare the current value at a memory address; if it equals `expected`, replace it with `update`; return whether the swap happened.**

```java
// Conceptual model (NOT actual code — this is what the CPU does atomically):
boolean cas(MemoryLocation location, int expected, int update) {
    if (location.value == expected) {
        location.value = update;
        return true;   // success
    }
    return false;       // lost the race — value was changed by another thread
}
```

On x86, CAS compiles to the `LOCK CMPXCHG` instruction — single cycle, hardware-atomic.

### 1.2 The CAS Retry Loop

```java
import java.util.concurrent.atomic.AtomicInteger;

// Lock-free increment: keep retrying until CAS succeeds
public class LockFreeCounter {
    private final AtomicInteger value = new AtomicInteger(0);

    public int increment() {
        int current, next;
        do {
            current = value.get();           // Step 1: read current
            next = current + 1;              // Step 2: compute new value
            // Step 3: CAS — if value is still 'current', swap to 'next'
            // If another thread changed value between Step 1 and Step 3, retry
        } while (!value.compareAndSet(current, next));
        return next;
    }
}
```

**Why this is lock-free:** If thread A fails its CAS, it's because thread B succeeded — thread B made progress. At least one thread always advances. No thread can block all others.

**Why this is NOT wait-free:** Thread A might retry indefinitely under extreme contention (though in practice, retries are rare — usually 0-2).

### 1.3 When CAS Degrades

```
CONTENTION LEVEL     CAS BEHAVIOR
──────────────────────────────────────────────
Low (< 4 threads)    ~3-5 ns/op — faster than synchronized (~14 ns)
Medium (4-8 threads) ~12 ns/op — comparable to synchronized
High (> 16 threads)  ~80+ ns/op — CAS retry storm, consider LongAdder
```

For high-contention counters, `LongAdder` (striped CAS, reduces contention by factor N) outperforms `AtomicLong` by 10-100x.

---

## 2. The ABA Problem

### 2.1 What It Is

CAS checks `current == expected`. But what if the value went from A → B → A between your read and CAS? Your CAS succeeds, but you missed an intermediate change.

```java
// ABA scenario:
// Stack: [A, B, C]
// Thread 1: reads head = A, paused by OS

// Thread 2: pops A, pops B, pushes A back
// Stack: [A, C]  ← A is back, but B is gone!

// Thread 1 resumes: CAS(head, A, B) — succeeds! (head is still A)
// Stack now: [B, ...dangling garbage...] — CORRUPTION
```

ABA most commonly corrupts linked data structures (stacks, queues, trees) where node reuse causes an "old" pointer to accidentally look valid.

### 2.2 Fix: `AtomicStampedReference`

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABAFixDemo {
    // Pair (reference, stamp) — stamp is a monotonically increasing version
    private final AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("A", 0);

    public void demonstrateABAFix() {
        int[] stampHolder = new int[1];
        String current = ref.get(stampHolder);   // read both value AND stamp
        int currentStamp = stampHolder[0];

        // Even if value goes A → B → A, the stamp goes 0 → 1 → 2
        // CAS fails because stamp 0 != stamp 2
        boolean success = ref.compareAndSet(
            current, "B",           // expected value, new value
            currentStamp, currentStamp + 1  // expected stamp, new stamp
        );

        System.out.println("CAS succeeded: " + success);
        System.out.println("New stamp: " + ref.getStamp());
    }
}
```

**Trade-off:** `AtomicStampedReference` stores two values — reference + int stamp — in a single heap object. The CAS operates on the object reference, not the contained values directly. This adds one pointer indirection and one object allocation per reference. Use only when ABA is a real concern (linked structures with node reuse).

---

## 3. Lock-Free Queue: Michael-Scott Algorithm

The Michael-Scott queue (1996) is the algorithm inside `ConcurrentLinkedQueue`. Understanding it reveals how lock-free data structures actually work.

### 3.1 Structure

```
┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
│dummy│───▶│  A  │───▶│  B  │───▶│  C  │───▶ null
└─────┘    └─────┘    └─────┘    └─────┘
   ▲                                 ▲
  head                             tail
```

Key invariant: `tail` points to last node OR second-to-last (momentarily during enqueue). `head` always points to a dummy sentinel node.

### 3.2 Simplified Implementation

```java
import java.util.concurrent.atomic.AtomicReference;

public class MichaelScottQueue<T> {

    private record Node<T>(T value, AtomicReference<Node<T>> next) {
        Node(T value) { this(value, new AtomicReference<>(null)); }
    }

    private final AtomicReference<Node<T>> head;
    private final AtomicReference<Node<T>> tail;

    public MichaelScottQueue() {
        var sentinel = new Node<T>(null);  // dummy head
        head = new AtomicReference<>(sentinel);
        tail = new AtomicReference<>(sentinel);
    }

    public void enqueue(T value) {
        var newNode = new Node<>(value);
        while (true) {
            Node<T> last = tail.get();
            Node<T> next = last.next().get();

            if (last == tail.get()) {          // tail hasn't moved — consistent read
                if (next == null) {
                    // Tail points to actual last — try to append
                    if (last.next().compareAndSet(null, newNode)) {
                        // Success: try to advance tail (ok if this fails — other thread will fix it)
                        tail.compareAndSet(last, newNode);
                        return;
                    }
                } else {
                    // Tail is lagging — help advance it before retrying
                    tail.compareAndSet(last, next);
                }
            }
        }
    }

    public T dequeue() {
        while (true) {
            Node<T> first = head.get();
            Node<T> last = tail.get();
            Node<T> next = first.next().get();

            if (first == head.get()) {          // consistent read
                if (first == last) {
                    if (next == null) return null;  // queue empty
                    tail.compareAndSet(last, next); // help advance lagging tail
                } else {
                    T value = next.value();
                    if (head.compareAndSet(first, next)) return value;
                }
            }
        }
    }
}
```

**Key insight:** When a thread gets preempted mid-enqueue (tail is lagging), other threads detect this and help complete the operation (`tail.compareAndSet(last, next)`). This "helping" is what makes the algorithm lock-free — a paused thread cannot stall others indefinitely.

---

## 4. StampedLock: Optimistic Reads

### 4.1 The Read-Write Lock Problem

`ReadWriteLock` (and `ReentrantReadWriteLock`) give shared reads / exclusive writes. But even a read acquires a lock, which means contention between readers under high load.

```java
// ReadWriteLock: every read is still a lock acquisition
private final ReadWriteLock rwl = new ReentrantReadWriteLock();

public double read() {
    rwl.readLock().lock();         // contention with other readers!
    try { return value; }
    finally { rwl.readLock().unlock(); }
}
```

### 4.2 StampedLock Optimistic Read

`StampedLock` (Java 8+) adds a third mode: **optimistic read** — read without acquiring ANY lock, then validate the read was consistent.

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // WRITE: exclusive lock
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try { x += deltaX; y += deltaY; }
        finally { lock.unlockWrite(stamp); }
    }

    // OPTIMISTIC READ: no lock — just check if a write happened during our read
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // returns 0 if write lock held
        double curX = x, curY = y;             // read fields
        if (!lock.validate(stamp)) {            // was a write lock acquired?
            // Fallback: read lock (always consistent)
            stamp = lock.readLock();
            try { curX = x; curY = y; }
            finally { lock.unlockRead(stamp); }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }
}
```

**When optimistic reads win:** Read-heavy workloads where writes are rare. The optimistic path has no CAS, no cache line invalidation — it's as fast as a plain field read (~1-2 ns). Under low write contention, validation almost always succeeds.

---

## 5. Source Archaeology

### 5.1 `ConcurrentLinkedQueue` — Michael-Scott in Production

```java
// java.util.concurrent.ConcurrentLinkedQueue (OpenJDK 21)
// Line ~310 (offer method):

public boolean offer(E e) {
    final Node<E> newNode = newNode(Objects.requireNonNull(e));
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (NEXT.compareAndSet(p, null, newNode)) {
                // Successful CAS; if p != t, try to set tail to newNode
                if (p != t)
                    TAIL.weakCompareAndSet(this, t, newNode);  // "weak" — may fail
                return true;
            }
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;  // handle sentinel
        else
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
// Note: weakCompareAndSet may spuriously fail — that's intentional.
// Advancing tail is a best-effort optimization; correctness doesn't depend on it.
```

### 5.2 `LongAdder` — Striped CAS for High Contention

```java
// java.util.concurrent.atomic.LongAdder (OpenJDK 21)
// Inherits from Striped64

// Internal structure: base + Cell[] array
// Each thread targets a random Cell index — reduces contention by factor N
// Sum = base + sum(cells[i])

// Striped64.longAccumulate() — the CAS loop with cell-level striping:
// 1. Try CAS on target cell
// 2. If contention detected, create new cell or resize array
// 3. Result: contention distributed across N cells
```

---

## 6. Code Lab

### Lab: StampedLock vs ReadWriteLock Benchmark

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
@Threads(8)  // 8 concurrent threads — read-heavy simulation
public class LockBenchmark {

    private double value = 1.0;
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final StampedLock stampedLock = new StampedLock();

    @Benchmark
    public double rwlRead() {
        rwLock.readLock().lock();
        try { return value; }
        finally { rwLock.readLock().unlock(); }
    }

    @Benchmark
    public double stampedOptimisticRead() {
        long stamp = stampedLock.tryOptimisticRead();
        double v = value;
        if (!stampedLock.validate(stamp)) {
            stamp = stampedLock.readLock();
            try { v = value; }
            finally { stampedLock.unlockRead(stamp); }
        }
        return v;
    }
}

/*
Benchmark results (8 reader threads, no writes, x86):
──────────────────────────────────────────────────────
Benchmark                    Throughput   Units
────────────────────────────────────────────────
rwlRead                      28,450       ops/ms   ← lock contention
stampedOptimisticRead       312,800       ops/ms   ← no lock, 11x faster!

With 1% write traffic mixed in:
rwlRead                      22,100       ops/ms
stampedOptimisticRead        89,400       ops/ms   ← fallback kicks in ~1% of reads
*/
```

---

## 7. Production Lens

### Incident: LongAdder vs AtomicLong Under Spike Traffic

At a payments processor, a `AtomicLong` request counter became a hot spot during Black Friday: 64 threads incrementing the same `AtomicLong` caused a CAS retry storm.

```
OBSERVED:
  AtomicLong.incrementAndGet() at 64-thread contention: 180 ns/op
  LongAdder.increment() at 64-thread contention: 8 ns/op — 22x faster

ROOT CAUSE:
  AtomicLong = single CAS target = linear contention
  LongAdder  = 64 cells = each thread hits its own cell = near-zero contention

TRADE-OFF:
  LongAdder.sum() is eventually-consistent — reads may lag writes by ~100ns
  AtomicLong.get() is immediately consistent
  For a rate counter (approximate OK), LongAdder wins by orders of magnitude
```

### Red Flags in Code Review

```
❌ AtomicReference.get() + .set() in separate lines  → not atomic, use CAS
❌ AtomicLong in a high-contention counter            → use LongAdder
❌ Custom lock-free code without ABA analysis         → stamp or version every node reference
❌ StampedLock without validate() in optimistic path  → silent stale data bug
❌ lock.readLock().lock() without try/finally         → deadlock if exception thrown
```

---

## 8. Exercises

**1.** What is the difference between **lock-free** and **wait-free**? Give an example of each in Java's standard library.

**2.** The ABA problem: Draw a timeline showing two threads operating on an `AtomicReference<Node>` stack where ABA causes a corruption. Then fix the code using `AtomicStampedReference`.

**3.** Explain why `ConcurrentLinkedQueue.offer()` uses `weakCompareAndSet` for the tail update but a regular `compareAndSet` for the next pointer. What is the behavioral difference, and why is it safe here?

**4. Coding challenge:** Implement a thread-safe `OneTimeLatch` — a boolean flag that can only be set to `true` once, and any thread can query it. Use only `AtomicBoolean`. It must be lock-free. Write a concurrent test that fires 100 threads all trying to set it simultaneously and verifies exactly one succeeds.

**5.** When should you NOT use lock-free algorithms? List three scenarios where `synchronized` or a blocking `Lock` is the better choice.

---

## Exercise Solutions

<details>
<summary>Exercise 1 — Lock-free vs wait-free: definition and Java examples</summary>

**Lock-free:** Guarantees that *at least one thread* makes progress in a finite number of steps. Individual threads may starve (retry their CAS loop indefinitely) but the system as a whole always makes forward progress. A thread can be preempted and its CAS will fail, but some other thread will succeed.

**Wait-free:** Guarantees that *every thread* completes its operation in a bounded number of steps, regardless of what other threads do. No thread can starve.

Wait-free is strictly stronger than lock-free. All wait-free algorithms are lock-free, but not vice versa.

**Java standard library examples:**
- **Lock-free:** `ConcurrentLinkedQueue` — `offer()` and `poll()` use CAS retry loops; under high contention a thread may retry many times, but globally progress is always made
- **Wait-free:** `AtomicInteger.get()` — a simple volatile read that completes in O(1) steps regardless of contention; no CAS, no retry, always terminates in one step

**Staff-level phrasing:** "Lock-free = system progress guaranteed; wait-free = per-thread bounded-step guarantee — `ConcurrentLinkedQueue.offer()` is lock-free (CAS retry, individual threads may spin); `AtomicInteger.get()` is wait-free (volatile read, O(1) unconditional)."

</details>

<details>
<summary>Exercise 2 — ABA problem: timeline and AtomicStampedReference fix</summary>

**ABA timeline on a lock-free stack (`AtomicReference<Node>`):**
```
Initial stack: A → B → C
Thread 1 reads head = A, about to CAS(A, B)
Thread 1 is preempted.
Thread 2 pops A, pops B (stack = C)
Thread 2 pushes A back (stack = A → C, but B is freed/reused)
Thread 1 resumes: CAS(A, B) SUCCEEDS (A is still head)
Stack is now: B → C — but B was freed! Dangling pointer.
```

The CAS succeeded because the reference value matched, but the structure underneath had changed.

**Fix with `AtomicStampedReference`:**
```java
AtomicStampedReference<Node> head = new AtomicStampedReference<>(nodeA, 0);

// Pop:
int[] stampHolder = new int[1];
Node current = head.get(stampHolder);
int stamp = stampHolder[0];
// ... compute next ...
head.compareAndSet(current, next, stamp, stamp + 1);  // fails if stamp changed
```

Each mutation increments the stamp. Thread 1's CAS now checks `(A, stamp=0)`. After Thread 2's operations, `stamp=2`. Thread 1's CAS fails because stamp 0 ≠ 2, forcing a retry with the correct current state.

**Staff-level phrasing:** "ABA lets a CAS succeed on a structurally changed structure because reference equality alone can't distinguish 'same object returned' from 'never changed'; `AtomicStampedReference` pairs a version counter with the reference so the CAS fails on any intermediate modification even if the pointer looks the same."

</details>

<details>
<summary>Exercise 3 — ConcurrentLinkedQueue: weakCompareAndSet vs compareAndSet</summary>

In `ConcurrentLinkedQueue.offer()`, two CAS operations occur:
1. **`node.next.compareAndSet(null, newNode)`** — links the new node into the queue. This MUST succeed exactly once per offer; using `compareAndSet` ensures that if two threads race, exactly one wins and the other retries from the top.
2. **`tail.compareAndSet(t, newNode)`** — advances the tail pointer to the new node. This uses `weakCompareAndSet` (or a relaxed CAS).

`weakCompareAndSet` may fail **spuriously** — it can return `false` even when the expected value matches. This is safe here because the algorithm is designed with a "lazy tail" — the tail is allowed to lag one step behind the actual last node. If the tail CAS fails spuriously, the next `offer()` call will advance the tail as part of its own operation (the "helping" pattern). The correctness invariant (a node is always linked before any tail update) is preserved by the `compareAndSet` on `node.next`. The tail CAS is a best-effort optimization, not a correctness requirement.

**Staff-level phrasing:** "The `next` pointer CAS uses strong CAS because correct linking is a correctness invariant — one winner required; the `tail` CAS uses weak CAS because tail lag is intentional — a spurious failure is recovered by the next enqueue's helping step, so spurious failure is free."

</details>

<details>
<summary>Exercise 4 — Coding challenge: OneTimeLatch with AtomicBoolean</summary>

```java
// Reference implementation (Java 21+, compilable standalone)
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class OneTimeLatch {
    private final AtomicBoolean set = new AtomicBoolean(false);

    /** Returns true if this call was the one that set it; false if already set. */
    public boolean trySet() {
        return set.compareAndSet(false, true);  // CAS: only one thread wins
    }

    public boolean isSet() {
        return set.get();
    }

    public static void main(String[] args) throws InterruptedException {
        var latch = new OneTimeLatch();
        var successCount = new AtomicInteger(0);
        int threadCount = 100;
        var startLatch = new CountDownLatch(1);
        var doneLatch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            Thread.ofVirtual().start(() -> {
                try { startLatch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                if (latch.trySet()) successCount.incrementAndGet();
                doneLatch.countDown();
            });
        }

        startLatch.countDown();  // release all threads simultaneously
        doneLatch.await();

        assert successCount.get() == 1 : "Expected exactly 1 success, got " + successCount.get();
        assert latch.isSet() : "Latch should be set";
        System.out.println("All assertions passed. One winner out of " + threadCount + " threads.");
    }
}
```

**Why this works:** `AtomicBoolean.compareAndSet(false, true)` is a single atomic CAS: exactly one thread will observe `false` and successfully write `true`; all others see `true` on their CAS (either because the winner already wrote it, or because they read the updated value) and return `false`. No lock, no synchronization — the CAS itself is the serialization point.

**Common mistake:** Using `if (!set.get()) { set.set(true); return true; }` — this has a TOCTOU race: two threads can both read `false`, then both set `true` and both return `true`. The check-then-act must be atomic, which only CAS provides.

</details>

<details>
<summary>Exercise 5 — When NOT to use lock-free algorithms</summary>

Three scenarios where blocking locks are better:

1. **Long critical sections with complex state.** Lock-free algorithms work by retrying on CAS failure — if the work inside the "critical section" is expensive (e.g., sorting a list, calling an external service), retrying it on every collision is MORE expensive than holding a lock for the duration. Lock-free is optimal for short, fixed-cost atomic operations.

2. **Low contention or single-threaded contexts.** Lock-free algorithms have higher constant overhead (CAS, volatile reads, retry loop setup) than a simple `synchronized` block on an uncontended monitor. On x86, an uncontended `synchronized` is a few nanoseconds; a CAS loop with bookkeeping can be slower. Use `synchronized` for objects accessed by one or two threads.

3. **Fairness is required.** Lock-free algorithms (including wait-free `AtomicInteger`) give no ordering guarantee — a thread can be starved if other threads keep succeeding their CAS. `ReentrantLock(true)` (fair lock) guarantees FIFO ordering among waiting threads. Use a fair lock for scenarios like task schedulers, rate limiters, or any system where starvation is unacceptable.

**Staff-level phrasing:** "Avoid lock-free when: (1) the protected operation is long-running — CAS retry multiplies cost under contention; (2) contention is low — uncontended `synchronized` is faster than CAS overhead; (3) fairness/FIFO ordering is required — CAS gives no ordering guarantee, use `ReentrantLock(fair=true)`."

</details>

---

## 9. Summary / Flashcard

- **CAS is the CPU primitive for lock-free**: atomically compare-then-swap; if contention causes failure, the retry loop ensures at least one thread always makes progress — this is the definition of lock-free
- **ABA corrupts linked structures**: when a value goes A→B→A, naïve CAS sees A and succeeds even though the structure changed; fix with `AtomicStampedReference` which pairs a stamp (version counter) alongside the reference
- **Michael-Scott queue is lock-free via helping**: an enqueue leaves tail momentarily lagging; other threads detect and fix this rather than blocking — cooperative progress is the key
- **StampedLock optimistic reads remove all locking overhead**: read with no lock, validate consistency afterward; 10-11x throughput over `ReadWriteLock` in read-heavy (>95%) workloads; fall back to read-lock if validation fails
- **LongAdder beats AtomicLong above 4-thread contention**: striped CAS distributes pressure across N cells; `sum()` is approximate but sufficient for metrics/counters; at 64-thread contention: 22x faster than AtomicLong
