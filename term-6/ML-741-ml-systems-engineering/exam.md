# ML-741 — Written Examination

**Time allowed: 3 hours. Closed book, no machine.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Where a question asks for a mechanism, give the mechanism and not the name. Where it asks about a
trade-off, name what is given up. Answers that recommend a tool without stating the problem it
solves will be marked down.

---

## Section A — answer FOUR

**A1.** Give five properties that distinguish ML systems from ordinary software, with an engineering
consequence of each. Then state which of them is responsible for the fact that ordinary monitoring
cannot detect a degraded model. *(15)*

**A2.** State the CACE principle and explain why it defeats the usual practice of isolating a change
for evaluation. Then define correction cascades and undeclared consumers, giving the mechanism by
which each accumulates and the fix for each. *(15)*

**A3.** Define training/serving skew and give its four mechanisms. Explain why label leakage is the
most damaging, and give the four mitigations, identifying which are prevention and which is
detection. *(15)*

**A4.** Give the five categories of testing an ML system needs, with an example of each. Then
explain behavioural testing, distinguishing invariance from directional tests, and say what these
catch that a held-out metric does not. *(15)*

**A5.** Give five conditions under which machine learning is the wrong tool. Then explain what a
model adds beyond capability, and why that belongs in the decision. *(15)*

**A6.** Define point-in-time correctness and give five mechanisms by which it is violated. Give the
heuristic that catches leakage most often in practice, and explain why it works. *(15)*

**A7.** Give the four-way transformation taxonomy for features and say where each can be computed.
Then state the one thing a feature store reliably delivers, the thing it usually fails to deliver,
and the costs it introduces. *(15)*

**A8.** Give five properties of labels that create engineering problems. Explain in particular why
label delay bounds the whole system's operational tempo, and what specifically it bounds. *(15)*

**A9.** Give five kinds of data validation for an ML pipeline, and say which catches a silent
upstream failure that all the others miss. Then explain why a pipeline that fails loudly is
preferable to one that produces subtly wrong data. *(15)*

**A10.** Give the four levels of the reproducibility spectrum with their relative costs, and say
which you would target for a production system and why. Then give five sources of non-determinism
in training. *(15)*

**A11.** Explain what must be recorded for provenance and give the test for whether the record is
complete. Then explain why a container image tag is insufficient and why preprocessing must be
versioned with the model. *(15)*

**A12.** Explain why a noise floor must be established before any improvement can be claimed, and
how you would measure one. Then give five failures of experiment tracking discipline. *(15)*

**A13.** Explain why random search beats grid search at equal budget. Then state the multiple
comparisons hazard in hyperparameter search, what it does to the reported metric, and what controls
it. *(15)*

**A14.** Give five bottleneck categories for a training job with a diagnostic symptom and a fix for
each. Then explain why buying a bigger accelerator is the wrong response to low utilisation. *(15)*

**A15.** Give the four forms of parallelism and the condition that selects each. Then explain why
scaling to N devices can produce a *worse* model, and the standard fix. *(15)*

**A16.** State what a training checkpoint must contain, and explain what goes wrong if the data
loader position is omitted. Then explain why checkpoint writes must be atomic, and what gates the
use of preemptible capacity for training. *(15)*

**A17.** Decompose model serving latency into its components and say which is most often dominant
and most often ignored. Then explain why cold start constrains your capacity strategy. *(15)*

**A18.** Explain dynamic batching, its two parameters, and the shape of the latency/throughput
trade-off. Then explain length-bucketed batching and continuous batching, and the problem each
solves. *(15)*

**A19.** Give five inference optimisations in the order you would attempt them, with the accuracy
cost of each. Explain why every accuracy trade must be measured on slices, and why unstructured
pruning is often a theoretical speedup. *(15)*

**A20.** Distinguish shadow deployment, canary and A/B testing by the question each answers. Say
which most teams skip, and give the requirement that all three depend on. *(15)*

**A21.** Give five failure cases a serving system must have an explicit answer for, with a
reasonable answer to each. Then explain why an untested fallback path is worse than no fallback.
*(15)*

**A22.** State the central operational fact about model failure. Then give the three different
things that can go wrong, their detection mechanisms, and explain why the asymmetry between them is
the whole difficulty of ML monitoring. *(15)*

**A23.** Define covariate shift, label shift and concept drift formally, with an example of each.
Explain why concept drift is undetectable from inputs and what follows for a monitoring design.
*(15)*

**A24.** Give four numeric drift detection methods and the multivariate technique that also
identifies which features moved. Then give three engineering problems that make drift monitoring
produce alert fatigue, and the fix for each. *(15)*

**A25.** Give four approaches to performance monitoring under label delay, with the limitation of
each. Explain specifically why unlabelled performance estimation methods fail in the case you most
need them. *(15)*

**A26.** Explain why aggregate metrics lie, and give the practical approach to slice monitoring
including the engineering problems it creates. Then give the decision tree for responding to a drift
alert. *(15)*

**A27.** Give five reasons offline and online evaluation disagree, with a remedy for each. Then
explain when calibration matters more than discrimination, and how you would measure it. *(15)*

**A28.** Explain peeking, multiple comparisons, Simpson's paradox and interference, each with the
wrong conclusion it produces and the correct method. Then explain what an A/A test validates and
what a failure of one means. *(15)*

**A29.** Explain why the trivial baseline is the most informative comparison and why it is usually
omitted. Then explain guardrail metrics and what they catch. *(15)*

**A30.** Draw the feedback loop and state the bandit feedback property. Then explain the popularity
loop and why it looks like success on every dashboard the team has. *(15)*

**A31.** Give five reasons feedback loops are hard to detect, and four signals that do indicate one.
Say which signal is most informative and why almost nobody tracks it. *(15)*

**A32.** Explain what must be true of a logging policy for off-policy evaluation to be possible, and
why this cannot be retrofitted. Compare IPS, doubly robust and direct-method estimation. *(15)*

**A33.** Explain why exploration is necessary, what it costs, and how you would justify that cost to
a stakeholder. Then give the cases where exploration is not acceptable and what follows honestly for
evaluation. *(15)*

**A34.** Explain the predictive policing feedback loop in full, and explain precisely why "the model
is just predicting what happens" is false. Then say what measurement would reveal the loop. *(15)*

**A35.** Give six properties that make an LLM an unusual dependency, mapping each to a problem
covered elsewhere in this course. Then explain why exact-match testing is impossible and what
replaces it. *(15)*

**A36.** Give the RAG components and the failure mode of each. Explain why retrieval and generation
must be evaluated separately, and what hybrid retrieval fixes that dense retrieval alone does not.
*(15)*

**A37.** Explain LLM-as-judge, its three known biases with their mitigations, and the validation it
requires before being trusted. Then explain grounding validation and why it is the most effective
mechanical control against fabrication. *(15)*

**A38.** Explain why prompt injection cannot be solved at the prompt level, and give the
architectural mitigations. Then compute the success rate of a ten-step agent at 95% per-step
reliability and state the architectural conclusion. *(15)*

**A39.** Give the nine dimensions of an ML system review, and explain why weakness in the
ML-specific ones tends to be self-concealing. Then give the question to lead with and say why
everything else depends on its answer. *(15)*

---

## Section B — answer ONE

**B1. The silent failure.** A fraud model has been in production for eight months. The service is
healthy: latency normal, error rate zero, no alerts have fired. A quarterly finance review finds
that fraud losses have risen 40% over that period. The team's dashboards show stable precision on
reviewed cases and no input drift.

Write the analysis and the remediation. Your answer must: explain how every dashboard can be healthy
while the model has degraded, naming the specific mechanism; explain why the measured precision is
misleading and what population it is computed over; identify the feedback loop involved and how it
concealed the problem; state what monitoring would have caught this and how much earlier, being
specific about the label delay; explain what the team should do *now*, in order, including what they
must not do first; and give the six changes you would make to the system, ranked, each with the
failure it prevents and its cost. *(40)*

**B2. The design.** You are asked to design a system that predicts, for each incoming support
ticket, whether it will require escalation — so that likely escalations are routed to senior staff
immediately. Volume is 50,000 tickets a day. The ground truth (did it escalate?) is known within a
week. Senior staff capacity is fixed.

Design it. Your answer must: state whether this should be ML at all and what the rule baseline is;
specify the features and their point-in-time correctness requirements; state the metric, connect it
to the business outcome, and address the asymmetric costs of the two error types; specify the
serving architecture with its latency requirement and justify whether it needs to be online;
describe the monitoring including what you would do about the one-week label delay; **identify the
feedback loop** — routing changes which tickets escalate — and state how you would detect and
mitigate it; describe the evaluation from offline through shadow to experiment; and state what
happens when the model is unavailable. *(40)*

**B3. The loop.** A recommendation system has been running for two years. Engagement metrics have
improved steadily throughout. A new analyst observes that the number of distinct items ever
recommended has fallen by 80% over the same period, and that a user survey shows declining
satisfaction.

Write the analysis and the plan. Your answer must: explain the mechanism producing both the rising
engagement and the falling satisfaction; explain why no drift detector, offline metric or A/B test
run inside the system would have caught it; state which measurements would have, and why they are
rarely implemented; explain what data the system now lacks and whether it can be recovered; design
the exploration you would introduce, with its cost, and the argument you would make to a product
owner who objects to deliberately showing worse recommendations; describe the changes to the
objective and the serving policy; and explain how you would measure whether your intervention
worked, given that the metric you were previously trusting is the one that was misleading. *(40)*

**B4. The LLM system.** A company wants an internal assistant that answers employee questions from
its documentation. Requirements: answers must be grounded in real documents with citations; response
time under three seconds; cost under a stated monthly budget; and it must not disclose documents an
employee is not authorised to see.

Design it, and design its evaluation. Your answer must: give the architecture, being specific about
chunking, retrieval and reranking; explain how document-level authorisation is enforced and why
filtering after retrieval is the wrong place; specify the evaluation — the golden set, the
deterministic checks, the judge and its validation, and the retrieval evaluation separately; state
the cost model and the levers you would use to stay in budget, ranked; describe the failure
containment including what happens when retrieval finds nothing relevant and why that case matters
most; address prompt injection through the document corpus, with what your mitigations do and do not
cover; and state what you would do when the provider changes the model underneath you, including how
you would detect it. *(40)*

**B5. The review.** You are given an ML system for review. It has: a model retrained weekly on the
last 90 days of data, automatically deployed if its held-out AUC exceeds the current model's;
features computed by a Spark job for training and reimplemented in the Java serving path;
monitoring on service latency and error rate; predictions logged without features; and an A/B test
run once at launch eighteen months ago showing a positive result. Nobody is on call for the model
specifically.

Write the review. Your answer must: work through all nine dimensions, identifying the specific
defect in each; explain in particular what is wrong with the promotion gate as described, and what a
correct gate would require; identify the feedback loop implied by weekly retraining on recent data
and what would make it dangerous; explain what the missing feature logging forecloses, permanently
or otherwise; state what the eighteen-month-old A/B test does and does not still tell you; rank your
findings by risk × likelihood ÷ cost and present the top five; identify which findings are about the
team rather than the system; and state what you would ask before writing any of this down. *(40)*

---

*Marks in Section A are awarded for mechanisms rather than names, and for naming what a trade-off
costs. In Section B, an answer that does not identify a feedback loop where one exists, or that
cannot say how a silent failure would be detected, cannot reach the upper band however sound the
rest of the design.*
