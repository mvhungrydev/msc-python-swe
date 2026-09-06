# Program Handbook — MPSE

## 1. Philosophy of the program

A taught master's degree does three things that self-study usually fails to do.

**It sequences.** Topics arrive in an order where each one has somewhere to attach. You
learn the CPython object model before you learn why `functools.lru_cache` mutates your
memory profile; you learn linearizability before you learn why a distributed cache
invalidation scheme is subtly wrong. This program is sequenced. Do not shuffle it.

**It forces production.** You cannot pass a graduate course by reading. Every course here
has problem sets and an exam, and every term has a build artifact. If you read the lessons
and skip the problem sets you will finish with the *feeling* of expertise and not the
substance of it. This is the single most common failure mode of self-directed study.

**It exposes you to primary sources.** Textbooks and blog posts are compressions of
research. At the master's level you are expected to read the original — Lamport's *Time,
Clocks*, Parnas on modularity, Wadler on theorems for free, the Dynamo and Spanner papers.
Each course lists 3–6 required primary readings. They are short. Read them.

## 2. Prerequisites

You should already be able to:

- Write and ship Python programs of a few thousand lines.
- Use `git` fluently, including rebase and bisect.
- Read a stack trace and use a debugger.
- Operate on a Unix-like command line.
- Read basic mathematical notation (set-builder, summation, quantifiers). If not, work
  through `appendices/mathematical-notation.md` first — it takes an evening.

You do **not** need: a CS degree, calculus, or prior exposure to formal logic.

## 3. Structure and credit model

Each course is nominally **15 credits** in the ECTS-like sense used here: roughly
120–150 hours of work, split about 30% reading and lecture-equivalent, 50% problem sets
and code, 20% review and assessment.

| Term | Theme | Courses | Nominal hours |
|---|---|---|---|
| 1 | Foundations of Craft | PY-501, SE-511 | ~250 |
| 2 | Abstraction & Design | PY-502, SE-521 | ~250 |
| 3 | Systems & Performance | PY-601, PY-602 | ~250 |
| 4 | Theory Core | CS-621, CS-641 | ~280 |
| 5 | Distributed & Data-Intensive | DS-701, DI-721 | ~280 |
| 6 | Advanced Practice | CA-731, ML-741, FM-751 | ~330 |
| — | Capstone | CAP-799 | ~200 |

Total: **~1,840 hours**. At 8 hours a week that is roughly four and a half years; at 15
hours a week, about two and a half; at 25 hours a week, about eighteen months. Pick a
sustainable rate and hold it. A master's degree is not won by intensity, it is won by not
stopping.

### Prerequisite graph

```
PY-501 ──┬─→ PY-502 ──┬─→ PY-601 ──→ PY-602 ──┐
         │            │                        │
SE-511 ──┴─→ SE-521 ──┴──────────┐             ├─→ CA-731 ─┐
                                 │             │           │
CS-621 ──→ CS-641 ───────────────┼─→ DS-701 ───┤           ├─→ CAP-799
                                 │      │      │           │
                                 └─→ DI-721 ───┴─→ ML-741 ─┘
                                        │
                                    FM-751 (needs CS-641)
```

CS-621 and CS-641 have no Python prerequisites and may be pulled forward into Term 1 or 2
if you prefer to front-load theory. Everything else respects the arrows.

## 4. How to study a lesson

Each lesson follows the same shape, and each part has a job:

1. **Orientation** — what problem this exists to solve, and what breaks without it.
2. **Theory** — the model, stated precisely. Read this twice. The second reading is where
   the definitions actually land.
3. **Construction** — code built incrementally, naive version first. *Type it.* Run the
   broken version. Watch it fail in the specific way the text predicts.
4. **Failure modes** — the ways working engineers get this wrong. This section is where
   most of the transferable value is, because it is the part written from scar tissue.
5. **Exercises** — three tiers: **Warm-up** (checks comprehension, 10–20 min), **Core**
   (the real work, 45–120 min), **Challenge** (open-ended, sometimes multi-day).
6. **Self-check** — questions you should be able to answer aloud, without notes, before
   moving on. If you cannot, re-read the theory section. This is not optional.
7. **Primary sources** — go read at least one per lesson pair.

**A studied lesson takes 2–4 hours.** If you finished in forty minutes you read it; you
did not study it.

### The retrieval rule

At the start of each study session, before opening the new lesson, spend five minutes
writing — from memory, on paper — the key claims of the previous lesson. This single habit
is worth more than any note-taking system. Recall is what builds durable memory; re-reading
produces only the feeling of fluency.

### The build rule

Every term ends with an artifact you built and can demo. Not a toy, not a tutorial
follow-along: something with tests, docs, CI, and a README that argues for its design.
These accumulate into a portfolio that is, frankly, more persuasive than the degree would
have been.

| Term | Build artifact |
|---|---|
| 1 | A small library with 95%+ meaningful coverage, full typing, and a published package |
| 2 | A non-trivial domain model with a documented architecture and ADR log |
| 3 | A high-throughput async service, profiled and optimized with before/after numbers |
| 4 | An interpreter with a type checker, for a small language of your own design |
| 5 | A replicated key-value store with a consistency model you can state and defend |
| 6 | A platform component: a Terraform provider, an operator, or an ML serving system |
| Capstone | A substantial system plus a written thesis-equivalent defending it |

## 5. Assessment

There are no grades because there is no one to award them. There are **rubrics**, and you
apply them to yourself honestly. `00-program/assessment-and-rubrics.md` gives the criteria.

The functional test of mastery at this level is: **can you defend it?** For each course,
write a 1,500-word position paper answering the course's driving question, then find
someone competent and let them attack it. Failing to be able to defend a claim is the only
grade that matters.

Each course also has an `exam.md`: a closed-book written paper in the style of a real MSc
examination. Sit it under timed conditions. Mark it against the rubric a week later, when
you have forgotten what you meant.

## 6. Failure modes of self-directed graduate study

Named here so you recognize them when they happen to you.

- **Tutorial drift.** Substituting easy new material for hard current material. Symptom:
  you have started four courses and finished none. Fix: single-threading. One course at a
  time until its problem sets are done.
- **Collector's fallacy.** Accumulating bookmarks, books, and notes as a proxy for
  learning. Fix: the retrieval rule; nothing counts until you can produce it from memory.
- **Skipping the maths.** The theory courses are the ones that raise your ceiling. They
  are also the ones with no immediate payoff, which is exactly why people skip them and
  then plateau. Fix: schedule theory in your highest-energy slot, not your leftover slot.
- **Building without shipping.** The term artifacts must be finished and published, not
  perfected. Ship at 80%.
- **Solo isolation.** Find one person to review your term artifacts. One is enough.

## 7. Recommended cadence

A cadence that works for a full-time engineer:

- **Weekday mornings, 45 minutes** — one lesson section, plus retrieval on the last one.
- **Two weekday evenings, 90 minutes** — problem sets and code.
- **Saturday, 3 hours** — the deep block: theory reading, primary sources, term artifact.
- **Sunday, 30 minutes** — review week's notes, plan next week, update the log.

That is ~11 hours/week, which finishes the program in about three and a half years
including the capstone.

Keep a **learning log**: one file per week, three lines — what I studied, what I got
wrong, what I owe. `appendices/learning-log-template.md` has the format. When motivation
sags in month fourteen, the log is what shows you the distance travelled.

## 8. Course catalog

Full descriptions in each course's `syllabus.md`. In brief:

**PY-501 — The Python Object Model & Execution Semantics.** Objects, names, and binding;
the type/instance/metatype triangle; attribute lookup and the MRO; the memory model and
reference counting; the frame and evaluation model; bytecode; exceptions as control flow.
*Driving question: what actually happens when Python evaluates `a.b(c)`?*

**SE-511 — Software Construction: Testing, Types, and Tooling.** Testing as specification;
property-based testing; test doubles and the boundaries they imply; gradual typing theory
and practice; static analysis; packaging and dependency resolution; reproducible builds;
CI as a design constraint. *Driving question: what makes code trustworthy?*

**PY-502 — Advanced Abstraction.** Protocols and structural typing; the descriptor
protocol; class construction and metaclasses; `__init_subclass__` and modern alternatives;
generics and variance in Python; abstract interpretation of decorators; building internal
DSLs; the `dataclasses`/`attrs`/`pydantic` design space. *Driving question: when is
metaprogramming a better answer than repetition?*

**SE-521 — Software Architecture & Design.** Modularity and information hiding; coupling
and cohesion formalized; design patterns as a vocabulary and their critique; domain-driven
design; hexagonal/clean architecture and its costs; evolutionary architecture and fitness
functions; architectural decision records; legacy system strategy. *Driving question: what
distinguishes a structure that can absorb change from one that cannot?*

**PY-601 — Concurrency, Parallelism, and Asynchrony.** Concurrency vs parallelism; the
GIL and the free-threaded build; threads, locks, and the memory model; `asyncio` from the
event loop up; structured concurrency; cancellation and its correctness conditions;
multiprocessing and shared memory; actor and CSP models. *Driving question: what does it
mean for a concurrent program to be correct?*

**PY-602 — Performance Engineering & the Systems Interface.** Measurement methodology and
statistics; profiling (deterministic, sampling, memory); CPU caches and data layout;
CPython's object overheads; `__slots__`, arrays, and NumPy's memory model; the C API,
Cython, and Rust extensions; syscalls, I/O, and zero-copy; queueing theory for engineers.
*Driving question: where does the time actually go?*

**CS-621 — Algorithms, Complexity, and Computability.** Asymptotics and amortized
analysis; divide and conquer and recurrences; greedy and exchange arguments; dynamic
programming; graph algorithms; randomized algorithms and concentration; NP-completeness
and reductions; approximation; computability, halting, and Rice's theorem. *Driving
question: which problems are hard, and how would you know?*

**CS-641 — Programming Languages & Type Systems.** Syntax, semantics, and the lambda
calculus; operational semantics; simply-typed lambda calculus and progress/preservation;
polymorphism and Hindley–Milner inference; subtyping and variance; algebraic data types
and pattern matching; effects and monads; a working interpreter and type checker. *Driving
question: what does a type actually guarantee?*

**DS-701 — Distributed Systems.** Failure models; time, clocks, and causality; consistency
models from linearizability to eventual; consensus (Paxos, Raft); replication and quorums;
CAP, PACELC and their frequent misuse; distributed transactions and sagas; failure
detection; observability and distributed tracing; chaos engineering. *Driving question:
what can you guarantee when parts of your system are lying to you?*

**DI-721 — Data-Intensive Systems.** Storage engines (B-trees vs LSM); indexing;
transactions and isolation levels; MVCC; column stores and analytical processing; batch vs
stream processing; exactly-once semantics; event sourcing and CDC; schema evolution; data
modelling for warehouses and lakehouses. *Driving question: how does data keep its meaning
as it moves?*

**CA-731 — Cloud & Platform Architecture at Scale.** Cloud as a distributed system with a
billing model; multi-tenancy; identity and the trust boundary; network architecture;
infrastructure as code and provider design; Kubernetes as a control-plane pattern;
reliability engineering, SLOs and error budgets; capacity and cost modelling; platform
engineering as a product discipline. *Driving question: what is the platform's actual
contract with its users?*

**ML-741 — Machine Learning Systems Engineering.** ML systems as software systems; data
and feature engineering pipelines; training infrastructure; experiment tracking and
reproducibility; model serving and inference optimization; drift, monitoring, and
evaluation in production; feedback loops and their hazards; LLM systems — retrieval,
evaluation, and cost control. *Driving question: what breaks when a model becomes a
dependency?*

**FM-751 — Formal Methods & Verification for Practitioners.** Specifications as artifacts;
Hoare logic and invariants; model checking; TLA+ and PlusCal; refinement; property-based
testing as lightweight verification; SMT solvers and Z3; contract and runtime verification.
*Driving question: how do you know a design is right before you build it?*

**CAP-799 — Capstone.** A substantial system, a written defence, and an oral examination
you arrange for yourself.

## 9. Errata and revision

Nothing here is infallible. Keep `appendices/errata.md` updated with anything you find
wrong or out of date — especially Python version behaviour, which moves. Finding an error
and being able to prove it is itself a passing grade on that lesson.
