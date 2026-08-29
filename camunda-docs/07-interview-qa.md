# 07 · Interview Q&A Bank

> ~90 questions with model answers. Try to answer **before** expanding. Keep spoken answers to
> 30–60 seconds; the written answers here are longer so you can pick what to say.

Legend: 🟢 fundamentals · 🟡 intermediate · 🔴 senior/architect · 🔶 version-sensitive

---

## A. BPM & concepts

<details><summary>🟢 1. What is BPM?</summary>

The discipline of designing, executing, monitoring and continuously improving business processes. It
combines a **management practice** (ownership, governance, improvement), a **notation/method**
(BPMN, DMN), and **technology** (a process engine, modeler, monitoring). Its goal is to make how
work flows through an organisation visible, consistent, measurable, and changeable.
</details>

<details><summary>🟢 2. What are the phases of the BPM lifecycle?</summary>

Identify → Discover (as-is) → Analyse → Redesign (to-be) → Implement → Monitor & control → back to
Identify. It's a loop, not a project.
</details>

<details><summary>🟢 3. BPMN vs DMN vs CMMN?</summary>

**BPMN** models *how work flows* (tasks, gateways, events). **DMN** models *how decisions are made*
(decision tables, DRDs) so rules live outside the flow. **CMMN** models *unpredictable knowledge
work* where the caseworker decides what to do next. All three are OMG standards. 🔶 Camunda 8
supports BPMN and DMN but **not** CMMN — use ad-hoc subprocesses instead.
</details>

<details><summary>🟡 4. Orchestration vs choreography?</summary>

Orchestration = a central component explicitly drives participants; the flow is visible in one
model, failure handling and timeouts are centralised. Choreography = services react to each other's
events with no central brain; more autonomy but the end-to-end flow is implicit and only
reconstructable from logs. Camunda advocates *decentralised orchestration*: many small team-owned
process models, each orchestrating its own scope — you get visibility without a monolith.
</details>

<details><summary>🟡 5. When would you NOT use a process engine?</summary>

Single-step logic; purely synchronous, sub-100ms request/response paths; workflows that never span
systems, teams or time; ultra-high-frequency data processing (that's a stream processor's job). If
there is no long-running state, no human step, no cross-system coordination and no need for
visibility, an engine is pure overhead.
</details>

<details><summary>🔴 6. How do you convince a team that "this is just if-statements" that they need an engine?</summary>

Point at what they'd have to build anyway: durable state that survives restarts, timers spanning
days, retries with backoff, compensation, versioning of in-flight work, an audit trail, and a view
non-developers can read. Then show the cost of *not* having it: "where is order 4711?" requiring a
developer and three log searches. The engine isn't replacing their logic; it's replacing the
undifferentiated plumbing around it.
</details>

---

## B. BPMN

<details><summary>🟢 7. Explain token semantics.</summary>

A token is a conceptual marker representing a thread of execution. A start event creates one; it
moves along sequence flows; a parallel split multiplies it; a parallel join merges it; the instance
completes when all tokens are consumed. Most "what happens if…" questions are answered by tracing
tokens.
</details>

<details><summary>🟢 8. Difference between a message event and a signal event?</summary>

Message = **1:1**, targeted, routed by a **correlation key** to one specific waiting instance.
Signal = **1:N** broadcast; every subscriber reacts, with no correlation. Message = letter, signal =
radio announcement.
</details>

<details><summary>🟡 9. Error event vs escalation event vs incident?</summary>

**Error** = a modelled *business* failure, always interrupting, handled by an error boundary event
or error event subprocess. **Escalation** = a modelled notification to a higher scope, and it can be
**non-interrupting** so the main flow continues. **Incident** = an unmodelled *technical* failure
raised by the engine when retries are exhausted; a human resolves it in Operate/Cockpit. Rule: if a
developer must look at it, it's an incident; if the business anticipated it, it's an error event.
</details>

<details><summary>🟡 10. Interrupting vs non-interrupting boundary events?</summary>

Interrupting (solid border) cancels the attached activity and the token leaves via the boundary
path. Non-interrupting (dashed border) leaves the activity running and **spawns an additional
token** on the boundary path — so you now have two tokens. Non-interrupting is for reminders,
alerts, and SLA warnings that must not stop the work.
</details>

<details><summary>🟡 11. Exclusive vs inclusive vs parallel vs event-based gateway?</summary>

**Exclusive (X)** takes exactly one path — first matching condition (always define a default).
**Inclusive (O)** takes every path whose condition is true, and its join waits for all tokens that
could still arrive. **Parallel (+)** always takes all paths and its join waits for all of them.
**Event-based (⬠)** waits, and the first event to occur decides the path — must be followed by
catching events or receive tasks.
</details>

<details><summary>🟡 12. What happens if no condition matches on an exclusive gateway?</summary>

Without a default flow the engine raises an error/incident and the instance stops there. Always
model a default flow (or an explicit "else" path) — it's the most common modelling defect.
</details>

<details><summary>🔴 13. What happens if you join a parallel split with an exclusive gateway?</summary>

The exclusive gateway doesn't synchronise, so each arriving token passes straight through and the
downstream path executes **once per branch** — duplicate work, duplicate side effects. The mirror
mistake (exclusive split joined by a parallel gateway) deadlocks, because the join waits for tokens
that will never arrive. Keep splits and joins symmetric.
</details>

<details><summary>🟢 14. Embedded subprocess vs call activity?</summary>

Embedded stays inside the same process instance and gives you a scope (for boundary events, variable
scoping, error handling) but isn't reusable. A call activity starts a **separate process instance**
of a separately deployed definition — reusable, independently versioned and visible as its own
instance in Operate/Cockpit, with explicit variable in/out mapping.
</details>

<details><summary>🟡 15. What is an event subprocess and when do you use it?</summary>

A subprocess with a dotted border that has no incoming sequence flow; it's triggered by its own
start event within the parent scope. Interrupting → cancels the whole parent scope, then handles it
(e.g. "order cancelled"). Non-interrupting → runs alongside (e.g. a timer that nudges an approver
every two days). Unlike a boundary event it covers the entire scope, not one activity.
</details>

<details><summary>🟡 16. What is an ad-hoc subprocess and why does it matter in Camunda 8?</summary>

A container of activities with no sequence flows between them; at runtime something decides which
inner elements to activate, and a completion condition decides when it's done. It's Camunda 8's
answer to CMMN-style case work, and it's the foundation of **AI agents** — the LLM picks which inner
elements (tools) to run, while the model constrains what's even possible.
</details>

<details><summary>🔴 17. How do you handle distributed transactions across microservices?</summary>

You don't use 2PC — you implement a **Saga**. Model each step with a compensation boundary event and
a compensating activity; on failure, throw a compensation event (or use a transaction subprocess)
and the engine runs the handlers in reverse completion order. The process instance is the durable
saga log, and Operate gives you the visibility that hand-rolled sagas lack.
</details>

<details><summary>🟡 18. Multi-instance: parallel vs sequential?</summary>

Parallel (three vertical bars) creates all instances at once — use for independent work like
notifying five suppliers. Sequential (three horizontal bars) runs them one at a time — use when
order matters or when a downstream system can't take concurrency. Both take an input collection,
input element, optional output collection/element, and an optional completion condition for early
exit.
</details>

<details><summary>🟢 19. Pool vs lane?</summary>

A pool is an independent participant — in Camunda, one pool = one executable process; sequence flows
never cross pool boundaries, only message flows do. A lane is a subdivision inside a pool (role,
department, system) and has **no execution semantics** — it's organisational documentation, though
it's often used as a convention for user-task assignment.
</details>

<details><summary>🟡 20. Terminate end event vs normal end event?</summary>

A normal end event consumes **one** token; other branches keep running. A terminate end event kills
**all** tokens in its scope immediately — the whole instance (or the whole subprocess if it's inside
one). Use it for hard aborts like "order cancelled, stop everything".
</details>

<details><summary>🔴 21. Review this model: a service task calling a payment API with no boundary events and no async marker. What's wrong?</summary>

No timeout handling (a hung API hangs the instance), no distinction between technical failure and
business decline, and in Camunda 7 no async marker means it runs in the caller's transaction — so a
failure rolls back the API call that started the instance and there are no retries. Fix: add
`asyncBefore` (C7), a timer boundary event for the timeout, an error boundary event for
"declined", and make the call idempotent.
</details>

---

## C. Camunda 7

<details><summary>🟢 22. Name the main engine services.</summary>

`RepositoryService` (deployments/definitions), `RuntimeService` (running instances, variables,
messages), `TaskService` (user tasks), `HistoryService` (audit), `IdentityService` (users/groups),
`ManagementService` (jobs, incidents, ops), `AuthorizationService`, `FormService`, `FilterService`,
`ExternalTaskService`, `DecisionService`, `CaseService` (CMMN).
</details>

<details><summary>🟢 23. What do the ACT_ table prefixes mean?</summary>

`ACT_GE_` general (deployments, byte arrays), `ACT_RE_` repository (versioned definitions),
`ACT_RU_` runtime (live state — deleted when the instance ends), `ACT_HI_` history (append-only
audit), `ACT_ID_` identity (users/groups). `ACT_RU_EXECUTION` is effectively the token table.
</details>

<details><summary>🟡 24. Four ways to implement a service task in Camunda 7?</summary>

`camunda:class` (engine instantiates a `JavaDelegate`), `camunda:delegateExpression` (resolves a
Spring bean — the idiomatic choice), `camunda:expression` (direct method call), and
`camunda:type="external"` with a topic (fetch-and-lock by an external client in any language). The
external-task option is the one that maps cleanly onto Camunda 8 job workers.
</details>

<details><summary>🔴 25. What do asyncBefore / asyncAfter actually do?</summary>

They create a **job** and end the current transaction at that point, so the job executor picks the
work up in a new transaction. Benefits: bounded transaction scope, automatic retries and incidents
(only async work gets them), crash-safe save points, load distribution across cluster nodes, and
**genuine parallelism** across parallel branches. Without async markers a parallel gateway executes
its branches sequentially in one thread.
</details>

<details><summary>🔴 26. A parallel gateway splits into three service tasks — do they run concurrently?</summary>

No, not by default: BPMN parallelism is about tokens, not threads, so the engine walks the branches
sequentially in one transaction. Add `asyncBefore` on each branch so each becomes a job and the job
executor can run them concurrently (and on different nodes in a cluster).
</details>

<details><summary>🟡 27. How does the job executor work?</summary>

A thread pool acquires due jobs from `ACT_RU_JOB` (async continuations, timers), locks them with an
owner and lock expiry, executes them, then deletes or retries. On failure `retries` decrements
(default 3) with a backoff; at zero an **incident** is created and the job stops until someone
increases retries. Tuning: pool sizes, `maxJobsPerAcquisition`, `lockTimeInMillis`, backoff, and
**exclusive jobs** (serialise jobs of the same instance to avoid optimistic-locking storms).
</details>

<details><summary>🔴 28. What causes OptimisticLockingException and is it bad?</summary>

Every row has a `REV_` revision column; two transactions updating the same execution/variable row
collide and the loser rolls back. It's **expected and handled** — the job is simply retried. A high
*rate* signals contention: too many concurrent branches on one instance, non-exclusive jobs, or hot
variables written from parallel paths.
</details>

<details><summary>🟡 29. What are history levels and why do they matter?</summary>

`none`, `activity`, `audit` (default), `full`, or custom — they control how much goes into the
`ACT_HI_*` tables. History is the main driver of database growth, so production setups must set
`historyTimeToLive` on definitions and enable the **history cleanup** job.
</details>

<details><summary>🟡 30. How do you version processes and migrate running instances in C7?</summary>

Each deployment creates a new version; running instances stay on the version they started with. To
move them, use **process instance migration** (`RuntimeService.newMigration()`), mapping activity
IDs old → new; Cockpit provides a UI (Enterprise). For call activities, control which version is
invoked with the **binding** setting (`latest`, `deployment`, `version`, `versionTag`).
</details>

<details><summary>🟡 31. BpmnError vs RuntimeException in a delegate?</summary>

`BpmnError` is caught by an error boundary event / error event subprocess and continues along a
modelled path — no incident. A plain `RuntimeException` in async work decrements retries and
eventually creates an incident; in synchronous work it propagates to the caller and rolls the
transaction back.
</details>

<details><summary>🟡 32. Execution listener vs task listener?</summary>

Execution listeners attach to flow elements and sequence flows and fire on `start`, `end`, `take`.
Task listeners attach to user tasks and fire on `create`, `assignment`, `complete`, `delete`,
`update`, `timeout`. Use them for cross-cutting concerns (logging, metrics, derived variables), not
business logic — logic hidden in listeners is invisible in the diagram, which defeats BPM.
</details>

<details><summary>🟢 33. Cockpit, Tasklist, Admin, Optimize — who uses what?</summary>

Cockpit = ops/devs monitor and fix instances (incidents, retries, modification, migration).
Tasklist = business users work on human tasks. Admin = user/group/authorization administration.
Optimize = analysts get heatmaps, durations, bottlenecks and KPIs (Enterprise only).
</details>

---

## D. Camunda 8 / Zeebe

<details><summary>🟢 34. Biggest architectural difference from Camunda 7?</summary>

Camunda 7 embeds the engine in your JVM with a relational database; Camunda 8 is a separate,
distributed cluster you talk to as a client. State lives in a partitioned, Raft-replicated,
append-only event log with RocksDB, and query data is exported to Elasticsearch. You gain horizontal
scale and fault tolerance; you lose embeddability and shared transactions.
</details>

<details><summary>🟡 35. What is a partition?</summary>

An independent log plus state machine — Zeebe's unit of parallelism and sharding. Each process
instance lives entirely in one partition. More partitions = more throughput. 🔶 The partition count
is fixed at cluster creation and can't be changed later without a migration, so size it up front.
</details>

<details><summary>🟡 36. How does replication work?</summary>

Each partition is replicated across brokers with **Raft**. One replica is the **leader** and
processes commands; the others are followers. A write commits once a **quorum**
(`replicationFactor/2 + 1`) acknowledges it. Replication factor must be odd; with 3 you survive one
broker loss per partition, with 5 you survive two.
</details>

<details><summary>🔴 37. Why is Zeebe faster than a database-backed engine?</summary>

Sequential appends to a log instead of random-access transactional row updates; state sharded across
partitions with no cross-partition transactions or global locks; no SQL round trips in the hot path;
replication via Raft rather than DB clustering; and reads served by a separate model
(Elasticsearch), so reporting never competes with execution.
</details>

<details><summary>🟡 38. What are exporters and why are they critical?</summary>

Exporters stream every record off the broker into secondary storage (Elasticsearch/OpenSearch, or a
custom target like Kafka). Operate, Tasklist and Optimize read only from that store. Critically,
**the broker cannot compact/delete log segments that haven't been exported yet** — so if
Elasticsearch stalls, broker disks fill and the cluster eventually fails. Monitoring exporter lag is
a top operational priority.
</details>

<details><summary>🟡 39. What is backpressure in Zeebe?</summary>

When a broker can't keep up it rejects new commands with `RESOURCE_EXHAUSTED` rather than degrading
for everyone. Clients retry with backoff. It's a health signal meaning "add partitions/brokers or
throttle producers", not a bug.
</details>

<details><summary>🟢 40. What is a job worker?</summary>

A client that subscribes to a **job type** declared on a service task, activates jobs (getting the
job key, variables and headers), does the work, and then completes the job, throws a BPMN error, or
fails it with a retry count. In Spring it's a `@JobWorker`-annotated method; the returned map
becomes output variables.
</details>

<details><summary>🔴 41. Why must job workers be idempotent?</summary>

Job delivery is **at-least-once**. If the worker crashes, the network drops the completion, or the
job timeout expires mid-processing, the job is re-activated and the work runs again. Guard side
effects with an idempotency key (job key, a business key, or a dedicated variable) and size timeouts
above realistic p99 durations.
</details>

<details><summary>🟡 42. Job timeout, maxJobsActive, fetchVariables — what do they control?</summary>

`timeout` is how long the job stays locked to this worker (too short → duplicate execution).
`maxJobsActive` caps in-flight jobs per worker (too high → memory pressure and timeouts).
`fetchVariables` restricts which variables are sent (bandwidth, and avoids leaking data the worker
shouldn't see).
</details>

<details><summary>🟡 43. Polling vs job streaming?</summary>

Classic clients poll for jobs on an interval, which trades latency against load. 🔶 Modern Camunda 8
clients use **job streaming**: the worker opens a long-lived stream and the broker pushes jobs as
they're created, cutting latency dramatically, with polling retained as a fallback for jobs created
while no stream existed.
</details>

<details><summary>🟡 44. What are connectors and when do you build a custom one?</summary>

Connectors are reusable integrations: **outbound** (process → system, e.g. REST, Kafka producer,
Slack, AI) and **inbound** (system → process, e.g. webhook start events, Kafka consumer). Under the
hood an outbound connector is a job worker with a well-known type plus an **element template** for
the Modeler UI. Use OOTB first, the generic REST connector next, a **custom connector** when the
logic must be reused across teams with a friendly template, and a plain **job worker** when it needs
deep access to your own domain code.
</details>

<details><summary>🟢 45. How do messages correlate in Camunda 8?</summary>

By **message name + correlation key** (a FEEL expression like `= orderId` declared in a
`zeebe:subscription`). Published messages can be **buffered for a TTL** so they still correlate if
the instance isn't waiting yet, and a `messageId` gives deduplication. Only one instance may wait on
a given name+key pair at a time.
</details>

<details><summary>🔴 46. How do you handle very large payloads?</summary>

Don't put them in variables — there are hard record/message size limits and variables are copied and
persisted on every state change. Store the document in object storage or a document service and pass
an ID/URL. 🔶 Camunda 8.8+ adds a document-handling API for exactly this pattern.
</details>

<details><summary>🟡 47. How do you monitor a Camunda 8 cluster?</summary>

Prometheus metrics from brokers and gateway. Watch: backpressure/rejection rate, **exporter lag**,
partition health and Raft leader changes, disk usage and free space, job activation latency,
incident counts, and Elasticsearch cluster health. Plus Operate for business-level incident
visibility.
</details>

<details><summary>🔴 48. Sizing: how many partitions and what replication factor?</summary>

Replication factor 3 for production (odd, tolerates one broker loss per partition). Partitions
should be at least the number of brokers and sized for **peak** throughput and the parallelism you
need — and remember they're fixed at creation, so over-provision slightly rather than under. Fast
local NVMe disks matter more than CPU because the log is the hot path; leave plenty of RAM for
RocksDB and the OS page cache.
</details>

<details><summary>🟡 49. How is data retained/cleaned up in Camunda 8?</summary>

There's no `ACT_HI_*` cleanup job. Broker logs are compacted once exported; the long-term data lives
in Elasticsearch, where you configure ILM policies and Operate's archiver to move and eventually
delete old instance data. Backups must snapshot brokers and Elasticsearch **together**.
</details>

<details><summary>🟡 50. How do you test Camunda 8 processes?</summary>

**Camunda Process Test (CPT)** — `camunda-process-test-spring` with `@CamundaSpringProcessTest` and
`CamundaAssert`, running against a real engine (Testcontainers or embedded runtime). Mock the
workers and connectors and assert on **routing and coverage** (which elements activated, which
variables were set), not on external-system data correctness.
</details>

<details><summary>🔴 51. What are AI agents in Camunda 8.8+?</summary>

An AI Agent (sub-process/task) where an LLM decides which activities to invoke, built on the
**ad-hoc subprocess**: the inner elements are the tool catalogue, the agent activates the ones it
needs, receives results and loops until a completion condition is met. Providers are pluggable,
including any OpenAI-compatible endpoint. The key architectural point is that **the process model is
the guardrail** — the model bounds what the LLM can do, and every step is in the audit log.
</details>

---

## E. Cross-version & architecture

<details><summary>🟢 52. Should a new project use Camunda 7 or 8?</summary>

Camunda 8, unless there's a hard blocker such as a strict requirement for a fully embedded engine
with no external cluster, or an unavoidable need for same-transaction consistency or CMMN. 🔶 Camunda
7 is in maintenance/EOL wind-down, so new investment should target 8.
</details>

<details><summary>🔴 53. How would you migrate a large C7 estate to C8?</summary>

Inventory and classify processes by migration cost (external-task-based = cheap,
JavaDelegate-heavy = medium, CMMN/scripting = redesign). Convert models with the Migration
Analyzer/Modeler, rewrite delegates as job workers with idempotency, translate JUEL → FEEL, replace
SQL reporting with REST v2 / Optimize. Then use the **strangler pattern**: new instances start on C8
while C7 drains its in-flight work; there is no in-place live-instance migration. Prove the
operational model on one low-risk, high-volume process first.
</details>

<details><summary>🔴 54. You lose shared transactions in C8. How do you keep data consistent?</summary>

Design for eventual consistency: make every worker idempotent, use business keys, prefer
"do the side effect, then complete the job" with deduplication on the downstream system, and model
compensation for steps that can't be retried safely. Where a strict local transaction is needed,
keep it inside the service and treat the process as the coordinator, not the transaction manager.
</details>

<details><summary>🟡 55. Where should business logic live?</summary>

In services (or DMN), not in the model and not in listeners. The model should express **flow,
timing, and exception paths**; DMN expresses **rules**; services express **domain behaviour**. If a
gateway condition is a paragraph of FEEL, it belongs in a decision table; if a delegate contains 300
lines, it belongs in a domain service the worker calls.
</details>

<details><summary>🔴 56. How do you version process models safely in production?</summary>

Running instances stay on their original version. Make changes **additive** where possible; use
`versionTag` and explicit call-activity binding so subprocess versions don't drift unexpectedly; for
breaking changes either let old instances drain or plan an explicit migration (C7 instance
migration / C8 Operate migration) with activity mappings. Always keep worker code backward
compatible with both versions during the overlap window.
</details>

<details><summary>🟡 57. How do you secure a Camunda 8 deployment?</summary>

OAuth2 client credentials for API/worker clients, Identity + Keycloak/OIDC for user SSO,
role/resource-based authorizations for who can see and operate which processes, tenant isolation for
multi-tenant setups, TLS everywhere, and network isolation of brokers (only the gateway is exposed).
Also restrict which variables workers fetch.
</details>

<details><summary>🔴 58. Design question: order fulfilment with payment, inventory, shipping, and a 48h cancellation window. Sketch it.</summary>

One process: start (order placed) → parallel split → [reserve inventory] and [authorise payment] →
join → user/automated fraud check if amount is high (DMN decides) → capture payment → ship. Attach a
**cancellation event subprocess** (interrupting message start "cancel-order") that runs compensation
handlers (release inventory, refund) and ends with a terminate end event. The 48h window is a
**timer boundary event** or an event-based gateway racing "cancel-order" against "48h elapsed". Each
service task is a job worker with idempotent side effects; payment capture uses an idempotency key.
</details>

---

## F. DMN & FEEL

<details><summary>🟢 59. What is DMN and why separate decisions from the process?</summary>

DMN models business decisions — usually as decision tables in a DRD — outside the process flow.
Rules change more often than flows, business analysts can safely edit a table, decisions become
independently testable, versioned and auditable, and the BPMN model avoids gateway explosion.
</details>

<details><summary>🟡 60. Name the hit policies.</summary>

Single result: **U**nique (default, overlaps are errors), **A**ny (overlaps allowed if outputs are
identical), **P**riority (highest-priority output wins), **F**irst (first matching row wins).
Multi-result: **R**ule order, **C**ollect (with `C+` sum, `C<` min, `C>` max, `C#` count), **O**
utput order.
</details>

<details><summary>🟡 61. What's the difference between an input entry and an output entry?</summary>

An input entry is a **unary test** — `< 1000`, `[1..10]`, `"A","B"`, `not("X")`, `-` for any. An
output entry is a **value expression** — `"VP"`, `amount * 0.1`. Confusing the two is the most
common DMN mistake.
</details>

<details><summary>🟡 62. FEEL gotchas you've hit?</summary>

Lists are **1-indexed** (`items[1]` is the first, `items[-1]` the last); `=` is equality not
assignment; strings need **double quotes**; `if` always needs an `else`; missing variables evaluate
to `null` and silently break conditions (guard with `is defined()` / `get or else()`); built-in
names contain spaces (`string length`, `distinct values`).
</details>

<details><summary>🟢 63. Where is FEEL used in Camunda 8?</summary>

Everywhere: gateway conditions, input/output mappings, timer definitions, correlation keys,
multi-instance collections, connector fields, completion conditions, and DMN. There is no JUEL and
no in-engine scripting. 🔶 In Camunda 7, FEEL is used in DMN while the process engine uses JUEL.
</details>

---

## G. Scenario / troubleshooting drills

<details><summary>🔴 64. Instances are piling up at one service task and nothing happens. Diagnose.</summary>

Check: (1) is a worker actually subscribed to that **job type** — typos between
`zeebe:taskDefinition type` and `@JobWorker(type=...)` are the #1 cause; (2) are jobs being created
at all (Operate shows the token waiting); (3) are workers erroring out and creating incidents; (4)
job timeout too short causing thrash; (5) `maxJobsActive` saturation or a blocked worker thread
pool; (6) cluster backpressure; (7) wrong tenant. In C7 the equivalents are job executor exhaustion,
suspended job definitions, and jobs with `retries = 0`.
</details>

<details><summary>🔴 65. A customer was charged twice. Root cause and fix?</summary>

At-least-once delivery plus a job timeout shorter than the actual processing time: the broker
re-activated the job while the first attempt was still running. Fix: idempotency key propagated to
the payment provider, timeout sized above p99, keep worker logic short (offload long work and
complete asynchronously), and never assume "completed the job" means "the side effect happened
exactly once".
</details>

<details><summary>🔴 66. Broker disks are filling up. Why?</summary>

Almost always **exporter lag** — Elasticsearch is down, slow, in read-only mode (disk watermark), or
misconfigured, so the broker can't compact exported log segments. Fix Elasticsearch/retention first;
also check snapshot configuration and that no exporter is disabled but still referenced.
</details>

<details><summary>🟡 67. An instance is stuck at a parallel join. Why?</summary>

One incoming branch never produced a token — typically because an upstream exclusive gateway skipped
it, or a branch ended at an end event before reaching the join. Fix the model (symmetric split/join,
or use an inclusive gateway), and for the stuck instances use modification (C7) / Operate modify
(C8) to inject or cancel tokens.
</details>

<details><summary>🟡 68. Message correlation fails intermittently. Causes?</summary>

The instance isn't waiting yet (fix with message buffering/TTL, or restructure so the subscription
exists first), the correlation key value is of the wrong type or is null, key values aren't unique
so two instances match, or the message name doesn't match exactly. In C7 you'd see
`MismatchingMessageCorrelationException`.
</details>

<details><summary>🔴 69. Throughput is fine but latency is terrible. Where do you look?</summary>

Job activation latency (polling interval vs streaming), worker concurrency and thread pool sizing,
`maxJobsActive` too low, backpressure, downstream system latency inside workers, and — in C7 — job
executor acquisition intervals and lock durations. Also check whether workers are doing long
blocking I/O on the same threads that poll.
</details>

<details><summary>🟡 70. History/Elasticsearch storage is exploding. What do you do?</summary>

C7: lower the history level where possible, set `historyTimeToLive` on every definition and enable
history cleanup with a batch window. C8: configure Elasticsearch ILM and Operate's archiver, reduce
retained record types where supported, and stop storing large payloads as variables.
</details>

<details><summary>🔴 71. How would you load-test a Camunda 8 deployment?</summary>

Model realistic instance mix and variable sizes, drive load through the gateway (not synthetic
single-partition traffic), scale workers separately so they're not the bottleneck, and measure
throughput, p99 job activation latency, backpressure rate and exporter lag together. Test failure
modes too: kill a broker and confirm quorum survives; stall Elasticsearch and observe disk growth.
</details>

---

## H. Rapid-fire (one-liners)

| Q                                     | A                                                              |
|---------------------------------------|----------------------------------------------------------------|
| Who maintains BPMN/DMN/CMMN?          | OMG                                                            |
| Current BPMN version?                 | 2.0                                                            |
| Does Camunda 8 support CMMN?          | No — use ad-hoc subprocesses                                   |
| Camunda 8 engine name?                | Zeebe                                                          |
| Camunda 7 origin?                     | Fork of Activiti, 2013                                         |
| Camunda founded?                      | 2008, Berlin (Freund & Rücker)                                 |
| Camunda 8 GA?                         | 2022                                                           |
| C8 client protocols?                  | gRPC + REST API v2                                             |
| C8 secondary storage?                 | Elasticsearch/OpenSearch (🔶 RDBMS option in 8.8+)             |
| C8 consensus protocol?                | Raft                                                           |
| C8 embedded state store?              | RocksDB                                                        |
| C7 token table?                       | `ACT_RU_EXECUTION`                                             |
| C7 default retries?                   | 3                                                              |
| C7 default history level?             | `audit`                                                        |
| Replication factor rule?              | Odd number; quorum = RF/2 + 1                                  |
| Can you change partition count later? | No (fixed at creation)                                         |
| C8 expression language?               | FEEL only                                                      |
| C7 expression language?               | JUEL (+FEEL in DMN)                                            |
| Default/safest hit policy?            | Unique (U)                                                     |
| Sum aggregator hit policy?            | `C+`                                                           |
| FEEL list indexing?                   | 1-based                                                        |
| Message vs signal?                    | 1:1 correlated vs 1:N broadcast                                |
| Error vs incident?                    | Modelled business failure vs technical failure needing a human |
| C7 monitoring app?                    | Cockpit                                                        |
| C8 monitoring app?                    | Operate                                                        |
| C8 test framework?                    | Camunda Process Test (CPT)                                     |
| C7 test framework?                    | camunda-bpm-assert                                             |
| Job delivery guarantee?               | At-least-once                                                  |

---

## I. Questions to ask *them*

Asking good questions is scored. Pick two or three:

1. Camunda 7 or 8 — and if 7, what's the migration plan and timeline?
2. Self-managed or SaaS? If self-managed, who owns the Elasticsearch and backup story?
3. How many process definitions, and what's the peak instance rate?
4. Are workers Java-only, or polyglot? Connectors or hand-written workers?
5. Who owns the models — engineers, or business analysts in Web Modeler?
6. How do you test processes today, and is process testing part of CI?
7. How are incidents triaged — does an ops team watch Operate, or is it alert-driven?
8. Are you using DMN, and are business users actually editing the tables?

➡️ Next: [`08-glossary-and-cheatsheet.md`](08-glossary-and-cheatsheet.md)

