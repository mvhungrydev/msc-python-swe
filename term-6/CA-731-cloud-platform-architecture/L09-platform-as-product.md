# CA-731 · Lesson 09 — Platform Engineering as a Product Discipline

**Estimated study time:** 4 hours
**Prerequisites:** L01–L08; SE-521 L09 (service boundaries)

---

## 1. Orientation

Every organisation past a certain size builds an internal platform, whether or not anyone calls it
that: the shared way to deploy, the shared logging setup, the Terraform modules everyone copies,
the CI templates, the base images. The question is never *whether* there is a platform; it is
whether it is designed or accreted.

The claim this lesson makes:

> **A platform is a product. Its users are engineers, they have alternatives, and adoption is
> voluntary in practice even when it is mandatory on paper.** A platform nobody wants to use gets
> routed around, and the routes around it become the real platform — undocumented, unsupported and
> inconsistent.

That single reframing changes the work. A product has users whose needs you must actually
understand, an interface you must keep stable, documentation that must be good enough to succeed
without you, a support model, a roadmap, and a measure of success that is *adoption and user
outcomes*, not features shipped.

The failure mode this lesson is written against is specific and extremely common: a platform team
that builds what it finds interesting, mandates its use, measures its success by the number of
services onboarded, and cannot understand why the engineers it serves are unhappy. The technical
work in L01–L08 is necessary and not sufficient; this lesson is the part that determines whether
any of it is used.

## 2. Theory

### 2.1 What a platform is for

The economic argument: **a platform exists to reduce the cognitive load on the teams that build
products**, so they can spend their attention on their domain rather than on the twenty decisions
between "we wrote a service" and "it is running reliably in production with observability,
security, and a deployment pipeline."

The corollary, which is the test for any proposed platform feature: **if it does not reduce
cognitive load for a product team, it is not platform work.** A feature that requires every team to
learn a new abstraction *in addition to* the underlying technology has increased cognitive load, not
reduced it — this is the leaky-abstraction failure, and it is what turns a platform into a tax.

*Team Topologies* frames this well: a platform team's job is to provide a **compelling internal
product** that other teams *choose* to use because it is genuinely the easiest path — not because
they are required to.

### 2.2 Golden paths

A **golden path** is a supported, opinionated, well-documented route through a common task. Not the
only path — the *paved* one.

Properties that make a golden path work:

- **It is opinionated.** It makes the decisions so users do not have to: this language runtime,
  this deployment pattern, this observability stack, these defaults. Choice is a cost you are
  removing.
- **It is complete.** From "I have code" to "it is in production with monitoring, alerting, a
  pipeline, secrets, and an on-call rotation". A path that gets 80% of the way and abandons the user
  at the hard part is worse than none, because it wastes the effort before the hard part.
- **It is fast.** Minutes to a running service, not days. The time-to-first-success is the number
  that determines whether people adopt it.
- **It is escapable.** A team with a genuine reason to deviate can, without leaving the platform
  entirely. A platform that forbids deviation gets forked.
- **It is the default and the easiest option.** If the golden path is harder than doing it yourself,
  it will not be used, and no mandate fixes that.

The organisational subtlety: golden paths work when the platform team *supports* them. A path with
no support is a template, and a template with no support rots as everyone's copy diverges — which
is exactly the accreted-platform problem you were trying to solve.

### 2.3 Self-service, and the interface

The measure of a platform's maturity is what a product team can do **without talking to the platform
team**. If provisioning an environment, adding a dependency, changing a resource limit or rotating a
credential requires a ticket and a human, the platform is a bottleneck wearing a platform's clothes.

Self-service requires:

- **An API or a declarative interface** (this is where L06's control-plane pattern earns its place:
  a CRD plus a controller is a self-service interface with authorisation, validation, audit and
  reconciliation built in).
- **Guardrails rather than gates** (L03 §2.4). A gate is a human approving each request; a guardrail
  is a policy that makes the wrong thing impossible and lets everything else through automatically.
  Gates do not scale and they make the platform team a queue.
- **Good errors.** A self-service system that fails with an unclear message generates the ticket you
  were trying to avoid. **Error message quality is a platform feature**, and it is the one most
  consistently under-invested in.
- **Visibility.** Users can see what they have, what it costs (L08), and whether it is healthy
  (L07), without asking.

### 2.4 The interface is a contract

Everything from SE-511 L09 and SE-521 applies, with one aggravating factor: **your users cannot
easily be changed, and there may be hundreds of them.**

- **Version the interface.** Module versions (L05), API versions, base image tags. Consumers pin;
  you do not break pinned versions.
- **Deprecate properly**: announce with a date, provide a migration path *and ideally a migration
  tool*, support both for a stated period, then remove. "We announced it in Slack" is not a
  deprecation process.
- **Understand that a default is an interface.** Changing a default changes behaviour for everyone
  who did not specify it, which is everyone. Treat default changes as breaking changes.
- **Hide implementation** (SE-521 L01). If your interface exposes that you use a particular managed
  service, you can never change it. If it exposes "a queue with these semantics", you can.

The hardest interface question in platform work: **how much of the underlying system do you expose?**
Too little and every non-standard need becomes an escalation to your team. Too much and you have
built a thin wrapper that provides no abstraction and breaks whenever the underlying system changes.
The workable answer is a layered interface — a simple default path, a more detailed path for teams
that need it, and a documented escape hatch — with the honest acknowledgement that the boundary will
be wrong somewhere and will need to move.

### 2.5 Measuring a platform

The wrong metrics: number of services onboarded (mandated adoption is not adoption), features
shipped, tickets closed.

The right ones fall into three groups:

**Adoption, honestly measured**: what fraction of eligible teams *chose* the golden path when they
had an alternative, and — the informative one — how many built their own thing instead, and why.
The teams that routed around you are your most valuable source of product feedback.

**User outcomes**, which is where the DORA metrics belong: deployment frequency, lead time for
change, change failure rate, and time to restore. These measure whether the platform made teams
faster and safer, which is its actual purpose. Measure them per team and look at the *distribution*
— a platform that helps strong teams and abandons weak ones has a bimodal distribution and a
problem.

**Platform health**: time-to-first-success for a new service, support ticket volume per team
(falling is good; it means self-service works), the platform's own SLOs (L07 — the platform is a
dependency, and its availability is a ceiling on everyone else's), and cost per unit served (L08).

The one qualitative signal worth institutionalising: **talk to your users regularly and watch them
work.** A platform team that has not watched an engineer onboard a service in six months does not
know what its product is like to use. This is the standard product discipline of user research,
applied to an internal audience, and it is the practice most often skipped.

### 2.6 Documentation as a deliverable

Platform documentation is the interface for most users most of the time, and it fails in
predictable ways. The Diátaxis framework's four types, each with a distinct job:

- **Tutorial**: a beginner completes a real task successfully. Learning-oriented, prescriptive,
  guaranteed to work. This is the golden path's front door.
- **How-to guide**: solves a specific problem for someone who already has context.
- **Reference**: complete and accurate specification. Generated where possible.
- **Explanation**: why the system is like this, what the trade-offs were. This is the type that
  makes users able to reason rather than just follow — and it is the type nearly always missing.

The failures: one enormous README that tries to be all four; a tutorial that does not actually work
end to end (test it in CI — an untested tutorial is broken within a month); reference generated but
explanation absent, so users cannot make decisions; and no owner, so it rots.

The standard worth holding: **a new engineer should be able to get a service into production from
the documentation alone, without asking anyone.** If they cannot, the documentation is incomplete,
and the evidence is in your support channel — every repeated question is a documentation defect with
a specific location.

### 2.7 The support model

Say explicitly what the platform supports and what it does not. Without it, the platform team's
work is defined by whoever asks most loudly.

- **Tiers**: what is fully supported (on-call, SLO, break/fix), what is best-effort, what is
  unsupported-but-permitted.
- **Channels and expectations**: where to ask, and what response time to expect.
- **On-call**: the platform team is on call for the platform. A platform team that is not on call
  for its own product does not experience its failure modes, and will not prioritise fixing them.
- **Escalation and the shared incident model**: when a product team's incident is caused by the
  platform, how does that work?

The practice that distinguishes good platform teams: **treat every support request as a product
defect.** A question asked twice is a documentation gap; a manual intervention performed twice is
an automation gap; a failure diagnosed twice is an error-message gap. Tracking support volume by
category and driving it down is the mechanism by which a platform team stops being a help desk.

## 3. Construction: build the platform as a product

Build in `mpse/ca731/l09/`, integrating everything from L01–L08 into something a hypothetical
product team could actually use.

**Stage 1 — user research.** Identify three "user teams" (real colleagues if possible; otherwise
write three detailed personas from teams you have worked with). For each, document what they need to
get a service into production, what they currently find hard, and what they would do if your
platform did not exist. Then rank the pain points by frequency × severity. Build for the top three,
not the ones you find most interesting.

**Stage 2 — the golden path.** Build it end to end: a single command (or template repository) that
takes a service from nothing to running in a non-production environment with a pipeline (L05),
observability (DS-701 L10), SLOs (L07), secrets (L03), network placement (L04) and a cost tag (L08).
Measure the **time to first success** and drive it down. Under fifteen minutes is a reasonable
target; if it is hours, find out where the time goes.

**Stage 3 — the self-service interface.** Expose it as a declarative interface — the `TenantEnvironment`
CRD and controller from L06 is the natural vehicle. A user declares what they want and the platform
converges. Then count: how many things still require a human from your team? Eliminate the top two.

**Stage 4 — error message quality.** Deliberately break the golden path in eight distinct ways
(missing permission, quota exceeded, invalid config, name collision, dependency unavailable, policy
violation, resource limit, malformed manifest). For each, record the error a user actually sees.
Then rewrite each message to say: what happened, why, and what to do next. Re-test with someone who
has not seen your platform, and count how many they could resolve alone. This stage produces more
adoption per hour of effort than any other in the lesson.

**Stage 5 — documentation.** Write all four Diátaxis types for the golden path. Then test the
tutorial: have someone follow it with no help and no prior knowledge, and record every point where
they stopped, asked a question or guessed. Fix each. Then automate the tutorial as a CI test so it
cannot silently rot.

**Stage 6 — the interface contract.** Version your platform's interface (modules, CRD version, base
images). Write the compatibility and deprecation policy: what is stable, what is experimental, what
notice you give, and what migration support you provide. Then execute a real deprecation: change a
default, provide a migration tool, and run the deprecation to completion. Note how much of the work
was the migration tool — that ratio is why deprecations without tools do not finish.

**Stage 7 — measurement.** Instrument: time to first success, DORA metrics per team, support
requests by category, platform SLOs, and cost per service. Build the dashboard. Then answer the
uncomfortable question with data: **is any team better off?** If you cannot tell, your measurement
is wrong.

**Stage 8 — the support model.** Write it: tiers, channels, response expectations, on-call, and
escalation. Then run a simulated week: handle ten support requests, categorise each as documentation
gap, automation gap, error-message gap, or genuine novel problem, and produce the backlog of product
defects that follows. That backlog is your roadmap, and it will look nothing like the roadmap you
would have written from your own preferences.

**Stage 9 — the escape hatch.** Design and document how a team with a legitimate non-standard need
deviates without leaving the platform. Then implement it for one real case. Then write the policy:
who approves a deviation, what support it gets, and how deviations feed back into the roadmap —
because a deviation that three teams request is a missing feature, and noticing that is the
mechanism by which a platform stays relevant.

## 4. Failure modes

- **Building what the platform team finds interesting.** The most common failure, and the hardest
  to see from inside.
- **Mandating adoption instead of earning it.** Produces compliance and workarounds, and destroys
  your feedback signal.
- **Measuring services onboarded.** Mandated adoption measured as success.
- **A golden path that stops before the hard part.** Worse than none.
- **A thin wrapper with no abstraction.** All the maintenance cost, none of the benefit, and it
  breaks whenever the underlying system changes.
- **Gates instead of guardrails.** The platform team becomes a queue.
- **Unclear errors.** Every one generates the ticket self-service was meant to prevent.
- **Untested tutorials.** Broken within a month, and the first thing a new user encounters.
- **Reference documentation with no explanation.** Users can follow but cannot decide.
- **Changing a default without treating it as breaking.** It is breaking.
- **Deprecation without a migration tool.** Never completes; you support both forever.
- **A platform team not on call for its platform.** Does not feel its own failure modes.
- **No escape hatch.** Teams fork, and the fork becomes a second platform you do not control.
- **Never watching a user work.** You do not know what your product is like.

## 5. Exercises

### Warm-up (30 min)

1. State the platform-as-product claim and the test for whether something is platform work.
2. Give five properties of a golden path, and say which one is most often missing.
3. Distinguish guardrails from gates, and explain why the distinction determines whether a platform
   scales.

### Core (3 h)

4. Complete Stage 1 and deliver the ranked pain points, with an explicit note on which one you
   *wanted* to build and whether it made the top three.
5. Complete Stages 2–3 and report your time to first success, plus what still requires a human.
6. Complete Stage 4 — the eight errors, rewritten, with the independent-resolution count before and
   after.
7. Complete Stage 5, including the observed tutorial run and the CI-tested tutorial.

### Challenge

8. Complete Stages 6–8: the deprecation executed to completion, the measurement dashboard with an
   honest answer to "is anyone better off?", and the support-request categorisation with the
   resulting backlog.
9. Complete Stage 9, then write a **platform strategy document** (1,500 words) for a real or
   hypothetical organisation: what the platform will and will not do; which teams are its users and
   which are explicitly out of scope; the golden paths it will support; the interface stability
   commitment; the support model; the measures of success and the thresholds at which you would
   consider the platform to have failed; and — the section that most such documents omit — **what
   you will delete.** A platform that only accretes becomes the legacy system it was built to
   replace, and having a written policy for removing paths, deprecating modules and sunsetting
   features is what prevents that.

## 6. Self-check

1. Why is a platform a product, and what follows from that about mandates?
2. What is the test for whether a proposed feature is platform work?
3. Give the five properties of a golden path, and explain why escapability matters.
4. What can a mature platform's users do without talking to the platform team?
5. Why are guardrails better than gates for a platform's scalability?
6. Why is error message quality a platform feature?
7. Give the four documentation types and the job of each, and say which is most often missing.
8. Why is changing a default a breaking change?
9. Give three groups of platform metrics and one wrong metric with the reason it misleads.
10. Why should a platform team be on call for its own platform?

## 7. Primary sources

- **Skelton & Pais, *Team Topologies*** — the platform team as an enabling structure, and the
  cognitive-load argument.
- **Evan Bottcher, "What I Talk About When I Talk About Platforms"** (martinfowler.com) — the
  clearest short statement of the compelling-internal-product idea.
- Forsgren, Humble & Kim, *Accelerate* — the DORA metrics and the research behind them.
- **Procida, the Diátaxis framework** (diataxis.fr) — short, and it will change how you write
  documentation.
- Fowler & contributors on internal developer platforms; and Spotify's Backstage documentation read
  as a product rather than a tool.
- Google, *Site Reliability Engineering*, chapter 32 ("The Evolving SRE Engagement Model") — how a
  platform-like team decides what to support.
- Kim, Humble, Debois & Willis, *The DevOps Handbook* — the organisational argument this lesson
  assumes.
- Your own organisation's internal platform, examined honestly with §2.5's metrics. This is the most
  valuable source available to you.

---

**Previous:** [L08](L08-capacity-and-cost.md) · **Next:**
[L10 — Reviewing a Design: Failure, Cost, Security, Operability](L10-design-review.md)
