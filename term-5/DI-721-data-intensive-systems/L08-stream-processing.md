# DI-721 · Lesson 08 — Stream Processing, Event Time, and Windows

**Estimated study time:** 5 hours
**Prerequisites:** L07; DS-701 L02 (time and causality), DS-701 L08 (idempotence)

---

## 1. Orientation

Batch processing (L07) assumes a bounded input: you can wait for all of it, and "the answer" is
well defined. Streaming removes that assumption, and the removal breaks something more fundamental
than latency.

> **In an unbounded stream, you can never know that you have seen all the data for a given period.**
> A phone that was in a tunnel will upload its events tomorrow. So "the count of events in the
> 09:00–09:05 window" is not a fact you can compute; it is a fact you must *decide when to
> approximate*, and then decide what to do when late data arrives.

Everything difficult about stream processing follows from that sentence. The Dataflow model's
contribution was to decompose the difficulty into four independent questions, and once you have
them you can reason about any streaming system:

1. **What** results are computed? (the transformations)
2. **Where** in event time are they computed? (windowing)
3. **When** in processing time are they emitted? (triggers, driven by watermarks)
4. **How** do refinements relate? (accumulation mode — discard, accumulate, or accumulate-and-retract)

Most confusion about streaming dissolves when these four are separated, because most bad designs
answer two of them with the same mechanism.

## 2. Theory

### 2.1 The two times

- **Event time**: when the event actually occurred, according to the source.
- **Processing time**: when your system observed it.

The gap between them — **skew** — is unbounded and variable: network delays, retries, mobile
devices offline for days, a backfill replaying a month of history in ten minutes, a consumer that
was down and is now catching up.

Processing-time windows are trivial to implement and almost always wrong for analytics, because
the same input produces different results depending on when it happened to arrive. That single
property makes them non-reproducible, which makes them untestable and unbacktestable. Event-time
windows give the same answer every time, which is what makes reprocessing (L07 §2.6's Kappa
argument) possible at all.

Use processing time only when the question really is about your system's behaviour ("requests
handled per second by this service"). Use event time whenever the question is about the world.

### 2.2 Windows

- **Tumbling**: fixed size, non-overlapping. Every event in exactly one window.
- **Hopping / sliding**: fixed size, fixed advance, overlapping. An event belongs to size/advance
  windows, which multiplies state.
- **Session**: gap-based — events grouped while consecutive gaps are below a threshold, with
  windows merging when a late event bridges two of them. Data-dependent, and the most expensive to
  maintain, but the right model for user activity.
- **Global**: one window, with triggers doing all the work.

The state cost is the thing to keep in view: a 24-hour sliding window advancing every minute means
1,440 open windows per key, and with a million keys that is 1.44 billion window-key entries. Window
choice is a state-size decision as much as a semantic one.

### 2.3 Watermarks

> A **watermark** at time W is an assertion: *no more events with event time < W are expected.*

It is a heuristic — a *guess*, made from observed lateness — and the two error directions matter:

- **Too aggressive** (watermark advances too early): windows close before their data arrives, and
  you get late data or lost data.
- **Too conservative** (advances too slowly): results are delayed, and state accumulates because
  windows stay open.

You cannot escape this trade-off. What you can do is make the behaviour explicit:

- **Allowed lateness**: keep window state for a period past the watermark, so late events can update
  the result (with a retraction or an updated value emitted). Costs state proportional to the
  lateness allowance.
- **Late data side output**: route events later than the allowance to a separate sink rather than
  dropping them silently. **A pipeline that silently drops late data is lying about its results**,
  and this is the single most common correctness defect in production streaming systems.
- **Percentile watermarks**: advance to the 99th percentile of observed event times rather than the
  minimum, deliberately trading a small amount of data for large latency gains — acceptable for
  monitoring, not for billing.

Watermarks propagate through a pipeline: an operator's output watermark is the minimum over its
inputs (and its own held-back state). Which yields a practical fact that surprises people: **one
idle or lagging partition holds back the entire pipeline's watermark**, so an empty partition can
stall everything. Systems provide idleness detection for exactly this reason, and it is worth
knowing before you meet it at 3 a.m.

### 2.4 Triggers and accumulation

Watermarks say when data is *complete*; **triggers** say when to *emit*, and they are separate
questions. Common triggers: at the watermark (the default); early, on a repeating processing-time
interval or after N elements (for low-latency partial results); and late, on each late element.

The typical production pattern is all three: speculative early results every minute, an
"on-time" result at the watermark, and updates for late data within the allowance.

Which makes the fourth question unavoidable — **accumulation mode**:

- **Discarding**: each pane contains only new data since the last. The consumer must sum panes.
  Cheap in state; requires an additive consumer.
- **Accumulating**: each pane is the complete result so far, superseding the previous. Requires an
  idempotent, overwrite-capable sink keyed by window.
- **Accumulating and retracting**: emits both a retraction of the previous value and the new one.
  Correct for non-idempotent downstreams and for anything that aggregates your output further; the
  most expensive.

The mistake to avoid is emitting accumulating panes into a sink that *adds* them. The count triples
and nothing errors. Match the accumulation mode to the sink's semantics, deliberately, and write
down which you chose.

### 2.5 State, and exactly-once

Stateful stream processing keeps per-key state — window contents, aggregates, join buffers,
deduplication sets. The state is often larger than any single machine's memory, so it is
partitioned by key and backed by an embedded store (RocksDB, an LSM — L02, in the place you would
least expect it).

Fault tolerance requires **consistent checkpointing**: a snapshot of the state of every operator
that corresponds to a single, consistent cut of the input. The algorithm is Chandy–Lamport
distributed snapshots, adapted: **barriers** are injected into the stream, flow through the
operators, and each operator snapshots its state when it has received the barrier on all inputs.
Recovery restores the snapshot and rewinds the source to the corresponding offsets.

Now the claim that requires care:

> **"Exactly-once" in stream processing means exactly-once *effect on managed state*, not
> exactly-once delivery.** DS-701 L01 established that exactly-once delivery is impossible.

The mechanism is: replayable source (Kafka offsets), checkpointed state, and *transactional or
idempotent* output. On recovery, the source rewinds and events are reprocessed — so the state is
restored to a consistent point, and the *output* must be either transactional (a two-phase commit
between the checkpoint and the sink, as Kafka's transactional producer provides) or idempotent
(keyed upserts).

The honest boundary: **the moment your pipeline calls an external system that is neither
transactional nor idempotent, the guarantee ends.** A sink that sends emails is at-least-once no
matter what the framework's documentation says. Knowing where the guarantee stops is the practical
skill.

### 2.6 Stream joins

Three kinds, of increasing difficulty:

**Stream–table (enrichment).** Join a stream against a slowly-changing reference table, usually
kept as local state fed by that table's changelog (L09's CDC). The subtlety is *temporal
correctness*: should an event be enriched with the table's value as of the event's time, or as of
now? If a customer's tier changed yesterday, an event from last week should probably use last
week's tier — which requires versioned state, and most implementations quietly use "now" instead.
This is a common, invisible correctness defect.

**Stream–stream (windowed join).** Two unbounded streams joined within a time bound: "the click
within 30 minutes of the impression". Requires buffering both sides for the window, so state is
bounded only by the window, and the window must be finite for the join to be implementable at all.
Watermarks determine when buffered state can be released.

**Table–table.** Joining two changelogs to produce a third; the domain of materialised views
(L09).

The general principle: **a join over unbounded inputs is only implementable if something bounds the
state** — a window, a retention policy, or a key space small enough to hold entirely.

### 2.7 Reprocessing

The Kappa argument (L07 §2.6) depends on being able to replay history through the same code and get
the same answer. Making that true requires:

- **A durable, replayable log** with sufficient retention (L09).
- **Event-time semantics**, so the results do not depend on when processing happened. This is why
  §2.1 matters so much: processing-time logic makes reprocessing meaningless.
- **Deterministic transformations**: no `now()`, no unseeded randomness, no dependence on the
  arrival order within a key beyond what event time establishes.
- **A strategy for the switchover**: run the new version in parallel into a separate output, compare
  the two, then switch consumers. The comparison step is what turns "we redeployed" into "we
  verified", and skipping it is how silent data corruption ships.

## 3. Construction: a stream processor

Build in `mpse/di721/l08/`. Kafka in Docker (from the lab setup) as the log; your own processor on
top. The point is to implement the semantics yourself before using a framework that hides them.

**Stage 1 — the source.** A producer generating events with *deliberately skewed* event times: 90%
arrive within a second, 9% within a minute, 1% up to an hour late, and a configurable trickle that
is days late. Every later stage depends on this generator being realistic, so build it first and
verify the distribution.

**Stage 2 — processing-time windows, and their failure.** Tumbling windows on arrival time.
Then replay the identical input at a different speed and show the results differ. That
irreproducibility is the argument for event time, demonstrated rather than asserted.

**Stage 3 — event-time windows.** Assign events to windows by event time; buffer; emit on a
watermark. Implement the watermark as "max observed event time − fixed delay". Show that replay at
any speed now yields identical results.

**Stage 4 — lateness.** Measure the fraction of data dropped at watermark delays of 1 s, 10 s, 1
min, 10 min, and plot completeness against latency. This curve *is* the trade-off, drawn from your
own data. Then implement allowed lateness with updated emissions and a late-data side output, and
verify that nothing is silently dropped.

**Stage 5 — triggers and accumulation.** Add early triggers (every 10 s) and late triggers. Then
implement all three accumulation modes and, for each, write a consumer that produces the correct
final answer. Then deliberately mismatch one — accumulating panes into an adding consumer — and
observe the silently wrong result. Write it up; this is the failure that ships to production most
often.

**Stage 6 — session windows.** Implement gap-based sessions including the merge case where a late
event bridges two existing sessions. Test with the interleaving that requires a merge. Measure
state size against tumbling windows on the same data.

**Stage 7 — state and checkpointing.** Back your window state with an embedded key-value store
(your L02 engine, or RocksDB via `python-rocksdb`). Implement barrier-based checkpointing and
offset-committing recovery. Kill the processor mid-stream and verify that no window result is lost
or double-counted. Measure checkpoint duration against state size, and note the pause it imposes.

**Stage 8 — exactly-once, and its boundary.** Make the sink idempotent (upsert keyed by window and
key) and demonstrate correct results across three induced failures. Then swap in a non-idempotent
sink (append to a file) and demonstrate duplicates. Write the paragraph naming precisely where the
guarantee ends and why.

**Stage 9 — stream–stream join.** Join impressions and clicks within a 30-minute event-time window.
Measure state size against window length. Then construct the temporal-correctness case in a
stream–table join: enrich events with a customer tier that changes, once using the current value and
once using the value as of event time, and show that the results differ. Decide which is correct for
a billing use case and defend it.

## 4. Failure modes

- **Processing-time windows for an event-time question.** Non-reproducible, unbackfillable results.
- **Silently dropping late data.** The results are wrong and nothing says so. Always route late data
  to a side output and monitor its volume.
- **A watermark tuned by guesswork.** Measure the lateness distribution (Stage 4) and choose.
- **One idle partition stalling the watermark.** Know about idleness detection before you need it.
- **Unbounded state.** No window, no TTL, no retention — the job runs for a month and then dies. The
  most common streaming outage.
- **Accumulation mode mismatched to the sink.** Triple-counted results, silently.
- **Believing "exactly-once" covers external side effects.** It does not.
- **Non-deterministic transformations**, which make reprocessing produce different answers than the
  original run.
- **Stream–table joins with "current" state** where event-time state was required. Invisible, and
  wrong.
- **Checkpoint interval untuned.** Too frequent and throughput suffers; too rare and recovery
  replays a long stretch.
- **No plan for schema change in a running job** (L10).

## 5. Exercises

### Warm-up (30 min)

1. Give the Dataflow model's four questions and answer them for "hourly revenue by region, results
   within a minute, corrected for data up to a day late".
2. Define a watermark and give the failure in each direction.
3. Explain why "exactly-once" is achievable for state but not for delivery, and where the boundary
   sits.

### Core (3.5 h)

4. Complete Stages 1–4 and deliver the completeness-versus-latency curve.
5. Complete Stage 5, including the deliberate accumulation-mode mismatch and its write-up.
6. Complete Stages 6–7 with the state-size and checkpoint measurements.
7. For a streaming pipeline you know (or Stage 4's): determine its watermark policy, what it does
   with late data, and whether anyone monitors the late-data volume. Write 400 words on what you
   found. If the answer is "we do not know", that is the finding.

### Challenge

8. Complete Stages 8–9, including the temporal-correctness demonstration and the defence.
9. Implement **reprocessing end to end**: change your aggregation logic, replay the full history
   into a second output, compare old and new results record by record, and produce a difference
   report distinguishing expected changes (from the logic change) from unexpected ones. Then
   implement the consumer switchover with no downtime and no double-counting. This is the exercise
   that makes the Kappa architecture real rather than theoretical, and the difference report is
   where you will find out whether your pipeline was as deterministic as you believed.

## 6. Self-check

1. Distinguish event time and processing time, and say why the choice determines reproducibility.
2. Name four window types and the state cost of each.
3. Define a watermark, state that it is a heuristic, and give both error directions.
4. What is allowed lateness and what does a late-data side output prevent?
5. Why does one idle partition stall a pipeline's watermark?
6. Give the three accumulation modes and the sink each requires.
7. Describe barrier-based checkpointing and what it snapshots.
8. State precisely what "exactly-once" means in a stream processor and where it stops.
9. Give the three join types and the thing that must bound state in each.
10. List four requirements for reprocessing to be meaningful.

## 7. Primary sources

- **Akidau et al., "The Dataflow Model" (VLDB 2015)** — the four questions; the most important
  paper in streaming.
- **Akidau, "Streaming 101" and "Streaming 102"** (O'Reilly Radar) — the same material, readable in
  an evening; read these first.
- Carbone et al., "Lightweight Asynchronous Snapshots for Distributed Dataflows" (2015) — Flink's
  checkpointing.
- Chandy & Lamport, "Distributed Snapshots: Determining Global States of Distributed Systems"
  (TOCS 1985) — the original algorithm.
- Kreps, "Questioning the Lambda Architecture" (2014) — the Kappa argument.
- Akidau, Chernyak & Lax, *Streaming Systems* — the book-length treatment; excellent.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 11.
- Sax et al., "Streams and Tables: Two Sides of the Same Coin" (BIRTE 2018) — the duality, which
  L09 develops.

---

**Previous:** [L07](L07-batch-processing.md) · **Next:**
[L09 — Event Sourcing, CDC, and the Log as Truth](L09-event-sourcing-and-cdc.md)
