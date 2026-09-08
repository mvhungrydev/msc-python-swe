# CA-731 — Problem Sets

The three problem sets build one internal platform and then subject it to the review a staff
engineer would give it. Do them in order; each depends on the last.

**A note on the deliverables.** More of this course's output is documents than in any other course
in the programme, and that is deliberate: the work of a platform architect is substantially the
work of writing things down clearly enough that other people can act on them. A design that exists
only in your head has not been designed, and a trade-off nobody recorded will be re-litigated
annually forever. The documents here — the isolation statement, the blast radius record, the error
budget policy, the design review — are the artifacts you would actually produce in the job.

**A note on the cloud account.** Everything can be done on a free tier with small resources, or
locally with LocalStack and kind. Where a part requires spending money, an alternative is given.
Set a billing alarm before you start Problem Set 1; L08 explains why, and doing it first is the
practical version of that lesson.

---

## Problem Set 1 — A Multi-Tenant Service With a Stated Isolation and Trust Model
**Covers L01–L04 · Budget: 22–26 hours**

**Part A — Reading services (L01).** Stages 1–2: three managed services read against the seven
questions with citations, the gaps in the documentation noted, and one guarantee experimentally
verified or falsified. Report what the experiment found, including if it confirmed the docs exactly.

**Part B — The dependency graph (L01).** Stages 3–4: the full graph including DNS, identity,
secrets, certificates, registry, CI and observability; every shared node marked with what happens
when it is unavailable; and the control-plane audit with a decision per recovery scenario about
whether you would pay for static stability.

**Part C — Limits and cost (L01).** Stages 5–7: the throttling experiment with your client library's
default behaviour compared against correct behaviour; the parameterised cost model with the dominant
term identified at three scales; and the availability arithmetic compared against a real incident
distribution, with the gap explained in 400 words.

**Part D — The multi-tenant service (L02).** Stages 1–3: the service, the adversarial isolation test
suite (including the 404-versus-403 question), the deliberately-introduced missing filter, and
row-level security demonstrated to contain it — plus the precise statement of what RLS does not
protect against.

**Part E — Noisy neighbours (L02).** Stages 4–6: the baseline degradation chart; the four
mitigations each measured against it; and shuffle sharding with its analytical prediction verified
empirically, with and without client retry, compared against simple sharding.

**Part F — The isolation statement (L02).** Stage 7: the one-page document a customer's security
team would receive, then interrogated by someone technical, with the unanswerable questions recorded
as design gaps. This is a required deliverable.

**Part G — Identity (L03).** Stages 1–3: the credential inventory categorised by the workload
identity hierarchy; one static credential eliminated via OIDC federation with the trust relationship
documented; and the three-layer SSRF demonstration with a statement of which layer you would rely on
if you could have only one.

**Part H — Authorisation and privilege (L03).** Stages 4, 6 and 8: the policy engine with a test
suite including the safe-looking change caught by a negative test; least privilege derived from
observed API calls with an honest account of what the technique missed; and the trust boundary
document with the boundary you had not drawn identified.

**Part I — Network (L04).** Stages 1–6: the address plan with the peering failure demonstrated;
the three-tier VPC in code; default-deny including egress with every breakage properly fixed; the
NAT/endpoint/cross-AZ cost numbers; and the five-fault connectivity debugging exercise with the
symptom-to-cause table.

**Design note (1,500–2,000 words).** The trade-off you found hardest in this set — isolation versus
multiplexing gain, or availability versus transfer cost, or least privilege versus velocity — and
how you resolved it, with the numbers. Then: the dependency you did not know you had; the thing your
credential inventory found; and the sentence you would now say to a customer asking whether their
data is isolated.

---

## Problem Set 2 — Infrastructure as Code, a Controller, and an SLO With Teeth
**Covers L05–L07 · Budget: 24–28 hours**

**Part A — The module library (L05).** Stages 1–2: three modules with justified interfaces (every
variable defended, undefendable ones deleted), root modules per environment with no environment
conditionals inside modules, and state split by blast radius with the reasoning recorded.

**Part B — The pipeline (L05).** Stages 3–4: CI with static checks, plan-on-PR and
apply-the-saved-plan; the forced-replacement visibility test with the pipeline fixed if it failed;
and testing at three levels with timings and a note of which level caught what.

**Part C — Drift and refactoring (L05).** Stages 5, 7: drift detected and handled three ways with
the decision runbook; and the module rename done wrongly and then correctly with `moved` blocks,
plus the versioning policy that treats state as part of the contract.

**Part D — The provider (L05).** Stage 6: a Terraform provider for your tenant registry with full
CRUD, import and drift detection — plus all three deliberate bugs (erroring `Read`, normalisation
producing a perpetual diff, create returning before consistency) with their operator-visible
symptoms written up.

**Part E — Recovery and blast radius (L05).** Stages 8–9: the state-loss recovery with a measured
time, the preventive controls implemented, and the per-module blast radius document with recovery
procedures and times. Required deliverable.

**Part F — Kubernetes as a control plane (L06).** Stages 1–2: the eight-step path narrated with
commands, five deliberate breakages with the symptom-to-command table, and the convergence-from-
arbitrary-state demonstration that shows level-triggering working.

**Part G — Your own controller (L06).** Stages 3–6: the `TenantEnvironment` CRD; the reconciler with
owner references, garbage collection and restart convergence; `observedGeneration` with the
stale-belief case demonstrated and fixed; and all three failure modes produced — the hot loop with
API server request rates before and after backoff, the two-controller fight, and the stuck finalizer
with its manual recovery documented.

**Part H — Admission and the alternative (L06).** Stages 7–8: the validating webhook enforcing
organisational policy, its `failurePolicy: Fail` outage demonstrated, properly scoped, with a
runbook; and the same reconciliation implemented off Kubernetes with the comparison of what you had
to build versus what you got for free. Required deliverable.

**Part I — SLOs (L07).** Stages 1–5: three SLIs with their gaming vulnerabilities named; the
historical baseline measured before the target was chosen; the dependency ceiling with a *tested*
degradation path; multi-window burn-rate alerting replayed against four incident shapes and compared
against a naive rule; and per-tenant SLIs with the global-healthy-tenant-down case demonstrated and
an alerting design that handles it.

**Part J — The budget with consequences (L07).** Stages 6–9: the price-of-a-nine table for your own
service; the error budget policy including overrides, dependencies and shared budgets; a simulated
quarter that exhausts the budget with your honest account of what you would have done; a postmortem
including the "what was lucky" section; and progressive rollout with automatic rollback, with the
error budget consumed by a bad deploy measured.

**Design note (1,500–2,000 words).** What writing a provider changed about how you read plans. What
the off-Kubernetes comparison told you about the pattern versus the product. And the answer to: for
your service, what availability target is right, what does the next one cost, and what would you
tell a product owner who asked for it?

---

## Problem Set 3 — A Costed, Reviewed Platform
**Covers L08–L10 · Budget: 22–26 hours**

**Part A — The cost model (L08).** Stages 1–3: the parameterised model validated against a real or
realistic bill with its error reported; the sensitivity analysis with the tornado chart and the
dominant term identified at expected scale and at 10×; and unit economics with the cost-per-tenant
curve, including the superlinear term identified or its absence explained.

**Part B — Capacity (L08).** Stages 4–5: the knee found by load test with utilisation at the knee
computed; Little's Law applied and checked against your configured pool sizes (report what that
comparison found); and the N+1 headroom arithmetic with its cost compared against the cost of the
outage it prevents.

**Part C — Scaling (L08).** Stages 6–7: autoscaling on CPU with all three failure scenarios
constructed; the same on concurrency or queue depth compared; and the buffered alternative with the
condition under which each approach is correct.

**Part D — Optimisation and denial of wallet (L08).** Stages 8–9: the optimisation pass with the
saving-per-hour table and risk noted per step; and the denial-of-wallet demonstration with rate
limits, spend caps, anomaly alerting and a five-minute runbook.

**Part E — The platform as product (L09).** Stages 1–3: user research with ranked pain points and an
explicit note on whether what you *wanted* to build made the top three; the golden path end to end
with time-to-first-success measured and reduced; and the self-service interface with a count of what
still requires a human.

**Part F — Errors and documentation (L09).** Stages 4–5: eight deliberate breakages with their
error messages rewritten and the independent-resolution count before and after; and all four
Diátaxis document types, with the tutorial observed being followed by someone with no prior
knowledge, fixed, and then automated as a CI test.

**Part G — Contract and support (L09).** Stages 6, 8–9: the versioning and deprecation policy with a
real deprecation executed to completion including a migration tool; the support model with a
simulated week of ten requests categorised into the product-defect backlog that follows; and the
escape hatch designed, implemented for one case, and its policy written.

**Part H — Measurement (L09).** Stage 7: time-to-first-success, DORA metrics per team, support
requests by category, platform SLOs and cost per service, with an honest answer to "is any team
better off?" — and if you cannot tell, an explanation of what measurement is missing.

**Part I — The review (L10).** Stages 1–4: the full ten-section design document for your platform;
the self-review gap list from §2.3's questions; the complete failure table including the row you
could not fill; and the pre-mortem with the three design changes that followed.

**Part J — Reviewing and being reviewed (L10).** Stages 5–9: a written review of someone else's
design with concerns ranked and labelled, plus your misunderstanding ratio; your own document
reviewed with a recorded response to every concern and a note of which you had already found; five
ADRs plus one superseded; the one-way door classification with the proportionality check; and the
one-page organisational review checklist, revised after real use.

**Design note (2,000–2,500 words).** The platform you built, assessed honestly across all four
dimensions: what it does well, what it costs, what its blast radius is, and what it would be like to
be on call for. Then the two questions this course exists to make answerable: **what did you trade
away, and what would have to be true for this to be the wrong design?**

---

## Course position paper (1,500 words)

Choose one:

1. **"Static stability is worth its cost for any system with a meaningful availability target."**
   Defend or refute using your L01 and L08 numbers.
2. **"Kubernetes is over-adopted: most organisations running it would be better served by managed
   container services and good infrastructure as code."** Argue it with reference to your L06 Stage
   8 comparison, and address the strongest counter-argument fairly.
3. **"A platform team that mandates adoption has already failed."** Use L09's material and a real
   organisation you know.
4. **"Cloud cost is an engineering responsibility that engineering has systematically declined."**
   Argue from your L08 measurements and the incentives involved.
5. **"Identity has replaced the network as the security boundary — and the network still matters
   for reasons identity cannot address."** Reconcile the two halves with reference to L03 and L04.
6. **"Availability targets above 99.9% are usually bought for reasons that are not about users."**
   Defend or refute using L07's price-of-a-nine analysis.

The structure is the one from `00-program/assessment-and-rubrics.md`: claim, grounds, the strongest
rebuttal you can construct, and the limits of your position. A paper that does not name a condition
under which its claim fails has not made a claim.

---

## Submission checklist (per set)

- [ ] Code in `courses/ca731/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does not tell
      you here.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented with a reason.
- [ ] All measurements reproducible: every cost figure states the assumptions and the date, since provider pricing moves.
- [ ] Charts and tables as files, not as descriptions — a plot referred to but not produced does not
      count.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the five-criterion rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated with everything you got wrong on the way, including the predictions
      that were incorrect.
