# ML-741 · Lesson 04 — Training Infrastructure

**Estimated study time:** 4.5 hours
**Prerequisites:** L03; PY-601 (concurrency), PY-602 L06 (caches), CA-731 L08 (cost)

---

## 1. Orientation

Training is a batch job (DI-721 L07) with three unusual properties: it is expensive per run, it
uses accelerators whose economics differ sharply from CPUs, and it fails in ways that waste hours
rather than seconds.

This lesson is about the engineering around it: making training jobs efficient, resumable,
schedulable and affordable. It is not about distributed training algorithms in depth — that is a
specialist topic — but you should finish able to reason about *when* to distribute, *which* form of
parallelism applies, and what it costs.

The organising observation, which is where most training infrastructure effort should go:

> **Most training jobs are not compute-bound. They are input-bound, memory-bound, or bound by the
> engineer's iteration time.** Buying a bigger GPU for a job whose data loader cannot keep it fed is
> the most common and most expensive mistake in this area.

## 2. Theory

### 2.1 Find the bottleneck first

Before optimising anything, determine what is limiting. The categories, with their diagnostics:

- **Input-bound**: the accelerator waits for data. Symptom — low GPU utilisation with high CPU
  utilisation, or a step time that does not improve when you use a faster GPU. Extremely common,
  and the fix is usually cheap: more loader workers, prefetching, a better on-disk format, caching
  decoded data, or moving augmentation onto the GPU.
- **Compute-bound**: the accelerator is saturated. Symptom — high utilisation, and step time scales
  with model FLOPs. The fixes are mixed precision, a better kernel, a bigger batch, or more devices.
- **Memory-bound**: it does not fit, or it fits only at a batch size that wastes the device. Fixes —
  gradient checkpointing (trade compute for memory), gradient accumulation (simulate a larger batch),
  mixed precision, model parallelism, or optimiser state sharding.
- **Communication-bound** (distributed): time is spent in gradient synchronisation. Fixes —
  gradient compression, overlapping communication with computation, larger batches per device, or a
  better interconnect.
- **Human-bound**: the engineer waits. Often the real bottleneck, and the one nobody measures.

The measurement discipline is PY-602's: **profile, do not guess.** The framework profilers give a
timeline showing exactly which of the above you have, and the first hour of any training
optimisation should be spent looking at one.

The number worth internalising: **accelerator utilisation.** If your GPU is at 30%, you are paying
for a GPU and using a third of it, and the fix is almost never a bigger GPU.

### 2.2 The input pipeline

Usually the first bottleneck, and usually fixable without buying anything:

- **File format matters enormously.** Millions of small files on network storage is the worst case —
  per-file latency dominates (DI-721 L06's small-files problem, again). Use sharded sequential
  formats (TFRecord, WebDataset, Parquet) with records batched into large files.
- **Overlap loading with computation.** Prefetch, so the next batch is prepared while the current
  one trains. This is PY-601's pipelining, and it is often a 2–3× win alone.
- **Parallel loading.** Multiple workers, with the count tuned — too many causes memory pressure and
  contention, and the optimum is measured, not assumed.
- **Do the work once.** Expensive deterministic preprocessing should be done ahead of time and
  cached, not repeated every epoch.
- **Augmentation placement.** CPU augmentation can starve the GPU; moving it to the GPU or using a
  library designed for throughput often removes the bottleneck entirely.
- **Storage locality.** Reading from object storage across a network for every epoch is slow and
  expensive (CA-731 L08's transfer costs). Cache to local NVMe on first epoch.

### 2.3 When and how to distribute

**Do not distribute until a single device is genuinely saturated and insufficient.** Distributed
training adds substantial complexity, new failure modes, and communication overhead; a well-tuned
single-GPU job frequently beats a badly-tuned four-GPU one.

The forms:

- **Data parallelism**: the model is replicated; each device processes a different batch shard;
  gradients are all-reduced. The default, and it works until the model does not fit on one device.
  Scaling efficiency degrades as communication grows relative to compute.
- **Model / tensor parallelism**: the model is split across devices. Necessary when the model does
  not fit, and communication-heavy, so it wants a fast interconnect.
- **Pipeline parallelism**: layers are split across devices and micro-batches flow through. Reduces
  communication relative to tensor parallelism, at the cost of pipeline bubbles.
- **Sharded data parallelism** (ZeRO / FSDP): shards optimiser state, gradients and parameters
  across data-parallel workers, dramatically reducing per-device memory for a modest communication
  increase. Usually the right first step beyond plain data parallelism.

**Batch size and learning rate interact**, and this is where distributed training silently goes
wrong: scaling to N devices multiplies the effective batch size, which changes the optimisation
problem. The linear scaling rule with warmup (Goyal et al.) is the standard practice, and large
batches often need it plus a longer schedule to reach the same accuracy. **A distributed run that
converges to a worse model is a common and under-diagnosed outcome**, and the cause is usually here
rather than in the infrastructure.

Efficiency should be measured, not assumed: **scaling efficiency** = (throughput on N devices) /
(N × throughput on one). Report it. Anything below about 70% deserves investigation before adding
more devices — you are paying for hardware that is producing communication overhead.

### 2.4 Failure and resumption

Training jobs run for hours or days, on hardware that fails, sometimes on preemptible capacity that
is *designed* to be interrupted. So:

- **Checkpoint regularly**, including the optimiser state, the learning rate schedule position, the
  epoch and step, and the data loader's position. **A checkpoint without the data position resumes
  onto a different data order**, which is a subtle reproducibility break.
- **Checkpoint atomically** — write to a temporary path and rename, or you will eventually load a
  half-written checkpoint (DI-721 L07 §2.5, again).
- **Choose the interval by arithmetic**: the expected work lost is roughly half the interval times
  the failure probability, against the checkpoint's write cost. On preemptible capacity with a
  short notice period, checkpoint frequently and on the preemption signal.
- **Make resumption the normal path**, exercised in tests. A resume path that is only used during a
  failure will fail during a failure.
- **Handle the accelerator-specific failures**: OOM (usually a batch size or a memory leak in the
  loop), NCCL timeouts in distributed runs (usually one worker died and the others are waiting
  forever — set timeouts), and silent data corruption on faulty hardware, which is rare and
  extremely confusing when it happens.

### 2.5 Scheduling and utilisation

Training jobs contend for expensive resources, so a shared cluster needs:

- **Queueing with priorities**, and **gang scheduling** for distributed jobs — a job needing eight
  GPUs must get all eight or none, or partially-scheduled jobs deadlock holding resources.
- **Preemption policy**: a higher-priority job can evict a lower-priority one, which requires the
  evicted job to checkpoint and resume.
- **Quotas per team** (CA-731 L02's multi-tenancy, applied to GPUs).
- **Utilisation monitoring**, because the dominant waste in shared GPU clusters is not scheduling
  inefficiency — it is **allocated-but-idle**: a notebook holding a GPU for three days while someone
  thinks. Measure allocated-versus-utilised, and reclaim aggressively.

The cost decisions (CA-731 L08):

- **Spot/preemptible instances** are 60–90% cheaper and fit training well *if* checkpointing and
  resumption work. This is the single largest cost lever available, and it is gated entirely on
  §2.4 being done properly.
- **Reserved capacity** for the predictable baseline, on-demand for peaks.
- **Right-size the accelerator.** The newest, largest GPU is not always the best price-performance,
  and an input-bound job on an expensive GPU is pure waste.
- **The full cost is not just the GPU-hours**: storage, data transfer, idle allocation, and the
  engineer's time all count, and engineer time is frequently the largest term.

### 2.6 Continuous and incremental training

Models decay (L06), so retraining is an ongoing operation and should be engineered as a pipeline
rather than performed as a task:

- **Triggering**: on a schedule, on a data volume threshold, or on a drift signal. Schedule is
  simplest and adequate for most; drift-triggered sounds better and requires the drift detection to
  be trustworthy (L06).
- **Cadence is bounded by label delay** (L02 §2.4). You cannot retrain more often than labels
  arrive.
- **Retrain from scratch or continue from the last checkpoint?** From scratch is more reproducible
  and avoids accumulating drift in the weights; continuing is cheaper. Note that continued training
  makes provenance a chain rather than a record, which complicates L03's requirements.
- **The gate is not optional.** A retrained model must pass evaluation (against the current
  production model, on a fresh held-out set, including slice metrics) *before* deployment.
  **Automatic retraining without an automatic gate is a mechanism for automatically deploying a bad
  model**, and it has caused real incidents.
- **Rollback** must be as automatic as the deployment.

## 3. Construction: make training fast, cheap and resumable

Build in `mpse/ml741/l04/`, on the L02/L03 pipeline. GPUs are optional — everything here can be
demonstrated on CPU with a small model, and the reasoning transfers. Where a GPU is available, use
it for the profiling stages.

**Stage 1 — profile.** Instrument your training loop and produce a timeline: data loading, forward,
backward, optimiser step, checkpointing. Determine which of §2.1's categories you are in. State the
evidence.

**Stage 2 — fix the input pipeline.** Whatever the profile says, first make the input pipeline fast:
convert to a sharded sequential format, add prefetching, parallelise loading with a tuned worker
count, and cache expensive preprocessing. Measure step time and accelerator utilisation after each
change. Report the sequence of improvements — you should find at least one that is much larger than
expected.

**Stage 3 — memory.** Find the largest batch size that fits. Then apply gradient accumulation to
simulate a larger effective batch, and gradient checkpointing to trade compute for memory. Measure
the memory/throughput trade-off curve for each and plot it.

**Stage 4 — mixed precision.** Enable it. Measure speedup, memory reduction, and — the part people
skip — **the effect on final model quality**, over multiple seeds against your L03 noise floor.
Report whether the difference is inside the noise.

**Stage 5 — checkpointing and resumption.** Implement checkpointing of model, optimiser, schedule,
epoch, step and data loader position, written atomically. Then kill the job at three different
points, resume, and verify that the resumed run produces the same model as an uninterrupted one —
within the noise floor, and ideally exactly. Then find the thing you forgot to checkpoint, because
there will be one.

**Stage 6 — preemptible training.** Run on spot/preemptible capacity (or simulate preemption with a
signal at random intervals). Handle the preemption notice by checkpointing immediately. Measure:
total wall-clock time, total cost, and work lost to preemption, against an on-demand baseline.
Report the cost saving and the wall-clock penalty, and state at what preemption rate the trade stops
being worth it.

**Stage 7 — distribution.** Run data-parallel training on two or more devices (or processes, on
CPU). Measure scaling efficiency. Then investigate whatever you lost: profile the communication,
and try overlapping it with computation. Then demonstrate the batch-size hazard: scale to N devices
without adjusting the learning rate and show the convergence difference; then apply linear scaling
with warmup and show it recovered.

**Stage 8 — the retraining pipeline.** Automate: triggered on a schedule, trains, evaluates against
the current production model on a fresh held-out set including slice metrics, and promotes only if
it passes the gate. Then deliberately feed it degraded data and verify the gate *blocks* promotion.
That test is the point of the whole stage.

**Stage 9 — cost.** Build the cost model for your training pipeline: GPU-hours, storage, transfer,
and idle allocation. Compute cost per training run and cost per point of metric improvement over
the last ten experiments. That second number is uncomfortable and is the right input to "should we
keep tuning this model or work on the data instead?"

## 4. Failure modes

- **Buying a bigger GPU for an input-bound job.** The most expensive common mistake.
- **Not profiling.** Optimising the part that is not the bottleneck.
- **Millions of small files on network storage.** Per-file latency dominates.
- **Distributing before saturating one device.** Complexity and communication overhead for nothing.
- **Scaling batch size without adjusting the learning rate.** Silently worse convergence.
- **Scaling efficiency unmeasured.** Paying for devices that produce communication overhead.
- **Checkpoints missing optimiser or data position.** Resumption is not equivalent to continuation.
- **Non-atomic checkpoint writes.** Eventually a corrupt checkpoint is loaded.
- **A resume path never exercised.** It fails when you need it.
- **Spot instances without working resumption.** Wasted work exceeds the saving.
- **Allocated-but-idle GPUs.** The dominant waste in shared clusters.
- **Automatic retraining with no evaluation gate.** Automatic deployment of bad models.
- **Continued training without provenance chaining.** L03's requirements quietly violated.

## 5. Exercises

### Warm-up (30 min)

1. Give five bottleneck categories with a diagnostic symptom and a fix for each.
2. Give the four forms of parallelism and the condition that selects each.
3. Explain why scaling to N devices can produce a worse model, and the standard fix.

### Core (3.5 h)

4. Complete Stages 1–3 and report the profile, the input-pipeline improvement sequence, and the
   memory/throughput curves.
5. Complete Stage 4 including the quality comparison against your noise floor.
6. Complete Stage 5, including the thing you forgot to checkpoint.
7. Complete Stage 8 and demonstrate the gate blocking a degraded model.

### Challenge

8. Complete Stages 6, 7 and 9: preemptible economics with the break-even preemption rate, scaling
   efficiency with the batch-size hazard demonstrated and fixed, and the cost per point of metric
   improvement.
9. Build a **training job analyser**: given a job's profile trace and its resource metrics,
   automatically classify the bottleneck, estimate the achievable speedup from each of a set of
   candidate interventions (more loader workers, mixed precision, larger batch, more devices), and
   rank them by expected improvement per unit of engineering effort and cost. Validate it against
   five jobs whose bottlenecks you determined by hand. Then report where it was wrong — automated
   bottleneck classification is genuinely hard when a job is bound by two things at once, and
   characterising that failure mode is the exercise.

## 6. Self-check

1. What is the most common cause of low accelerator utilisation, and what is the wrong response?
2. Give six input pipeline optimisations.
3. When should you distribute, and what should you tune first?
4. Compare data, tensor, pipeline and sharded data parallelism.
5. What is scaling efficiency, and what value should prompt investigation?
6. What must a checkpoint contain, and what goes wrong if the data position is omitted?
7. Why must checkpoint writes be atomic?
8. What gates the use of spot instances for training?
9. What is allocated-but-idle waste, and how do you find it?
10. Why is an evaluation gate mandatory for automated retraining?

## 7. Primary sources

- **Goyal et al., "Accurate, Large Minibatch SGD" (2017)** — the linear scaling rule and warmup.
- **Rajbhandari et al., "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models"
  (SC 2020)** — sharded data parallelism, clearly explained.
- Narayanan et al., "Efficient Large-Scale Language Model Training on GPU Clusters Using
  Megatron-LM" (SC 2021) — the combination of parallelism forms and how to reason about it.
- Micikevicius et al., "Mixed Precision Training" (ICLR 2018).
- Chen et al., "Training Deep Nets with Sublinear Memory Cost" (2016) — gradient checkpointing.
- Jeon et al., "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads"
  (USENIX ATC 2019) — where the utilisation actually goes in a real shared cluster; read it before
  designing one.
- The PyTorch profiler and FSDP documentation; the NVIDIA performance guides.
- CA-731 L08 for the cost and capacity material this lesson assumes.

---

**Previous:** [L03](L03-reproducibility-and-experiments.md) · **Next:**
[L05 — Serving and Inference Optimisation](L05-serving-and-inference.md)
