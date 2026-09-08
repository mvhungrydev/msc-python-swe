# PY-601 — Written Examination

**Time allowed: 3 hours. Closed book.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

---

## Section A — answer FOUR

### Correctness models (L01)

**A1.** State the concurrency/parallelism distinction and give the diagnostic question that
identifies which one a given problem needs. Then name the three things that make concurrency hard
and say which is deepest, with your reasoning. *(15)*

**A2.** Distinguish safety from liveness and explain why testing finds one far more easily than the
other. Define linearizability and give the two properties that make it the preferred correctness
condition. *(15)*

**A3.** Distinguish a race condition from a data race, and explain why Python programmers are
systematically misled about the distinction. State the four Coffman conditions and say which you
would normally break in practice and why. Then state Amdahl's law, Gustafson's law and the USL, and
what each explains that the others do not. *(15)*

### Threads and the GIL (L02)

**A4.** State precisely what the GIL protects and why reference counting made it necessary. Give the
two circumstances in which a thread releases it, and explain what the switch interval is and what
latency effect it causes. *(15)*

**A5.** Name three C-accelerated operations that do *not* release the GIL, and explain what that
means for a workload built around them. Then explain biased reference counting and immortal objects
and the problem each solves. *(15)*

**A6.** Explain what breaks when the GIL is removed, and why "the GIL made this bug rare" is the
worst possible property for a bug to have. Name four approaches to parallelism in Python other than
free-threading, and explain why an unretrieved `Future` swallows its exception and what you do about
it. *(15)*

### Shared state and locks (L03)

**A7.** Give the four bytecodes of `counter += 1` and identify exactly where a thread switch can
occur. Then name the three problems a memory model addresses and state what CPython actually
provides. *(15)*

**A8.** Give five operations that are atomic on CPython and five that are not, and then explain why
you should not rely on the first list. Name the three things a lock provides and the one thing it
does not. *(15)*

**A9.** Give four disciplines for avoiding deadlock in preference order. Give the three
condition-variable rules and the bug each prevents. Then state the design hierarchy for shared state
and say where most bugs actually originate, and explain what a cache stampede is with two
preventions. *(15)*

### Thread-safe design (L04)

**A10.** Give the five levels of thread-safety specification with an example of each, and explain
why publishing which level a class provides is part of its interface. *(15)*

**A11.** Explain why `frozen=True` on a dataclass is insufficient for immutability and what you do
instead. Describe copy-on-write publication and the situation in which it is the right pattern, and
name four kinds of confinement saying which is fragile. *(15)*

**A12.** Give five properties a bounded-queue pipeline provides for free. Explain why you need one
sentinel per worker. Then define idempotence and commutativity, say what each buys in a concurrent
system, and distinguish lock-free from wait-free explaining why you cannot write either in pure
Python. *(15)*

### The event loop (L05)

**A13.** Describe an event loop in one sentence, then in detail: give the three data structures a
scheduler needs and what each is for. Distinguish readiness-based from completion-based I/O
notification with an example of each. *(15)*

**A14.** State what `await x` requires of `x`, and what is at the bottom of every await chain.
Distinguish a coroutine from a task and say precisely where the concurrency comes from. *(15)*

**A15.** Give four categories of blocking call that will stall an event loop and the fix for each.
Explain why a fire-and-forget task is dangerous in two distinct ways, and why you must keep a
reference to a task you created. *(15)*

### Structured concurrency (L06)

**A16.** State the `goto` analogy for unstructured concurrency and explain what block structure
restores. Give four guarantees `TaskGroup` provides. *(15)*

**A17.** Explain what `gather` does on the first exception by default and why that default is
dangerous. Explain why `asyncio` primitives are not thread-safe, and why you need them less often
than you would need threading primitives. *(15)*

**A18.** State where the interleaving points are in async code and why that makes reasoning easier
than with threads. Give three facts about `asyncio.to_thread` that surprise people, explain why
async generator finalisation is awkward with its two mitigations, and say when `uvloop` is worth
adopting and what determines the size of the gain. *(15)*

### Cancellation and shutdown (L07)

**A19.** State what `task.cancel()` actually does and, more importantly, what it does not do.
Explain why `CancelledError` is a `BaseException` and give the two rules that follow. *(15)*

**A20.** State what `asyncio.timeout` provides that `wait_for` does not. Explain the cascading-timeout
problem and the deadline-propagation fix, and say what `shield` protects, what it does not, and why
it must itself have a timeout. *(15)*

**A21.** Give the two-phase cleanup pattern and explain why it is necessary. Give the seven steps of
graceful shutdown in order with the reason for the ordering of at least three of them, and explain
why idempotence is a better answer than more shutdown code. *(15)*

### Processes and subinterpreters (L08)

**A22.** Give the three process start methods with one distinguishing property each, and say which
should be the default and why. Explain precisely why `fork` in a threaded process can deadlock the
child. *(15)*

**A23.** Explain why `spawn` requires the `__main__` guard. Give five rules for reducing data
movement between processes, and state the `shared_memory` close/unlink discipline and what happens
if you get it wrong. *(15)*

**A24.** Compare threads, subinterpreters and processes across five dimensions. Explain what happens
when a pool worker is OOM-killed and where the evidence is, and give the decision tree for choosing
a parallelism mechanism — including the branch that sits above all of it. *(15)*

### Backpressure and flow control (L09)

**A25.** Define backpressure and explain why a single unbounded buffer anywhere breaks the whole
chain. State Little's Law and give three distinct uses of it. *(15)*

**A26.** Explain why a deeper queue does not improve throughput and what it does cost. Give five
load-shedding strategies and say when each applies. *(15)*

**A27.** Distinguish rate limiting from concurrency limiting and say which matters more for
protecting a downstream dependency and why. State the relationship between utilisation and queueing
latency and the capacity implication, explain why variability matters as much as the mean, and name
six places unbounded buffers hide. *(15)*

### Testing concurrent programs (L10)

**A28.** Give three structural reasons ordinary testing fails for concurrency. Name four ways to
make a stress test more effective, and identify the single most effective one. *(15)*

**A29.** Define deterministic simulation and state the property that makes it so valuable. Explain
what a linearizability checker checks that nothing else does, and what model checking establishes
and does not. *(15)*

**A30.** Give the seven-step hang-diagnosis procedure. Explain why naming tasks matters
operationally, and then — for each of stress testing, deterministic simulation, linearizability
checking and model checking — state in one sentence what it does *not* establish. *(15)*

---

## Section B — answer ONE

**B1. (40)** A service handles 2,000 requests/second. Each request calls three upstream
services and writes to Postgres. It is written with `asyncio`. Symptoms: p50 latency is
15 ms, p99 is 4 s and rising over the day; memory grows steadily and resets on deploy;
during an upstream slowdown last week the whole service became unresponsive and had to be
restarted; and one endpoint that does JSON parsing of large payloads seems to make unrelated
endpoints slower.

Write the investigation and remediation plan. It must include:

- Your ranked hypotheses, with the mechanism for each symptom.
- The specific measurement that would discriminate between them, in order, and what result
  eliminates which hypothesis. (Marks are for discrimination: three well-chosen
  measurements beat ten.)
- What you would instrument first, and what you would expect to see.
- For each likely root cause, the change and its cost.
- The structural changes that would prevent recurrence, distinguishing those that fix a
  symptom from those that fix a class.
- What you would not do, and why.

**B2. (40)** Design the concurrency architecture for a system that ingests a 50,000
events/second stream, enriches each event with two lookups (one cached, one a remote call),
performs a CPU-bound transformation, and writes batches to a columnar store. Bursts reach
5× the average. The remote lookup has a p99 of 200 ms and occasionally degrades to 5 s.
Events must not be lost; duplicates are acceptable if the store deduplicates.

Your answer must cover:

- The model and mechanism for each stage, justified against the alternatives.
- The queue between each pair of stages: bounded or not, depth derived from what.
- Where backpressure propagates to, and what happens when it reaches the source.
- The shedding policy, if any, and how it is observable.
- Concurrency and rate limits per dependency, with the sizing derived.
- Behaviour when the remote lookup degrades to 5 s — trace the effect through the whole
  pipeline.
- Cancellation and shutdown semantics: what is guaranteed about an in-flight event.
- What you would test with deterministic simulation, and the invariants you would assert.
- The capacity you would provision, and the utilization target, with the reasoning.

**B3. (40)** Argue for or against:

> "The removal of the GIL is a net negative for the Python ecosystem. It buys parallelism
> that most Python programs do not need, at the cost of single-thread performance, a decade
> of C-extension churn, and — most importantly — exposing every Python programmer to data
> races, a class of bug the language had accidentally protected them from and which the
> language provides no tools to reason about."

Required: state the opposing position at its strongest; give at least four specific technical
mechanisms as evidence; address separately the arguments from performance, from ecosystem
cost, and from correctness (they have different strengths); consider the alternatives
(subinterpreters, processes, GIL-releasing extensions) and whether they were sufficient; and
conclude with a falsifiable claim about what evidence would change your mind.

Even-handedness is assessed. An answer that does not state the opposing case fairly cannot
score above 24.

---

**B4. (40)** A team migrated a threaded service to `asyncio` six months ago. The results: p50
latency improved 30%; p99 latency got worse and is now spiky; CPU sits at 40% while throughput has
plateaued below the old system's; the service occasionally stops responding entirely and must be
restarted, with no error in the logs; and two data-corruption incidents have been traced to
"something with the cache".

Write the investigation. Your answer must: give your ranked hypotheses for each of the four
symptoms, noting where one cause could produce several of them; explain what the CPU-at-40%
observation rules in and out, precisely; give the seven-step procedure you would follow for the
hang, and say what evidence each step produces; identify the most likely cause of the p99
degradation given the other symptoms, with the mechanism; explain how the cache corruption could
occur in single-threaded async code, since the team believes async made locking unnecessary — name
the specific misconception; state the instrumentation you would add before changing anything; and
give the fix for each confirmed cause with the regression test that would prevent recurrence.

Marks are for connecting the symptoms to mechanisms. A list of general async advice scores in the
lowest band.

**B5. (40)** You must choose the concurrency architecture for a new service: it receives webhooks
(bursty, up to 5,000/second in spikes lasting under a minute, 200/second sustained), does a modest
amount of CPU work per event (about 8 ms of pure Python), writes to a database, and calls two
third-party APIs, one of which has a p99 of 4 seconds and occasionally hangs entirely.

Design it and defend every choice. Your answer must: state which concurrency model you would use for
each stage and why, working through the decision tree explicitly rather than asserting a conclusion;
do the Little's Law arithmetic for both the sustained and the spike load and state what it implies
for pool sizes and queue depths; state where the bounded queues are and what happens when each
fills — including what the webhook sender sees; address the 8 ms of CPU work specifically, since it
is the detail that decides the architecture, and give the two options with their costs; explain how
the hanging third-party API is prevented from consuming the whole system, naming the mechanisms and
their configuration; state your shutdown sequence and what it guarantees about in-flight events; and
identify the failure mode your design is *least* robust to, with what you would monitor to detect it.

Then state what you would change if the CPU work were 80 ms rather than 8 ms, and why that changes
the answer.

---

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision and correct edge
cases; 3 for an example not drawn from the lessons; 2 for a stated limitation of your own
answer or a connection to another course.

Section B: a correct, complete answer is 24/40. The rest is judgement, discrimination
between hypotheses, and honesty about what you do not know.
