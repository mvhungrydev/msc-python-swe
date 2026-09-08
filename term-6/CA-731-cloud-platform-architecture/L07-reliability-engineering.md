# CA-731 · Lesson 07 — Reliability Engineering: SLOs, Error Budgets, and Architecture

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02; DS-701 L09, DS-701 L10

---

## 1. Orientation

DS-701 L10 introduced SLOs and burn-rate alerting as an observability technique. This lesson takes
them seriously as an **architecture and organisation** technique, which is what they were actually
invented for.

The core claim, and it is a claim about organisations as much as systems:

> **"How reliable should this be?" is a question with a cost, and answering it with a number
> converts an argument into a measurement.** Without a target, reliability is negotiated
> case-by-case by whoever is most persuasive in the room. With one, the question becomes "are we
> within budget?", which has an answer.

The second claim, which is where most SLO adoptions fail:

> **An SLO with no consequence is a dashboard.** The error budget only does its job when exhausting
> it *changes what the team does* — and that change has to be agreed before it is needed, because
> nobody agrees to slow down in the middle of a quarter.

And the third, which is this lesson's specific contribution beyond DS-701:

> **Availability targets are architecture decisions with prices.** Going from 99.9% to 99.99% is
> not a matter of trying harder; it changes what you must build, what it costs, and what you must
> stop doing. Being able to price that step is a senior engineering skill.

## 2. Theory

### 2.1 SLI, SLO, SLA

- **SLI** (indicator): a measured ratio, `good events / valid events`. Not a raw count, not an
  average — a ratio, because a ratio is comparable across load.
- **SLO** (objective): a target for the SLI over a window. "99.9% of valid requests succeed over
  28 days."
- **SLA** (agreement): a contractual commitment with financial consequences. Always looser than the
  internal SLO, because you want to know you are in trouble before your customers get credits.

Choosing an SLI well is most of the work, and the criteria are:

1. **It measures what users experience.** CPU utilisation is not an SLI. "Requests that returned a
   non-5xx within 500 ms" is closer. "The user's page rendered" is closer still.
2. **It is measured where users are**, or as close as you can get. Server-side metrics miss
   everything between your load balancer and the user — DNS, network, the CDN, the client — which
   is often where the outage is.
3. **`valid` is defined carefully.** Health checks, bots, and requests rejected for malformed input
   are usually excluded. Every exclusion is a place to accidentally hide a real failure, so write
   them down.
4. **It is not gameable.** An SLI that improves when you shed load harder is measuring the wrong
   thing.

The common SLI shapes: **availability** (success ratio), **latency** (proportion of requests faster
than a threshold — note that this is a ratio, not a percentile, which makes it aggregatable and is
why it is preferred), **quality** (proportion served without degradation), **freshness** (proportion
of data newer than X), and **correctness** (proportion of records that pass a validation).

### 2.2 Error budgets, and the mechanism that makes them work

`error budget = 1 − SLO`. A 99.9% SLO over 28 days permits about 40 minutes of failure.

The reframing that makes this powerful: **the budget is not a limit to avoid, it is a resource to
spend.** Unspent budget means you were more reliable than you needed to be, which means you
overinvested in reliability relative to features — a real cost, not a virtue.

The mechanism, which must be agreed in advance:

- **Budget remaining**: ship features, take risks, deploy frequently, run chaos experiments.
- **Budget exhausted**: a **feature freeze** on that service until it is back in budget. Work goes
  to reliability. No negotiation, because the negotiation happened when the SLO was set.

That policy is the entire point, and it is where adoption usually fails: teams define SLOs, build
dashboards, exhaust the budget, and ship anyway. At that moment the SLO becomes decoration.

The policy needs real detail to survive contact with a business: who can override it (someone
senior, and the override is logged and reviewed); what happens with a *shared* budget across teams;
and what happens when the budget is consumed by a *dependency's* failure rather than your own — the
usual answer being that it still counts, because your users experienced it, and if that feels
unfair the correct response is to reduce your dependence or negotiate the dependency's SLO, both of
which are the intended outcomes.

### 2.3 Choosing the target

The number should come from evidence, not aspiration:

1. **What do users actually notice?** Below some level of reliability, additional nines are
   invisible against the background of the user's own network and device. For a consumer mobile
   app, the client's connection is less reliable than four nines, so four nines of server
   availability is invisible.
2. **What did you deliver historically?** The past twelve months' measured availability is the
   honest starting point. An SLO far above it is a promise to do work nobody has planned.
3. **What does each nine cost?** §2.4.
4. **What does the dependency chain permit?** (L01 §2.6.) If you depend serially on a 99.9%
   service, you cannot promise 99.95%, and a target above your ceiling is a fiction.

Start with a target you can meet, watch it for a quarter, and tighten it deliberately. An SLO that
is breached every month is not an SLO; it is a source of alert fatigue and cynicism.

### 2.4 The price of a nine

The step from one target to the next is an architecture change, and here is roughly what each buys
and costs:

| Target | Monthly downtime | Typically requires |
|---|---|---|
| 99% | 7.2 h | A single instance, backups, business-hours response |
| 99.9% | 43 min | Redundant instances, health-checked LB, automated deploys with rollback, on-call |
| 99.95% | 22 min | Multi-AZ, no single points of failure, tested failover, staged rollouts |
| 99.99% | 4.3 min | Static stability (L01 §2.3), automated failure response, no human in the recovery path, extensive testing including chaos |
| 99.999% | 26 s | Multi-region active-active, and near-total elimination of change-induced risk |

Three things this table is really saying:

- **Beyond about 99.9%, humans are too slow to be in the recovery path.** Four nines means 4.3
  minutes a month total; a human cannot be paged, wake up, orient and act inside that. So four nines
  requires automated recovery, which requires you to have anticipated the failure — which is
  expensive and only partially achievable.
- **Most downtime is caused by change.** Which means high targets constrain how you deploy far more
  than what hardware you buy: staged rollouts, canaries, automatic rollback on SLI regression, and
  smaller changes.
- **The dependency ceiling binds.** You cannot exceed the reliability of what you depend on unless
  you can operate degraded without it — which is a design decision (graceful degradation, caching,
  fallbacks) with its own costs and its own hazards (fallback paths are rarely exercised and
  therefore rarely work, which is why Brooker argues against fallback in distributed systems).

Being able to walk a product owner through this table, with your own system's costs attached, is
one of the more valuable things this course teaches.

### 2.5 Alerting on the budget

From DS-701 L10, stated here with the arithmetic because it is the practical core:

**Burn rate** = how fast you are consuming budget relative to spending it evenly across the window.
A burn rate of 1 exhausts the budget exactly at the window's end; a rate of 14.4 exhausts a 30-day
budget in about two days.

The standard multi-window, multi-burn-rate policy for a 99.9%/30-day SLO:

| Burn rate | Long window | Short window | Budget consumed | Action |
|---|---|---|---|---|
| 14.4 | 1 h | 5 min | 2% | Page |
| 6 | 6 h | 30 min | 5% | Page |
| 3 | 1 day | 2 h | 10% | Ticket |
| 1 | 3 days | 6 h | 10% | Ticket |

Why two windows: the long window controls the false-positive rate, and the short window makes the
alert stop firing quickly once the burn stops. Why multiple rates: a fast, severe burn deserves a
page in minutes, while a slow, grinding degradation deserves a ticket, and a single threshold
cannot give you both.

The result is what makes this worth the arithmetic: **fewer, more meaningful alerts.** Every page
corresponds to a user-visible problem consuming a defined fraction of an agreed budget, which is a
categorically different thing from "CPU above 80%".

### 2.6 The rest of the reliability practice

SLOs are the measurement; these are what act on it.

**Blameless postmortems.** Written for every significant incident, focused on the systemic causes
and the missing safeguards rather than on the person who typed the command. The rationale is
practical, not therapeutic: **people who expect blame hide information, and hidden information
means the cause is never found.** A postmortem has: a timeline, an impact statement (with SLI
numbers), contributing factors, what went well, what was lucky — the "lucky" section is
underrated — and action items with owners and dates. Action items without owners do not happen.

**The change discipline.** Because change causes most incidents: progressive rollout (canary, then
percentage waves), automatic rollback triggered by SLI regression, bake time between waves, and
feature flags decoupling deploy from release. The last one is the most valuable and the least
adopted: if turning a feature on is a config change rather than a deploy, your rollback is instant.

**Toil reduction.** Toil is manual, repetitive, automatable work that scales with service size and
has no enduring value. Google's guidance caps it at 50% of an SRE's time; the number matters less
than the principle that unbounded toil means the team cannot improve anything and the system
degrades by default.

**On-call as an engineering input.** Every page is a signal about the system. Track pages per shift,
the proportion that were actionable, and the proportion that were the *same* cause recurring. A
team that is paged nightly is being told something, and the correct response is engineering work,
not resilience.

## 3. Construction: SLOs with teeth

Build in `mpse/ca731/l07/`, applied to your L02 multi-tenant service.

**Stage 1 — pick the SLIs.** Define three for your service: availability, latency, and one
domain-specific (freshness or correctness). For each write: the exact numerator and denominator,
where it is measured, what is excluded from `valid`, and one way it could be gamed. Then implement
the measurement.

**Stage 2 — the historical baseline.** Run your service under a realistic workload with injected
faults (DS-701 L10's chaos harness) for long enough to get a distribution. Compute what SLO you
*currently* meet. Only now choose the target — and if your chosen target is above the baseline,
write down the specific work required to reach it.

**Stage 3 — the dependency ceiling.** From L01 Stage 3's dependency graph, compute your
theoretical availability ceiling. Then identify the dependency that binds, and design a graceful
degradation for it — what does your service do when that dependency is down, and what does the user
see? Implement it, and then test it, because an untested degradation path is a fallback that will
not work when needed.

**Stage 4 — burn-rate alerting.** Implement the four-rule multi-window policy. Then replay four
incident shapes through it — a 3-minute total outage, a 6-hour 5% error rate, a 20-minute 50% error
rate, and a slow degradation over three days — and record which rules fire and when. Compare
against a naive "error rate > 1% for 5 minutes" rule on false positives and detection time.

**Stage 5 — per-tenant SLOs.** Your L02 service is multi-tenant, so a global SLI can be healthy
while one tenant is entirely down. Implement per-tenant SLIs and demonstrate exactly that case:
global 99.95%, one tenant at 0%. Then design the alerting policy that catches it without paging on
every tenant's individual blip — this is a real design problem and the answer involves both a
per-tenant threshold and a "number of tenants affected" dimension.

**Stage 6 — the price of a nine.** For your service, cost the step from its current target to the
next one: what would have to be built, what it would cost per month, and what capability the team
would need. Then do the same for the step after that. Present it as the table you would take to a
product owner, and include the option they will not have considered — spending the budget you
already have rather than buying more.

**Stage 7 — the error budget policy.** Write it: the SLO, the window, the budget, what happens at
each burn level, who can override, what happens when a dependency consumes the budget, and how a
shared budget is handled. Then simulate a quarter — inject enough failure to exhaust the budget —
and follow the policy. Write up what you would actually have done, honestly, including whether you
would have overridden it.

**Stage 8 — a postmortem.** Cause a real incident in your system (an unsatisfiable spec from L06
Stage 6 is a good one, or a bad deploy), respond to it as if on-call, and write the postmortem to
the standard in §2.6 — including the "what was lucky" section, which is the one that most often
reveals the next incident.

**Stage 9 — change safety.** Implement progressive rollout with automatic rollback triggered by SLI
regression: deploy to 1% of traffic, watch the SLI for a bake period, proceed or roll back. Then
deploy a deliberately broken version and measure the total error budget consumed before rollback
completes. That number is your change risk, and reducing it is what a canary is *for*.

## 4. Failure modes

- **An SLO with no consequence.** A dashboard with extra steps.
- **A target chosen aspirationally.** Breached constantly, ignored quickly.
- **A target above the dependency ceiling.** Arithmetically impossible.
- **SLIs measured server-side only.** Misses the failures users are actually experiencing.
- **`valid` defined to exclude the failures.** Gaming your own measurement.
- **Percentile latency SLIs aggregated across instances.** Meaningless (DI-721/DS-701 L10).
- **Global SLIs on a multi-tenant service.** One tenant's total outage is invisible.
- **Threshold alerts rather than burn rates.** Noisy and slow, in different situations.
- **Blameful postmortems.** Information stops flowing and causes stop being found.
- **Action items without owners and dates.** Not action items.
- **Unbounded toil.** The team cannot improve anything, so the system degrades by default.
- **Buying nines with hardware when the downtime is caused by change.** Solving the wrong problem
  expensively.

## 5. Exercises

### Warm-up (30 min)

1. Define SLI, SLO and SLA, and give four criteria for a good SLI.
2. Compute the error budget in minutes for 99.95% over 28 days, and the burn rate that exhausts it
   in four hours.
3. Explain why an unspent error budget represents a cost.

### Core (3.5 h)

4. Complete Stages 1–3, including how each SLI could be gamed and the tested degradation path.
5. Complete Stage 4 and deliver the four-incident comparison table against the naive rule.
6. Complete Stage 5 and deliver the per-tenant alerting design.
7. Complete Stage 7: the error budget policy plus your honest account of the simulated quarter.

### Challenge

8. Complete Stages 6, 8 and 9 — the price-of-a-nine table, the postmortem, and the measured budget
   consumption of a bad deploy under progressive rollout.
9. Build an **error budget management system**: continuous SLI computation from your telemetry,
   budget tracking against multiple SLOs, burn-rate alerting, and a policy engine that gates
   deployments on remaining budget (integrated with your L05 pipeline, so an out-of-budget service
   cannot deploy features without a logged override). Then run it for two simulated quarters with
   varying incident loads and report: how often the freeze triggered, how much feature work was
   blocked, and whether the SLO you chose produced sensible behaviour — because an SLO that freezes
   a team for six weeks is as wrong as one that never binds, and finding that out in simulation is
   much cheaper than finding it out in a quarter.

## 6. Self-check

1. Distinguish SLI, SLO and SLA, and say why the SLA is looser.
2. Give the four criteria for a good SLI and an example of a gameable one.
3. Why is a latency SLI expressed as a ratio rather than a percentile?
4. State the error budget policy and explain why the agreement must precede the exhaustion.
5. Give four inputs to choosing a target.
6. What changes architecturally between 99.9% and 99.99%, and why can humans not be in the recovery
   path?
7. Why do high availability targets constrain deployment practice more than hardware?
8. Explain multi-window multi-burn-rate alerting and what each dimension buys.
9. Why must a multi-tenant service have per-tenant SLIs?
10. What is toil, why is it capped, and what does an unbounded amount of it imply?

## 7. Primary sources

- **Google, *The Site Reliability Engineering Workbook*, chapters 2, 4 and 5** — SLOs, budget
  policy, and the burn-rate arithmetic. Chapter 5 is the one to work through with a calculator.
- **Google, *Site Reliability Engineering*, chapters 3, 4, 5, 6 and 15** — the philosophy, the
  toil chapter, and the postmortem culture chapter.
- Beyer, Murphy, Rensin, Kawahara & Thorne, *The Site Reliability Workbook* case studies — how
  other organisations adapted it, including where it did not work.
- Brooker, "Avoiding Fallback in Distributed Systems" (AWS Builders' Library) — read before
  designing a degradation path.
- Allspaw, "Blameless PostMortems and a Just Culture" (Etsy) — the original argument, and the
  clearest.
- Dekker, *The Field Guide to Understanding 'Human Error'* — the theory behind blamelessness, from
  outside software.
- Nygard, *Release It!*, 2nd ed. — stability patterns and antipatterns, with DS-701 L09.

---

**Previous:** [L06](L06-control-planes-and-kubernetes.md) · **Next:**
[L08 — Capacity, Cost, and the Economics of Architecture](L08-capacity-and-cost.md)
