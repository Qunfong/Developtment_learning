# 05: Java Memory Model & Concurrency Primitives

## Learning Objectives

After this module you can:
- Explain the happens-before relationship and its five core rules with concrete code examples
- Identify CPU-cache visibility bugs and instruction-reordering bugs by reading code
- Choose correctly between `volatile`, `synchronized`, and `AtomicXxx` for a given access pattern
- Read OpenJDK source to understand how Java maps happens-before to hardware memory barriers
- Diagnose race conditions using the JMM formal model rather than trial-and-error

## Prerequisites

- Ch 1.0 Functional Foundations (understanding Java's type system and lambdas)
- Ch 1.3 Project Loom Revolution (virtual threads build on JMM — understand the base first)
- Basic familiarity with threads: `new Thread()`, `Runnable`, `ExecutorService`

---

## The Critical Dialogue

**Student:** I have an intermittent bug. A background thread updates a `boolean running` field to `false` to stop a worker thread. The worker thread never stops. I added `volatile` and it works, but I don't understand why. And now my colleague says `volatile` is not enough for a counter — we need `AtomicInteger`. Why does the same problem need three different solutions?

**Principal:** You just stumbled into the most misunderstood part of Java. The problem isn't the language — it's the hardware. Modern CPUs lie to your program. They have private caches, store buffers, and reorder instructions for performance. The JVM spec had to define *exactly* which lies are permitted and which are not — that spec is the **Java Memory Model**. Once you understand JMM, the difference between `volatile`, `synchronized`, and `AtomicInteger` becomes obvious: they each purchase different *guarantees* from the hardware at different *costs*.

---

## 1. Why the CPU Lies: Hardware Visibility Basics

### 1.1 The CPU Cache Problem

A modern x86 server has this memory hierarchy:

```
┌─────────────────────────────────────────────────────────────┐
│  Core 0                          Core 1                     │
│  ┌──────────┐                    ┌──────────┐               │
│  │ L1 Cache │ ← private, ~4 ns   │ L1 Cache │ ← private     │
│  │  32 KB   │                    │  32 KB   │               │
│  └────┬─────┘                    └────┬─────┘               │
│       │                               │                     │
│  ┌────▼─────┐                    ┌────▼─────┐               │
│  │ L2 Cache │ ~12 ns             │ L2 Cache │               │
│  │  256 KB  │                    │  256 KB  │               │
│  └────┬─────┘                    └────┬─────┘               │
│       └──────────────┬────────────────┘                     │
│                 ┌────▼─────┐                                 │
│                 │ L3 Cache │ ~40 ns, SHARED                  │
│                 │  8-32 MB │                                 │
│                 └────┬─────┘                                 │
│                 ┌────▼─────┐                                 │
│                 │   RAM    │ ~100 ns                         │
│                 └──────────┘                                 │
└─────────────────────────────────────────────────────────────┘
```

When Core 0 writes `running = false`, the value lands in Core 0's L1 cache and a **store buffer**. It may take microseconds before it flushes to L3 and Core 1's L1 invalidates its cached copy. Without a memory barrier, Core 1 reads stale data from its own L1.

### 1.2 Compiler and CPU Reordering

The JVM JIT compiler and the CPU both reorder instructions for performance, as long as the result looks correct **within a single thread**. They cannot see across thread boundaries.

```java
// Your code:
x = 1;
y = 2;

// CPU may execute as:
y = 2;   // reordered — single-thread result identical
x = 1;
```

This is safe for single-threaded code but catastrophic if another thread reads `x` expecting `y` to already be 2.

### 1.3 The Four Hardware Memory Barriers

To stop reordering, CPUs have memory fence instructions:

| Barrier | Prevents |
|---------|---------|
| `LoadLoad` | Load before barrier cannot be reordered after load after barrier |
| `LoadStore` | Load before barrier cannot be reordered after store after barrier |
| `StoreStore` | Store before barrier cannot be reordered after store after barrier |
| `StoreLoad` | Store before barrier cannot be reordered after load after barrier — most expensive |

`volatile` write inserts `StoreStore + StoreLoad`. `volatile` read inserts `LoadLoad + LoadStore`. This is why `volatile` costs more than a plain field access (~1 ns volatile read vs ~0.3 ns plain field read).

---

## 2. The Java Memory Model: Happens-Before

The JMM defines **happens-before** (HB): if action A happens-before action B, then A's effects are visible to B. This is the formal contract between your code and the JVM/CPU.

### 2.1 The Five Core Rules

```
RULE 1 — Program Order Rule
  Within a single thread, every action HB every subsequent action.
  (The CPU can reorder internally, but the result must appear ordered.)

RULE 2 — Monitor Lock Rule
  An unlock of a monitor HB every subsequent lock of that same monitor.
  → synchronized(lock) { write x } HB synchronized(lock) { read x }

RULE 3 — Volatile Variable Rule
  A write to a volatile field HB every subsequent read of that field.
  → volatile write HB volatile read (if read sees the write)

RULE 4 — Thread Start Rule
  Thread.start() on a thread HB any action in that started thread.
  → Everything before thread.start() is visible to the new thread.

RULE 5 — Thread Join Rule
  All actions in a thread HB thread.join() returning in another thread.
  → Everything the child thread did is visible after join() returns.
```

HB is **transitive**: if A HB B and B HB C, then A HB C.

### 2.2 Visibility Bug: Missing Happens-Before

```java
// BUG: No happens-before between writer and reader thread
public class StopFlag {
    private boolean running = true;  // NOT volatile

    public void stop() {
        running = false;             // Writer: Core 0
    }

    public void loop() {
        while (running) {            // Reader: Core 1 — may NEVER see false
            doWork();
        }
    }
}
```

The writer thread writes to `running` in Core 0's L1 cache. The reader thread on Core 1 reads its own cached copy — which is still `true`. No happens-before exists between the two threads for this field. **This is not a bug in the JVM; it is correct behavior according to JMM.**

```java
// FIX: volatile establishes happens-before via Rule 3
public class StopFlag {
    private volatile boolean running = true;  // volatile write HB volatile read

    public void stop() {
        running = false;             // Forces flush + StoreLoad barrier
    }

    public void loop() {
        while (running) {            // Forces cache invalidation + LoadLoad barrier
            doWork();
        }
    }
}
```

---

## 3. volatile vs synchronized vs AtomicInteger

### 3.1 Why volatile Is Not Enough for a Counter

```java
// BUG: volatile does NOT make compound operations atomic
private volatile int counter = 0;

public void increment() {
    counter++;   // This is THREE operations: read → increment → write
                 // Two threads can both read 0, both compute 1, both write 1
                 // Result: counter = 1, not 2
}
```

`volatile` gives you **visibility** (Rule 3) but NOT **atomicity**. `counter++` decomposes into read-modify-write, and another thread can interleave between read and write.

### 3.2 Decision Matrix

```
NEED                              USE
─────────────────────────────────────────────────────────────
Simple flag (write/read, no RMW)  volatile
                                  → StopFlag, published singleton

Single numeric counter (RMW)      AtomicInteger / AtomicLong
                                  → Uses CAS, lock-free

Object reference swap             AtomicReference
                                  → Publish immutable config

Guard multiple fields together    synchronized
                                  → Account balance + history together

Highest throughput, read-heavy    StampedLock (Ch 1.6)
```

### 3.3 synchronized and the Monitor Lock Rule

```java
public class SafeCounter {
    private int counter = 0;  // NOT volatile — guarded by lock

    public synchronized void increment() {
        counter++;             // unlock at exit: HB next synchronized enter
    }

    public synchronized int get() {
        return counter;        // lock acquire: sees all previous unlocks
    }
}
```

When `increment()` exits, the `unlock` happens-before any subsequent `lock` on the same monitor. So `get()` always sees the latest value. The `counter` field does NOT need `volatile` — the lock provides a stronger guarantee.

---

## 4. Source Archaeology

### 4.1 `sun.misc.Unsafe` — The Bridge to Hardware Barriers

All JDK concurrency primitives ultimately call `sun.misc.Unsafe` (or its successor `jdk.internal.misc.Unsafe`). Reading this reveals how Java maps HB rules to machine code.

```java
// jdk.internal.misc.Unsafe (OpenJDK 21)
// Source: jdk/src/hotspot/share/prims/unsafe.cpp

public native void putVolatile(Object o, long offset, Object x);
// Maps to: StoreStore barrier + StoreLoad barrier + store + LoadLoad barrier

public native void putOrdered(Object o, long offset, Object x);
// Maps to: StoreStore barrier only — CHEAPER, but only store-release semantics
// Used by: ConcurrentLinkedQueue, SynchronousQueue
```

Key insight: `putOrdered` (store-release) is 2-3x cheaper than `putVolatile` (full fence) because it skips `StoreLoad`. It provides enough guarantee for a single producer publishing to a single consumer, but NOT for arbitrary multi-producer scenarios.

### 4.2 `VarHandle` — The Modern API (Java 9+)

```java
// java.lang.invoke.VarHandle — replaces direct Unsafe usage
// Source: java/lang/invoke/VarHandleInts.java (OpenJDK 21)

// Four access modes (ordered by cost, cheapest first):
varHandle.get(obj)                // plain — no barrier
varHandle.getOpaque(obj)          // opaque — prevents JIT hoisting
varHandle.getAcquire(obj)         // acquire — LoadLoad + LoadStore
varHandle.getVolatile(obj)        // volatile — all 4 barriers

// AtomicInteger.get() internally calls:
// VALUE.getVolatile(this)  where VALUE is a VarHandle
```

### 4.3 `AbstractQueuedSynchronizer` — The Foundation of Everything

`ReentrantLock`, `Semaphore`, `CountDownLatch`, `ForkJoinPool` — all use AQS.

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer (OpenJDK 21)
// Line ~386:

protected final boolean compareAndSetState(int expect, int update) {
    return STATE.compareAndSet(this, expect, update);
    // STATE is a VarHandle with volatile access mode
    // This CAS provides full fence semantics on x86
}
```

The AQS state field is the single volatile variable that establishes happens-before for the entire lock. When `unlock()` writes state=0, and `lock()` reads state=0, Rule 3 kicks in — every write before the unlock is visible after the lock.

---

## 5. Code Lab

### Lab 1: Reproduce and Fix a Visibility Bug

```java
import java.util.concurrent.TimeUnit;

public class VisibilityBugLab {

    // ❌ BAD: missing volatile — worker may never stop
    static class BrokenWorker implements Runnable {
        private boolean running = true;  // NOT volatile

        public void stop() { running = false; }

        @Override
        public void run() {
            long count = 0;
            while (running) { count++; }
            System.out.println("Stopped at: " + count);
        }
    }

    // ✅ GOOD: volatile ensures visibility
    static class FixedWorker implements Runnable {
        private volatile boolean running = true;  // volatile

        public void stop() { running = false; }

        @Override
        public void run() {
            long count = 0;
            while (running) { count++; }
            System.out.println("Stopped at: " + count);
        }
    }

    public static void main(String[] args) throws Exception {
        // Test broken worker — will likely hang or run much longer
        var broken = new BrokenWorker();
        var t1 = new Thread(broken);
        t1.start();
        TimeUnit.MILLISECONDS.sleep(100);
        broken.stop();
        t1.join(500);  // timeout — may still be running!
        if (t1.isAlive()) {
            System.out.println("❌ BrokenWorker still running — visibility bug confirmed");
            t1.interrupt();
        }

        // Test fixed worker — stops reliably
        var fixed = new FixedWorker();
        var t2 = new Thread(fixed);
        t2.start();
        TimeUnit.MILLISECONDS.sleep(100);
        fixed.stop();
        t2.join(500);
        System.out.println(t2.isAlive() ? "❌ Still running" : "✅ FixedWorker stopped correctly");
    }
}
```

### Lab 2: volatile Is Not Atomic — Counter Race

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicityLab {

    // ❌ BAD: volatile counter — loses increments under contention
    static volatile int volatileCounter = 0;

    // ✅ GOOD: AtomicInteger — CAS-based, no lost increments
    static AtomicInteger atomicCounter = new AtomicInteger(0);

    public static void main(String[] args) throws Exception {
        int threads = 4, incrementsPerThread = 100_000;
        int expected = threads * incrementsPerThread;

        // Test volatile counter
        volatileCounter = 0;
        var latch1 = new CountDownLatch(threads);
        for (int i = 0; i < threads; i++) {
            new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) volatileCounter++;
                latch1.countDown();
            }).start();
        }
        latch1.await();
        System.out.printf("volatile:  expected=%d, actual=%d, lost=%d%n",
            expected, volatileCounter, expected - volatileCounter);
        // Output: volatile: expected=400000, actual=312847, lost=87153 (varies)

        // Test AtomicInteger
        atomicCounter.set(0);
        var latch2 = new CountDownLatch(threads);
        for (int i = 0; i < threads; i++) {
            new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) atomicCounter.incrementAndGet();
                latch2.countDown();
            }).start();
        }
        latch2.await();
        System.out.printf("atomic:    expected=%d, actual=%d, lost=%d%n",
            expected, atomicCounter.get(), expected - atomicCounter.get());
        // Output: atomic: expected=400000, actual=400000, lost=0 ✅
    }
}
```

### Lab 3: JMH Benchmark — volatile vs synchronized vs AtomicInteger (uncontended)

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class MemoryOrderingBenchmark {

    private int plainField = 0;
    private volatile int volatileField = 0;
    private final AtomicInteger atomicField = new AtomicInteger(0);
    private final Object lock = new Object();
    private int syncField = 0;

    @Benchmark
    public int plainRead() { return plainField; }

    @Benchmark
    public int volatileRead() { return volatileField; }

    @Benchmark
    public int atomicRead() { return atomicField.get(); }

    @Benchmark
    public int synchronizedRead() {
        synchronized (lock) { return syncField; }
    }

    @Benchmark
    public void plainWrite() { plainField = 42; }

    @Benchmark
    public void volatileWrite() { volatileField = 42; }

    @Benchmark
    public void atomicIncrement() { atomicField.incrementAndGet(); }

    @Benchmark
    public void synchronizedIncrement() {
        synchronized (lock) { syncField++; }
    }
}

/*
Typical results on x86 (uncontended, single thread):
──────────────────────────────────────────────────────
Benchmark                   Score   Units
─────────────────────────── ─────── ──────
plainRead                    0.28   ns/op   ← register/L1 cache
volatileRead                 1.12   ns/op   ← barrier forces cache check
atomicRead                   1.15   ns/op   ← equivalent to volatile read
synchronizedRead            12.40   ns/op   ← lock acquisition overhead

plainWrite                   0.28   ns/op
volatileWrite                6.80   ns/op   ← StoreStore + StoreLoad fence
atomicIncrement              3.90   ns/op   ← CAS (no StoreLoad on x86 — free!)
synchronizedIncrement       14.20   ns/op   ← lock + unlock

KEY INSIGHT: On x86, CAS is cheaper than volatile write because x86 TSO
(Total Store Order) memory model gives StoreLoad for free on LOCK prefix.
On ARM/RISC-V, the difference narrows.
*/
```

---

## 6. Production Lens

### Incident 1: The Double-Checked Locking Classic (Pre-Java 5)

Before Java 5's JMM revision, this pattern was broken:

```java
// ❌ BROKEN before Java 5 — still wrong without volatile even on modern JVMs
// if compiled against pre-Java5 semantics
public class BrokenSingleton {
    private static BrokenSingleton instance;

    public static BrokenSingleton getInstance() {
        if (instance == null) {              // Check 1
            synchronized (BrokenSingleton.class) {
                if (instance == null) {      // Check 2
                    instance = new BrokenSingleton();  // 3 steps: alloc + init + publish
                    // CPU can reorder: alloc + PUBLISH (partial ref) + init later
                    // Another thread sees non-null instance, calls methods on half-init object
                }
            }
        }
        return instance;
    }
}

// ✅ CORRECT: volatile prevents publish-before-init reordering
public class SafeSingleton {
    private static volatile SafeSingleton instance;  // volatile!

    public static SafeSingleton getInstance() {
        if (instance == null) {
            synchronized (SafeSingleton.class) {
                if (instance == null) {
                    instance = new SafeSingleton();  // volatile write: StoreStore before
                }
            }
        }
        return instance;
    }
}

// ✅ BEST: Initialization-on-demand holder — no volatile, no synchronized in hot path
public class IdealSingleton {
    private static class Holder {
        static final IdealSingleton INSTANCE = new IdealSingleton();
        // Class loading is atomic (Rule 4: thread-start equivalent)
        // JVM guarantees class init completes before any thread can see the field
    }
    public static IdealSingleton getInstance() { return Holder.INSTANCE; }
}
```

**Production benchmark:** At 500k req/s, the DCL `volatile` read adds ~0.6ms/req overhead vs the holder pattern. At this scale: 300ms/sec of wasted CPU just from unnecessary volatile reads in a hot path.

### Incident 2: The False Sharing Tax

```java
// ❌ BAD: counters share a cache line — cores invalidate each other's cache
public class FalseSharingCounters {
    volatile long counter0 = 0;  // offset 16
    volatile long counter1 = 0;  // offset 24 — same 64-byte cache line as counter0!
    volatile long counter2 = 0;  // offset 32
    volatile long counter3 = 0;  // offset 40
}
// Result: 4 threads incrementing "independent" counters see 10x throughput drop
// because each write invalidates the entire cache line in all other cores

// ✅ FIXED: @Contended annotation adds 128 bytes of padding
import jdk.internal.vm.annotation.Contended;

public class PaddedCounters {
    @Contended volatile long counter0 = 0;  // own cache line
    @Contended volatile long counter1 = 0;  // own cache line
    @Contended volatile long counter2 = 0;  // own cache line
    @Contended volatile long counter3 = 0;  // own cache line
}
// Requires JVM flag: -XX:-RestrictContended
// Benchmark: 8x throughput improvement on 4-core write-heavy workload
```

**Key number:** A cache line invalidation costs ~40 cycles (~13 ns at 3GHz). With 4 threads invalidating per increment, false sharing turns a 1.1 ns volatile write into an effective 53 ns operation.

### Red Flags in Code Review

```
❌ volatile counter that gets counter++        → race condition, use AtomicLong
❌ synchronized method on 'this' object        → too coarse, use dedicated lock
❌ double-checked locking without volatile     → broken publication
❌ long/double fields in shared state          → not atomic on 32-bit JVMs (JMM §17.7)
❌ ThreadLocal used to avoid synchronization   → correct, but verify proper remove() on return-to-pool
```

---

## 7. Exercises

**1.** Explain why `volatile` is not sufficient for a thread-safe counter but IS sufficient for a stop flag. What is the fundamental property that distinguishes the two use cases?

**2.** Given this code, identify ALL happens-before relationships:
```java
int x = 0;
volatile int flag = 0;

// Thread A:         // Thread B:
x = 42;             while (flag == 0) {}
flag = 1;           System.out.println(x);  // What does this print?
```

**3.** A colleague says: "I can use a `HashMap` in a multi-threaded app as long as reads are more frequent than writes." Evaluate this claim using JMM reasoning. What is the correct solution?

**4. Coding challenge:** Implement a thread-safe lazy-initializing cache using only `volatile` (no `synchronized`, no `AtomicReference`). The cache computes a value once and returns it forever. Write a test that verifies correctness under concurrent access.

**5.** Why does `x86` make `AtomicInteger.incrementAndGet()` cheaper than a `volatile` write, while `ARM` does not? What property of `x86 TSO` explains this?

---

## Exercise Solutions

<details>
<summary>Exercise 1 — volatile: sufficient for stop flag, insufficient for counter</summary>

`volatile` provides two guarantees: **visibility** (a write is immediately flushed to main memory, subsequent reads see the new value) and **ordering** (writes/reads cannot be reordered across the volatile variable). It does NOT provide **atomicity** for compound operations.

A **stop flag** (`volatile boolean stopped`) needs only visibility: Thread A writes `stopped = true`, Thread B reads `stopped` and sees `true`. This is a single atomic write followed by a single atomic read — no compound operation involved. `volatile` is exactly sufficient.

A **counter** (`volatile int count; count++`) is a compound operation: read current value, increment, write new value. With two threads both executing `count++` concurrently: both may read the same value `42`, both increment to `43`, both write `43`. The result is `43` instead of the correct `44`. The read-increment-write sequence is not atomic even with `volatile`. Use `AtomicInteger.incrementAndGet()` which uses a CAS loop to make the entire operation atomic.

**Staff-level phrasing:** "`volatile` provides visibility + ordering, not atomicity; `stopped = true` is a single atomic write so `volatile` suffices; `count++` is read-modify-write requiring CAS via `AtomicInteger` — mixing up these two cases is one of the most common JMM bugs in production."

</details>

<details>
<summary>Exercise 2 — Happens-before relationships in the flag example</summary>

```java
int x = 0;
volatile int flag = 0;
// Thread A:         // Thread B:
x = 42;             while (flag == 0) {}
flag = 1;           System.out.println(x);
```

Happens-before relationships:
1. `x = 42` **HB** `flag = 1` — program order rule within Thread A
2. `flag = 1` (write) **HB** `while(flag == 0)` exit (read that observes the write) — volatile write HB subsequent volatile read
3. By transitivity: `x = 42` **HB** `System.out.println(x)`

**Result:** Thread B prints **42** — guaranteed. The volatile write to `flag` establishes a happens-before edge that carries all preceding writes in Thread A (including `x = 42`) to Thread B. This is the canonical pattern for safe publication via `volatile`.

**Staff-level phrasing:** "Volatile write-to-read establishes HB; by transitivity all Thread A writes before `flag=1` are visible to Thread B after it reads `flag==1` — result is deterministically 42, not a data race."

</details>

<details>
<summary>Exercise 3 — HashMap in multi-threaded app with mostly reads</summary>

The claim is wrong. `HashMap` is not safe under any concurrent use, regardless of read/write ratio.

The specific danger: a `HashMap.put()` that triggers a **resize** (`rehash`) restructures the internal array while readers are iterating buckets. A reader can:
- Follow a `next` pointer into a **cycle** (infinite loop — historically from Java 6 resize, fixed in Java 8's tree-ification of long chains, but still possible with corrupt state)
- Observe a partially-written entry and read `null` from a non-null key
- Miss a key that was fully inserted before the current thread started

The "mostly reads" heuristic is dangerous: even one concurrent write invalidates all concurrent reads without synchronization.

**Correct solutions:**
- Read-heavy, write-rare: `ConcurrentHashMap` (lock-striped, readers never block, O(1) per segment)
- Read-only after construction: plain `HashMap` wrapped in `Collections.unmodifiableMap()`, accessed after a safe publication guarantee (e.g., `final` field or `volatile`)
- Never: `Collections.synchronizedMap()` for high-concurrency — coarse lock serializes all access

**Staff-level phrasing:** "Any concurrent write to `HashMap` — even one — can corrupt the structure for all concurrent readers; use `ConcurrentHashMap` which provides lock-free reads via `volatile` array slots and CAS-based writes."

</details>

<details>
<summary>Exercise 4 — Coding challenge: volatile-only lazy-init cache</summary>

```java
// Reference implementation (Java 21+, compilable standalone)
import java.util.concurrent.*;
import java.util.function.*;

public class VolatileCache<T> {
    private volatile T value;
    private final Supplier<T> supplier;

    public VolatileCache(Supplier<T> supplier) {
        this.supplier = supplier;
    }

    public T get() {
        T v = value;                  // read once from volatile field
        if (v == null) {
            synchronized (this) {     // only one thread initializes
                v = value;            // re-read under lock
                if (v == null) {
                    v = supplier.get();
                    value = v;        // volatile write — publishes to all threads
                }
            }
        }
        return v;
    }

    // Test
    public static void main(String[] args) throws InterruptedException {
        var counter = new java.util.concurrent.atomic.AtomicInteger(0);
        var cache = new VolatileCache<>(() -> {
            counter.incrementAndGet();
            return "computed";
        });

        int threads = 100;
        var latch = new CountDownLatch(threads);
        var results = new java.util.concurrent.CopyOnWriteArrayList<String>();

        for (int i = 0; i < threads; i++) {
            Thread.ofVirtual().start(() -> {
                results.add(cache.get());
                latch.countDown();
            });
        }
        latch.await();

        assert counter.get() == 1 : "Supplier called " + counter.get() + " times, expected 1";
        assert results.stream().allMatch("computed"::equals) : "Some thread got wrong value";
        System.out.println("All assertions passed. Supplier called: " + counter.get());
    }
}
```

**Why this works:** This is the **double-checked locking** (DCL) pattern. The `volatile` read on the fast path avoids synchronization after initialization. The `synchronized` block ensures only one thread computes the value. The re-check inside the lock handles the race where two threads both saw `null` before synchronizing. The `volatile` write to `value` ensures the constructed object is **safely published** — all threads see a fully initialized object, not a partially constructed one (critical: without `volatile`, the JIT could reorder the write to `value` before the constructor completes).

**Common mistake:** Implementing DCL without `volatile` — `private T value` without volatile allows the JIT to cache the field in a register or reorder construction, making it possible for another thread to see a non-null but partially initialized object.

</details>

<details>
<summary>Exercise 5 — x86 TSO vs ARM: AtomicInteger cost difference</summary>

**x86 TSO (Total Store Order):** x86 has a very strong memory model. Every store is immediately visible to other cores (there is a store buffer, but stores are written through to the coherence protocol in order). `AtomicInteger.incrementAndGet()` on x86 compiles to a single `LOCK XADD` instruction — the `LOCK` prefix makes the read-modify-write atomic at the hardware level. There is NO additional memory barrier needed because x86 loads already have acquire semantics and stores have release semantics by default.

A plain `volatile` write on x86 emits a `SFENCE` (store fence) or `MFENCE` (full fence) which is MORE expensive than `LOCK XADD` — because the fence stalls the entire store buffer whereas `LOCK XADD` only serializes one address.

**ARM (Weak Memory Model):** ARM allows loads and stores to be reordered freely. `AtomicInteger.incrementAndGet()` on ARM requires: a `LDADD` (load-acquire-add) instruction plus a `STLR` (store-release) — or a CAS loop using `LDAXR`/`STLXR` with full barrier semantics. Both acquire and release barriers are explicitly emitted. A plain `volatile` write on ARM also requires only a `STLR` (store-release) — so a `volatile` write is cheaper or similar cost to an atomic increment on ARM, unlike on x86.

**Staff-level phrasing:** "On x86 TSO, `LOCK XADD` for atomic increment is cheaper than a full `MFENCE` volatile write because x86 already has strong ordering — no separate fence needed; on ARM's weak model, atomic increment requires explicit acquire+release barriers, matching or exceeding the cost of a volatile write."

</details>

---

## 8. Summary / Flashcard

- **JMM exists because CPUs cache and reorder**: without barriers, threads see stale values from private L1 caches; `volatile` forces cache flush and disables reordering via hardware fences
- **Happens-before (HB) is the formal contract**: volatile write HB volatile read; unlock HB next lock; start() HB child thread — if A HB B, A's writes are guaranteed visible to B
- **volatile = visibility only, not atomicity**: `counter++` is read+increment+write — two threads can both read the same old value; use `AtomicInteger` for compound operations
- **synchronized provides both**: the monitor unlock HB next lock establishes full visibility; the lock itself serializes execution for atomicity
- **False sharing destroys throughput**: two `volatile` fields on the same 64-byte cache line cause cross-core invalidation storms; `@Contended` pads fields to separate cache lines
