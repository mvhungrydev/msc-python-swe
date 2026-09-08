# PY-601 · Lesson 09 — Backpressure, Rate Limiting, and Flow Control

**Estimated study time:** 3.5 hours
**Prerequisites:** L04, L06

---

## 1. Orientation

```python
queue = asyncio.Queue()          # unbounded
```

That is an outage with a delay fuse.

If the producer is ever faster than the consumer — a traffic spike, a slow database, a GC
pause, one bad deploy — the queue grows. Memory grows. Latency grows, because every item now
waits behind a longer queue. Eventually the process is OOM-killed, and it is killed with
*all* the queued work, which was the work you were trying to protect.

**An unbounded queue does not absorb load. It converts a latency problem into a memory
problem and then into a total failure.** Backpressure is the discipline of not doing that.

## 2. Theory

### 2.1 What backpressure is

**Backpressure is a signal from a slow consumer to its producer that says: slow down.**

Without it, the only way a system can respond to overload is to accumulate — and
accumulation is unbounded, so the system fails completely rather than degrading.

The mechanism is almost always the same: **a bounded buffer**. When it is full, the producer
blocks (or is refused), which propagates the pressure upstream — to the previous stage, then
to the network socket, then to TCP's window, and finally to the client, which experiences a
slower response and, if well behaved, slows down.

```python
queue = asyncio.Queue(maxsize=100)
await queue.put(item)             # blocks when full — this is the whole idea
```

The chain matters: **backpressure only works if it propagates end to end.** One unbounded
buffer anywhere in the chain breaks it, because that buffer absorbs the pressure until it
kills the process. Auditing a pipeline for unbounded buffers — including the invisible ones
in HTTP client connection pools, logging handlers, and metrics clients — is the practical
exercise.

### 2.2 Little's Law

The single most useful formula in this area:

> **L = λ W** — items in the system = arrival rate × time in the system.

It holds for any stable system, with no assumptions about distributions. Three ways it earns
its keep:

**Sizing a pool.** If requests arrive at 500/s and each takes 40 ms, you need
`500 × 0.04 = 20` concurrent workers to keep up. Fewer and the queue grows without bound;
many more and you are just adding queueing latency and contention.

**Sizing a queue.** A queue of depth L adds `L / λ` to latency. A 1,000-item queue at
500/s adds two seconds of latency at the tail, *even when nothing is wrong*. If your SLO is
200 ms, your queue cannot be deeper than 100 items. **Queue depth is a latency budget, and
should be derived from the SLO rather than picked as a round number.**

**Diagnosing.** Measure any two of L, λ, W and derive the third. If measured L is much larger
than λW, you have a leak or a stall.

The corollary that surprises people: **a deep queue does not improve throughput.** Throughput
is set by the slowest stage. A deeper queue only smooths bursts and adds latency; past the
burst size you actually need, it is pure harm.

### 2.3 When you cannot slow the producer

Backpressure requires a producer that can be slowed. Sometimes there is none: a UDP feed, a
market data stream, a webhook sender that will time out and retry, a Kafka partition that
keeps being written.

Then you must **shed load**, deliberately:

| Strategy | Drop | Use when |
|---|---|---|
| **Reject new** | newest | Fair-ish, simple; the standard HTTP 429/503 |
| **Drop oldest** | oldest | Freshness matters more than completeness (metrics, prices, sensor data) |
| **Sample** | randomly | Statistical work; tracing |
| **Downgrade** | quality, not items | Lower resolution, cached answer, partial result |
| **Prioritize** | low-priority items | You have a real priority signal; beware starvation |

The rule: **decide the shedding policy explicitly and make it observable.** Every system
sheds load under enough pressure; the only question is whether it does so by design or by
crashing. A dropped-item counter with an alert is the difference between "we shed 3% of
telemetry for four minutes" and "the service disappeared".

Note also the **timeout/queue interaction**: an item that has been queued longer than the
client's timeout is *worthless work*. Checking the deadline on dequeue (L07 §2.3's
propagated budget) and discarding expired items is often the highest-leverage single change
under overload — it converts a death spiral into graceful degradation.

### 2.4 Rate limiting

Limiting the rate *you* impose on something else. Two algorithms cover nearly everything:

**Token bucket.** Tokens accrue at rate `r` up to capacity `b`. Each request takes one.
Allows bursts up to `b`, then settles to `r`.

```python
class TokenBucket:
    def __init__(self, rate: float, capacity: float) -> None:
        self.rate, self.capacity = rate, capacity
        self.tokens = capacity
        self.updated = time.monotonic()

    def _refill(self) -> None:
        now = time.monotonic()
        self.tokens = min(self.capacity, self.tokens + (now - self.updated) * self.rate)
        self.updated = now

    async def acquire(self, n: float = 1) -> None:
        while True:
            self._refill()
            if self.tokens >= n:
                self.tokens -= n
                return
            await asyncio.sleep((n - self.tokens) / self.rate)
```

Note `time.monotonic()`, not `time.time()`: a wall-clock adjustment must not grant or revoke
tokens.

**Leaky bucket.** Requests queue and drain at a fixed rate. No bursts; perfectly smooth
output. Right when the downstream cannot tolerate bursts at all.

**Sliding window** (log or counter) is what most API gateways implement for
"N requests per minute", because it is what the contract usually says. A fixed window has
the well-known boundary problem: 100 requests at 11:59:59 and 100 more at 12:00:01 is 200 in
two seconds while satisfying "100 per minute".

**Distributed rate limiting** is a different problem: coordinating a limit across N
instances needs shared state (Redis with a Lua script for atomicity) or per-instance
allocation (each instance gets `limit/N`, which wastes capacity when load is uneven).
Approximate approaches — each instance enforcing `limit/N × 1.2` and accepting some overage
— are usually correct in practice, and saying so explicitly beats pretending the limit is
exact.

### 2.5 Concurrency limiting versus rate limiting

Frequently conflated, and they protect against different things:

- **Rate limit** — requests per second. Protects against volume.
- **Concurrency limit** — requests in flight. Protects against *slowness*.

If a downstream slows from 50 ms to 5 s, a rate limit of 100/s lets you accumulate 500
concurrent requests — exhausting your connection pool, memory, and threads. A concurrency
limit of 20 caps the damage regardless of how slow the dependency gets.

**Concurrency limiting is the more important of the two for resilience**, and it is the one
more often missing. `asyncio.Semaphore` or a bounded pool per dependency is the mechanism —
this is the **bulkhead** of SE-521 L09 §2.6.

Adaptive concurrency limits (gradient/Vegas-style, as in Netflix's `concurrency-limits`) go
further: measure latency, and reduce the limit when latency rises, which finds the right
value automatically. Worth knowing about; a fixed, measured limit gets you most of the
benefit.

### 2.6 Queueing theory, minimally

You do not need the mathematics, but two results change how you think.

**Utilization and latency.** For an M/M/1 queue, mean waiting time is proportional to
`ρ / (1 − ρ)` where ρ is utilization. At 50% utilization, wait ≈ service time. At 90%, ≈ 9×.
At 99%, ≈ 99×. **Latency does not degrade gracefully as you approach saturation; it explodes.**

The engineering consequence: **plan for 60–70% utilization, not 95%.** The last 30% of
capacity costs you an order of magnitude in tail latency, and it is where every capacity
plan that optimizes for cost ends up.

**Variability makes it worse.** The Kingman approximation says waiting time scales with the
*sum of the squared coefficients of variation* of arrivals and service times. Bursty
arrivals or highly variable service times multiply the queueing delay at the same
utilization. This is why a p99 service time 50× the median is so damaging, and why reducing
*variance* often helps more than reducing the mean.

**Tail amplification.** A request that fans out to N backends waits for the slowest. With
N = 100 and a p99 of 1 s, roughly 63% of requests hit at least one slow backend. Dean &
Barroso's "The Tail at Scale" is the reference; the mitigations are hedged requests,
tied requests, and reducing variance at the source.

### 2.7 Where the invisible buffers are

Audit checklist for any pipeline. Each of these is a place backpressure silently stops:

- `asyncio.Queue()` / `queue.Queue()` with no `maxsize`.
- An unbounded thread or process pool work queue (`ThreadPoolExecutor`'s internal queue is
  unbounded — submitting faster than workers complete grows it without limit).
- A logging handler with a queue (`QueueHandler` with an unbounded queue), or synchronous
  logging to a slow sink.
- A metrics or tracing client buffering in memory.
- An HTTP client with unlimited connections, or a connection pool that queues acquisitions
  without limit.
- `asyncio.create_task` in a loop — every task is an unbounded work item.
- A list accumulating results before a batch write.
- Kernel socket buffers and TCP windows (bounded, but larger than you think).
- The database's own connection queue.

The exercise of finding all of them in a real service is C3, and it usually finds three or
four.

## 3. Construction: a pipeline that degrades gracefully

Build an ingest pipeline: HTTP endpoint → validation → enrichment (calls a slow upstream) →
batch write to a database. Then make it survive overload.

**Stage 1 — the naive version.** Unbounded queues everywhere, `create_task` per request,
no limits. Load-test it: ramp arrival rate until it fails. Record *how* it fails — it will be
memory growth then OOM, with latency climbing beforehand. Plot memory, latency, and
throughput against arrival rate.

**Stage 2 — bound everything.** Bounded queues sized from Little's Law and your latency SLO,
a semaphore per upstream, a bounded pool for the database. Re-run the ramp. Now the failure
mode should be: throughput plateaus, latency rises to a bound, and `put` blocks — which
propagates to the HTTP layer.

**Stage 3 — propagate to the client.** When the entry queue is full, return 503 with
`Retry-After` rather than blocking the request forever. Measure: at 2× capacity, what
fraction is served and what fraction is rejected? A correct system serves ~100% of capacity
and rejects the rest promptly; a broken one serves 40% slowly and times out the rest.

**Stage 4 — deadline checking.** Attach an arrival timestamp and a budget to each item.
Discard on dequeue if expired. Re-run at 2× capacity and compare with stage 3: goodput
should rise, because you stop spending capacity on work nobody is waiting for any more.
This is usually the largest single improvement and it is rarely implemented.

**Stage 5 — shed deliberately.** Add a load-shedding policy for the case where even the
bounded queue is full: choose from §2.3's table, implement it, count what you drop, and
alert on the rate. Justify the choice.

**Stage 6 — the utilization curve.** Plot p50/p99/p999 latency against utilization from 10%
to 99%. You should see §2.6's explosion. Mark on the chart where you would set your capacity
target, and defend it.

**Stage 7 — rate-limit the upstream.** Add a token bucket for the enrichment call, sized to
the upstream's published limit. Verify you never exceed it, including under a burst. Then
make the upstream return 429s and verify your client backs off correctly rather than
retrying into the wall (PY-501 L10 §3).

## 4. Failure modes

- **An unbounded queue anywhere.** The whole chain's backpressure stops there.
- **Deep queues chosen by round number** instead of from the latency SLO.
- **Believing a deep queue improves throughput.**
- **Blocking the client forever instead of rejecting.** A rejected request is a better
  outcome than a timed-out one, for both sides.
- **No deadline check on dequeue.** Capacity spent on work nobody wants.
- **Rate limit without concurrency limit.** A slow dependency still exhausts you.
- **`ThreadPoolExecutor.submit` in a loop** without limiting submissions.
- **`create_task` per item** with no semaphore.
- **Retrying into an overloaded dependency.** Amplifies the overload; this is how partial
  outages become total (add a circuit breaker).
- **Load shedding by crashing.** Undesigned, and it drops everything.
- **Planning for 95% utilization.**
- **`time.time()` in a rate limiter.** NTP adjustments corrupt it.
- **No metric for drops or rejections.** You cannot see the degradation you designed.

## 5. Exercises

### Warm-up (25 min)

**W1.** Build a producer/consumer with an unbounded queue and a consumer 2× slower. Plot
memory over time. Then bound the queue and plot again.

**W2.** Use Little's Law to size a pool for a workload you know. Then measure the actual
concurrency and compare.

**W3.** Show the fixed-window rate limiter boundary problem with a concrete request pattern.

### Core (2.5 h)

**C1 — The pipeline.** Complete §3, all seven stages. Deliverable: the failure-mode plots for
stages 1 and 2, the goodput comparison for stages 3 and 4, the shedding policy with its
justification and metrics, the utilization curve with a defended capacity target, and the
rate-limiter verification.

**C2 — Find the unbounded buffers.** Audit a real service against §2.7's checklist. Report
every unbounded buffer found, its potential size, and the fix. Include the ones inside
libraries — logging, metrics, and HTTP clients are where they hide.

**C3 — Rate limiter implementations.** Implement token bucket, leaky bucket, and sliding
window. Test each against: a steady rate, a burst, and a burst at a window boundary. Report
which allows what. Then implement a distributed version with Redis and measure its accuracy
and its latency cost under contention.

**C4 — Adaptive concurrency.** Implement a simple gradient-based adaptive concurrency limit:
track latency, reduce the limit when latency rises above a baseline, increase it when it does
not. Compare against a fixed limit under a dependency whose latency changes by 10× midway.
Report throughput and p99 for both.

### Challenge

**X1.** Read Dean & Barroso, "The Tail at Scale". Implement and measure three of its
mitigations — hedged requests, tied requests, and micro-partitioning with load-aware
assignment — on a fan-out workload with a realistic slow tail. Report the latency improvement
and the extra load each costs. State when each is worth it.

**X2.** Build a load-testing harness that ramps arrival rate and records throughput, goodput,
p50/p99/p999 latency, memory, and rejection rate, producing the full characteristic curve for
a service. Run it against a real service before and after adding backpressure. The harness is
reusable; the before/after chart is the artifact that gets backpressure prioritized.

## 6. Self-check

1. Define backpressure and explain why one unbounded buffer breaks the whole chain.
2. State Little's Law and give three distinct uses.
3. Why does a deeper queue not improve throughput, and what does it cost?
4. Give five load-shedding strategies and when each applies.
5. Distinguish rate limiting from concurrency limiting, and say which matters more for
   resilience and why.
6. What is the relationship between utilization and queueing latency, and what capacity
   target follows?
7. Why does variability matter as much as the mean?
8. Name six places unbounded buffers hide.

## 7. Primary sources

- Dean & Barroso, "The Tail at Scale" (CACM, 2013).
- Little, "A Proof for the Queuing Formula L = λW" (1961) — two pages.
- Gunther, *Guerrilla Capacity Planning* — the USL and utilization curves.
- Nygard, *Release It!*, 2nd ed. — the stability patterns chapters.
- Netflix Technology Blog, "Performance Under Load" (adaptive concurrency limits).
- Beyer et al., *Site Reliability Engineering*, ch. 21 (handling overload) and ch. 22
  (addressing cascading failures). Chapter 22 is the best short treatment of this subject
  anywhere.

---

**Previous:** [L08](L08-processes-and-subinterpreters.md) · **Next:**
[L10 — Testing and Debugging Concurrent Programs](L10-testing-concurrent-programs.md)
