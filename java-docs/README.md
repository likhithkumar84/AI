# Java Mastery

Core Java interview preparation, organized as numbered topic guides. Start with the
[Core Java index](00-core-java-index.md), then use the
[cheat sheet](09-core-java-cheat-sheet.md) for revision. The baseline is Java 17;
features introduced in Java 21 are identified where they appear.

| Order | Guide |
|---|---|
| 00 | [Core Java index and reading order](00-core-java-index.md) |
| 01 | [Language foundations and OOP](01-language-foundations-oop.md) |
| 02 | [JVM memory, GC and OOM](02-jvm-memory-gc.md) |
| 03 | [Collections and generics](03-collections-generics.md) |
| 04 | [Exceptions](04-exceptions.md) |
| 05 | [Functional Java](05-functional-java.md) |
| 06 | [Concurrency](06-concurrency.md) |
| 07 | [IO and files](07-io-files.md) |
| 08 | [Modern Java](08-modern-java.md) |
| 09 | [Interview cheat sheet](09-core-java-cheat-sheet.md) |

## From Java source to a running application

**Compile, package, and launch are different operations.** For a simple `Main.java` with no
package, these are illustrative commands, not steps performed by this document:

```powershell
javac Main.java
jar --create --file app.jar --main-class Main Main.class
java -jar app.jar
```

`javac` turns source into `.class` bytecode. `jar` **archives existing** classes and writes a
manifest with `Main-Class: Main`; it does not compile Java. `java -jar app.jar` starts the
application's JVM and uses that manifest to select the entry class. `java app.jar` without
`-jar` is not the command to launch an executable JAR.

```mermaid
flowchart TB
    subgraph BUILD["Build and package -<br/>no application JVM yet"]
        direction TB
        SRC["Main.java<br/>source"] -->|"javac Main.java"| CLASS["Main.class<br/>portable bytecode"]
        CLASS -->|"jar --create (packages bytecode)"| JAR["app.jar<br/>classes +<br/>META-INF/MANIFEST.MF<br/>Main-Class: Main"]
    end
    subgraph RUN["Launch and execute"]
        direction TB
        START["java launcher + OS<br/>new application JVM process"]
        MEMORY["JVM runtime data areas<br/>heap, class metadata,<br/>stacks, PC"]
        ENTRY["Read Main-Class<br/>from JAR manifest"]
        LOADING["Class loaders find and define classes<br/>application loader: Main;<br/>built-in loaders: JDK classes"]
        LINK["JVM links classes<br/>verify, prepare;<br/>resolve references as needed"]
        INIT["JVM initializes Main<br/>on active use<br/>run static initializers"]
        MAIN["Invoke<br/>Main.main(String[] args)"]
        EXEC["Interpreter executes bytecode<br/>JIT may compile hot paths"]
        CPU["CPU executes<br/>native instructions"]
        HEAP["Heap objects"]
        GC["Garbage collector"]

        START --> MEMORY
        START --> ENTRY
        ENTRY --> LOADING --> LINK --> INIT --> MAIN --> EXEC --> CPU
        MEMORY -.-> EXEC
        EXEC -.->|"allocates"| HEAP
        GC -.->|"reclaims unreachable"| HEAP
    end
    JAR -->|"java -jar app.jar"| START

    classDef build fill:#2E75B6,stroke:#2E75B6,color:#FFFFFF,stroke-width:2px
    classDef runtime fill:#007880,stroke:#007880,color:#FFFFFF,stroke-width:2px
    classDef memory fill:#6B5B95,stroke:#6B5B95,color:#FFFFFF,stroke-width:2px
    classDef support fill:#17365D,stroke:#17365D,color:#FFFFFF,stroke-width:2px
    classDef accent fill:#E67E22,stroke:#E67E22,color:#111827,stroke-width:2px
    class SRC,CLASS build
    class JAR,ENTRY support
    class START,LOADING,LINK,INIT,MAIN,EXEC,GC runtime
    class MEMORY,HEAP memory
    class CPU accent
```

| Step | What actually happens |
|---|---|
| Compile | `javac` writes `Main.class` to disk. Its compiler process is separate from the application JVM, which has not started yet. |
| Package | `jar` puts compiled classes, optional resources, and a `Main-Class` manifest entry into `app.jar`; bytecode is not recompiled. |
| Launch | The OS starts the `java` process. The launcher reads the JAR's entry point; the JVM establishes its heap, class metadata storage, and thread state. |
| Load and link | Built-in loaders normally delegate parent-first; the application loader finds `Main` and other application classes as needed. The JVM verifies bytecode, prepares static defaults, and may resolve references lazily. |
| Initialize and enter `main` | On first active use, the JVM runs class initialization, then invokes `public static void main(String[] args)`. Other classes may load later as execution reaches them. |
| Execute and exit | The interpreter executes bytecode; HotSpot may JIT-compile hot paths to native code. Threads use stack frames and the heap; GC reclaims unreachable heap objects while the app runs. On process exit, the OS reclaims its resources. |

For packaged classes, use their fully qualified name with `--main-class`. A plain JAR does
not automatically include third-party dependencies; a build tool can assemble more complex
applications. Without a `Main-Class` manifest entry, use `java -cp app.jar Main` instead.

## Java architecture at a glance

The JDK supplies development tools and a runtime image; the runtime includes the JVM and
standard libraries. A compatible JVM loads portable bytecode and executes it on its platform.
This map shows **component containment and interactions**, not the time-ordered build and
launch steps shown above. Class loaders find and define classes; the JVM performs linking
and initialization.

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 70, "rankSpacing": 80, "padding": 30}}}%%
flowchart TB
    TITLE["THE COMPLETE JAVA<br/>ARCHITECTURE MAP"]
    STORAGE["STORAGE / SSD<br/>Stores raw text files (.java)<br/>and compiled bytecode (.class)"]
    TITLE --> STORAGE

    subgraph JDK["JDK (Java Development Kit)<br/>-> Used to BUILD and compile code"]
        direction TB
        TOOLS["Development Tools:<br/>javac (Compiler),<br/>jdb (Debugger), jar (Packager)"]
        subgraph JRE["JRE (Java Runtime Environment)<br/>-> Package needed to RUN code"]
            direction TB
            LIBS["Core Libraries & Classes:<br/>java.lang, java.util,<br/>java.io, Math, etc."]
            subgraph JVM["JVM (Java Virtual Machine)<br/>-> The engine loaded directly into RAM"]
                direction LR
                subgraph AREAS["ALLOCATED SPACE INSIDE<br/>YOUR PHYSICAL RAM<br/>JVM RUNTIME DATA AREAS"]
                    direction LR
                    subgraph SHARED["SHARED MEMORY<br/>(All parts of your app can see this):"]
                        direction TB
                        METHOD["METHOD AREA<br/>Stores Class Blueprints, Code<br/>structures, and Static Vars."]
                        HEAP["HEAP<br/>Stores all dynamic Objects and Arrays<br/>created via the 'new' keyword.<br/>(Main target of Garbage Collection)."]
                        METHOD ~~~ HEAP
                    end
                    subgraph PERTHREAD["PER-THREAD MEMORY<br/>(Private workspaces created for<br/>every concurrent path of execution):"]
                        direction TB
                        STACK["JVM STACK<br/>Stores Local Variables,<br/>Method Frames, and tracking<br/>for active mathematical state."]
                        PC["PC REGISTER (Prog)<br/>Stores memory address<br/>of the JVM bytecode<br/>instruction executing<br/>code tracking."]
                        NATIVE["NATIVE METHOD STACK<br/>Holds instruction state<br/>for native C/C++ source<br/>code tracking."]
                        STACK ~~~ PC
                        PC ~~~ NATIVE
                    end
                    SHARED ~~~ PERTHREAD
                end
                subgraph ENGINE["JVM EXECUTION ENGINE"]
                    direction TB
                    INTERPRETER["INTERPRETER:<br/>Reads bytecode line-by-line<br/>and executes it immediately on the fly."]
                    JIT["JIT COMPILER:<br/>Compiles heavily repeated bytecode<br/>straight into 1s and 0s for peak speed."]
                    GC["GARBAGE COLLECTOR:<br/>Automatically deletes unused objects<br/>from Heap to keep RAM clear."]
                    INTERPRETER ~~~ JIT
                    JIT ~~~ GC
                end
                AREAS --> ENGINE
            end
        end
    end

    CODE["Native Machine<br/>Language Code"]
    subgraph HARDWARE["PHYSICAL HARDWARE<br/>LAYER"]
        CPU["PHYSICAL CPU:<br/>Processes raw binary instructions<br/>using its internal physical registers."]
    end
    STORAGE --> JDK
    TOOLS ~~~ LIBS
    LIBS -.-> JVM
    JDK --> CODE
    CODE --> HARDWARE

    classDef build fill:#2E75B6,stroke:#2E75B6,color:#FFFFFF,stroke-width:2px
    classDef runtime fill:#007880,stroke:#007880,color:#FFFFFF,stroke-width:2px
    classDef memory fill:#6B5B95,stroke:#6B5B95,color:#FFFFFF,stroke-width:2px
    classDef support fill:#17365D,stroke:#17365D,color:#FFFFFF,stroke-width:2px
    classDef accent fill:#E67E22,stroke:#E67E22,color:#111827,stroke-width:2px
    class TITLE,LIBS,CODE support
    class STORAGE,TOOLS build
    class METHOD,HEAP,STACK,PC,NATIVE memory
    class INTERPRETER,JIT,GC runtime
    class CPU accent
```

In HotSpot, class metadata uses native Metaspace. Static field values are associated with
heap-resident `Class` mirrors; interned strings also live on the heap. The
[memory and GC guide](02-jvm-memory-gc.md) expands on memory areas, collectors, leaks and
out-of-memory diagnosis. The build/run lifecycle is [documented above](#from-java-source-to-a-running-application).

### Which component accesses which memory area?

These arrows show **runtime access**, not the order in which Java is compiled. The heap and
method area are shared; the JVM stack, PC register, and native method stack belong to
individual threads.

```mermaid
flowchart LR
    LOADING["Class loading"] -->|"class metadata"| META["Method area / Metaspace<br/>shared class metadata"]
    LOADING -.->|"Class mirror"| HEAP["Heap<br/>shared objects, Class mirrors<br/>static field values, interned strings"]
    EXECUTION["Java execution<br/>interpreter or JIT-compiled code"] -->|"type information"| META
    EXECUTION -->|"objects"| HEAP
    EXECUTION -->|"frames and locals"| STACK["JVM stack<br/>per-thread frames and references"]
    EXECUTION -->|"bytecode position"| PC["PC register<br/>per-thread instruction position"]
    NATIVE_CALL["Native method calls"] -->|"native frames"| NATIVE_STACK["Native method stack<br/>per-thread native call state"]
    GC["Garbage collector"] -.->|"scan roots"| STACK
    GC -->|"reclaim unreachable objects"| HEAP

    classDef runtime fill:#007880,stroke:#007880,color:#FFFFFF,stroke-width:2px
    classDef memory fill:#6B5B95,stroke:#6B5B95,color:#FFFFFF,stroke-width:2px
    classDef support fill:#17365D,stroke:#17365D,color:#FFFFFF,stroke-width:2px
    classDef accent fill:#E67E22,stroke:#E67E22,color:#111827,stroke-width:2px
    class LOADING,EXECUTION runtime
    class NATIVE_CALL support
    class META,HEAP,STACK,PC,NATIVE_STACK memory
    class GC accent
```

See the [JVM memory guide](02-jvm-memory-gc.md#3-runtime-memory-shared-vs-per-thread) for
shared versus per-thread behavior and HotSpot implementation details.
