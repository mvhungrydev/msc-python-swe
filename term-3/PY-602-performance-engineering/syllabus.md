# PY-602 — Performance Engineering & the Systems Interface

**Term:** 3 · **Credits:** 15 · **Nominal hours:** 125
**Prerequisites:** PY-501 (especially L08, L09), PY-601
**Co-requisite:** PY-601

---

## Driving question

> Where does the time actually go?

Almost nobody knows. Engineers optimize what they *believe* is slow, and the belief is wrong
often enough that the profession has a proverb about it. The discipline of performance
engineering is the discipline of replacing belief with measurement — and then of knowing
which measurements mean anything.

This course is as much about *method* as about techniques. Half of it is measurement:
statistics, profiling, attribution, and the traps that make numbers lie. The other half is
the mechanism: what a Python object costs, what a cache line is, what a syscall costs, and
what changes when you drop to C or Rust.

## Learning outcomes

On completion you will be able to:

1. **Design** a benchmark that measures what you intend, with a defensible statistical
   treatment and stated confounds.
2. **Profile** with the right tool — deterministic, sampling, memory, or system-level — and
   explain what each can and cannot attribute.
3. **Read** a flame graph and a `perf` profile and locate the cost.
4. **Quantify** CPython's object overheads and predict the memory profile of a data
   structure before building it.
5. **Reason** about the memory hierarchy: cache lines, locality, prefetching, and why data
   layout dominates instruction count.
6. **Use** NumPy's memory model deliberately — views, strides, copies, and dtype — and
   predict when an operation allocates.
7. **Extend** Python in C, Cython, or Rust, with a measured justification and correct GIL
   handling.
8. **Analyse** I/O costs: syscalls, buffering, zero-copy, and the difference between
   throughput and latency optimization.
9. **Model** capacity: utilization, queueing, and what a benchmark number means for
   production.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Measurement: Methodology and Statistics | 4 |
| L02 | Profiling: Deterministic, Sampling, and Attribution | 4 |
| L03 | Memory Profiling and Allocation Behaviour | 3.5 |
| L04 | CPython Object Costs and Data Layout | 4 |
| L05 | Vectorization and NumPy's Memory Model | 4 |
| L06 | Caches, Locality, and Data-Oriented Design | 4 |
| L07 | Extending Python: C API, Cython, and Rust | 4 |
| L08 | I/O, Syscalls, and Zero-Copy | 3.5 |
| L09 | Capacity, Queueing, and Performance in Production | 3.5 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L03): a rigorous optimization study, end to end | 20% |
| Problem set 2 (L04–L06): a data-structure redesign with predicted and measured results | 20% |
| Problem set 3 (L07–L09): an extension module and a production capacity model | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 3 build artifact (shared with PY-601)

The high-throughput service, with a **measured** before/after performance report: full
methodology, the profile that identified the bottleneck, the change, the re-measurement, and
an honest account of what did not work.

## Required reading

- Bryant & O'Hallaron, *Computer Systems: A Programmer's Perspective*, 3rd ed., chs. 5–6.
- Gregg, *Systems Performance*, 2nd ed., chs. 1–2 (methodology) and 6 (CPUs).
- Drepper, "What Every Programmer Should Know About Memory" (2007), parts 1–3.
- Knuth, "Structured Programming with go to Statements" (1974) — the passage around
  "premature optimization", in context.
- Amdahl (1967); Gustafson (1988); Dean & Barroso, "The Tail at Scale" (2013).
- PEP 659 (specializing adaptive interpreter), PEP 744 (JIT), PEP 3118 (buffer protocol).

## Recommended

- Gregg, *BPF Performance Tools*.
- Gorelick & Ozsvald, *High Performance Python*, 2nd ed.
- Fog, *Optimizing Software in C++* / the microarchitecture manuals — for L06.
- Mytkowicz et al., "Producing Wrong Data Without Doing Anything Obviously Wrong!"
  (ASPLOS 2009). Required for L01; it will change how you benchmark.

## A note on version sensitivity

Performance characteristics change between CPython releases more than semantics do. Every
number in this course is illustrative. **Run the measurement yourself**, record your
interpreter build and machine, and treat any figure you did not produce as a hypothesis.
