# Core Java - Interview Cheat Sheet

> **Baseline:** Java 17. Java 21 additions are labeled. Start with the
> [numbered Core Java index](00-core-java-index.md) for the full reading order.

| Topic | Answer to recall | Study guide |
|---|---|---|
| Java portability | `javac` produces bytecode; a **compatible** JVM version runs it on each platform. Native libraries can still be platform-specific. | [Core Java index](00-core-java-index.md) |
| JDK / runtime / JVM | Conceptual layers: tools + runtime; runtime = libraries + JVM. A standalone JRE is not required by modern JDK distributions. | [Architecture map](README.md#java-architecture-at-a-glance) |
| Variables | Java passes values; a method receives a copy of an object reference, never the caller's variable itself. | [Language and OOP](01-language-foundations-oop.md) |
| Identity and equality | `==` checks reference identity for objects; override `equals()` **and** `hashCode()` together when defining logical equality. | [Language and OOP](01-language-foundations-oop.md) |
| Strings | `String` is immutable; use `.equals()` for content and `StringBuilder` for repeated mutable concatenation. | [Language and OOP](01-language-foundations-oop.md) |
| Generics | Type erasure removes most type parameters at runtime; `List<String>` is not a subtype of `List<Object>`. | [Collections and generics](03-collections-generics.md) |
| Collections | `List` allows duplicates, `Set` enforces uniqueness, `Map` maps keys to values; `Map` is **not** a `Collection`. | [Collections and generics](03-collections-generics.md) |
| Exceptions | Checked exceptions must be caught or declared; `RuntimeException` and `Error` are unchecked. Prefer try-with-resources for closeable resources. | [Exceptions](04-exceptions.md) |
| Streams | Intermediate operations are lazy; one terminal operation consumes the stream. | [Functional Java](05-functional-java.md) |
| Threads | `start()` schedules work; direct `run()` executes on the caller. `volatile` gives visibility/ordering, not atomic `count++`. | [Concurrency](06-concurrency.md) |
| Virtual threads (Java 21) | Good for many I/O-bound tasks; do not pool them or treat them as faster CPU threads. Bound external resources separately. | [Concurrency](06-concurrency.md) |
| Files and encodings | Distinguish bytes from characters; choose a charset (for example UTF-8) and close resources. | [IO and files](07-io-files.md) |
| Modern types | Records (Java 16) and sealed types (Java 17) reduce boilerplate; records are shallowly immutable unless their components are immutable. | [Modern Java](08-modern-java.md) |
| GC and storage | GC reclaims **unreachable** objects. In HotSpot, class metadata is in Metaspace; static field values and interned strings are on the heap. | [JVM memory and GC](02-jvm-memory-gc.md) |

## Topics not covered in depth by the original guides

- **Exact decimal arithmetic:** Use `new BigDecimal("0.10")` rather than
  `new BigDecimal(0.1)` for decimal input. `equals()` considers scale; `compareTo()` compares value.
- **Date and time:** `Instant` is a timeline point; `LocalDate` is a calendar date without
  a zone; `ZonedDateTime` adds a zone. `Duration` and `Period` measure different kinds of amount.
- **Annotations and reflection:** `@Retention(RetentionPolicy.RUNTIME)` is needed to
  inspect an annotation at runtime. Module boundaries can limit reflective access.
- **Reachability and cleanup:** A weak reference does not keep an object alive, but
  files and sockets still need explicit closing; GC is not a resource-management strategy.
- **Regular expressions:** `Pattern` compiles a regex; `Matcher.matches()` tests the
  complete input while `Matcher.find()` searches for a substring.
- **Further study:** Standard networking (`URI`, `HttpClient`, sockets) and localization
  (`Locale`, `ResourceBundle`) are not covered in depth by the original topic guides.

For memory failures, study the [OOM diagnosis table](02-jvm-memory-gc.md#6-outofmemoryerror-diagnose-by-symptom)
and [prevention checklist](02-jvm-memory-gc.md#9-production-design-checklist) before an interview.
