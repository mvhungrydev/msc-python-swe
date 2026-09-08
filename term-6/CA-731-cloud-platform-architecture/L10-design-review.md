# CA-731 · Lesson 10 — Reviewing a Design: Failure, Cost, Security, Operability

**Estimated study time:** 4 hours
**Prerequisites:** L01–L09; SE-521 L10 (ADRs); DS-701 (all)

---

## 1. Orientation

Every previous lesson taught a dimension. This one is about holding them all at once, which is the
actual job of a senior engineer and the thing least often taught explicitly.

A design review is not a checklist exercise, and it is not a hunt for mistakes. Its purpose:

> **Surface the assumptions and the trade-offs while they are still cheap to change, and produce a
> written record of what was decided and why — so that in two years, when the context has changed,
> someone can tell whether the decision is still right.**

The failure mode of design review at most organisations is that it happens either too late (the
system is built; the review is theatre) or too shallow (a diagram is presented, nobody asks a hard
question, and it is approved). This lesson gives you a method for asking the questions that
actually find problems, and a format for recording the answers.

The second theme, which is a skill in itself: **reviewing well is a social act.** A review that
makes the author defensive produces less information than one that does not. The techniques for
that are not soft — they are what determines whether you learn anything.

## 2. Theory

### 2.1 The four dimensions, and why they must be reviewed together

- **Failure**: what breaks, how it is detected, what the user experiences, how it recovers.
- **Cost**: what it costs at expected scale and at 10×, and which term dominates.
- **Security**: what the trust boundaries are, what the blast radius of a compromise is.
- **Operability**: can a person on call at 3 a.m. understand and fix it?

They must be reviewed together because **they trade against each other**, and reviewing them
separately produces a design that is locally optimal in each and globally incoherent:

- Three AZs improve failure tolerance and add cross-AZ transfer cost (L04, L08).
- Aggressive caching improves cost and latency and adds a staleness failure mode and a cache
  invalidation problem (DI-721 L09).
- Fine-grained microservices improve team autonomy and multiply the failure surface, the network
  cost and the operational burden.
- Strong isolation between tenants improves security and reduces the multiplexing gain that makes
  the business work (L02).

The single most useful question in a design review is therefore: **"what did you trade away to get
that?"** A design presented with no trade-offs has not been thought through, or is being sold rather
than reviewed.

### 2.2 A review method

Work in this order. It is deliberately not the order the design is usually presented in.

**1. Establish the requirements before looking at the design.** What must this do? What are the
availability, latency, durability, consistency and cost targets? Who are the users and what do they
experience when it fails? **A surprising fraction of bad designs are correct solutions to
unexamined requirements**, and half the value of a review is extracted here, before the design is
discussed at all.

**2. Understand the design as the author intends it.** Have them walk the primary flow end to end.
Ask clarifying questions only. Do not critique yet — you will critique a design you have
misunderstood, and the conversation will never recover.

**3. Trace the failure paths.** For each component: what happens when it is slow? unavailable?
returns wrong data? What does the *user* see? How is it detected? How does it recover, and does
recovery require the control plane (L01 §2.3)? This is where most real problems are found, and
"slow" is the case people have not thought about — partial failure is harder than total failure and
much more common.

**4. Find the correlated failures.** What is shared? Which components fail together? What single
dependency appears in every path — DNS, identity, a config service, the deployment pipeline (L01
§2.4)?

**5. Cost it.** At expected scale and at 10×. Which term dominates? What grows superlinearly
(L08 §2.2)?

**6. Draw the trust boundaries.** At each crossing: authenticated, authorised, validated, logged?
What is the blast radius of one compromised credential (L03)?

**7. Assess operability.** Can it be debugged from its telemetry alone? What are the runbooks? What
does deployment look like, and what does rollback look like? What is the on-call burden?

**8. Ask what was considered and rejected.** A design with no rejected alternatives has not been
designed; it has been arrived at. The rejected options and the reasons are the most valuable part of
the record.

**9. Identify the assumptions.** What must remain true for this to be right? Which of them is most
likely to change? This is what a future reader needs most.

**10. Write it down.** §2.5.

### 2.3 The questions that find real problems

Collected from the preceding lessons; these are the ones that repeatedly surface things the author
had not considered.

**Failure**
- What happens when this dependency is *slow* rather than down? (Almost always the unconsidered
  case.)
- Does recovery require a control-plane call? What if the control plane is degraded?
- What is retried, at which layer, and what bounds the amplification? (DS-701 L09.)
- What does the user see during a partial failure? Is there a degraded mode, and has it been
  *tested*?
- How is a bad deploy detected and rolled back, and how much error budget does that consume?
  (L07.)
- What is the largest correlated failure this design can survive, and what is the smallest one it
  cannot?

**Cost**
- What is the unit cost, and does it rise or fall with scale?
- How much of the bill is data transfer, and which paths?
- What is the idle cost, and what is the duty cycle?
- What does this cost at 10× traffic? At 10× data?
- Which reliability decisions are we paying for, and did anyone price them?

**Security**
- What credentials exist, how long do they live, and what can each one do? (L03.)
- What is the blast radius if this component is fully compromised?
- Where does untrusted input enter, and where is it validated?
- What does the CI pipeline have access to?
- If a customer asks "is my data isolated from other tenants?", what is the honest sentence?
  (L02 §2.6.)

**Operability**
- Can I diagnose a problem from telemetry without adding code? (DS-701 L10.)
- What are the three most likely pages, and does a runbook exist for each?
- How do I roll this back? Has that been tested?
- What is the migration path from what exists today, and is each step reversible? (DI-721 L10.)
- Who is on call for this, and did they review it?

**Meta**
- What did you trade away?
- What would have to be true for this to be the wrong design?
- What is the reversibility of this decision — a one-way door or a two-way door? (Two-way doors
  deserve less review and faster decisions; one-way doors deserve more of both, and confusing them
  wastes everyone's time in one direction and causes disasters in the other.)

### 2.4 Reviewing well, socially

The techniques that produce more information:

- **Review the design, not the designer.** "What happens when the database fails over?" not "you
  forgot failover."
- **Ask rather than assert.** You may be wrong, and a question lets the author show you why without
  a confrontation. It also lets them find the problem themselves, which they will remember.
- **Distinguish blocking concerns from preferences**, explicitly and out loud. Reviewers who label
  everything as important get ignored; reviewers who label nothing get overruled.
- **Acknowledge the constraints.** The author may know something you do not. "Was there a reason
  you did not use X?" often gets a good answer.
- **Timebox and prioritise.** A review that raises forty points produces no action. Raise the five
  that matter and record the rest.
- **Write the concerns down** so they survive the meeting, and note which were accepted, which were
  rejected and why, and which were deferred with an owner.

And the structural point that determines whether reviews work at all: **review early, when changing
the design is still cheap.** A review of a built system is an audit, and audits produce either
theatre or rework. The right moment is when the shape is clear and the code is not written.

### 2.5 The record: ADRs and design documents

**Architecture Decision Records** (SE-521 L10) capture one decision: context, the decision, status,
consequences, and the alternatives considered. Short, immutable, superseded rather than edited.
Their value is entirely in the future — the reader in two years asking "why on earth is it like
this?"

A **design document** for a substantial system, with a structure that maps onto the review method:

1. **Context and requirements** — what problem, for whom, with what targets.
2. **The design** — the primary flow, the components, the data model, the interfaces.
3. **Failure analysis** — per component, per dependency, with detection and recovery.
4. **Cost analysis** — at expected scale and at 10×.
5. **Security analysis** — trust boundaries, credentials, blast radius.
6. **Operability** — telemetry, runbooks, deployment, rollback, on-call impact.
7. **Alternatives considered and rejected**, with reasons.
8. **Assumptions**, and which are most likely to change.
9. **Open questions and risks**, with owners.
10. **Migration plan**, if replacing something.

Sections 3–7 are the ones usually missing, and they are the ones that make a document worth reading.
Section 8 is the one that ages best.

### 2.6 The design review as a course capstone

Everything in this course meets in one document, and the exam question this lesson is really
preparing you for is: *given a design, can you find what is wrong with it, price it, and say what
you would change and why?*

That is the work of a staff engineer. It is not primarily about knowing more services; it is about
holding four dimensions simultaneously, asking the question that finds the assumption, and writing
it down clearly enough that someone else can act on it.

## 3. Construction: review, be reviewed, and write it up

Build in `mpse/ca731/l10/`. This lesson's construction is documentary, and the documents are the
deliverables.

**Stage 1 — the design document.** Write the full design document for your term artifact platform
(L01–L09) using §2.5's structure. All ten sections. Sections 3–7 in real detail, with the numbers
you measured in the earlier lessons rather than assertions.

**Stage 2 — self-review.** Work through §2.3's question list against your own document. Record every
question you could not answer well. That list is more valuable than the document.

**Stage 3 — the failure analysis, properly.** For every component and dependency in your design,
fill in a table: failure mode (slow / unavailable / wrong data), user impact, detection mechanism,
recovery mechanism, whether recovery needs the control plane, and estimated time to recover. Then
find the row you cannot fill in — there will be one — and fix the design or record it as a known
risk.

**Stage 4 — the pre-mortem.** Assume it is eighteen months from now and the system has failed
badly. Write the incident report for that failure, in the past tense, with a timeline. This
technique reliably surfaces risks that a forward-looking analysis does not, because it changes the
question from "what could go wrong?" to "what *did* go wrong?", which people answer more concretely.
Then take the top three causes and address them in the design.

**Stage 5 — review someone else's design.** Take a real design document — a colleague's, an
open-source project's design docs, or a public architecture write-up. Review it with the method in
§2.2, produce a written review with concerns ranked and each labelled blocking or preference, and
— if the author is available — deliver it and observe what happens. Record which of your concerns
turned out to be based on a misunderstanding; that ratio is a calibration measure for you.

**Stage 6 — be reviewed.** Have someone review your Stage 1 document. Record every concern. For
each, respond: accept and change, reject with reasoning, or defer with an owner. Note which
concerns you had already identified in Stage 2 and which you had not — the gap is what a second
reader is *for*.

**Stage 7 — the ADRs.** Write five ADRs for the significant decisions in your platform, each with
its rejected alternatives. Then write a sixth for a decision you now think was wrong, with a
`Superseded` status and a link to the replacement — practising the supersession mechanism is worth
more than writing five correct ones.

**Stage 8 — the one-way doors.** Go through your design and classify each significant decision as a
one-way or two-way door. For the one-way doors, state what would be required to reverse them and
what it would cost. Then check whether the review effort you spent was proportionate — most teams
over-review reversible decisions and under-review irreversible ones, and finding an instance of
each in your own work is the exercise.

**Stage 9 — the review checklist for your organisation.** Produce a one-page review checklist,
adapted from §2.3 to your actual context, that a team could use without having taken this course.
Then use it on a real design and revise it based on what it missed. A checklist that has never been
used is a wishlist.

## 4. Failure modes

- **Reviewing too late.** The system exists; the review is theatre or rework.
- **Reviewing the diagram, not the failure paths.** Boxes and arrows hide everything that matters.
- **Only considering total failure.** Slow is more common and harder.
- **Reviewing dimensions separately.** Produces a design that is locally sensible and globally
  incoherent.
- **No cost analysis.** Discovered at the invoice, years later.
- **No rejected alternatives recorded.** The next person cannot tell what was considered, and will
  re-litigate it.
- **No assumptions recorded.** Nobody can tell when the decision has expired.
- **Reviewing the designer.** Produces defensiveness and less information.
- **Forty concerns, unranked.** No action follows.
- **Concerns raised verbally and lost.** The meeting happened; nothing changed.
- **Equal review effort for one-way and two-way doors.** Slow on the reversible, careless on the
  irreversible.
- **The on-call engineer not in the review.** Operability is decided by people who will not
  experience it.

## 5. Exercises

### Warm-up (30 min)

1. Name the four dimensions and give a trade-off between each pair you can construct.
2. Give the ten-step review method and say why requirements come before design.
3. Explain one-way versus two-way doors and what each implies for review effort.

### Core (3 h)

4. Complete Stages 1–3: the design document, the self-review gap list, and the completed failure
   table including the row you could not fill.
5. Complete Stage 4 — the pre-mortem — and the three design changes that followed.
6. Complete Stage 5 and report your misunderstanding ratio.
7. Complete Stage 7: five ADRs plus one superseded.

### Challenge

8. Complete Stages 6, 8 and 9 — being reviewed with a recorded response to every concern, the
   one-way door classification with the proportionality check, and the organisational checklist
   revised after real use.
9. Take a **published system architecture** — a paper (Dynamo, Spanner, Borg, Kafka, Zanzibar), a
   detailed engineering blog post, or a public reference architecture — and write a full design
   review of it using this lesson's method. Identify: the trade-offs it made and what it gave up;
   the assumptions it depends on and which have since changed; what it would cost to run today; and
   what you would do differently now and why. Then find the paper's own limitations section (or its
   absence) and compare against your review. This exercise is the closest thing in the course to
   the judgement being examined, and doing it against a *good* system is more instructive than
   against a bad one — because finding the trade-offs in a design that is genuinely well made is
   what the skill actually consists of.

## 6. Self-check

1. State the purpose of a design review in one sentence.
2. Give the four dimensions and explain why they must be reviewed together.
3. Give the ten-step method in order.
4. Why is "what happens when it is slow?" more useful than "what happens when it fails?"
5. Give five failure-dimension questions and five security-dimension questions.
6. What does a design with no rejected alternatives tell you?
7. Why are recorded assumptions the section that ages best?
8. Give four techniques for reviewing in a way that produces more information.
9. What is a pre-mortem and why does it surface risks a forward analysis misses?
10. Why should review effort be proportionate to reversibility?

## 7. Primary sources

- **SE-521 L10** and Michael Nygard's original ADR post — the record format.
- Google's design document culture, as described in *Software Engineering at Google*, chapter 9
  (code review) and the design review material — read for the process, not the templates.
- **Klein, "Performing a Project Premortem" (HBR, 2007)** — two pages, and the technique works.
- Bezos's one-way/two-way door framing, and Amazon's PR/FAQ working-backwards process — read for
  the reversibility argument.
- Google, *Site Reliability Engineering*, chapter 27 ("Reliable Product Launches at Scale") — the
  launch checklist as an institutionalised review.
- Fowler, "Who Needs an Architect?" — on what architectural decisions actually are.
- The papers in this course's and DS-701's reading lists, re-read as *designs to be reviewed*
  rather than as results to be learned. That shift in reading stance is the last skill this course
  is trying to install.

---

**Previous:** [L09](L09-platform-as-product.md) ·
**Next:** [Problem sets](problem-sets.md) · [Exam](exam.md)

*This completes CA-731. The course began by claiming that a managed service is a distributed system
somebody else operates, and ends by asking you to review a design across four dimensions at once.
Those are the same skill: refusing to accept an abstraction at face value, and asking what it costs.*
