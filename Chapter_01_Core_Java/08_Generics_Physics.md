# 08: Generics Physics — Type Erasure, Variance & API Design

## Learning Objectives

After this module you can:
- Explain type erasure: what information the compiler removes and what survives to runtime
- Apply the PECS rule (Producer Extends, Consumer Super) to design correct, maximally flexible generic APIs
- Use bounded wildcards to write methods that accept `List<Integer>`, `List<Double>`, and `List<Number>` with a single signature
- Explain heap pollution: how it happens, why `@SuppressWarnings("unchecked")` is sometimes necessary, and why it requires a justification comment
- Read JDK source that uses raw types and unchecked casts and explain why those casts are safe

## Prerequisites

- Ch 1.0 Functional Foundations (`Function<T,R>`, `Optional<T>`, sealed interfaces — generics are the foundation of all of these)
- Ch 1.2 Modern Language Paradigms (records and sealed interfaces as generic types)

---

## The Critical Dialogue

**Student:** I wrote `void process(List<Number> numbers)` but the compiler rejects `process(myIntegerList)`. `Integer` extends `Number`, so why doesn't `List<Integer>` extend `List<Number>`?

**Principal:** Because Java generics are *invariant* by default — by design. If `List<Integer>` were a `List<Number>`, you could call `list.add(3.14)` through the `Number` reference, silently inserting a `Double` into what is actually a `List<Integer>`. The type system prevents this compile-time corruption. But "I only ever *read* from this list" is a different contract than "I read and write" — and bounded wildcards let you express exactly that distinction. Mastering PECS is what separates a developer who fights the type system from one who uses it to encode API contracts.

---

## 1. Type Erasure: What the Compiler Removes

Java generics are a compile-time construct only. Type parameters are **erased** before bytecode generation.

### 1.1 What erasure removes

```java
// At compile time — full type information:
List<String>  strings = new ArrayList<String>();
List<Integer> ints    = new ArrayList<Integer>();

// At runtime — after erasure:
List strings = new ArrayList();   // both become raw List
List ints    = new ArrayList();

strings.getClass() == ints.getClass(); // true — both are java.util.ArrayList
```

### 1.2 What you cannot do because of erasure

```java
public <T> void check(Object obj) {
    if (obj instanceof T) { /* ... */ }  // ❌ COMPILE ERROR: T erased, no runtime info
}

List<String>[] arr = new ArrayList<String>[10]; // ❌ COMPILE ERROR: generic array creation

public <T> T instantiate() {
    return new T(); // ❌ COMPILE ERROR: JVM doesn't know what T is at runtime
}

// ✅ Workaround: pass Class<T> as a runtime token
public <T> T instantiate(Class<T> clazz) throws Exception {
    return clazz.getDeclaredConstructor().newInstance(); // reflection uses runtime Class
}
```

### 1.3 Bridge methods: erasure in inheritance

When a generic type overrides a method from a parameterized supertype, the compiler generates a **bridge method** to satisfy the erased signature:

```java
public class StringBox implements Comparable<StringBox> {
    private final String value;

    @Override
    public int compareTo(StringBox other) {    // your method
        return value.compareTo(other.value);
    }
    // Compiler also generates (invisible in source, visible via reflection):
    // public int compareTo(Object o) {        ← bridge method (isBridge() == true)
    //     return compareTo((StringBox) o);    ← delegates to real method
    // }
    // This satisfies the erased Comparable.compareTo(Object) contract
}

// Verify bridge method exists:
for (Method m : StringBox.class.getDeclaredMethods()) {
    System.out.println(m.getName() + " isBridge=" + m.isBridge());
    // compareTo isBridge=false
    // compareTo isBridge=true   ← the generated bridge
}
```

---

## 2. Variance: Why `List<Integer>` Is Not a `List<Number>`

### 2.1 Arrays are covariant (and that was a mistake)

```java
// Arrays: covariant — Dog[] IS-A Animal[]
Animal[] animals = new Dog[3];   // allowed at compile time
animals[0] = new Cat();          // compiles! but: ArrayStoreException at runtime
// Runtime check required every time you write to an array — performance cost
```

Generics learned from this: they are **invariant** by default, enforcing correctness at compile time with zero runtime overhead.

### 2.2 Bounded wildcards: opt into covariance or contravariance

```
? extends T   (upper-bounded) → covariant     → you can READ T, cannot ADD
? super T     (lower-bounded) → contravariant  → you can ADD T, cannot READ precisely
```

```java
// Upper-bounded: "List of T or any subtype" — READING only
public double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();  // ✅ reading as Number
    // list.add(42);  // ❌ COMPILE ERROR — could be a List<Double>, adding int illegal
    return total;
}
sum(new ArrayList<Integer>());  // ✅
sum(new ArrayList<Double>());   // ✅
sum(new ArrayList<Number>());   // ✅

// Lower-bounded: "List of T or any supertype" — WRITING only
public void fill(List<? super Integer> dest, int start, int count) {
    for (int i = start; i < start + count; i++) dest.add(i);  // ✅ always valid
    // Integer n = dest.get(0);  // ❌ COMPILE ERROR — could be List<Number>, get() returns Object
    Object o = dest.get(0);      // ✅ Object is always safe
}
fill(new ArrayList<Integer>(), 1, 5);  // ✅
fill(new ArrayList<Number>(),  1, 5);  // ✅
fill(new ArrayList<Object>(),  1, 5);  // ✅
```

---

## 3. PECS: Producer Extends, Consumer Super

From *Effective Java* Item 31. One sentence that encodes every bounded-wildcard decision:

> **If a parameter PRODUCES values (you read from it) → `? extends T`**
> **If a parameter CONSUMES values (you write to it) → `? super T`**
> **If both → use exact `T`**

### 3.1 PECS in the JDK: `Collections.copy()`

```java
// java.util.Collections (OpenJDK 21)
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    //                           ↑ CONSUMER          ↑ PRODUCER
    // dest: we SET elements into it   → consumer  → ? super T
    // src:  we GET elements from it   → producer  → ? extends T
    int srcSize = src.size();
    for (int i = 0; i < srcSize; i++)
        dest.set(i, src.get(i));
}

// This allows copying from List<Integer> into List<Number>:
List<Integer> src  = List.of(1, 2, 3);
List<Number>  dest = new ArrayList<>(Arrays.asList(0, 0, 0));
Collections.copy(dest, src);   // ✅ Integer extends Number satisfies both bounds
```

### 3.2 PECS in practice: API before and after

```java
// ❌ BAD: too restrictive — callers cannot pass List<Integer>
public static double sumBad(List<Number> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}
// sumBad(integerList);  // COMPILE ERROR — unnecessary restriction

// ✅ GOOD: only reads → producer → extends
public static double sumGood(List<? extends Number> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}
// sumGood(integerList);  // ✅
// sumGood(doubleList);   // ✅

// ❌ BAD: too restrictive destination
public static void addIntegers(List<Number> dest, List<Integer> src) {
    dest.addAll(src);
}
// addIntegers(objectList, src);  // COMPILE ERROR — unnecessary restriction

// ✅ GOOD: only writes → consumer → super
public static void addIntegers(List<? super Integer> dest, List<Integer> src) {
    dest.addAll(src);
}
// addIntegers(numberList, src);  // ✅
// addIntegers(objectList, src);  // ✅
```

---

## 4. Heap Pollution and `@SuppressWarnings("unchecked")`

### 4.1 What heap pollution is

```java
List<String> strings = new ArrayList<>();
List rawList = strings;   // raw type — compiler warns but allows assignment
rawList.add(42);           // Integer inserted into what is "supposed to be" List<String>
// Heap is now polluted: strings variable promises String, but contains Integer

String s = strings.get(0); // ClassCastException at runtime — not here
```

### 4.2 Generic varargs creates heap pollution

```java
// T... is actually T[] — but generic arrays can't be created safely
// Compiler warns: "Possible heap pollution from parameterized vararg type"
public <T> List<T> listOf(T... elements) {
    return Arrays.asList(elements);
}

// @SafeVarargs: your assertion that the method doesn't pollute the heap
// Only valid if: you don't store the array reference, only read from it
@SafeVarargs
public final <T> List<T> safeListOf(T... elements) {
    return Arrays.asList(elements);  // safe: only reading, not storing array ref
}
```

### 4.3 When `@SuppressWarnings("unchecked")` is legitimate

```java
// Heterogeneous containers (type token pattern — used throughout Spring)
@SuppressWarnings("unchecked")  // safe: we only store/retrieve matching Class<T> + T pairs
public <T> T get(Class<T> type) {
    return (T) map.get(type);   // unchecked cast — but provably safe by construction
}
// ALWAYS add a comment explaining WHY the cast is safe.
// Without a comment, this is a maintenance hazard.
```

---

## 5. Source Archaeology

### 5.1 `Collections.sort` — PECS and `Comparable`

```java
// java.util.Collections (OpenJDK 21)
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}
// Dissecting the bound: T extends Comparable<? super T>
//
// Naive version: T extends Comparable<T>
//   Works for List<Integer> (Integer implements Comparable<Integer>) ✅
//   Fails for List<Dog> where Dog implements Comparable<Animal>     ❌
//   Dog implements Comparable<Animal>, not Comparable<Dog>
//
// With ? super T (PECS — Comparable<T> produces values for comparison):
//   Dog extends Comparable<? super Dog>
//   ? super Dog can be Dog, Animal, or Object
//   Dog implements Comparable<Animal> → Animal is ? super Dog → ✅
//
// Real-world: Employee class that implements Comparable<Person> (by salary)
// sort(List<Employee>) now works even though Employee is Comparable<Person>
```

### 5.2 `Class<T>` as a type token (used in Spring, Jackson, GSON)

```java
// java.lang.Class<T> is itself generic — used as a runtime type token
// Class<String> == the String class object

// Spring's BeanFactory:
// <T> T getBean(Class<T> requiredType)
// The Class<T> token carries T's type information past erasure into runtime

// Problem: Class<T> only works for non-parameterized types
// Class<List<String>> does NOT exist — you'd get Class<List> (erased)

// Solution: ParameterizedTypeReference (Spring) / TypeToken (Guava)
// Uses anonymous subclass to capture generic type in supertype signature:
new ParameterizedTypeReference<List<String>>() {}
// The anonymous class's getGenericSuperclass() returns the parameterized type
// This is how Jackson's ObjectMapper.readValue(json, new TypeReference<List<User>>(){}) works
```

---

## 6. Code Lab

### Lab 1: Generic event bus with type-safe PECS subscriber

```java
import java.util.*;
import java.util.function.Consumer;

public class EventBus {

    // Raw types in the map are unavoidable (erased generics across heterogeneous event types)
    // Safe because subscribe() and publish() maintain the Class<E> ↔ Consumer<E> invariant
    @SuppressWarnings("rawtypes")
    private final Map<Class, List<Consumer<Object>>> handlers = new HashMap<>();

    // Consumer<? super E>: PECS consumer — handler can accept E or any supertype
    // e.g., a Consumer<Object> can handle any event, a Consumer<OrderCreated> handles only that
    @SuppressWarnings("unchecked")
    public <E> void subscribe(Class<E> eventType, Consumer<? super E> handler) {
        handlers.computeIfAbsent(eventType, k -> new ArrayList<>())
            .add((Consumer<Object>) handler); // safe: we only call this with E instances
    }

    @SuppressWarnings("unchecked")
    public <E> void publish(E event) {
        List<Consumer<Object>> eventHandlers = handlers.get(event.getClass());
        if (eventHandlers != null) {
            eventHandlers.forEach(h -> h.accept(event));
        }
    }
}

// Usage
record OrderCreated(String orderId) {}
record OrderShipped(String orderId, String trackingId) {}

public class EventBusDemo {
    public static void main(String[] args) {
        var bus = new EventBus();

        bus.subscribe(OrderCreated.class, e -> System.out.println("Created: " + e.orderId()));
        bus.subscribe(OrderShipped.class, e -> System.out.println("Shipped: " + e.trackingId()));

        // Consumer<Object> can listen to everything (? super OrderCreated includes Object)
        bus.subscribe(OrderCreated.class, (Object e) -> System.out.println("Audit: " + e));

        bus.publish(new OrderCreated("ORD-001"));
        // prints: Created: ORD-001
        //         Audit: OrderCreated[orderId=ORD-001]
    }
}
```

### Lab 2: Why the naive version fails — and the PECS fix

```java
// Scenario: utility methods for a number crunching service

// ❌ Version 1: overly restrictive
public static double average(List<Number> list) {         // rejects List<Integer>
    return list.stream().mapToDouble(Number::doubleValue).average().orElse(0);
}

public static void addDefaults(List<Number> list, int n) { // rejects List<Object>
    for (int i = 0; i < n; i++) list.add(0);
}

// ✅ Version 2: PECS applied
public static double average(List<? extends Number> list) {   // producer → extends
    return list.stream().mapToDouble(Number::doubleValue).average().orElse(0);
}

public static void addDefaults(List<? super Integer> list, int n) { // consumer → super
    for (int i = 0; i < n; i++) list.add(0);
}

// Now this compiles:
List<Integer> ints    = new ArrayList<>(List.of(1, 2, 3));
List<Number>  numbers = new ArrayList<>();
List<Object>  objects = new ArrayList<>();

System.out.println(average(ints));      // ✅ was COMPILE ERROR before
addDefaults(numbers, 3);                // ✅ was COMPILE ERROR before
addDefaults(objects, 3);                // ✅ was COMPILE ERROR before
```

---

## 7. Production Lens

### Incident: Raw type causes silent ClassCastException in production

In a large monolith during a refactoring sprint, a `@SuppressWarnings` annotation was removed without investigating why it existed:

```
OBSERVED:
  Service ran fine for 3 weeks post-deploy
  Then: ClassCastException in serialization layer
  Stack trace pointed to random-looking line far from the actual bug
  NPE in a downstream null-check masked the real exception type

ROOT CAUSE:
  Legacy cache used raw Map — no type parameters
  New code added a generic helper that assumed Map<String, CacheEntry>
  Raw map had stored Integer keys in some code paths
  Type mismatch surfaced only when that particular cache entry was evicted/reloaded

BEFORE (dangerous):
  @SuppressWarnings("rawtypes")
  private Map cache = new HashMap();

AFTER (safe):
  private Map<String, CacheEntry> cache = new HashMap<>();  // compiler catches mismatches

LESSON: Never remove @SuppressWarnings without reading the context.
        Raw types trade compile-time safety for runtime ClassCastException.
```

### Benchmark: Generic method dispatch cost

```
Type erasure erases to Object + compiler-inserted cast. Cost:

Benchmark (JMH, Java 21, 10M iterations):
  Non-generic method call:              1.2 ns/op
  Generic method call (erased to cast): 1.2 ns/op   ← identical
  
Generics have ZERO runtime overhead compared to non-generic code.
The cost of raw types is correctness risk, not performance.
Heap pollution bugs cost days of debugging, not nanoseconds.
```

---

## 8. Exercises

**1.** Why does `new T()` fail to compile inside a generic method? Name two workarounds, and explain which one `ApplicationContext.getBean(Class<T>)` uses.

**2.** Explain why `Arrays.asList("a", "b")` returns `List<String>` but `Arrays.asList()` returns `List<Object>`. Trace through generic type inference. What does the compiler infer for `T` in each case?

**3.** `Collections.reverse(List<?>)` uses an unbounded wildcard. Can it modify the list? Why does the unbounded wildcard allow `set()` here when normally `List<?>` forbids it? (Hint: look at how it's implemented.)

**4. Coding challenge:** Implement `<T> Optional<T> findFirst(List<? extends T> source, Predicate<? super T> predicate)`. Apply PECS correctly for both parameters. Explain in a comment why each wildcard is the correct choice.

**5.** Write a class with a generic `compareTo` method that triggers bridge method generation, then verify the bridge method exists using reflection. Print `m.getName() + " " + m.isBridge()` for all declared methods.

---

## Exercise Solutions

<details>
<summary>Exercise 1 — Why new T() fails; two workarounds</summary>

Generics are erased at runtime — `T` becomes `Object` in bytecode. `new T()` would compile to `new Object()`, which is not what the caller wants. The compiler rejects it because it cannot know the correct constructor to call, and the JVM has no type token at runtime to dispatch on.

**Workaround 1 — Class token (`Class<T>`):**
```java
public static <T> T create(Class<T> type) throws Exception {
    return type.getDeclaredConstructor().newInstance();
}
// call: create(MyService.class)
```
The caller passes the `Class<T>` explicitly; reflection uses it to find and invoke the constructor. This is exactly what `ApplicationContext.getBean(Class<T>)` does — Spring holds the class token in its `BeanDefinition` registry and uses it to instantiate beans.

**Workaround 2 — Factory function (`Supplier<T>`):**
```java
public static <T> T create(Supplier<T> factory) {
    return factory.get();
}
// call: create(MyService::new)
```
No reflection, fully type-safe, preferred in modern code where the constructor is accessible at the call site.

**Staff-level phrasing:** "`new T()` fails because erasure removes `T` at runtime — pass a `Class<T>` token (reflection path, Spring's approach) or a `Supplier<T>` (factory path, preferred for type safety and no checked exceptions)."

</details>

<details>
<summary>Exercise 2 — Arrays.asList type inference: String vs Object</summary>

`Arrays.asList` signature: `public static <T> List<T> asList(T... a)`.

**`Arrays.asList("a", "b")`:** The compiler sees two `String` literals. Type inference infers `T = String` → returns `List<String>`. The varargs array created is `String[]`.

**`Arrays.asList()`:** No arguments are passed. The compiler cannot infer `T` from the arguments (there are none). With no type constraint from the arguments or the call site, `T` is inferred as its upper bound: `Object`. The call is equivalent to `Arrays.<Object>asList()` → returns `List<Object>`. The varargs array created is `Object[]`.

This is pure compile-time type inference — no runtime information involved. The fix if you want an empty `List<String>` is: `Collections.<String>emptyList()` or `new ArrayList<String>()` or to use a target-type hint: `List<String> list = Arrays.asList()` (explicit target type forces `T = String`).

**Staff-level phrasing:** "Type inference derives `T` from argument types; with no arguments, `T` falls back to `Object` — provide an explicit type witness `Arrays.<String>asList()` or use a typed target variable to force `T = String`."

</details>

<details>
<summary>Exercise 3 — Collections.reverse with unbounded wildcard: can it modify?</summary>

`Collections.reverse(List<?>)` CAN and DOES modify the list. Here is why:

Normally, `List<?>` (unbounded wildcard) forbids `add()` and `set()` because the compiler doesn't know the actual type parameter — calling `list.set(i, x)` would require knowing `x` is of the right type. However, `reverse()` uses an internal trick: it casts to a raw `List` to perform the swap:

```java
@SuppressWarnings("unchecked")
public static void reverse(List<?> list) {
    // simplified
    List rawList = list;
    for (int i = 0, j = size - 1; i < j; i++, j--) {
        Object tmp = rawList.get(i);
        rawList.set(i, rawList.get(j));
        rawList.set(j, tmp);
    }
}
```

The key insight: `reverse()` only stores back elements that it just read from the same list — it never introduces a new value of a different type. This is type-safe even without the compiler knowing the type parameter. The `@SuppressWarnings("unchecked")` is the contract: the implementer proves by inspection that the operation is safe; the wildcard on the API prevents callers from passing their own arbitrary values.

**Staff-level phrasing:** "`List<?>` forbids external `set()` calls because the type is unknown to callers, but `reverse()` sidesteps this via a raw cast — it only puts back elements it read from the same list, which is provably type-safe regardless of `?`; the wildcard is an API-boundary guarantee, not a runtime enforcement."

</details>

<details>
<summary>Exercise 4 — Coding challenge: findFirst with PECS</summary>

```java
// Reference implementation (Java 21+, compilable standalone)
import java.util.*;
import java.util.function.*;

public class PecsExample {

    // PECS: source produces T → extends; predicate consumes T → super
    public static <T> Optional<T> findFirst(
            List<? extends T> source,       // Producer Extends: we READ T from source
            Predicate<? super T> predicate  // Consumer Super: predicate CONSUMES T
    ) {
        for (T item : source) {
            if (predicate.test(item)) {
                return Optional.of(item);
            }
        }
        return Optional.empty();
    }

    public static void main(String[] args) {
        // source is List<Integer> (subtype of Number) — PECS allows this
        List<Integer> integers = List.of(1, 2, 3, 4, 5);

        // predicate is Predicate<Number> (supertype of Integer) — PECS allows this
        Predicate<Number> greaterThanThree = n -> n.doubleValue() > 3.0;

        Optional<Number> result = findFirst(integers, greaterThanThree);
        assert result.isPresent() && result.get().equals(4) : "Expected 4, got " + result;

        System.out.println("Found: " + result.get());
    }
}
```

**Why each wildcard is correct:**
- `List<? extends T>`: we only READ from `source` (producer) — `extends` allows subtypes of `T` to be passed. Without it, `findFirst(List<Integer>, ...)` would fail when `T=Number`.
- `Predicate<? super T>`: the predicate CONSUMES elements of type `T` — `super` allows supertypes of `T` to be passed. Without it, `Predicate<Number>` would fail when `T=Integer`.

**Staff-level phrasing:** "PECS: `? extends T` on input collections you read from (producer); `? super T` on functional interfaces that consume values — maximizes call-site flexibility without sacrificing type safety at the implementation level."

</details>

<details>
<summary>Exercise 5 — Bridge method generation via reflection</summary>

```java
// Reference implementation (Java 21+, compilable standalone)
import java.lang.reflect.*;
import java.util.*;

public class BridgeMethodDemo {

    // Generic base: erases to Comparable<Object> at runtime
    static class OrderId implements Comparable<OrderId> {
        final int value;
        OrderId(int v) { this.value = v; }

        @Override
        public int compareTo(OrderId other) {
            return Integer.compare(this.value, other.value);
        }
    }

    public static void main(String[] args) {
        System.out.println("Methods on OrderId:");
        for (Method m : OrderId.class.getDeclaredMethods()) {
            System.out.printf("  name=%-20s bridge=%s%n", m.getName(), m.isBridge());
        }
    }
}
/*
Expected output:
  name=compareTo           bridge=false   ← our implementation: compareTo(OrderId)
  name=compareTo           bridge=true    ← compiler bridge: compareTo(Object) → delegates to compareTo(OrderId)
*/
```

**Why the bridge method exists:** After erasure, `Comparable<OrderId>` becomes `Comparable<Object>`. The JVM requires a method with signature `compareTo(Object)` to satisfy the erased interface contract. But our class defines `compareTo(OrderId)`. The compiler generates a synthetic bridge method `compareTo(Object)` that casts the argument to `OrderId` and delegates — this makes the class usable as `Comparable<Object>` at the raw type level (e.g., `Collections.sort(List<OrderId>)` which calls `compareTo(Object)` via the raw `Comparable` interface).

**Staff-level phrasing:** "Bridge methods bridge the gap between the erased generic interface (`compareTo(Object)`) and the specific override (`compareTo(OrderId)`) — the compiler generates a synthetic cast-and-delegate method, visible via `method.isBridge() == true`, enabling correct generic dispatch through the erased type system."

</details>

---

## 9. Summary / Flashcard

- **Generics are compile-time only — erased at runtime**: `List<String>` and `List<Integer>` share the same `ArrayList` class at runtime; `instanceof List<String>` is impossible; `new T()` is impossible
- **Invariance prevents heap corruption**: `List<Integer>` is not `List<Number>` because adding a `Double` through a `Number` reference would corrupt the `Integer` list — use wildcards to express partial substitutability
- **PECS — Producer Extends, Consumer Super**: if you READ from a generic container use `? extends T`; if you WRITE to it use `? super T`; maximises API flexibility without sacrificing type safety
- **Bridge methods are compiler-generated**: when a generic type overrides an erased supertype method, the compiler generates a synthetic bridge method visible via `method.isBridge()` — this is how generic inheritance satisfies raw-type contracts
- **`@SuppressWarnings("unchecked")` requires a justification comment**: unchecked casts are sometimes unavoidable (type tokens, heterogeneous containers) but must be proven safe; without a comment they become invisible time bombs that cause `ClassCastException` months later
