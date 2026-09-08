# PY-601 — Concurrency, Parallelism, and Asynchrony

**Term:** 3 · **Credits:** 15 · **Nominal hours:** 130
**Prerequisites:** PY-501, PY-502 (especially L05–L07)
**Co-requisite:** PY-602

---

## Driving question

> What does it mean for a concurrent program to be correct?

Sequential correctness is a property of a program's output given its input. Concurrent
correctness is a property of *every possible interleaving*, and there are exponentially many.
You cannot test them all, most of them never occur in development, and the ones that occur
in production do so at 3 a.m. under load.

So concurrency is the area where "it works" carries the least information, and where
reasoning — about what is shared, what is atomic, what happens-before what — has to replace
it.

## Learning outcomes

On completion you will be able to:

1. **Distinguish** concurrency from parallelism and select the right model for a workload.
2. **Explain** the GIL precisely: what it protects, what it does not, why removing it is
   hard, and what the free-threaded build changes.
3. **Reason** about a memory model: atomicity, visibility, ordering, and what CPython
   guarantees versus what your code assumes.
4. **Design** thread-safe components, choosing between locks, immutability, confinement,
   and queues, and stating the invariant each protects.
5. **Build** an event loop from primitives, and therefore explain `asyncio` rather than
   using it by rote.
6. **Apply** structured concurrency: task groups, scopes, and the correctness conditions of
   cancellation.
7. **Choose** between threads, `asyncio`, processes, and subinterpreters with a stated cost
   model.
8. **Implement** flow control: backpressure, bounded queues, and rate limiting.
9. **Test** concurrent code: stress, property-based, deterministic simulation, and model
   checking — and state what each can and cannot establish.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Concurrency, Parallelism, and Models of Correctness | 3.5 |
| L02 | Threads, the GIL, and the Free-Threaded Build | 4 |
| L03 | Shared State, Locks, and the Memory Model | 4 |
| L04 | Thread-Safe Design: Confinement, Immutability, and Queues | 3.5 |
| L05 | The Event Loop from First Principles | 4 |
| L06 | `asyncio` in Practice and Structured Concurrency | 4 |
| L07 | Cancellation, Timeouts, and Shutdown | 4 |
| L08 | Processes, Subinterpreters, and Shared Memory | 4 |
| L09 | Backpressure, Rate Limiting, and Flow Control | 3.5 |
| L10 | Testing and Debugging Concurrent Programs | 3.5 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a thread-safe component with a stated correctness argument | 20% |
| Problem set 2 (L05–L07): an event loop, then a service with correct cancellation | 20% |
| Problem set 3 (L08–L10): a parallel pipeline with flow control, tested adversarially | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 3 build artifact (with PY-602)

A high-throughput service: an async I/O-bound front end, a parallel CPU-bound stage, bounded
queues with backpressure end to end, correct shutdown, and a measured before/after
performance report with methodology (PY-602 L01).

## Required reading

- Herlihy & Shavit, *The Art of Multiprocessor Programming*, 2nd ed., chs. 1–3, 7, 9–10.
- Arpaci-Dusseau, *Operating Systems: Three Easy Pieces*, the Concurrency section (free).
- Hoare, "Communicating Sequential Processes" (CACM 1978).
- Herlihy & Wing, "Linearizability" (TOPLAS 1990).
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess
  Programs" (1979).
- PEPs 492, 525, 530, 654, 684, 703, 734.

## Recommended

- Beazley, "Understanding the Python GIL" (PyCon 2010) and "Concurrency from the Ground Up"
  (PyCon 2015).
- Nathaniel J. Smith, "Notes on structured concurrency, or: Go statement considered harmful"
  (2018). Required in spirit; read it before L06.
- `trio` documentation — the clearest exposition of structured concurrency in Python.
