# PY-602 · Lesson 02 — Profiling: Deterministic, Sampling, and Attribution

**Estimated study time:** 4 hours
**Prerequisites:** L01

---

## 1. Orientation

A profiler answers "where does the time go?" — but each kind of profiler answers a slightly
different question, and using the wrong one produces a confident, wrong answer.

The most common failure: running `cProfile` on a program dominated by a few very frequent
small calls. `cProfile`'s per-call overhead inflates exactly those calls, so the profile
*creates* the hot spot it reports. You then optimize a function that was never the problem.

This lesson is about choosing the right instrument and reading its output correctly.

## 2. Theory

### 2.1 The two families

**Deterministic (tracing) profilers** hook every function call and return.

- Exact call counts and the full call graph.
- Overhead per call: `cProfile` adds roughly 1–3 µs per call, which for a program making
  10⁷ calls is 10–30 seconds of pure overhead.
- **Systematically biased toward small, frequently-called functions.** This is the failure
  above.
- Cannot profile a running process; you must start under the profiler.

**Sampling (statistical) profilers** interrupt periodically and record the stack.

- Overhead proportional to sample rate, not call count — typically 1–5%.
- Unbiased with respect to function size.
- Can attach to a *running* process (`py-spy`), which is the only option in production.
- No exact counts; rare-but-slow events may be missed. A function taking 100 ms once in a
  10-minute run may not appear at all.

**Rule: use a sampling profiler by default.** Reach for `cProfile` when you need exact call
counts (e.g. "how many times do we call the database?"), which is a *different question*
from "where does the time go".

### 2.2 `cProfile`, read correctly

```python
import cProfile, pstats
with cProfile.Profile() as pr:
    main()
pstats.Stats(pr).sort_stats("cumulative").print_stats(30)
```

The columns, and what each means:

| Column | Meaning |
|---|---|
| `ncalls` | number of calls (`a/b` = total/primitive, i.e. non-recursive) |
| `tottime` | time in this function **excluding** callees |
| `cumtime` | time in this function **including** callees |
| `percall` | the respective time divided by ncalls |

- **Sort by `tottime`** to find where work is actually done.
- **Sort by `cumtime`** to find which high-level operation is expensive.
- A function with high `cumtime` and low `tottime` is a *router*: the cost is below it.

Visualize rather than read tables: `snakeviz` (an icicle chart), `gprof2dot | dot`, or
export to `pstats` and load in a tool. A 300-row table hides structure that a picture shows
in seconds.

`cProfile` does not see: C-level time attributed to the caller, time in other threads (it
profiles one thread unless you set it per-thread), or time spent waiting outside a call.

### 2.3 `py-spy` and flame graphs

```bash
py-spy record -o profile.svg --duration 60 --rate 200 -- python app.py
py-spy record -o profile.svg --pid 1234              # attach to a running process
py-spy top --pid 1234                                # live, top-like
py-spy dump --pid 1234                               # all thread stacks right now
```

Key options that change what you learn:

- `--native` includes C stacks — essential when time is inside NumPy, a database driver, or
  your own extension. Without it, the whole cost appears as one Python frame.
- `--idle` includes threads that are not running, which is how you see where an async or
  threaded program is *waiting*. By default `py-spy` shows on-CPU time only, so a program
  that is 95% waiting looks almost idle.
- `--subprocesses` follows children.
- `--rate` — 100/s is the default; raise it for short runs, lower it for long ones.

**Reading a flame graph.** The x-axis is *not time*; it is the population of samples, sorted
alphabetically for stability. Width is proportion of samples; height is stack depth. What to
look for:

- **Wide leaves.** A wide frame with nothing above it is where the CPU actually is.
- **Wide plateaus.** A wide frame with many narrow children is dispatch overhead.
- **Unexpected width.** Something you did not expect to be there at all — logging,
  serialization, `__repr__`, a validator running on every access.
- **Tall thin towers.** Deep recursion or over-layered abstraction; not necessarily costly.

The most common surprise in a real Python flame graph: **serialization and logging**. In
services, JSON encoding, ORM object construction, and log formatting routinely account for
more time than the "business logic".

**Differential flame graphs** — comparing before and after — are the highest-value form and
are underused. `flamegraph.pl --negate` or `speedscope`'s comparison view.

### 2.4 `perf` and system-level profiling

Python profilers stop at the Python/C boundary in a useful way but cannot tell you about
cache misses, branch mispredictions, or kernel time.

```bash
perf stat -d python bench.py                  # counters: IPC, cache misses, branch misses
perf record -g --call-graph dwarf python bench.py
perf report
```

`perf stat` output worth understanding:

- **IPC (instructions per cycle).** Below ~1.0 on modern hardware suggests stalls — memory,
  branches, or dependencies. Above ~2 suggests you are compute-bound and well-pipelined.
- **Cache miss rate.** LLC misses are ~200–300 cycles each; a high rate means the working set
  or the layout is wrong (L06).
- **Branch miss rate.** ~15–20 cycles each; a high rate means unpredictable branching.

For Python specifically, CPython built with `--enable-shared` and frame-pointer support gives
`perf` readable Python frames; without it you see interpreter internals. Since 3.12 there is
`PYTHONPERFSUPPORT=1` / `-X perf`, which emits a JIT map so `perf` can resolve Python
function names — a genuinely useful addition and worth enabling for any `perf` work on
Python.

`bpftrace`/eBPF (Gregg) goes further: syscall latency, block I/O, scheduler wakeups, off-CPU
time. Off-CPU profiling is the complement to on-CPU profiling and answers "why is this not
running?" — which for most services is the actual question.

### 2.5 Attribution: on-CPU versus off-CPU

A service spending 95% of wall clock waiting for a database will show an almost-empty CPU
profile. You must know which regime you are in:

```python
t0, c0 = time.perf_counter(), time.process_time()
work()
wall, cpu = time.perf_counter() - t0, time.process_time() - c0
print(f"wall={wall:.3f}s cpu={cpu:.3f}s cpu_frac={cpu/wall:.1%}")
```

- **cpu_frac near 1** → CPU-bound. On-CPU profiling is the right tool.
- **cpu_frac near 0** → waiting. Use `py-spy --idle`, off-CPU profiling, or distributed
  tracing.
- **In between** → both, and you need to split the analysis.

This one measurement, taken first, prevents the most common wasted afternoon in performance
work.

### 2.6 Tracing in production

Profilers answer "where does the time go across many requests". **Distributed tracing**
answers "where did the time go in *this* request", which is a different and often more
useful question.

OpenTelemetry spans give you a per-request waterfall across services. What to instrument:

- Every outbound call (database, cache, HTTP), with the target and the operation.
- Significant internal phases.
- Attributes that let you slice: tenant, endpoint, cache hit/miss, batch size.

Two things people get wrong:

- **Sampling head-based at 1%** loses exactly the slow requests you want. Use tail-based
  sampling (decide after the trace completes, keeping all slow and error traces) or a
  latency-triggered sampler.
- **Spans too coarse.** A single span for "handle request" tells you nothing. A span per
  outbound call is the minimum useful granularity.

**Continuous profiling** (Pyroscope, Parca, cloud profilers) samples production continuously
at low overhead and lets you diff profiles across deploys or between the p99 and p50
populations. This is now the state of the art for services, and diffing "this week versus
last week" catches regressions that no benchmark would.

### 2.7 The Python-specific things profiles reveal

Recurring findings, worth knowing so you recognize them:

- **Attribute access in loops.** `self.config.timeout` inside a hot loop is three dict
  lookups per iteration (PY-501 L03). Hoisting it out is a real win — and PY-501 L08 §2.6
  notes that inline caches have reduced how much.
- **`isinstance` in hot paths**, particularly against ABCs, which run `__instancecheck__`.
- **Logging.** `log.debug(f"...{expensive!r}")` formats the string *even when debug is
  disabled*, because the f-string is evaluated before the call. Use
  `log.debug("...%r", x)` — lazy formatting.
- **`__repr__` called by logging or by a debugger-friendly library.**
- **Serialization.** Usually larger than expected.
- **ORM object construction.** Frequently the dominant cost of a "database-bound" endpoint —
  the query took 3 ms and building 5,000 objects took 200 ms.
- **Exception construction in a loop.** Building a traceback is not free.
- **Regex compilation** not being cached (it is cached by `re`, but only for the last 512
  patterns and only for identical pattern strings).
- **Import time.** For CLIs and serverless, `python -X importtime` often finds seconds.

### 2.8 Profiling async and multi-threaded code

- `cProfile` profiles the calling thread only. For threads, use `threading.setprofile`, or
  better, `py-spy` which sees all threads.
- For `asyncio`, on-CPU profiles show the event loop and your handlers interleaved; use
  `--idle` to see waiting, and rely on tracing for per-request attribution.
- `yappi` handles threads and coroutines with wall-clock or CPU-clock modes and is the best
  deterministic option for concurrent code.
- **Never profile a single request in isolation and conclude about production.** Contention
  effects only appear under load.

## 3. Construction: profiling a real application

Take a real, slow application — an endpoint, a batch job, a CLI.

**Step 1 — the regime.** Measure `perf_counter` versus `process_time` (§2.5). Write down
which regime you are in *before* choosing a profiler. This determines everything that
follows.

**Step 2 — sample it.** `py-spy record --native --idle` for a representative run. Read the
flame graph. Write down your top three suspects *and* the one thing you did not expect to be
there.

**Step 3 — confirm with counts.** For the top suspect, use `cProfile` to answer a count
question: how many times is it called? Often the surprise is not the per-call cost but the
call count — an N+1 query, a validator running per attribute, a `__eq__` in a linear scan.

**Step 4 — check the algorithm.** Before optimizing the hot function, ask whether it should
be called that many times at all. This is L01 §2.8 step 3, and it is where the large wins
are. Measure how the cost scales with input size; if it is superlinear, stop profiling and
fix the complexity (CS-621).

**Step 5 — system counters.** `perf stat -d` on the same workload. Report IPC and cache miss
rate. If IPC is below 1, the problem may be memory layout (L06) rather than instruction
count.

**Step 6 — import time.** `python -X importtime app.py 2>&1 | sort -k2 -n | tail -20`.
For a CLI or serverless function this is often the entire problem.

**Step 7 — the differential.** Make one change. Re-profile. Produce a differential flame
graph. Verify that the frame you meant to shrink shrank *and* that nothing else grew — the
common outcome is that the cost moved rather than disappeared.

**Step 8 — the report.** The regime, the profile, the surprise, the change, the differential,
and the end-to-end measurement from L01's harness. A profile without an end-to-end
measurement is not evidence that anything got faster.

## 4. Failure modes

- **`cProfile` on call-heavy code.** The profiler creates the hot spot.
- **On-CPU profiling a waiting program.** An empty profile, wrongly interpreted.
- **Reading a flame graph's x-axis as time.**
- **No `--native`**, so all the cost appears as one opaque frame.
- **No `--idle`**, so waiting is invisible.
- **Profiling one request and generalizing to production under load.**
- **Optimizing the top frame without asking why it is called so often.**
- **Head-based trace sampling at 1%**, discarding the slow requests.
- **Spans too coarse to attribute anything.**
- **Eager string formatting in disabled log calls.**
- **Profiling with unrealistic data**, so the profile is of a different program.
- **No before/after differential**, so you cannot show the cost moved rather than went.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write a function making millions of small calls. Profile with `cProfile` and with
`py-spy`. Report the difference in both the wall clock and the attribution, and explain it.

**W2.** Profile a program that spends 95% of its time in `time.sleep`. Show what `py-spy`
reports with and without `--idle`.

**W3.** Demonstrate eager f-string formatting in a disabled `log.debug` call, and measure the
cost of 10⁶ such calls with and without lazy formatting.

### Core (2.5 h)

**C1 — Profile a real application.** Complete §3, all eight steps. Deliverable: the regime
measurement, the flame graph with your three suspects and the surprise, the count confirmation,
the complexity check, the `perf stat` counters, the import-time report, the differential flame
graph, and the end-to-end result.

**C2 — Profiler comparison.** For one workload, profile with `cProfile`, `py-spy`, `yappi`,
`scalene`, and `perf`. Produce a table: overhead, what each attributed to the top three
functions, and what each *cannot* see. Then state which you would reach for in five different
situations.

**C3 — Production profiling.** Set up continuous profiling on a running service (Pyroscope,
Parca, or a cloud profiler; a local Docker setup is fine). Capture profiles for a week or
across a deploy. Produce a differential. Report anything it found that a benchmark would not
have.

**C4 — Off-CPU profiling.** Using `bpftrace` or an equivalent (Linux/WSL), produce an
off-CPU profile of a service: where does it wait, and for how long? Compare with the on-CPU
profile of the same run. Write 400 words on which one answered the question you actually had.

### Challenge

**X1.** Build a sampling profiler for Python yourself: a thread that periodically walks
`sys._current_frames()` and aggregates stacks, with configurable rate, plus flame-graph
output. Measure its overhead. Then compare its attribution with `py-spy`'s on the same
workload and explain any differences (hint: `sys._current_frames` needs the GIL, which biases
the sample toward moments when the GIL is available).

**X2.** Take a service and instrument it with OpenTelemetry to the granularity of §2.6.
Implement tail-based sampling that keeps all traces above p95 latency plus 1% of the rest.
Then analyse a week of traces: what fraction of p99 latency is attributable to each downstream,
and how does that differ from the p50 breakdown? The difference between the p50 and p99
attribution is usually the whole story and is invisible in aggregate profiles.

## 6. Self-check

1. Distinguish deterministic from sampling profilers, and give the specific bias of each.
2. Explain `tottime` versus `cumtime`, and what a high-cumtime/low-tottime function is.
3. What is on the x-axis of a flame graph? What do you look for?
4. What do `--native` and `--idle` change about what `py-spy` shows?
5. What does IPC below 1.0 suggest, and what would you investigate next?
6. Give the one measurement to take before choosing a profiler, and what its two outcomes
   imply.
7. Why does head-based trace sampling defeat the purpose, and what replaces it?
8. Name five Python-specific costs that profiles routinely reveal.

## 7. Primary sources

- Gregg, *Systems Performance*, 2nd ed., chs. 2 (methodology), 5 (applications), 6 (CPUs),
  and his flame-graph writing.
- Gregg, *BPF Performance Tools* — off-CPU analysis.
- `py-spy`, `scalene`, `yappi` documentation; `cProfile`/`pstats` docs.
- PEP 669 (low-impact monitoring) — the mechanism modern Python profilers use.
- CPython's `-X perf` / `PYTHONPERFSUPPORT` documentation.
- Berger, "Scalene: Scripting-Language Aware Profiling for Python" (2020) — the paper behind
  the tool, and a good account of why Python profiling is hard.

---

**Previous:** [L01](L01-measurement-and-statistics.md) · **Next:**
[L03 — Memory Profiling and Allocation Behaviour](L03-memory-profiling.md)
