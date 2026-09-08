# PY-601 · Lesson 10 — Testing and Debugging Concurrent Programs

**Estimated study time:** 3.5 hours
**Prerequisites:** all of PY-601; SE-511 L03, L04

---

## 1. Orientation

A concurrency test that passes tells you: *one* interleaving out of exponentially many
produced the right answer, on this machine, under this load, on this scheduler, once.

That is nearly no information. It is why "we tested it" is a weaker claim here than
anywhere else in software, and why the techniques in this lesson exist: they are all
attempts to convert "we ran it and it worked" into something with actual evidential weight.

## 2. Theory

### 2.1 Why the usual approach fails

Three reasons, all structural:

- **The state space is exponential.** N threads with M steps each have on the order of
  (NM)!/(M!)^N interleavings. Random execution samples a vanishingly small fraction.
- **The sampling is biased.** Your machine, your scheduler, and your load are not
  production's. Real schedulers are *not* adversarial, which sounds good and is actually the
  problem: they reliably explore the same small region, so the rare interleavings stay
  unexplored until production finds them.
- **Failures are not local.** A race corrupts state now and the symptom appears three
  operations later, in different code.

So the techniques below are ordered by how much of the space they cover, and you should be
able to say which one you are relying on for any given claim.

### 2.2 Stress testing

Run it hard and see if it breaks.

```python
def test_counter_under_contention() -> None:
    c = Counter()
    with ThreadPoolExecutor(64) as ex:
        list(ex.map(lambda _: [c.incr() for _ in range(1000)], range(64)))
    assert c.value == 64_000
```

Cheap and it does find gross errors. Its limits are the point: passing establishes almost
nothing, and a failure rate of 1-in-10,000 needs ~30,000 runs for reasonable confidence of
seeing it once.

Make it more effective:

- **Increase the switch rate.** `sys.setswitchinterval(0.000001)` forces far more thread
  switches, widening race windows dramatically. This single line turns many "never
  reproduces" bugs into "fails every time".
- **Add jitter.** Random `time.sleep(0)` or tiny sleeps at suspicious points widens windows.
- **Vary the thread count** across runs, including counts above the core count.
- **Run it many times** in CI as a *nightly* job, not a PR gate (a flaky-looking gate gets
  disabled — SE-511 L01 §2.7).
- **Run under contention**: pin to fewer cores than threads, or add background load.

### 2.3 Deterministic simulation

The strongest practical technique, and the most under-used.

The idea: **make the scheduler an input.** Replace real time, real I/O, and real scheduling
with deterministic fakes driven by a seed. Then a failing run is fully reproducible from its
seed, and you can explore the interleaving space systematically instead of randomly.

For `asyncio` this is tractable because the loop is already yours:

- Replace the clock with a virtual clock that jumps to the next scheduled deadline instead
  of sleeping. (Tests run in microseconds regardless of how much time the code "waits".)
- Replace network I/O with in-memory transports.
- Make the ready-queue ordering seeded and shuffleable, so a different seed produces a
  different legal interleaving.

`trio`'s testing support (`trio.testing.MockClock`, `wait_all_tasks_blocked`) is built for
this, and it is one of the strongest arguments for trio. `anyio` exposes similar
facilities. For `asyncio`, `asyncio.EventLoop` subclassing plus `time.monotonic` injection
gets you most of the way.

FoundationDB's deterministic simulation is the canonical industrial example: they built the
database *inside* a simulator that controls the clock, the network, the disk, and process
failures, and run millions of seeded simulated years. It is why their fault tolerance is
trusted. The technique transfers to ordinary services at much lower cost than people assume,
and DS-701 L10 returns to it.

**The key property**: with a seed, `pytest --seed=91827` reproduces the failure exactly. That
converts a heisenbug into an ordinary bug.

### 2.4 Property-based and stateful testing

`hypothesis`'s `RuleBasedStateMachine` (SE-511 L04 §2.4) generates *sequences of
operations*, and with a model to compare against, it finds violations that no example test
covers. For concurrency, use it in two ways:

- **Generate schedules.** Model the concurrent operations as interleavings and have
  `hypothesis` generate them, checking a linearizability property (L01 §2.4) against a
  sequential model.
- **Generate operation sequences** against a deterministic simulator (§2.3), so the schedule
  and the operations are both generated and both shrinkable.

The payoff is shrinking: a failure in a 40-operation interleaving shrinks to the minimal
3-operation sequence that breaks it, which is usually diagnostic on sight.

### 2.5 Linearizability checking

Record a history of invocations and responses with timestamps, then ask: does a
linearization exist (L01 §2.4)?

```
[t0] T1 invoke  push(1)
[t1] T2 invoke  pop()
[t2] T1 response ok
[t3] T2 response 1          → is there a valid sequential order? yes: push(1); pop()→1
```

Checking is NP-complete in general, but for small histories brute force works. Jepsen's
`knossos`/`elle` checkers do this for distributed systems; for a single concurrent object,
a hundred lines of Python suffices for histories of a dozen operations.

This is the *only* technique here that checks the property you actually care about rather
than a proxy, and it is worth building once so you know what it involves.

### 2.6 Model checking

Write a model of the concurrent algorithm and let a tool explore *all* interleavings.
TLA+/PlusCal (FM-751) does this and finds deadlocks, invariant violations, and liveness
failures that no amount of testing would.

It checks the *model*, not your code — a real limitation and a real strength. The model is
small enough to be exhaustively checked, and the process of writing it forces you to state
the invariants precisely, which frequently finds the bug before you run anything.

Use it for: lock protocols, cache coherence, distributed algorithms, state machines with
concurrent transitions. Not for: ordinary application code.

### 2.7 Dynamic analysis and tooling

**Thread sanitizer (TSan)** detects data races in C/C++ at runtime. Relevant to Python when
you write extensions (PY-602 L07) or run a free-threaded build with native code.

**`faulthandler`** — the first tool for a hung process:

```python
import faulthandler
faulthandler.enable()
faulthandler.dump_traceback_later(30, repeat=True, exit=False)   # every 30s
```

Also `faulthandler.register(signal.SIGUSR1)` so you can dump a stack on demand in
production without a debugger.

**`py-spy`** — sample or dump the stacks of a *running* process without modifying or
restarting it:

```bash
py-spy dump --pid 1234              # all thread stacks, right now
py-spy top --pid 1234               # live profile
py-spy record -o out.svg --pid 1234 # flame graph
```

This is the single most valuable production debugging tool for Python concurrency. It works
on a hung process and tells you exactly where every thread is stuck.

**`asyncio` debug mode** — `PYTHONASYNCIODEBUG=1` or `loop.set_debug(True)`: logs slow
callbacks, un-awaited coroutines, and tasks destroyed while pending. Set
`loop.slow_callback_duration` to something meaningful (10 ms) and log it in production.

**`asyncio.all_tasks()`** — dump every live task with its stack:

```python
for t in asyncio.all_tasks():
    print(t.get_name(), t.get_coro()); t.print_stack()
```

Wire this to a signal handler. When an async service hangs, this tells you what every task is
awaiting, which is nearly always enough.

**Name your tasks.** `asyncio.create_task(coro, name="ingest:batch-42")`. Unnamed tasks in a
dump are `Task-17`, which is useless. This costs nothing and is the difference between a
five-minute and a two-hour incident.

### 2.8 Debugging a hang

A procedure, in order:

1. **Is it hung or slow?** `py-spy top` — if the stacks are moving, it is slow.
2. **Dump all stacks.** `py-spy dump` for threads; `all_tasks()` + `print_stack()` for
   asyncio.
3. **Look for two threads waiting on locks.** A cycle in the wait-for graph is a deadlock.
4. **Look for a thread in a syscall with no timeout.** A `recv` with no timeout on a dead
   connection is the most common "hang" in practice, and TCP will not tell you for hours.
5. **Look for a full queue.** A producer blocked on `put` and a consumer blocked on
   something downstream is backpressure working as designed and the real problem is
   downstream.
6. **Check for a task awaiting a future nobody will resolve.** The classic async deadlock,
   usually caused by a cancelled or crashed task that was supposed to set the result.
7. **Check the executor.** A blocked thread-pool thread that cannot be cancelled (L07 §2.7).

Practice this on a *deliberately* hung process before you need it in production. That is
exercise C4.

## 3. Construction: testing a concurrent component

Take a component with real concurrency — the cache from L03 §3, the pipeline from L04 §2.4,
or a connection pool.

**Level 1 — stress.** Write the stress test. Run it 1,000 times. Then add
`sys.setswitchinterval(1e-6)` and run again. Report the failure rate at each setting. If the
low switch interval finds a bug the default does not, you have learned exactly why stress
testing at default settings is weak evidence.

**Level 2 — deterministic simulation.** Build a harness where the scheduler is seeded:

- A virtual clock that advances to the next deadline.
- Seeded ordering of the ready queue.
- In-memory transports for any I/O.

Then run 10,000 seeds. Report the failure rate and the seeds. Verify that a failing seed
reproduces exactly, ten times out of ten. **That reproducibility is the deliverable.**

**Level 3 — properties.** Define the invariants your component must satisfy (L01 §2.3:
which are safety, which are liveness) and assert them after every operation in the
simulation. Typical: no lost updates, no duplicate processing, bounded size, no item
processed after cancellation, every enqueued item eventually processed or explicitly
dropped.

**Level 4 — linearizability.** Record a history of invocations and responses in the
simulation, and check that a valid linearization exists. Implement the checker (brute force
over permutations, pruned by the real-time constraints). Then deliberately break the
component and confirm the checker catches it — a checker you have not seen fail is a checker
you cannot trust.

**Level 5 — fault injection.** Inject: a cancelled task at each `await`, a worker crash, a
slow dependency, a full queue, and a clock jump. For each, assert the invariants still hold.
The cancellation injection (L07 X2) finds the most.

**Level 6 — the model.** Write the core protocol in PlusCal and model-check it (FM-751 L03,
or do it now with the TLA+ Toolbox). Report anything the model check found that levels 1–5
did not — in a non-trivial component there is usually something, and it is usually a liveness
property.

**Level 7 — report.** For each level, state what it establishes and what it does not. That
table is the point of the whole exercise: it is how you talk honestly about concurrency
confidence.

| Technique | Covers | Establishes | Does not establish |
|---|---|---|---|
| Stress | a biased sample of interleavings | gross errors absent | anything about rare interleavings |
| Sim + seeds | a wide random sample, reproducibly | rare bugs findable and fixable | exhaustiveness |
| Properties | invariants over generated sequences | the invariants you thought of | the ones you did not |
| Linearizability | the correctness condition itself | the object behaves atomically | liveness |
| Fault injection | failure paths | robustness to injected faults | faults you did not inject |
| Model checking | all interleavings of the model | the model is correct | the code matches the model |

## 4. Failure modes

- **A passing stress test read as evidence of correctness.**
- **Retrying a flaky concurrency test in CI.** You have found a real bug and hidden it.
- **`time.sleep()` for synchronization in tests.** Slow, and still flaky — the sleep is a
  guess about timing. Use events, barriers, or `wait_all_tasks_blocked`.
- **No seed, so failures are not reproducible.**
- **Testing only the happy path.** Cancellation, crash, and timeout paths are where the bugs
  are.
- **Unnamed tasks**, making production dumps unreadable.
- **No `faulthandler` and no way to dump stacks in production.**
- **Debug mode off in production**, so slow callbacks are invisible.
- **Testing with fewer threads than production.**
- **Ignoring liveness.** Every technique above except model checking is weak on it.
- **Believing the model check covers the code.** It covers the model.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a racy component. Show its stress test passing at the default switch interval
and failing at 1e-6.

**W2.** Hang a process deliberately (a deadlock, and a socket read with no timeout). Diagnose
each with `py-spy dump` alone.

**W3.** Dump all asyncio tasks with their stacks from a signal handler in a running service.

### Core (3 h)

**C1 — All seven levels.** Complete §3 on one component. Deliverable: the code for each
level, the failure rates, a reproducible failing seed, the linearizability checker with a
demonstrated catch, the fault-injection results, and the level-7 table filled in for your
component.

**C2 — A deterministic simulator.** Build a reusable `asyncio` test harness: virtual clock,
seeded ready-queue ordering, in-memory transports, and a `pytest` fixture exposing it.
Publish it in your study repo. Then use it to find a bug in a component you believed was
correct.

**C3 — Production readiness.** For a real service, add: named tasks everywhere,
`faulthandler` on a signal, an `all_tasks` dump endpoint, `slow_callback_duration` logging,
and a documented hang-diagnosis runbook following §2.8. Then have a colleague hang it
without telling you how, and diagnose it using only your instrumentation. Time yourself.

**C4 — Flaky test forensics.** Find a flaky test in a real suite. Do not delete or retry it.
Determine the actual race by: running it 1,000 times with a low switch interval, adding
jitter, and reading the failure. Report the root cause. (In a large suite this is a
genuinely valuable contribution and a good way to build a reputation.)

### Challenge

**X1.** Implement a linearizability checker that handles histories of up to ~20 operations
efficiently (prune by real-time precedence, memoize on state). Test it against known-correct
and known-broken concurrent objects. Then read about Jepsen's `elle` and write 800 words on
how it scales beyond brute force.

**X2.** Take a concurrent algorithm from your own system, specify it in PlusCal, and
model-check it for both safety and liveness. Report everything the checker found. Then write
600 words on the gap between model and implementation and how you would narrow it (refinement
mapping, generated tests from the model, or runtime assertion of the model's invariants).

## 6. Self-check

1. Give three structural reasons ordinary testing fails for concurrency.
2. Name four ways to make a stress test more effective, and the single most effective one.
3. What is deterministic simulation, and what property makes it so valuable?
4. What does a linearizability checker check that nothing else does?
5. What does model checking establish and what does it not?
6. Give the seven-step hang-diagnosis procedure.
7. Why does naming tasks matter?
8. For each technique in the §3 table, state in one sentence what it does not establish.

## 7. Primary sources

- FoundationDB, "Testing Distributed Systems w/ Deterministic Simulation" (Rowley, Strange
  Loop 2014). The best talk on the subject.
- Kyle Kingsbury (aphyr), the Jepsen reports — read two, any two.
- `trio.testing` documentation, and Smith's writing on testing async code.
- Musuvathi & Qadeer, "Iterative Context Bounding for Systematic Testing of Multithreaded
  Programs" (PLDI 2007) — the theory behind bounded systematic exploration.
- `py-spy`, `faulthandler`, and `asyncio` debug-mode documentation.

---

**Previous:** [L09](L09-backpressure-and-flow-control.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), and
[PY-602](../PY-602-performance-engineering/syllabus.md).
