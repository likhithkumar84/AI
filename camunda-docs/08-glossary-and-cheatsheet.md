# 08 · Glossary & Cheat Sheet

> Day-before-the-interview revision. Everything here is compressed on purpose.

---

## 1. One-page architecture recall

### Camunda 7

```
Your JVM ──► Process Engine (library) ──► RDBMS (ACT_* tables)
                 │
                 ├─ RepositoryService  (definitions)
                 ├─ RuntimeService     (instances, variables, messages)
                 ├─ TaskService        (user tasks)
                 ├─ HistoryService     (audit)
                 ├─ ManagementService  (jobs, incidents)
                 └─ Job Executor       (async work + timers, thread pool polling ACT_RU_JOB)

Web apps: Cockpit (ops) · Tasklist (humans) · Admin (users) · Optimize 💰 (analytics)
```

### Camunda 8

```
Client (job workers / REST)
        │ gRPC + REST v2
        ▼
   Zeebe Gateway  (stateless, routes to partitions)
        ▼
   Zeebe Brokers  (stateful)
        ├─ Partitions   → unit of parallelism; instance lives in exactly one
        ├─ Raft         → leader + followers, quorum = RF/2 + 1, RF must be odd
        ├─ Event log    → append-only, source of truth, replayable
        └─ RocksDB      → materialised current state
        ▼
   Exporters ──► Elasticsearch / OpenSearch (🔶 or RDBMS 8.8+)
        ▼
   Operate · Tasklist · Optimize 💰 · Identity · Connector Runtime
```

**Three facts that carry a whole answer:**

1. Log is the source of truth; RocksDB is a derived view; replay rebuilds state.
2. Log segments can't be compacted until **exported** → exporter lag fills broker disks.
3. Partition count is fixed at cluster creation.

---

## 2. BPMN symbol cheat sheet

| Shape           | Element                   | Key point                                      |
|-----------------|---------------------------|------------------------------------------------|
| ⭕ thin         | Start event               | Creates a token                                |
| ⊙ double thin   | Intermediate **catching** | Waits                                          |
| ⊙ double filled | Intermediate **throwing** | Emits                                          |
| ⭕ thick        | End event                 | Consumes **one** token                         |
| ⭘ dashed        | Non-interrupting          | Spawns an **extra** token                      |
| ▭ rounded       | Task                      | Work                                           |
| ▭ thick border  | Call activity             | Separate process instance                      |
| ▭ dotted border | Event subprocess          | Triggered by its start event, covers the scope |
| ◇ `X`           | Exclusive                 | One path — **always add a default**            |
| ◇ `+`           | Parallel                  | All paths; join waits for all                  |
| ◇ `O`           | Inclusive                 | Conditional subset; expensive join             |
| ◇ ⬠             | Event-based               | First event wins                               |
| → solid         | Sequence flow             | Never crosses pools                            |
| ⇢ dashed        | Message flow              | Only between pools                             |

**Event icons:** ✉ message (1:1) · ⏰ timer · ⚡ error (business) · ▲ signal (1:N) · ▲ escalation · ⏪
compensation · ⬤ terminate (kills all tokens) · ↪ link.

**Task markers:** ≡ parallel MI · ⋯ sequential MI · ↻ loop · ⊞ subprocess · ~ ad-hoc · ⏪
compensation.

---

## 3. Decision tables at a glance

| Hit policy         | Result | Use                                         |
|--------------------|--------|---------------------------------------------|
| **U** Unique       | single | default, safest                             |
| **A** Any          | single | overlaps allowed if outputs identical       |
| **P** Priority     | single | highest-priority output wins                |
| **F** First        | single | row order **is** the rule                   |
| **R** Rule order   | list   | ordered by rows                             |
| **C** Collect      | list   | `C+` sum · `C<` min · `C>` max · `C#` count |
| **O** Output order | list   | ordered by output priority                  |

Input cells are **unary tests** (`< 1000`, `[1..10)`, `"A","B"`, `not("X")`, `-`). Output cells are
**values** (`"VP"`, `amount * 0.1`).

---

## 4. FEEL micro-reference

```feel
= amount > 1000 and tier = "GOLD"
= if approved then "OK" else "NOK"           // else is mandatory
= items[1]        items[-1]      items[2..4] // 1-BASED indexing
= items[price > 100]                          // filter
= for i in items return i.price * i.qty       // projection
= sum(...)  count(...)  min(...)  max(...)
= some i in items satisfies i.flag
= every i in items satisfies i.valid
= is defined(x)      get or else(x, 0)        // null safety
= string(x)   number("1.5")   upper case(s)
= date and time("2026-08-16T10:00:00@UTC")
= duration("PT30M")
= { total: 10, currency: "EUR" }
```

Traps: `=` is equality · double quotes only · 1-based lists · `else` required · missing vars →
`null`
· built-ins have spaces in their names.

---

## 5. Conversion table (C7 → C8)

| C7                                         | C8                                         |
|--------------------------------------------|--------------------------------------------|
| `camunda:class` / `delegateExpression`     | `zeebe:taskDefinition type` + `@JobWorker` |
| `camunda:type="external"` + topic          | `zeebe:taskDefinition type` (near 1:1)     |
| `camunda:expression="${...}"`              | job worker / connector                     |
| `${x > 1}` (JUEL)                          | `= x > 1` (FEEL)                           |
| `camunda:inputOutput`                      | `zeebe:ioMapping`                          |
| Script task (Groovy/JS)                    | job worker / connector                     |
| `asyncBefore` / `asyncAfter`               | not needed — jobs are inherently async     |
| `RuntimeService.startProcessInstanceByKey` | `newCreateInstanceCommand()`               |
| `runtimeService.correlateMessage`          | `newPublishMessageCommand()`               |
| Cockpit                                    | Operate                                    |
| `camunda-bpm-assert`                       | Camunda Process Test (`CamundaAssert`)     |
| CMMN                                       | ad-hoc subprocess / event subprocesses     |

---

## 6. Glossary

_Grouped by concept area and ordered the way you'd actually learn them — study path `01 → 02 → 06 →
03 → 04`, not alphabetical._

### Core process & notation concepts

| Term                   | Meaning                                                     |
|------------------------|--------------------------------------------------------------|
| **BPMN**               | Business Process Model and Notation (OMG), currently 2.0   |
| **DMN**                | Decision Model and Notation (OMG) — decision tables & DRDs |
| **CMMN**               | Case Management Model and Notation (C7 only)                |
| **Process definition** | Deployed, versioned BPMN model                              |
| **Process instance**   | One running execution of a definition                       |

### BPMN modelling elements

| Term                  | Meaning                                                                                 |
|------------------------|------------------------------------------------------------------------------------------|
| **Token**             | Conceptual marker representing a thread of execution                                   |
| **Activity**          | Any unit of work — task or subprocess                                                  |
| **Boundary event**    | Event attached to an activity's border; listens while it runs                          |
| **Signal**            | Broadcast event (1:N)                                                                  |
| **Escalation**        | Modelled hand-up to a higher scope; can be non-interrupting                            |
| **Correlation key**   | Value that routes a message to the right waiting instance                              |
| **Multi-instance**    | Repeat an activity per collection item, parallel or sequential                         |
| **Gateway**           | Routing element; performs no work                                                      |
| **Call activity**     | Invokes a separately deployed process as a child instance                              |
| **Event subprocess**  | Subprocess triggered by an event within the parent scope                               |
| **Ad-hoc subprocess** | Container whose inner activities have no fixed order; runtime decides what to activate |
| **Compensation**      | Explicit undo of a completed activity (Saga)                                           |
| **Saga**              | Long-running transaction implemented via compensating actions                          |

### DMN & FEEL

| Term           | Meaning                                                                       |
|----------------|----------------------------------------------------------------------------------|
| **DRD**        | Decision Requirements Diagram — how decisions depend on inputs and each other |
| **Hit policy** | How a decision table resolves multiple matching rules                        |
| **FEEL**       | Friendly Enough Expression Language — the C8 expression language             |

### Runtime concepts (apply to both engines)

| Term            | Meaning                                                                              |
|-----------------|-----------------------------------------------------------------------------------------|
| **Wait state**  | Point where the engine persists and stops (user task, timer, message, async marker) |
| **Job**         | A unit of work offered to a worker (C8) / an async continuation or timer (C7)       |
| **Incident**    | Technical failure needing human intervention (retries exhausted, correlation error) |
| **Idempotency** | Same operation applied twice has the same effect — mandatory for job workers        |

### Camunda 7 specifics

| Term              | Meaning                                                                        |
|-------------------|------------------------------------------------------------------------------------|
| **JavaDelegate**  | C7 interface implementing service task logic in the engine JVM               |
| **JUEL**          | C7 expression language (`${...}`)                                             |
| **External task** | C7 pattern: fetch-and-lock work by an outside client (ancestor of job workers) |
| **Job executor**  | C7 thread pool that polls and executes jobs                                   |
| **Business key**  | Domain identifier attached to an instance (C7); in C8 typically a variable    |

### Camunda 8 / Zeebe architecture

| Term                      | Meaning                                                                    |
|---------------------------|-------------------------------------------------------------------------------|
| **Zeebe**                 | The Camunda 8 engine                                                       |
| **Broker**                | Stateful Zeebe node holding partitions                                     |
| **Partition**             | Independent Zeebe log + state machine; unit of scaling                    |
| **Raft**                  | Consensus protocol replicating Zeebe partitions                           |
| **Quorum**                | Majority needed to commit in Raft = RF/2 + 1                              |
| **Replication factor**    | Copies of each partition; must be odd                                     |
| **Stream processor**      | Zeebe component applying log records to state                             |
| **RocksDB**               | Embedded KV store holding Zeebe's materialised state                      |
| **Exporter**              | Streams Zeebe records to secondary storage (Elasticsearch, Kafka, custom)  |
| **Backpressure**          | Zeebe rejecting commands (`RESOURCE_EXHAUSTED`) when overloaded           |
| **Orchestration Cluster** | 🔶 8.8+ unified deployment of Zeebe + Operate + Tasklist + Identity       |

### Camunda 8 apps, connectors & tooling

| Term                 | Meaning                                                                    |
|----------------------|---------------------------------------------------------------------------|
| **Operate**          | C8 monitoring & operations web app                                        |
| **Tasklist**         | Human task inbox web app                                                  |
| **Optimize**         | Analytics/BI product (Enterprise)                                         |
| **Connector**        | Reusable integration — inbound (into the process) or outbound (out of it) |
| **Element template** | JSON descriptor giving a Modeler property panel for a connector/task      |
| **CPT**              | Camunda Process Test — the C8 testing framework                           |
| **zbctl / c8ctl**    | Camunda 8 CLI tools                                                       |

---

## 7. 60-second self-audit before the interview

Can you, from memory:

- [ ] Define BPM and list the lifecycle phases?
- [ ] Explain token semantics and predict a deadlock from an asymmetric gateway?
- [ ] Distinguish message / signal / error / escalation / incident?
- [ ] Explain interrupting vs non-interrupting with a concrete use case?
- [ ] Name the C7 engine services and the `ACT_*` prefixes?
- [ ] Explain `asyncBefore` and why parallel branches aren't parallel without it?
- [ ] Draw the C8 architecture end-to-end and name every box?
- [ ] Explain partitions, Raft, quorum, and why exporters gate log compaction?
- [ ] Explain at-least-once delivery and the idempotency requirement?
- [ ] Recite the hit policies and what `C+` does?
- [ ] Write a FEEL filter and remember lists are 1-based?
- [ ] Give a credible C7 → C8 migration plan using the strangler pattern?
- [ ] Say honestly what Camunda 8 gives up versus Camunda 7?

If any box is unticked, jump back to the relevant document — the index is in
[`README.md`](README.md).

---

## 8. Official references

| Topic                  | Link                                                                    |
|------------------------|-------------------------------------------------------------------------|
| Camunda 8 docs         | https://docs.camunda.io                                                 |
| Camunda 7 docs         | https://docs.camunda.org                                                |
| BPMN 2.0 spec (OMG)    | https://www.omg.org/spec/BPMN/2.0/                                      |
| DMN spec (OMG)         | https://www.omg.org/spec/DMN/                                           |
| CMMN spec (OMG)        | https://www.omg.org/spec/CMMN/                                          |
| FEEL Playground        | https://play.camunda.io                                                 |
| Camunda BPMN reference | https://camunda.com/bpmn/reference/                                     |
| Camunda 8 REST API v2  | https://docs.camunda.io/docs/apis-tools/orchestration-cluster-api-rest/ |
| Migration (7 → 8)      | https://docs.camunda.io/docs/guides/migrating-from-camunda-7/           |

🔶 Always confirm version numbers, EOL dates, and API signatures against these before relying on them
in a design decision or an interview claim.

