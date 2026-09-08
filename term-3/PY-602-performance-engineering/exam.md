# PY-602 — Written Examination

**Time allowed: 3 hours. Closed book, no interpreter.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Numerical answers should be order-of-magnitude; you are being examined on the model, not on
arithmetic.

---

## Section A — answer FOUR

### Measurement and statistics (L01)

**A1.** Give Knuth's claim in full and both halves of what it actually requires — including the half
that is almost always dropped when the quotation is used. Then give the eight steps of the
performance methodology and name the two most often skipped. *(15)*

**A2.** Explain why throughput and latency are different questions, and give two changes that
improve one while worsening the other. Explain why percentiles must not be averaged and what you do
instead. *(15)*

**A3.** Explain coordinated omission: the mechanism, the conditions that produce it, and its typical
magnitude. Then name five sources of measurement variance with the mitigation for each. *(15)*

**A4.** Explain why median and median absolute deviation are preferable to mean and standard
deviation for latency data, and when reporting the minimum is defensible. Then give the eight-step
optimisation loop and the typical payoff of each step. *(15)*

### Profiling (L02)

**A5.** Distinguish deterministic from sampling profilers and give the specific bias of each.
Explain `tottime` versus `cumtime` and say what a high-cumtime/low-tottime function indicates. *(15)*

**A6.** Explain what is on the x-axis of a flame graph — being precise, since the common answer is
wrong — and what you look for in one. State what `--native` and `--idle` change about what `py-spy`
shows you. *(15)*

**A7.** State the one measurement to take before choosing a profiler and what each of its two
outcomes tells you to do. Explain why head-based trace sampling defeats the purpose and what
replaces it, and name five Python-specific costs that profiles routinely reveal. *(15)*

### Memory profiling (L03)

**A8.** State what `sys.getsizeof` measures and what it omits. Name the four consumers of a Python
process's memory and say which tools can see which. *(15)*

**A9.** Give the six-step leak diagnosis procedure and identify the step people skip. Distinguish a
peak problem from a steady-state problem and give the fix ladder for each. *(15)*

**A10.** Explain why reducing allocations is often a CPU optimisation as much as a memory one. Give
the seven-rung memory-reduction ladder in order, explain why Python-level accounting will never
equal RSS and what you do about it, and say what `gc.freeze()` does and when to call it. *(15)*

### Object costs and layout (L04)

**A11.** State what is in a CPython object header and how large it is including the GC header. Then
give the approximate size of each of: a small `int`, a 20-character ASCII string, a five-element
tuple, an empty dict, and an instance of a plain class with three attributes. *(15)*

**A12.** Explain PEP 393's compact string representation and the emoji effect it produces. State how
much `__slots__` actually saves and name four things it costs. *(15)*

**A13.** Give the order-of-magnitude cost of a local variable read, a dict lookup, a function call,
an attribute access, and an integer addition. State what dominates the cost of a hot Python loop and
what follows for where to optimise. *(15)*

**A14.** Explain array-of-structs versus struct-of-arrays and say which access patterns favour each,
with a concrete example of each. Then work through the per-record memory cost prediction for a
four-field record stored as a dict, and compare it against three alternative representations. *(15)*

### Vectorisation and NumPy (L05)

**A15.** Give the three reasons vectorised code is faster, in order of importance — and note that
the order surprises most people. Explain strides, and give five operations that produce views and
five that produce copies. *(15)*

**A16.** State the broadcasting rules and the trap they create. Explain why a Numba loop can beat
chained NumPy operations on large arrays, and why `a.sum(axis=0)` is slower than `a.sum(axis=1)` for
a C-order array. *(15)*

**A17.** Explain what is wrong with an `object`-dtype array. Give five situations where vectorisation
does not apply, and say when you should reach for a columnar query engine instead of NumPy. *(15)*

### Caches and locality (L06)

**A18.** Give the latency hierarchy from register to network in relative terms. State what a cache
line is and give the three consequences of its size. *(15)*

**A19.** Distinguish spatial, temporal and instruction locality with an example of each. Explain why
a list of Python objects has almost no spatial locality and what follows for data-heavy code. *(15)*

**A20.** State what an IPC below 1.0 tells you and what you would check next. Explain tiling and how
to choose a tile size. *(15)*

**A21.** Explain why sorting random accesses before performing them helps, and the condition under
which it does not pay. Then state the universal locality principle and give three levels of the
system at which it applies. *(15)*

### Extending Python (L07)

**A22.** Give the six extension approaches and the decision procedure for choosing between them.
Name the three things the C API requires you to get right and the failure mode of each. *(15)*

**A23.** Explain why untyped Cython is slow and what `cython -a` shows you. State what
`boundscheck(False)` and `wraparound(False)` trade away, and what PyO3's `py.allow_threads` makes
*unrepresentable* that C permits. *(15)*

**A24.** Identify the most common PyO3 performance mistake and its mechanism. Explain what the
buffer protocol lets your extension accept and why you should design for it, and give five costs of
shipping an extension beyond writing it. *(15)*

### I/O and syscalls (L08)

**A25.** State what a syscall costs and where it sits in the latency hierarchy. Name the layers
between `for line in f` and the block device, and the batching each layer performs. *(15)*

**A26.** Give four ways to read a large file, ordered by suitability, and say specifically when
`mmap` does *not* help. Distinguish buffered, flushed and fsync'd, and give the cost of the last.
*(15)*

**A27.** Explain why parallelism helps disk throughput despite the disk being one device. Explain
Nagle's algorithm interacting with delayed ACK and what it costs, why Arrow's speed is a difference
in kind rather than degree, and give the rule for when compression pays. *(15)*

### Capacity and production (L09)

**A28.** Give the four limits in a capacity model and explain why identifying the binding one comes
first. State the Universal Scalability Law and say what α and β mean physically. *(15)*

**A29.** State what utilisation you should plan for and why not 95%. Explain why variance matters as
much as the mean for queueing, and what is wrong with closed-loop load testing. *(15)*

**A30.** Give the USE and RED methods and name the metric most often missing in practice. Explain
why cost per unit of work is a good performance metric, and give four stopping criteria for
performance work. *(15)*

---

## Section B — answer ONE

**B1. (40)** A data-processing service loads 40 million records into memory, indexes them,
and answers filter-and-aggregate queries. It uses 22 GB of RAM (the instance has 32 GB),
p99 query latency is 1.8 s against a 500 ms SLO, and it runs on eight instances at 45% CPU.
Cost is $18,000/month.

Write the analysis and plan. It must include:

- Which of the four capacity limits binds, how you would establish that, and what follows for
  where to spend effort.
- A memory prediction: compute the expected bytes per record for a plausible current
  representation, and for two alternatives, showing your work.
- What you would profile and what you expect to find, distinguishing on-CPU from off-CPU.
- The specific changes, in order, with a predicted improvement for each and the reasoning.
- Which changes reduce *cost* and which merely reduce CPU with no cost effect.
- What you would measure to validate each change, and the criterion for accepting it.
- What you would deliberately not do.

Marks are for the memory arithmetic, the identification of the binding constraint, and the
distinction between changes that reduce cost and changes that do not.

**B2. (40)** You are asked to review this claim from a colleague:

> "I rewrote our scoring function in Rust. The microbenchmark shows 47× speedup — from
> 380 µs to 8 µs per call. We should merge this and I'd like to rewrite the feature
> extraction next."

Write the review. It must include:

- The questions you would ask before accepting the 47× figure, covering methodology
  (specific: what would make the microbenchmark misleading here).
- The Amdahl analysis you would require, and what data you need for it.
- The alternatives that should have been ruled out first, and what each would plausibly have
  achieved.
- The full cost side: packaging, the wheel matrix, contributor barrier, maintenance,
  free-threaded support, and debugging.
- The conditions under which you *would* approve it — there are some, and stating them fairly
  is worth marks.
- What you would want measured before the second rewrite.

An answer that simply rejects the proposal scores poorly; so does one that accepts it.

**B3. (40)** Design the performance strategy for a new service expected to grow 10× over two
years: 5,000 rps today, p99 SLO of 300 ms, a Postgres database, an object store, and one
third-party API in the request path.

Your answer must cover:

- The instrumentation you would build on day one, and why each item (be specific about
  saturation metrics).
- The load-testing approach: open versus closed loop, the workload mix, the data volume, and
  what the characteristic curve would tell you.
- The capacity model, and how you would establish the binding constraint at each growth
  stage — noting that it will change.
- The regression defences, ordered by value per unit effort, with the threshold you would set
  for a CI tripwire and why it should be loose.
- Which architectural decisions are expensive to reverse later and therefore need deciding
  now (connect to SE-521 L08 §2.5).
- The stopping criterion for performance work.
- The one number you would put on a dashboard for the leadership team, and why that one.

---

**B4. (40)** A batch job that processes a day of events has grown from 40 minutes to 6 hours over
eighteen months. Data volume grew 3×. The team has responded by moving it to a machine with four
times the CPU and eight times the RAM; runtime improved to 5 hours. CPU utilisation during the run
sits at 15%. Nobody has profiled it.

Write the investigation and the plan. Your answer must: explain what the 15% CPU utilisation rules
out immediately and what it points to; state your ranked hypotheses with the reasoning, given that
3× data produced a 9× slowdown — being specific about what superlinear growth implies; give the
measurements you would take in order, and what result would eliminate which hypothesis, aiming to
discriminate in as few measurements as possible; explain why the hardware upgrade produced so little
improvement and what that tells you; identify the two most likely root causes given the shape of the
degradation and give the mechanism for each; state the fix for each with its expected effect and its
risk; and give the stopping criterion — at what point do you declare the work finished, and why that
point.

Marks are for the discrimination and for the superlinearity reasoning, not for the list of
optimisations.

**B5. (40)** You are asked to establish the performance engineering practice for a team of twelve
that has none. Symptoms today: performance work happens only after a customer complains; there is no
benchmark suite; "it's faster" is asserted in pull requests without numbers; a previous optimisation
effort made the code substantially harder to read for a gain nobody measured; and there is
disagreement about whether the system is fast enough, which nobody can settle because there is no
target.

Write the plan. It must cover: what you would establish first and why — noting that the answer is
not a profiler; how you would set a performance target that is defensible rather than arbitrary, and
what evidence you would use; the benchmark discipline, addressing variance, coordinated omission,
and what runs in CI versus nightly; the rule for what claims a pull request may make and what
evidence they require; how you would handle the readability-versus-speed conflict as a matter of
policy rather than case by case; the production measurement you would add and which method (USE,
RED, or both) applies where; and the four stopping criteria, expressed so that a team member can
apply them without you.

Then identify which element you expect to be resented, and give the argument for it. Finally, state
what you would measure after six months to know whether the practice is working — and what result
would tell you it is not worth continuing.

---

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision, correct
edge cases, and correct arithmetic where required; 3 for an example not drawn from the
lessons; 2 for a stated limitation of your own answer or a connection to another course.

Section B: a correct, complete answer is 24/40. The rest is judgement, the quality of the
quantitative reasoning, and honesty about uncertainty.
