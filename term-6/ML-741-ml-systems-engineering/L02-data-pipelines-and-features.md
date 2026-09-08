# ML-741 · Lesson 02 — Data Pipelines, Features, and Contracts

**Estimated study time:** 4.5 hours
**Prerequisites:** L01; DI-721 L07–L10

---

## 1. Orientation

The consensus finding across every survey of deployed ML systems is that **data problems dominate**.
Not modelling problems, not serving problems — the pipeline that produces the data, the quality of
that data, and the contracts (usually absent) governing it.

This is good news for someone who has done DI-721, because the machinery is the same machinery:
batch and stream processing, schemas and compatibility, contracts and lineage, event time and
correctness. What ML adds is a set of specific requirements that ordinary data pipelines do not
have, and this lesson is about those:

- **Point-in-time correctness.** Training data must reflect what was knowable *at the time of the
  event*, or the model learns from the future and fails in production (L01 §2.3).
- **The same computation in two very different environments** — batch training and per-request
  serving — with identical results.
- **Labels**, which are a data source of their own, usually delayed, often noisy, and sometimes
  produced by the system being trained (L08).
- **Data as a versioned artifact**, because it is part of the source.

The framing claim:

> **A feature is a function of data as of a point in time. Every hard problem in this lesson comes
> from taking that definition seriously.**

## 2. Theory

### 2.1 Point-in-time correctness

The requirement: for a training example with event time *t*, every feature must be computed using
only data available at *t*.

Violating it produces **leakage**, and leakage is the most damaging bug in ML engineering because
it makes the model look *better*, not worse. The offline metric improves, the team celebrates, and
production performance is poor for reasons nobody can find.

The mechanisms, all of which occur in real pipelines:

- **Aggregates computed over the whole dataset**, including rows after *t*. `customer_total_orders`
  computed as a `GROUP BY` over the full table includes orders that had not happened yet.
- **Late-arriving corrections.** A record updated after the fact — a refund, a chargeback, a
  reclassification — where training reads the corrected value and serving sees the original.
- **The label leaking into a feature.** `account_closed_reason` as a feature for predicting churn.
  Obvious when stated, and easy to introduce through a join two steps removed.
- **Preprocessing fitted on all the data.** Scaling or imputing using statistics computed across
  train and test, so the test set influenced the transformation.
- **Splitting randomly on temporal data.** A random train/test split on time-series data trains on
  the future and tests on the past. Split by time.

The mitigations:

1. **Model everything as an event log with timestamps** (DI-721 L09), and compute features as
   as-of joins. The event-sourced representation makes point-in-time correctness natural rather
   than something you must remember.
2. **Use as-of / temporal joins** rather than plain joins: "the value of this attribute as of the
   event's timestamp", not "the current value".
3. **Fit preprocessing inside the cross-validation fold**, never outside it.
4. **Split by time**, always, for anything with a temporal structure — which is nearly everything.
5. **Be suspicious of large offline improvements.** A feature that adds ten points of AUC is
   leakage until proven otherwise. This heuristic catches more leakage than any tool.

### 2.2 Feature engineering as software

Features are code, and everything from SE-511 applies. The specific requirements:

- **A feature is a named, versioned, documented, tested transformation.** With an owner. A feature
  named `f_47` computed by a fragment of a notebook is a liability.
- **Feature definitions must be shared between training and serving** (L01 §2.3). Sharing the
  literal code is the strongest form; sharing computed values through a store is the next.
- **Features have dependencies** — on data sources, on other features — and the dependency graph
  must be explicit or you cannot reason about what breaks when a source changes.
- **Features can be removed.** Run leave-one-out evaluation periodically; unused features are pure
  cost and pure risk (L01 §2.2).

The **transformation taxonomy**, which determines where each computation can live:

- **Row-level, stateless** (`log(amount)`, `hour_of_day(ts)`): computable anywhere, identically.
  Easiest, and the ones to prefer where possible.
- **Aggregations over a window** (`count_transactions_last_7d`): require history and are where
  point-in-time correctness bites. Computed in batch for training, and either precomputed or
  computed on-line for serving.
- **Cross-entity joins** (`merchant_average_amount`): require another entity's state as of *t*.
- **Learned transformations** (encodings, embeddings, target encoding): are themselves fitted on
  data, so they are models, and they must be versioned as models and fitted inside the fold.

### 2.3 Feature stores, assessed honestly

A feature store is claimed to provide: a central registry of definitions, an **offline store** for
training (historical values, point-in-time correct), an **online store** for serving (low-latency
current values), consistency between the two, sharing across teams, and monitoring.

The honest assessment, because this is one of the most over-sold categories in the field:

**The real value is one thing: consistency between training and serving.** If a feature store gives
you one definition producing both the training rows and the serving values, it has solved L01's
worst bug class, and that alone can justify it.

**The claimed value that rarely materialises is reuse across teams.** Feature reuse requires
teams to agree on semantics, to trust each other's data quality, and to accept a shared dependency —
all organisational problems that a piece of infrastructure does not solve. Most feature stores
end up with one team's features in them.

**The costs are real**: another distributed system to operate, another failure mode in the serving
path (the online store is now in your latency budget and your availability chain), a
dual-write/consistency problem between offline and online stores, and a substantial amount of glue.

The judgement: **a feature store is worth it when you have many models sharing features, or
significant training/serving skew risk with windowed aggregations, and a team to operate it.** For
a single model with row-level features, sharing a Python module between the training and serving
code achieves the same consistency for a tiny fraction of the cost. Start there and add the store
when the pain is real — and note that this advice is the opposite of most vendor material.

### 2.4 Labels

Labels are their own data source and their own set of problems, and they are underrated as a
source of failure:

- **Delay.** Whether a loan defaults is known in months; whether a recommendation was good may be
  known in seconds. **The label delay bounds how fast you can detect a problem and how fast you can
  retrain**, and it is the number that most constrains an ML system's operational tempo. Compute it
  and state it.
- **Noise.** Human labellers disagree. Measure inter-annotator agreement; if it is low, your model's
  achievable performance is capped and you are partly measuring labelling noise.
- **Bias.** Labels reflect the process that produced them — including the previous model's decisions
  (L08). A fraud model trained on "transactions our old model flagged and an analyst confirmed"
  learns the old model's blind spots as ground truth.
- **Proxy labels.** You want "the user found this useful"; you have "the user clicked". The gap is
  where recommender systems go wrong, and the gap is a design decision that deserves to be written
  down and revisited.
- **Weak supervision** (programmatic labelling functions combined with a generative model, e.g.
  Snorkel) is the practical answer when labels are expensive — it trades label quality for volume,
  and whether that trade wins is empirical.

### 2.5 Data validation and contracts

DI-721 L10's data contracts, with ML-specific additions.

**Schema validation**: types, presence, allowed values, ranges. Cheap and catches a great deal.

**Distribution validation**: the statistical properties of the data are within expectation. This is
where ML differs — a value can be schema-valid and still wrong for training. Checks worth having:

- Per-feature summary statistics within historical bounds.
- Null rate within bounds. A feature whose null rate jumps from 2% to 40% has an upstream failure,
  and the model will happily train on it.
- Cardinality of categoricals within bounds; new categories flagged.
- Distribution distance from a reference (KS statistic, population stability index, or a divergence
  measure) above a threshold (L06 develops this).
- Cross-feature relationships that must hold.

**Volume validation**: the row count is within expectation. A pipeline that silently produces 10% of
the usual rows because an upstream partition was missing will train a model on a biased sample, and
nothing else will catch it.

**Where to enforce**: at ingestion (reject bad data before it enters), before training (refuse to
train on data that fails validation — an underused control), and at serving (per-request feature
validation, with a decision about what to do when it fails: reject, impute, or serve a fallback).

The rule that saves the most: **a pipeline that fails loudly is much better than one that produces
subtly wrong data quietly.** ML pipelines default to the latter, because most transformations
succeed on bad input.

### 2.6 Versioning data

Data is source, so it needs the properties source has: identify an exact version, reproduce a past
version, diff two versions, and record which version produced which artifact.

Approaches:

- **Immutable, partitioned storage.** Write date-partitioned, never mutate, and a version is a
  partition range. Simple, robust, and sufficient for most needs.
- **Table formats with time travel** (Iceberg, Delta — DI-721 L06 §2.5): snapshot isolation and
  querying as of a version, over object storage.
- **Content-addressed artifacts** (DVC, LakeFS): hash the data, store the hash in git, so a commit
  identifies both code and data.
- **The event log as the source** (DI-721 L09): any past state is a replay to an offset. The most
  general, and it requires the discipline of L09.

Whatever you choose, the requirement to satisfy is L03's: **given a model artifact, recover the
exact data it was trained on.** If you cannot, you cannot debug it, cannot reproduce it, and cannot
answer a regulator.

## 3. Construction: a feature pipeline that is correct

Build in `mpse/ml741/l02/`, on the DI-721 event log and stream processor. The domain — fraud
detection on a transaction stream — is chosen because it has all the hard properties: windowed
aggregations, delayed labels, and a feedback loop for L08.

**Stage 1 — the leakage demonstration.** Build a training set the naive way: features computed with
`GROUP BY` over the whole table, a random train/test split, and preprocessing fitted on everything.
Measure the offline metric. Then build it correctly — as-of joins, temporal split, preprocessing
inside the fold — and measure again. Report both numbers. The gap is your leakage, quantified, and
it will be larger than you expect.

**Stage 2 — as-of joins.** Implement point-in-time correct feature computation over your event log:
for each labelled event at time *t*, compute a windowed aggregation using only events before *t*.
Do it twice — once naively (a loop, correct and slow) and once efficiently (a sorted merge or a
window function) — and verify they agree exactly on a sample. The naive version is your oracle;
keep it as a test.

**Stage 3 — the feature module.** Define features as named, versioned, documented functions with
declared dependencies and types. Write unit tests for each. Then build the dependency graph and
render it. Then remove a data source and verify that your tooling tells you which features break.

**Stage 4 — training and serving, from one definition.** Implement the serving path so it uses
*literally the same feature code* as training. Then write the skew test: for a sample of historical
events, compute features through the training path and through the serving path and assert equality.
Run it in CI. Then deliberately break the equivalence — change one path's default for a missing
value — and confirm the test catches it.

**Stage 5 — validation.** Implement schema, distribution, null-rate, cardinality and volume checks.
Wire them into the pipeline so training *refuses to run* on failing data. Then inject six data
faults — a schema change, a distribution shift, a null-rate spike, a new category, a 90% volume
drop, and a subtly wrong unit conversion — and verify which are caught. The unit conversion is the
interesting one; design a check that catches it.

**Stage 6 — labels.** Implement label generation with a realistic delay (a chargeback arrives 30–90
days after the transaction). Then confront the consequences: compute your effective retraining
cadence given the delay; determine how much data is unlabelled at any moment; and design what the
system does about recent events for which no label exists. Write up how the label delay constrains
everything downstream.

**Stage 7 — the online store.** Add a low-latency store for serving-time features, populated by the
stream processor. Measure the added serving latency and the staleness distribution. Then handle the
failure case: what does the serving path do when the online store is unavailable or a feature is
missing? Implement and test the fallback, and state its accuracy cost.

**Stage 8 — versioning.** Make your data versioned such that a model artifact identifies its exact
training data. Demonstrate reproducing a training set from three months ago. Then diff two data
versions and report what changed — a diff tool for data is more useful than it sounds and almost
nobody has one.

**Stage 9 — the feature store decision.** Having built Stages 3–7 by hand, evaluate whether a
feature store (Feast, or a managed one) would be worth adopting for your system. Write the
assessment: what it would replace, what it would add, what it would cost to operate, and what
failure modes it introduces into the serving path. Reach a conclusion and defend it. A defensible
"no" is as good an answer as a defensible "yes".

## 4. Failure modes

- **Leakage.** Inflates offline metrics; looks like success.
- **Random splits on temporal data.** Trains on the future.
- **Preprocessing fitted outside the fold.** Test data influenced the transformation.
- **Different feature code in training and serving.** L01's worst bug class.
- **No volume check.** A pipeline producing 10% of the usual rows trains on a biased sample
  silently.
- **No null-rate check.** An upstream failure becomes a feature of all-nulls, and training
  proceeds.
- **Unversioned data.** The model cannot be reproduced, debugged or explained.
- **Label delay unmeasured.** The system's whole operational tempo is bounded by a number nobody
  computed.
- **Proxy labels unexamined.** Optimising for clicks when you wanted usefulness.
- **A feature store adopted for reuse.** The reuse is organisational and the store does not deliver
  it.
- **The online store in the serving path with no fallback.** A new availability dependency nobody
  costed.
- **Features nobody removes.** Cost and risk with no benefit.

## 5. Exercises

### Warm-up (30 min)

1. Define point-in-time correctness and give five mechanisms by which it is violated.
2. Explain why leakage is more dangerous than most bugs, and give the heuristic that catches it
   most often.
3. Give the four-way transformation taxonomy and say where each can be computed.

### Core (3.5 h)

4. Complete Stages 1–2: the leakage gap quantified, and the as-of join verified against the naive
   oracle.
5. Complete Stages 3–4: the feature module with its dependency graph, and the CI skew test with the
   deliberate break caught.
6. Complete Stage 5 with all six injected faults, including a check that catches the unit
   conversion.
7. Complete Stage 6 and write up how label delay constrains your system's operational tempo.

### Challenge

8. Complete Stages 7–9, including the fallback with its accuracy cost and the feature store
   assessment with a defended conclusion.
9. Build a **leakage detector**: given a training pipeline and its data, automatically flag
   suspicious features. Techniques to combine: features whose individual predictive power is
   implausibly high; features whose value distribution differs between labelled and unlabelled
   populations; features whose computation touches data with timestamps after the event; and
   features whose importance collapses when the split is made temporal rather than random. Validate
   it against the deliberate leakage from Stage 1 and against three leaks you introduce
   deliberately. Then report its false positive rate on clean features — and state honestly what it
   cannot detect, because leakage through a chain of joins is not always visible from the data
   alone.

## 6. Self-check

1. State the definition of a feature that generates this lesson's difficulties.
2. Give five leakage mechanisms and the mitigation for each.
3. Why must preprocessing be fitted inside the fold?
4. Give the transformation taxonomy and say which type causes point-in-time problems.
5. What is the one thing a feature store reliably delivers, and what does it usually fail to
   deliver?
6. Give five properties of labels that cause engineering problems.
7. Why does label delay bound the whole system's tempo?
8. Give five kinds of data validation and say which one catches a silent upstream failure.
9. What is the requirement that data versioning must satisfy?
10. Why is a pipeline that fails loudly better than one that produces subtly wrong data?

## 7. Primary sources

- **Polyzotis, Zinkevich, Roy, Breck & Whang, "Data Validation for Machine Learning" (MLSys 2019)**
  — the TFX data validation design; the practical reference for §2.5.
- Polyzotis, Roy, Whang & Zinkevich, "Data Management Challenges in Production Machine Learning"
  (SIGMOD 2017 tutorial).
- **Kaufman, Rosset & Perlich, "Leakage in Data Mining: Formulation, Detection, and Avoidance"
  (KDD 2011)** — the definitive treatment of leakage.
- Ratner et al., "Snorkel: Rapid Training Data Creation with Weak Supervision" (VLDB 2017).
- Huyen, *Designing Machine Learning Systems*, chapters 3–5.
- Sambasivan et al., "'Everyone wants to do the model work, not the data work'" (CHI 2021) — read
  it for the argument that data quality is under-resourced structurally, not accidentally.
- The Feast documentation, and Uber's Michelangelo and Airbnb's Zipline write-ups — the origin of
  the feature store idea, described by teams with genuinely many models.
- DI-721 L07–L10, which is the substrate this lesson sits on.

---

**Previous:** [L01](L01-ml-systems-are-software-systems.md) · **Next:**
[L03 — Reproducibility, Experiments, and Provenance](L03-reproducibility-and-experiments.md)
