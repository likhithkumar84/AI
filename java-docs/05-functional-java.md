# Java Mastery — Functional Java

> **Goal:** Write expressive, concise Java using lambdas, functional interfaces, the Stream API, and Optional.
> This doc is a standalone companion to the [numbered Core Java index](00-core-java-index.md). Prerequisites: Language Foundations + OOP + Collections.

---

## 0) What this covers

1. Lambdas — syntax, scope, effectively-final variables.
2. Method references — four forms.
3. Functional interfaces — built-in and custom.
4. The Stream API — creation, intermediate operations, terminal operations, collectors.
5. `Optional<T>` — safe null handling.
6. Composition — function chaining and pipelines.
7. Interview questions woven throughout.

---

## 1) Lambdas

A **lambda** is an anonymous function — a concise way to represent a block of code that can be passed as data. Lambdas implement **functional interfaces** (interfaces with exactly one abstract method).

### 1.1 Syntax forms

```java
// Full form
(String s) -> { return s.toUpperCase(); }

// Inferred parameter type (compiler infers from context)
(s) -> { return s.toUpperCase(); }

// Single parameter — parens optional
s -> { return s.toUpperCase(); }

// Expression body (single expression — no braces, no return keyword)
s -> s.toUpperCase()

// No parameters
() -> System.out.println("Hello")
() -> 42

// Multiple parameters
(a, b) -> a + b
(String a, String b) -> a.compareTo(b)
```

### 1.2 Lambdas and effectively-final variables

Lambdas can capture local variables from the enclosing scope, but those variables must be **effectively final** — never reassigned after their declaration.

```java
String prefix = "Hello";           // effectively final — never changed
Runnable r = () -> System.out.println(prefix + " World");

// prefix = "Hi";  // COMPILE ERROR if uncommented — breaks effectively-final
r.run();           // "Hello World"

// Instance fields and static fields CAN be mutated freely (not covered by the restriction)
class Counter {
    private int count = 0;
    Runnable incrementor = () -> count++;  // OK — count is a field, not a local var
}
```

### 1.3 Lambda scope — `this` refers to the enclosing class

```java
class Formatter {
    private String prefix = ">>>";

    Runnable makeRunner() {
        return () -> System.out.println(this.prefix); // 'this' = Formatter instance
    }
}
```

---

## 2) Method References

A method reference is a shorthand for a lambda that simply calls an existing method. Four forms:

| Form | Syntax | Equivalent Lambda |
|------|--------|-------------------|
| Static method | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` |
| Instance method (bound) | `instance::method` | `(args) -> instance.method(args)` |
| Instance method (unbound) | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` |
| Constructor | `ClassName::new` | `(args) -> new ClassName(args)` |

```java
// 1. Static method reference
List<String> words = List.of("banana", "apple", "cherry");
words.stream()
     .map(String::toUpperCase)         // s -> s.toUpperCase()
     .forEach(System.out::println);    // s -> System.out.println(s)

// 2. Bound instance method reference
String prefix = "Hello";
Predicate<String> startsWithPrefix = prefix::startsWith;  // s -> prefix.startsWith(s)  -- wait, inverted
// Actually: bound = prefix::concat  → s -> prefix.concat(s)

// 3. Unbound instance method reference
Comparator<String> byLength  = Comparator.comparingInt(String::length);
// String::length = s -> s.length()  (s is supplied by the stream)

Function<String, Integer> len = String::length;
System.out.println(len.apply("hello")); // 5

// 4. Constructor reference
Function<String, StringBuilder> sbFactory = StringBuilder::new;
// s -> new StringBuilder(s)
StringBuilder sb = sbFactory.apply("hello");

Supplier<List<String>> listFactory = ArrayList::new;
// () -> new ArrayList<>()
List<String> newList = listFactory.get();
```

---

## 3) Functional Interfaces

A **functional interface** has exactly one abstract method (SAM — Single Abstract Method). It may have any number of `default` or `static` methods.

### 3.1 The `@FunctionalInterface` annotation

```java
@FunctionalInterface
interface Transformer<T, R> {
    R transform(T input);
    // Only one abstract method — lambda-compatible
    
    default Transformer<T, R> andLog() {
        return input -> {
            R result = this.transform(input);
            System.out.println(input + " → " + result);
            return result;
        };
    }
}

Transformer<String, Integer> strLen = s -> s.length();
System.out.println(strLen.transform("hello")); // 5
```

### 3.2 Built-in functional interfaces (`java.util.function`)

```java
// ── Predicate<T> — boolean test(T t) ──────────────────────────────
Predicate<String> isLong     = s -> s.length() > 5;
Predicate<String> startsWithA = s -> s.startsWith("A");

isLong.test("hello world");        // true
isLong.and(startsWithA).test("Amsterdam");  // true — conjunction
isLong.or(startsWithA).test("Ant");         // true — disjunction
isLong.negate().test("hi");                 // true — negation
Predicate.not(String::isBlank);            // Java 11+ static factory

// ── Function<T, R> — R apply(T t) ────────────────────────────────
Function<String, Integer>  len     = String::length;
Function<Integer, Boolean> isEven  = n -> n % 2 == 0;

Function<String, Boolean> isEvenLength = len.andThen(isEven);   // compose right
Function<String, Boolean> composed     = isEven.compose(len);   // compose left
System.out.println(isEvenLength.apply("hello")); // false (5 is odd)
Function.identity();     // t -> t — passes through unchanged

// ── Consumer<T> — void accept(T t) ───────────────────────────────
Consumer<String> print  = System.out::println;
Consumer<String> log    = s -> logger.info(s);
Consumer<String> both   = print.andThen(log);  // chain consumers
both.accept("event");

// ── Supplier<T> — T get() ────────────────────────────────────────
Supplier<List<String>> newList  = ArrayList::new;
Supplier<LocalDate>    today    = LocalDate::now;
List<String> list = newList.get();

// ── BiFunction<T, U, R> — R apply(T t, U u) ─────────────────────
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
repeat.apply("ab", 3); // "ababab"

// ── UnaryOperator<T> extends Function<T,T> ───────────────────────
UnaryOperator<String> trim    = String::trim;
UnaryOperator<String> upper   = String::toUpperCase;
UnaryOperator<String> trimUpper = trim.andThen(upper);
trimUpper.apply("  hello  "); // "HELLO"

// ── BinaryOperator<T> extends BiFunction<T,T,T> ─────────────────
BinaryOperator<Integer> add = Integer::sum;
add.apply(3, 4); // 7

// ── Primitive specializations (avoid boxing overhead) ────────────
IntPredicate  isPositive = n -> n > 0;
IntFunction<String> intToStr = Integer::toString;
IntUnaryOperator    doubler  = n -> n * 2;
IntBinaryOperator   sum      = Integer::sum;
IntSupplier         random   = () -> new Random().nextInt(100);
IntConsumer         print2   = System.out::println;
// Also: LongXxx, DoubleXxx variants
```

### 3.3 Common built-in interfaces quick reference

| Interface           | Method          | In → Out                | Common use                     |
|---------------------|-----------------|-------------------------|--------------------------------|
| `Predicate<T>`      | `test(T)`       | T → boolean             | Filtering                      |
| `Function<T,R>`     | `apply(T)`      | T → R                   | Mapping / transformation       |
| `Consumer<T>`       | `accept(T)`     | T → void                | Side effects                   |
| `Supplier<T>`       | `get()`         | () → T                  | Lazy creation                  |
| `UnaryOperator<T>`  | `apply(T)`      | T → T                   | In-place transformation        |
| `BinaryOperator<T>` | `apply(T,T)`    | (T,T) → T               | Combining two values           |
| `BiFunction<T,U,R>` | `apply(T,U)`    | (T,U) → R               | Two-arg mapping                |
| `BiPredicate<T,U>`  | `test(T,U)`     | (T,U) → boolean         | Two-arg filtering              |
| `BiConsumer<T,U>`   | `accept(T,U)`   | (T,U) → void            | Two-arg side effects           |
| `Comparator<T>`     | `compare(T,T)`  | (T,T) → int             | Ordering                       |
| `Runnable`          | `run()`         | () → void               | Thread tasks                   |
| `Callable<V>`       | `call()`        | () → V (throws)         | Thread tasks with return value |

---

## 4) The Stream API

A **Stream** is a pipeline of operations on a sequence of elements. Streams are:
- **Not a data structure** — they don't store data; they process data from a source.
- **Lazy** — intermediate operations are not executed until a terminal operation is reached.
- **Non-reusable** — once a terminal operation is called, the stream is consumed.

```mermaid
flowchart LR
    SRC["Source<br/>collection, array, generator"]
    I1["filter()"]
    I2["map()"]
    I3["sorted()"]
    T["Terminal<br/>collect, count, reduce, forEach"]
    RESULT["Result"]

    SRC --> I1 --> I2 --> I3 --> T --> RESULT

    classDef build fill:#2E75B6,stroke:#2E75B6,color:#FFFFFF,stroke-width:2px
    classDef runtime fill:#007880,stroke:#007880,color:#FFFFFF,stroke-width:2px
    classDef support fill:#17365D,stroke:#17365D,color:#FFFFFF,stroke-width:2px
    classDef accent fill:#E67E22,stroke:#E67E22,color:#111827,stroke-width:2px
    class SRC build
    class I1,I2,I3 runtime
    class T accent
    class RESULT support
```

### 4.1 Creating streams

```java
// From a Collection
List<String> words = List.of("hello", "world", "java");
Stream<String> s1 = words.stream();
Stream<String> s2 = words.parallelStream();  // parallel processing

// From an array
Stream<String> s3 = Arrays.stream(new String[]{"a", "b", "c"});
IntStream      s4 = Arrays.stream(new int[]{1, 2, 3});

// Static factories
Stream<String>  s5  = Stream.of("a", "b", "c");
Stream<String>  s6  = Stream.empty();
Stream<Integer> s7  = Stream.iterate(0, n -> n + 1);        // infinite: 0, 1, 2, ...
Stream<Integer> s8  = Stream.iterate(0, n -> n < 10, n -> n + 1); // bounded (Java 9+)
Stream<Double>  s9  = Stream.generate(Math::random);        // infinite random doubles

// Primitive streams (avoid boxing overhead)
IntStream    ints    = IntStream.range(0, 10);      // [0, 9]
IntStream    intsInc = IntStream.rangeClosed(1, 10); // [1, 10]
LongStream   longs   = LongStream.of(1L, 2L, 3L);
DoubleStream doubles = DoubleStream.of(1.5, 2.5);

// String → IntStream of chars
"hello".chars()   // IntStream of char values
       .filter(Character::isLetter)
       .count();
```

### 4.2 Intermediate operations (lazy — return a new Stream)

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "alex", "");

// filter(Predicate) — keep elements that match
names.stream()
     .filter(s -> !s.isEmpty())              // remove blanks
     .filter(s -> s.startsWith("A"))        // keep "A" names

// map(Function) — transform each element
names.stream()
     .map(String::toUpperCase)              // Stream<String>

// mapToInt / mapToLong / mapToDouble — transform to primitive stream
names.stream()
     .mapToInt(String::length)             // IntStream

// flatMap(Function<T, Stream<R>>) — flatten nested streams
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4), List.of(5));
nested.stream()
      .flatMap(Collection::stream)          // Stream<Integer>: 1,2,3,4,5
      .toList();

// distinct() — unique elements (uses equals/hashCode)
Stream.of(1,2,2,3,3,3).distinct()          // 1, 2, 3

// sorted() / sorted(Comparator)
names.stream().sorted()
names.stream().sorted(Comparator.comparingInt(String::length).reversed())

// limit(n) — take first n elements
Stream.iterate(1, n -> n+1).limit(5)       // 1, 2, 3, 4, 5

// skip(n) — discard first n elements
Stream.of(1,2,3,4,5).skip(2)              // 3, 4, 5

// peek(Consumer) — side-effect for debugging (does not consume the stream)
names.stream()
     .filter(s -> s.length() > 3)
     .peek(s -> System.out.println("After filter: " + s))
     .map(String::toUpperCase)
     .peek(s -> System.out.println("After map: " + s))
     .toList();

// takeWhile(Predicate) — Java 9+ — take while condition holds (short-circuits)
Stream.of(1,2,3,4,5,1,2)
      .takeWhile(n -> n < 4)              // 1, 2, 3

// dropWhile(Predicate) — Java 9+ — drop while condition holds
Stream.of(1,2,3,4,5,1,2)
      .dropWhile(n -> n < 4)             // 4, 5, 1, 2
```

### 4.3 Terminal operations (eager — consume the stream)

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna");

// ── Collect ───────────────────────────────────────────────────────
List<String>          list   = names.stream().filter(s->s.length()>3).collect(Collectors.toList());
List<String>          list2  = names.stream().toList();              // Java 16+ immutable
Set<String>           set    = names.stream().collect(Collectors.toSet());
String                joined = names.stream().collect(Collectors.joining(", ", "[", "]")); // "[Alice, Bob, Charlie, Anna]"

Map<Integer, List<String>> byLen = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=["Bob"], 4=["Anna"], 5=["Alice"], 7=["Charlie"]}

Map<Boolean, List<String>> partition = names.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 4));
// {false=["Bob","Anna"], true=["Alice","Charlie"]}

Map<Integer, Long> countByLen = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));

Map<String, String> upperByName = names.stream()
    .collect(Collectors.toMap(
        s -> s,                   // key extractor
        String::toUpperCase       // value extractor
    ));

// ── Count / Match ──────────────────────────────────────────────────
long count = names.stream().filter(s -> s.startsWith("A")).count(); // 2

boolean anyA   = names.stream().anyMatch(s -> s.startsWith("A"));   // true
boolean allA   = names.stream().allMatch(s -> s.startsWith("A"));   // false
boolean noneZ  = names.stream().noneMatch(s -> s.startsWith("Z"));  // true

// ── Find ────────────────────────────────────────────────────────────
Optional<String> first = names.stream().filter(s -> s.length() > 5).findFirst(); // "Charlie"
Optional<String> any   = names.stream().filter(s -> s.length() > 5).findAny();  // parallel-friendly

// ── Min / Max ──────────────────────────────────────────────────────
Optional<String> shortest = names.stream().min(Comparator.comparingInt(String::length));
Optional<String> longest  = names.stream().max(Comparator.comparingInt(String::length));

// ── Reduce ─────────────────────────────────────────────────────────
// identity + accumulator
int sum = IntStream.rangeClosed(1, 5).reduce(0, Integer::sum);  // 15

// without identity — returns Optional (stream may be empty)
Optional<String> longest2 = names.stream()
    .reduce((a, b) -> a.length() >= b.length() ? a : b);

// identity + accumulator + combiner (for parallel streams)
int parallelSum = names.stream()
    .parallel()
    .reduce(0,
            (acc, s) -> acc + s.length(),  // accumulator
            Integer::sum);                  // combiner (merges partial results)

// ── forEach / forEachOrdered ────────────────────────────────────────
names.stream().forEach(System.out::println);
names.parallelStream().forEachOrdered(System.out::println); // respects encounter order

// ── Convert to array ────────────────────────────────────────────────
Object[]   arr1 = names.stream().toArray();
String[]   arr2 = names.stream().toArray(String[]::new);

// ── Primitive terminal operations ───────────────────────────────────
IntStream ints = IntStream.of(1, 2, 3, 4, 5);
ints.sum();       // 15
ints.average();   // OptionalDouble: 3.0
ints.min();       // OptionalInt: 1
ints.max();       // OptionalInt: 5
ints.summaryStatistics(); // count=5, sum=15, min=1, max=5, avg=3.0
```

### 4.4 Collectors cheat sheet

```java
// Basic
Collectors.toList()
Collectors.toSet()
Collectors.toUnmodifiableList()        // Java 10+
Collectors.toUnmodifiableSet()
Collectors.toMap(keyFn, valueFn)
Collectors.toMap(keyFn, valueFn, mergeFunction) // handle duplicate keys
Collectors.counting()
Collectors.joining()
Collectors.joining(delimiter)
Collectors.joining(delimiter, prefix, suffix)

// Statistical
Collectors.summingInt(fn)
Collectors.averagingInt(fn)
Collectors.summarizingInt(fn)          // IntSummaryStatistics

// Grouping / partitioning
Collectors.groupingBy(classifier)
Collectors.groupingBy(classifier, downstream)
Collectors.partitioningBy(predicate)
Collectors.partitioningBy(predicate, downstream)

// Downstream collectors
Collectors.counting()
Collectors.toList()
Collectors.mapping(fn, downstream)
Collectors.filtering(pred, downstream) // Java 9+
Collectors.collectingAndThen(downstream, finisher) // apply finisher to collected result

// Example: group by first letter, count each group
Map<Character, Long> freqByFirstLetter = names.stream()
    .collect(Collectors.groupingBy(
        s -> s.charAt(0),
        Collectors.counting()
    ));
```

### 4.5 Parallel streams

```java
// Use parallelStream() or .parallel() on an existing stream
long count = LongStream.rangeClosed(1, 1_000_000)
    .parallel()
    .filter(n -> n % 2 == 0)
    .count();

// Rules for safe parallel streams:
// ✅ Stateless intermediate operations (filter, map, flatMap)
// ✅ Non-interfering (don't modify the source during streaming)
// ✅ Associative and commutative reduce operations
// ❌ Avoid stateful lambdas (shared mutable state)
// ❌ forEach order is non-deterministic — use forEachOrdered if order matters
// ❌ Parallel overhead may not be worth it for small data sets
```

### 🎯 Interview Questions — Streams

- **Q: What is the difference between `map()` and `flatMap()`?**
  A: `map()` applies a 1-to-1 transformation (each element → one result). `flatMap()` applies a 1-to-many transformation (each element → a Stream of results) and flattens all resulting streams into one. Use `flatMap` to process nested collections.

- **Q: Are streams lazy? What does that mean?**
  A: Intermediate operations (`filter`, `map`, `sorted`, etc.) are **lazy** — they don't execute until a terminal operation is invoked. This enables optimizations like short-circuiting (`findFirst` stops once the first match is found) and fusing operations to avoid intermediate data structures.

- **Q: Can you reuse a stream?**
  A: No. Once a terminal operation consumes a stream, it is closed and cannot be reused. Create a new stream from the source if you need to process it again.

- **Q: What is the difference between `findFirst()` and `findAny()`?**
  A: `findFirst()` always returns the first element in encounter order. `findAny()` may return any element — it is optimized for parallel streams where the first-found is returned. For sequential streams, both return the first element.

- **Q: When should you use parallel streams?**
  A: For computationally expensive, stateless operations on large data sets with a data structure that splits well (e.g., `ArrayList`, arrays). Avoid for small collections, IO-bound work, or operations with shared mutable state.

- **Q: What is the difference between `reduce()` and `collect()`?**
  A: `reduce()` produces a single immutable result by repeatedly combining elements (fold). `collect()` produces a mutable container (List, Map) by accumulating elements via a `Collector`. For building collections, always use `collect()`.

---

## 5) Optional\<T\> — Safe Null Handling

`Optional<T>` is a container that either holds a non-null value or is empty. It forces callers to explicitly handle the "no value" case, eliminating `NullPointerException` risks.

### 5.1 Creating Optional

```java
Optional<String> present = Optional.of("hello");         // throws NPE if null!
Optional<String> nullable = Optional.ofNullable(null);   // empty if null — safe
Optional<String> empty    = Optional.empty();
```

### 5.2 Checking and accessing the value

```java
Optional<String> opt = Optional.ofNullable(getName());

// Presence check
opt.isPresent();         // true if has value
opt.isEmpty();           // true if empty (Java 11+)

// Unsafe get — throws NoSuchElementException if empty (avoid)
opt.get();

// Safe retrieval
opt.orElse("default");                          // return default value if empty
opt.orElseGet(() -> computeDefault());          // lazy — only called if empty
opt.orElseThrow();                              // throw NoSuchElementException if empty (Java 10+)
opt.orElseThrow(() -> new UserNotFoundException("not found"));

// Conditional action
opt.ifPresent(s -> System.out.println(s));     // only run if present
opt.ifPresentOrElse(                           // Java 9+
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);
```

### 5.3 Transforming Optional

```java
Optional<String> upper = opt.map(String::toUpperCase);     // Optional<String>
Optional<Integer> len  = opt.map(String::length);           // Optional<Integer>

// filter — returns empty Optional if predicate fails
Optional<String> longName = opt.filter(s -> s.length() > 5);

// flatMap — when the mapping function already returns Optional
Optional<String> result = opt.flatMap(s -> findRelated(s));

// or() — Java 9+ — provide alternative Optional
Optional<String> fallback = opt.or(() -> Optional.of("fallback"));

// stream() — Java 9+ — convert to Stream with 0 or 1 elements
opt.stream().forEach(System.out::println);
```

### 5.4 Optional in a stream pipeline

```java
// Java 9+ — flatMap with Optional::stream to filter out empty Optionals
List<String> names = List.of("Alice", null, "Bob", null, "Charlie");
List<String> present = names.stream()
    .map(Optional::ofNullable)
    .flatMap(Optional::stream)  // only non-empty Optionals
    .toList();
// ["Alice", "Bob", "Charlie"]
```

### 5.5 Optional best practices

```java
// ✅ DO — use Optional as a method return type when "no result" is possible
public Optional<User> findById(int id) {
    return users.stream().filter(u -> u.id() == id).findFirst();
}

// ✅ DO — chain operations fluently
String result = findById(42)
    .map(User::name)
    .filter(name -> !name.isBlank())
    .orElse("Anonymous");

// ❌ DON'T — use Optional as a field type (serialization issues, memory overhead)
// ❌ DON'T — use Optional as a method parameter (just use null check + overloading)
// ❌ DON'T — call get() without isPresent() check (defeats the purpose)
// ❌ DON'T — use Optional<Integer> when OptionalInt is available (avoids boxing)
```

### 🎯 Interview Questions — Optional

- **Q: What is the purpose of `Optional`?**
  A: To explicitly model the possibility of a missing value in a return type, forcing callers to handle both the present and absent cases. This eliminates implicit `null` returns that cause unexpected `NullPointerException`.

- **Q: What is the difference between `orElse()` and `orElseGet()`?**
  A: `orElse(value)` always evaluates `value`, even when the Optional is present — can be wasteful for expensive computations. `orElseGet(supplier)` only calls the supplier when the Optional is empty — prefer it for any non-trivial default.

- **Q: Should you use Optional as a field type?**
  A: No. `Optional` is not `Serializable`, has memory overhead as a field, and was designed as a return type, not a general-purpose container for fields. Use `@Nullable` annotations or null-checks for fields.

---

## 6) Function Composition

```java
Function<Integer, Integer> times2   = n -> n * 2;
Function<Integer, Integer> plus3    = n -> n + 3;

// andThen — apply this, then the other
Function<Integer, Integer> times2ThenPlus3 = times2.andThen(plus3);
times2ThenPlus3.apply(5);   // (5×2)+3 = 13

// compose — apply the other first, then this
Function<Integer, Integer> plus3ThenTimes2 = times2.compose(plus3);
plus3ThenTimes2.apply(5);   // (5+3)×2 = 16

// Predicate composition
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven     = n -> n % 2 == 0;

Predicate<Integer> isPositiveAndEven = isPositive.and(isEven);
Predicate<Integer> isPositiveOrEven  = isPositive.or(isEven);
Predicate<Integer> isNotPositive     = isPositive.negate();

// Consumer chaining
Consumer<String> log   = s -> System.out.println("[LOG] " + s);
Consumer<String> audit = s -> System.out.println("[AUDIT] " + s);
Consumer<String> both  = log.andThen(audit);
both.accept("user-login");
```

---

## 7) Practical Patterns

### 7.1 Strategy pattern with lambdas

```java
@FunctionalInterface
interface PricingStrategy { double calculate(double basePrice); }

class OrderService {
    private final PricingStrategy strategy;
    OrderService(PricingStrategy strategy) { this.strategy = strategy; }
    double price(double base) { return strategy.calculate(base); }
}

// Zero boilerplate — no need for separate classes
OrderService retail  = new OrderService(price -> price);
OrderService vip     = new OrderService(price -> price * 0.8);
OrderService employee = new OrderService(price -> price * 0.5);
```

### 7.2 Builder with lambdas

```java
class Email {
    private String to, subject, body;
    private Email() {}

    static Email create(Consumer<Email> builder) {
        Email e = new Email();
        builder.accept(e);
        return e;
    }
}

Email e = Email.create(mail -> {
    mail.to = "user@example.com";
    mail.subject = "Hello";
    mail.body = "World";
});
```

### 7.3 Common stream patterns

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "Barry");

// 1. Filter + map + collect
List<String> longUpperNames = names.stream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .toList();
// ["ALICE", "ANNA", "BARRY", "CHARLIE"]

// 2. Group + count
Map<Integer, Long> countByLen = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));

// 3. Frequency map (word count)
String text = "the quick brown fox jumps over the lazy dog the";
Map<String, Long> wordFreq = Arrays.stream(text.split("\\s+"))
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));

// 4. Flat list of all chars
List<Character> allChars = names.stream()
    .flatMapToInt(String::chars)
    .mapToObj(c -> (char) c)
    .toList();

// 5. Sum / average with primitives
int totalLength = names.stream().mapToInt(String::length).sum();
OptionalDouble avgLen = names.stream().mapToDouble(String::length).average();

// 6. Partition into two groups
Map<Boolean, List<String>> partitioned = names.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 4));
// {true=["Alice","Charlie","Barry"], false=["Bob","Anna"]}

// 7. Top N by criteria
List<String> top3Longest = names.stream()
    .sorted(Comparator.comparingInt(String::length).reversed())
    .limit(3)
    .toList();
```

---

## 8) Quick Checklist — Functional Java

- Lambdas only capture **effectively-final** local variables — never reassign a captured variable.
- Prefer method references over explicit lambdas when they improve readability.
- Use primitive stream specializations (`IntStream`, `LongStream`, `DoubleStream`) to avoid boxing overhead.
- Intermediate stream operations are lazy — no work is done until a terminal operation is called.
- Never reuse a stream after calling a terminal operation.
- Use `collect(Collectors.toList())` / `.toList()` (Java 16+) to materialize results; prefer `toList()` for immutable output.
- Use `Collectors.groupingBy()` for multi-group aggregation; `partitioningBy()` for binary splits.
- Prefer `orElseGet(supplier)` over `orElse(value)` for expensive defaults.
- Never call `Optional.get()` without checking `isPresent()` first — use `orElseThrow()` instead.
- Do not use `Optional` as a field, constructor parameter, or in collections — it is designed as a return type.
- Use `peek()` only for debugging — never for side effects that affect correctness.
- For parallel streams: ensure stateless, non-interfering operations and use associative combiners in `reduce()`.
