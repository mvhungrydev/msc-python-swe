# ML-741 — Machine Learning Systems Engineering

**Term:** 6 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** DI-721, PY-602, SE-521
**Co-requisites:** CA-731, FM-751

---

## Driving question

> What breaks when a model becomes a dependency?

This is not a machine learning course. It will not teach you to derive backpropagation or choose an
architecture; there are excellent courses for that and this is not one of them. It assumes you can
train a model, or can obtain one, and asks the question that the modelling courses do not:

**a trained model is a component in a software system — an unusual one, with properties no other
component has — and this course is about engineering the system around it.**

What makes it unusual, and what generates every problem in this course:

- **Its behaviour is determined by data, not by code.** So the data is part of the source, and the
  usual tools of software engineering — diffs, reviews, tests, version control — apply awkwardly or
  not at all without deliberate work.
- **It is correct only statistically.** There is no test that passes or fails; there is a
  distribution of outcomes, and "working" is a threshold on a metric that itself must be chosen.
- **It degrades silently.** Code that breaks throws an exception. A model whose input distribution
  has shifted returns confident, plausible, wrong answers indefinitely, and nothing in the stack
  notices.
- **It creates feedback loops.** The model's outputs influence the world that generates its next
  training data. This is a class of bug with no analogue in ordinary software, and it is the most
  dangerous material in the course.

The organising claim, from Sculley et al.'s paper that every lesson refers back to:

> **ML systems have all the maintenance problems of ordinary software, plus an additional set that
> are specific to them and largely invisible.** The model is a small box in the middle of a large
> diagram, and the rest of the diagram is where the engineering is.

## Learning outcomes

On completion you will be able to:

1. **Identify** the specific technical debt patterns that ML systems accumulate and design against
   them.
2. **Build** data and feature pipelines with contracts, validation and lineage (with DI-721).
3. **Make** training reproducible, and explain precisely what reproducibility can and cannot mean.
4. **Design** training infrastructure and reason about its cost and failure modes.
5. **Serve** models with stated latency, throughput and cost characteristics, and apply the
   standard inference optimisations with their accuracy trade-offs measured.
6. **Detect** drift and degradation in production, and distinguish the kinds.
7. **Evaluate** models in production — offline metrics, online experiments, and the gap between
   them.
8. **Recognise** feedback loops and design systems that do not silently corrupt their own training
   data.
9. **Engineer** LLM-based systems: retrieval, evaluation, cost and latency control, and failure
   containment.
10. **Review** an ML system as a system, across reliability, cost, and the failure modes that are
    specific to models.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | ML Systems Are Software Systems (And Worse) | 4 |
| L02 | Data Pipelines, Features, and Contracts | 4.5 |
| L03 | Reproducibility, Experiments, and Provenance | 4.5 |
| L04 | Training Infrastructure | 4.5 |
| L05 | Serving and Inference Optimisation | 5 |
| L06 | Monitoring, Drift, and Silent Degradation | 5 |
| L07 | Evaluation: Offline, Online, and the Gap | 4.5 |
| L08 | Feedback Loops and Their Hazards | 4 |
| L09 | LLM Systems: Retrieval, Evaluation, and Cost | 5 |
| L10 | Reviewing an ML System | 4 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a reproducible pipeline with data contracts | 20% |
| Problem set 2 (L05–L07): serving, monitoring, and an honest evaluation | 20% |
| Problem set 3 (L08–L10): feedback loops, an LLM system, and a review | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 6 build artifact (with CA-731 and FM-751)

**A production-shaped ML system** running on the platform you built in CA-731:

- A **feature pipeline** with data contracts, validation, and lineage, sharing the CDC and streaming
  machinery from DI-721 L08–L09.
- **Reproducible training**: given a commit and a data version, the same model, with the provenance
  recorded.
- A **serving layer** with a stated latency SLO (CA-731 L07), batching, and at least two inference
  optimisations with their accuracy cost measured.
- **Monitoring** that distinguishes input drift, prediction drift and performance degradation, with
  alerts that fire on the right one.
- An **evaluation harness** with an offline metric, a shadow deployment, and one online experiment
  correctly analysed.
- A **feedback loop analysis** for the system, with at least one loop identified and mitigated.
- An **LLM component** — retrieval-augmented, with an evaluation harness, a cost model, and
  failure containment.

The point of building it on CA-731's platform is that an ML system is a workload, and most of what
goes wrong with ML systems in production goes wrong for ordinary systems reasons.

## Required reading

- **Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)** — read it
  first, and re-read it at the end of the course. It is eight pages and it is the spine.
- **Sculley et al., "Machine Learning: The High-Interest Credit Card of Technical Debt" (2014)** —
  the longer earlier version, with more detail.
- **Breck, Cai, Nielsen, Salib & Sculley, "The ML Test Score" (IEEE Big Data 2017)** — a concrete
  rubric for ML system maturity; use it on your own artifact.
- Huyen, *Designing Machine Learning Systems* — the best single practitioner book on this material.
- Polyzotis, Roy, Whang & Zinkevich, "Data Management Challenges in Production Machine Learning"
  (SIGMOD 2017) and "Data Validation for Machine Learning" (MLSys 2019).
- **Bottou et al., "Counterfactual Reasoning and Learning Systems" (JMLR 2013)** — feedback loops
  and why naive logged data misleads. Hard, and worth it.
- Kohavi, Tang & Xu, *Trustworthy Online Controlled Experiments* — for L07; the standard reference.
- Shankar et al., "Operationalizing Machine Learning: An Interview Study" (2022) — what practitioners
  actually struggle with, as opposed to what vendors say they do.

## Recommended

- Google's *Rules of Machine Learning* (Zinkevich) — 43 rules, mostly about engineering; read it
  early.
- Amershi et al., "Software Engineering for Machine Learning: A Case Study" (ICSE-SEIP 2019).
- Paleyes, Urma & Lawrence, "Challenges in Deploying Machine Learning: A Survey of Case Studies"
  (ACM Computing Surveys, 2022).
- Sambasivan et al., "'Everyone wants to do the model work, not the data work'" (CHI 2021).
- Chip Huyen's and Eugene Yan's writing on ML systems and LLM applications.
- For L09: the RAG, evaluation and prompt-engineering literature moves fast — treat the lesson's
  principles as durable and its specifics as dated, and verify current practice.

## A note on scope and honesty

Two warnings about this course's material.

First, **L09 is the most perishable content in the entire programme.** LLM tooling and practice
change on a timescale of months. The lesson is written to emphasise the engineering principles —
evaluation, cost, latency, containment — which are durable, and to flag the specifics as
provisional. Check dates on everything, and record what has moved in `appendices/errata.md`.

Second, **this course is deliberately sceptical.** A large fraction of published ML systems
advice is vendor content, and a large fraction of ML projects fail for reasons that have nothing to
do with modelling. Where the course takes a position against the industry consensus — on feature
stores, on the value of most MLOps tooling, on offline metrics — the argument is given so you can
disagree with it on the evidence.
