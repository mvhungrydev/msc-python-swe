# ML-741 · Lesson 05 — Serving and Inference Optimisation

**Estimated study time:** 5 hours
**Prerequisites:** L02, L04; PY-601 L09, PY-602 (all), DS-701 L09, CA-731 L07

---

## 1. Orientation

A trained model becomes a **service**, and everything you know about services applies: it has a
latency SLO, a throughput ceiling, a failure mode, a cost per request and a deployment process. The
ML-specific additions are that its work is expensive and roughly constant per request, its cost is
dominated by an accelerator you are paying for whether or not it is busy, and there are several
optimisations available that trade accuracy for speed — which means **accuracy becomes a tuning
parameter of the serving system**, and that is unlike anything else in engineering.

The two claims the lesson is built on:

> **Batching is the single most important lever in model serving.** Accelerators are throughput
> devices; processing one request at a time wastes most of the hardware. But batching adds latency,
> so the design of the batching policy *is* the design of the latency/throughput/cost trade-off.

> **Every inference optimisation is an accuracy trade, and the trade must be measured rather than
> assumed.** "Quantisation costs about 1%" is not a measurement of your model on your data.

## 2. Theory

### 2.1 The serving shapes

- **Online, synchronous**: a request arrives, a prediction is returned, a user waits. Latency is
  the binding constraint.
- **Online, asynchronous**: the request is queued and the result delivered later. Converts a latency
  problem into a throughput problem, and is often available when people think it is not.
- **Streaming**: predictions on an event stream (DI-721 L08). Throughput-bound with a freshness
  requirement.
- **Batch / offline**: predictions computed on a schedule and stored for lookup. **The cheapest and
  simplest option by a wide margin, and the most under-used.** If the prediction depends only on
  data that changes daily, compute it daily and serve it from a key-value store — the serving
  problem disappears entirely.

The design question to ask first, before any optimisation: **does this need to be online at all?**
A large fraction of models serving live traffic are computing predictions that would be identical if
computed overnight.

### 2.2 Latency, decomposed

Total latency is not model time. Measure each part; the surprise is usually not where people expect:

1. Network and TLS to the service.
2. Request deserialisation and validation.
3. **Feature retrieval** — the online store lookup (L02 §2.3), often a network round trip, and
   frequently the largest term.
4. Preprocessing.
5. **Model inference.**
6. Post-processing (thresholds, business rules, formatting).
7. Response serialisation.

Two consequences: **optimising step 5 when step 3 dominates is wasted work** (PY-602's lesson,
again), and each step is a failure mode and a dependency in your availability chain (CA-731 L01
§2.6).

Two more properties specific to model serving:

- **Cold start.** Loading a model into memory (and onto an accelerator) takes seconds to minutes.
  This determines how fast you can scale, which determines whether autoscaling is a viable
  availability strategy (CA-731 L08 §2.4) — usually it is not, for models.
- **Tail latency.** Garbage collection, memory allocation, batch formation waits, and accelerator
  contention all produce a heavy tail. Measure the distribution, not the mean (PY-602 L01), and
  remember DS-701 L09's fan-out arithmetic if the model is called by a service that calls many.

### 2.3 Batching

Accelerators achieve high throughput by parallelising across a batch. Serving one request at a time
leaves most of the device idle, so **dynamic batching** — accumulate requests for a short window,
run them together, distribute the results — is the primary optimisation.

The parameters:

- **Maximum batch size**: bounded by memory and by the latency the largest batch costs.
- **Maximum wait time**: how long to wait for a batch to fill before running a partial one. This is
  a direct latency/throughput dial and it is where the design lives.

The shape of the trade-off: throughput rises steeply with batch size and then flattens (the device
saturates); latency rises roughly linearly with the wait time and then with batch execution time.
**The right operating point is the largest batch whose total latency still meets the SLO**, and
finding it is an empirical exercise you should do rather than guess.

Refinements that matter in practice:

- **Adaptive batching**: shorten the wait under low load (where waiting buys nothing) and lengthen
  it under high load (where batches fill quickly anyway).
- **Padding waste** for variable-length inputs: a batch is as slow as its longest member, so
  **length-bucketed batching** — grouping similar lengths — can be a large win for text models.
- **Continuous batching** for autoregressive generation (L09): rather than waiting for every
  sequence in a batch to finish, evict completed sequences and admit new ones each step. This is
  a several-fold throughput improvement for LLM serving and is now standard.
- **Priority classes**: interactive requests bypass or shorten batching; bulk requests batch
  aggressively.

### 2.4 Model optimisation, and its accuracy cost

Each of these trades accuracy for speed or memory. Each must be measured on **your** model and
**your** data, and each must be evaluated on **slices** — an optimisation that costs 1% overall may
cost 15% on a minority segment, and aggregate metrics hide that completely.

**Quantisation.** Represent weights (and often activations) in fewer bits: FP32 → FP16/BF16 → INT8
→ INT4. Roughly linear memory reduction and often superlinear speedup on hardware with dedicated
low-precision units.

- *Post-training quantisation* is easy, requires a small calibration set, and costs some accuracy.
- *Quantisation-aware training* simulates quantisation during training and recovers most of the
  loss, at the cost of retraining.
- FP16/BF16 is usually nearly free. INT8 usually costs a little. INT4 usually costs a lot, and
  "usually" is doing a lot of work in all three statements — measure.

**Pruning.** Remove weights. *Unstructured* pruning removes individual weights and gives sparsity
that most hardware cannot exploit, so the speedup is often theoretical. *Structured* pruning removes
whole channels or heads and gives real speedup at a higher accuracy cost. The practical lesson:
**check that your hardware and runtime actually exploit the sparsity you created**, because a
"90% sparse" model that runs at the same speed is a wasted week.

**Distillation.** Train a small student to imitate a large teacher, learning from its output
distribution rather than only from hard labels. Often the best accuracy-per-FLOP available, and it
costs a full training cycle.

**Compilation and graph optimisation.** Operator fusion, constant folding, layout optimisation,
kernel selection (TensorRT, ONNX Runtime, torch.compile, XLA). Frequently 2–5× for no accuracy cost
at all, which makes this the first thing to try. The costs are build time, a more complex
deployment, and occasional numerical differences that must be verified (L03 §2.6).

**Caching.** If inputs repeat, cache predictions. Trivially effective when the input distribution is
skewed, which it usually is. Requires a cache key that is exactly the model's input and an
invalidation policy tied to the model version — a stale cache serving a previous model's predictions
is a real and confusing bug.

The order to try them: **compilation first (free), then batching (free, costs latency), then
precision reduction (small cost), then distillation (large effort), then architectural change.**

### 2.5 Deployment patterns

Deploying a model is a statistical decision, so the release process differs from ordinary software:

- **Shadow deployment**: the new model receives real traffic in parallel; its predictions are logged
  and compared but not used. The safest way to evaluate on production traffic, with zero user risk,
  and it catches the failures offline evaluation misses — feature availability, latency, skew. **It
  should be the default first step for any model change.**
- **Canary**: a small percentage of real traffic uses the new model, with automatic rollback on SLI
  regression (CA-731 L07).
- **A/B test**: a proper randomised experiment measuring the business metric (L07). This is the only
  design that tells you whether the model is *better*, as opposed to *working*.
- **Multi-armed bandit**: dynamically shifts traffic toward the better arm. Efficient when the
  metric is fast and you care about regret during the experiment; complicates the statistics
  (L07).
- **Blue/green**: both versions ready, switch traffic instantly, switch back instantly. The most
  valuable property is the rollback speed.

The requirement underlying all of them: **you must be able to serve the previous model immediately.**
Keep the artifact, keep the environment, and test the rollback (L03 §2.3).

### 2.6 The serving architecture

- **Embedded in the application**: lowest latency, no network hop, but the model's memory and CPU
  are the application's, and every model update is an application deploy. Good for small models.
- **A dedicated model service**: scaled independently, updated independently, and reusable across
  callers. The default for anything non-trivial, at the cost of a network hop.
- **A managed inference endpoint**: the provider handles scaling and serving. Convenient, more
  expensive per request, and less controllable.
- **On-device**: no server cost, no network latency, full privacy — and update latency measured in
  app-release cycles, plus heterogeneous hardware.

**Multi-model serving** matters at scale: hundreds of small models (per-tenant, per-region) served
from a shared fleet, loading and evicting from device memory on demand. This is a caching problem
with a very expensive miss (cold start), and the standard tricks apply — plus one that does not
generalise: pin the models whose latency SLO cannot absorb a load.

**Hardware choice** is a cost decision (CA-731 L08). GPUs are not automatic: for small models,
low batch sizes, or latency-critical single-request inference, a CPU is frequently cheaper *and*
faster once you account for transfer overhead. Measure both before assuming.

### 2.7 Failure behaviour

What does the service do when things go wrong? These need explicit answers, decided in advance:

- **A feature is unavailable.** Impute, use a default, serve a fallback model, or reject? Each is a
  decision with an accuracy cost that should be measured (L02 Stage 7).
- **The model service is down.** A fallback to a simpler model, to a cached prediction, or to a
  rule-based default. **Note DS-701 L09's warning about fallbacks: an untested fallback path will
  not work**, so exercise it deliberately and regularly.
- **Inference times out.** Return a default, and make sure the timeout is inside the caller's budget
  (DS-701 L09 §2.2).
- **The input is out of distribution.** Ideally detect and flag it (L06), because this is where the
  silent-failure property bites hardest.
- **The prediction is absurd.** Bound the output. A model that can output a price of $10⁹ should be
  clamped, and the clamp should be alerted on rather than silent.

The general principle: **a model is an unreliable component that returns confident answers, so the
system around it must contain the consequences.** Post-processing rules, output bounds, and
fallbacks are not admissions of failure; they are the engineering that makes a statistical component
safe to depend on.

## 3. Construction: serve it properly

Build in `mpse/ml741/l05/`, on your L02–L04 model, deployed on the CA-731 platform.

**Stage 1 — the latency decomposition.** Instrument all seven steps of §2.2 and produce the
breakdown at p50, p95 and p99. Identify the dominant term. If it is not model inference — and it
often is not — say so prominently, because everything that follows should be prioritised by it.

**Stage 2 — the baseline.** Establish throughput and latency for single-request serving. Measure
accelerator (or CPU) utilisation. Compute cost per thousand predictions.

**Stage 3 — dynamic batching.** Implement it with configurable maximum batch size and wait time.
Then sweep both parameters and produce the surface: throughput and p99 latency as a function of
each. Mark the largest batch that meets a chosen SLO. This chart is the lesson's central
deliverable.

**Stage 4 — adaptive and bucketed batching.** Implement load-adaptive wait times and, for a
variable-length input model, length-bucketed batching. Measure the improvement over fixed batching
under a realistic bursty load profile. Report the padding waste you eliminated.

**Stage 5 — compilation.** Apply graph compilation (torch.compile, ONNX Runtime, or TensorRT).
Measure speedup and verify numerical equivalence on a real sample — report the maximum discrepancy
and whether any prediction changed class. This is the free win; quantify it before doing anything
that costs accuracy.

**Stage 6 — quantisation, measured properly.** Apply FP16 and INT8 post-training quantisation.
For each, measure: latency, throughput, memory, and accuracy **overall and on at least four slices**.
Produce the table. Then find the slice where the accuracy cost is worst, and decide whether the
optimisation is acceptable — with the decision written down, because this is exactly the kind of
trade that gets made implicitly and discovered later.

**Stage 7 — distillation.** Train a smaller student model from your production model. Compare
accuracy, latency and cost per prediction against both the teacher and a same-size model trained
directly on the labels. That third comparison is the one that tells you whether distillation
actually earned its complexity.

**Stage 8 — caching.** Measure your input distribution's repeat rate. Implement a prediction cache
keyed on the exact model input and the model version. Measure the hit rate and the cost saving.
Then construct the stale-cache bug: deploy a new model without invalidating, and show the old
predictions still being served.

**Stage 9 — deployment.** Implement shadow deployment: route a copy of production traffic to a new
model version, log both predictions, and produce a comparison report (agreement rate, disagreement
analysis, latency comparison, feature availability differences). Then implement canary with
automatic rollback on SLI regression, and demonstrate a rollback triggered by a deliberately worse
model.

**Stage 10 — failure behaviour.** Implement and test every case in §2.7: missing feature, model
service down, inference timeout, and absurd output. Measure the accuracy cost of each fallback path.
Then add the fallback exercises to your regular chaos runs (DS-701 L10), because that is the only
way they stay working.

## 4. Failure modes

- **Optimising inference when feature retrieval dominates.** Measure first.
- **Serving online what could be computed in batch.** The most avoidable complexity in ML serving.
- **No batching.** Most of an expensive accelerator sitting idle.
- **Batching with a fixed wait under variable load.** Latency penalty at low load for no benefit.
- **Padding waste on variable-length inputs.** The batch runs at the speed of its longest member.
- **Quantisation evaluated on aggregate accuracy only.** The minority slice regression is invisible.
- **Pruning without checking hardware support.** Theoretical sparsity, real effort, no speedup.
- **Autoscaling as the capacity strategy with a slow cold start.** The scaling arrives after the
  spike.
- **Prediction cache not keyed on model version.** The previous model keeps serving after a deploy.
- **No shadow deployment.** Offline evaluation missed the production-only failure.
- **No fast rollback.** Discovered during an incident.
- **Unbounded model outputs.** An absurd prediction propagates into a business decision.
- **Untested fallback paths.** They do not work when needed.

## 5. Exercises

### Warm-up (30 min)

1. Give the seven components of serving latency and say which is most often underestimated.
2. Explain the batching trade-off and how the maximum wait time controls it.
3. Give five inference optimisations in the order you would try them, with the accuracy cost of
   each.

### Core (3.5 h)

4. Complete Stages 1–3 and deliver the latency decomposition and the batching surface with the SLO
   operating point marked.
5. Complete Stages 5–6: the compilation speedup with numerical verification, and the quantisation
   table including slices with a written decision.
6. Complete Stage 8, including the stale-cache bug demonstrated.
7. Complete Stage 9: shadow deployment with a comparison report, and a canary rollback triggered
   automatically.

### Challenge

8. Complete Stages 4, 7 and 10 — adaptive and bucketed batching under bursty load, distillation
   with the three-way comparison, and every failure behaviour implemented, measured and added to
   chaos runs.
9. Build an **inference cost/latency optimiser**: given a model, a target latency SLO and a traffic
   profile, automatically evaluate the configuration space — batch size, wait time, precision,
   compilation, hardware type, replica count — measure accuracy on slices for each accuracy-affecting
   option, and output the Pareto frontier of cost against accuracy at the given SLO. Then pick the
   recommended point and justify it. Finally, run the recommended configuration under a realistic
   load and report how far the prediction was from reality — a configuration search that has not
   been validated against a real deployment is a simulation, and characterising its error is what
   makes it usable.

## 6. Self-check

1. Give the four serving shapes and the question to ask before choosing an online one.
2. Decompose serving latency and say which term is most often dominant.
3. Why is cold start a constraint on your capacity strategy?
4. Explain dynamic batching's two parameters and the shape of the trade-off.
5. What is continuous batching and why does it matter for generative models?
6. Give five model optimisations with their accuracy costs and the order to try them.
7. Why must optimisation accuracy be measured on slices?
8. Why is unstructured pruning often a theoretical speedup?
9. Distinguish shadow, canary and A/B deployment, and say what each tells you that the others do
   not.
10. Give five failure cases a serving system must have an explicit answer for.

## 7. Primary sources

- **Crankshaw et al., "Clipper: A Low-Latency Online Prediction Serving System" (NSDI 2017)** — the
  batching, caching and model-selection design, clearly reasoned.
- **Olston et al., "TensorFlow-Serving: Flexible, High-Performance ML Serving" (2017).**
- Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models"
  (OSDI 2022) — continuous batching.
- Jacob et al., "Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only
  Inference" (CVPR 2018).
- Hinton, Vinyals & Dean, "Distilling the Knowledge in a Neural Network" (2015).
- Blalock et al., "What is the State of Neural Network Pruning?" (MLSys 2020) — read this before
  believing any pruning claim, including your own.
- Dean & Barroso, "The Tail at Scale" (CACM 2013) — with DS-701 L09.
- The NVIDIA Triton, ONNX Runtime and vLLM documentation, read for the mechanisms rather than the
  APIs.

---

**Previous:** [L04](L04-training-infrastructure.md) · **Next:**
[L06 — Monitoring, Drift, and Silent Degradation](L06-monitoring-and-drift.md)
