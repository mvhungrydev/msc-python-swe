# SE-521 — Written Examination

**Time allowed: 3 hours. Closed book.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

---

## Section A — answer FOUR

### Modularity and information hiding (L01)

**A1.** State Parnas's criterion for decomposition. Explain precisely why decomposition by
processing step fails, and give a worked example of the same system decomposed both ways with the
change that distinguishes them. *(15)*

**A2.** What is a module's "secret", and what diagnostic question follows from the idea? Name six
things that are part of an interface beyond the function signatures, and give a change that breaks
each. *(15)*

**A3.** State Hyrum's Law and the correct response to it — which is not despair. Define deep and
shallow modules with an example of each, explain what "different layer, different abstraction" rules
out, and distinguish encapsulation from information hiding. *(15)*

### Coupling, cohesion, and change (L02)

**A4.** Give the coupling scale from worst to best with an example of each. Explain why control
coupling is worse than it appears and give the fix, and say when stamp coupling is acceptable and
when it is harmful. *(15)*

**A5.** Distinguish afferent from efferent coupling and say for which one a high value is fine and
why. Then give the connascence forms in order of strength and the three rules for using the
taxonomy. *(15)*

**A6.** Explain why dynamic connascence is worse than static, with an example of each that a type
checker can and cannot see. State the metric that all the coupling measures are proxies for and how
you would actually measure it, and name five couplings invisible in the import graph. *(15)*

### Dependency inversion (L03)

**A7.** State the dependency inversion principle precisely and identify which word does the work.
Explain where the interface must live, and why the common mistake — putting it beside the
implementation — inverts nothing. *(15)*

**A8.** Define a composition root and give the properties of a good one. Explain the fragile base
class problem and why composition cannot exhibit it. *(15)*

**A9.** Give six situations in which inverting a dependency makes the system worse. State the
criteria for when to invert and what they imply about how you identify ports. Then explain why a
service locator is worse than constructor injection, and state the functional core / imperative
shell idea and what it does to the need for inversion in the first place. *(15)*

### Design patterns (L04)

**A10.** Give Alexander's three-part definition of a pattern and say which part is usually dropped
when patterns are discussed in software. Then say which section of a GoF pattern description is the
most valuable and why. *(15)*

**A11.** Give the rule that decides between a function and a class when implementing Strategy in
Python. Name five patterns that remain genuinely useful in Python and say why each survives, and
explain why Singleton is not a pattern. *(15)*

**A12.** State Norvig's critique of design patterns at its strongest, and then state its limit —
what it does not account for. Give four consequences of the Observer pattern that are undefined by
default and must be specified, and give the criteria table for deciding whether to refactor a
dispatch conditional. *(15)*

### Ports and adapters (L05)

**A13.** State the dependency rule and describe how you would enforce it mechanically rather than by
review. Distinguish driving from driven ports and say which needs an interface and why. *(15)*

**A14.** Give the two properties of a good adapter. Explain where format validation and domain
validation each belong and why the split matters, and name three things that break persistence
ignorance in practice. *(15)*

**A15.** Give five honest costs of a ports-and-adapters architecture. Give the five adoption levels
and say which step has the best cost-to-benefit ratio, and explain why an external call inside a
database transaction is a defect and what the fix is. *(15)*

### Domain-driven design (L06)

**A16.** What is the ubiquitous language, and what does an ambiguity in it indicate about the
design? Distinguish an entity from a value object and explain why you should make more value
objects than you currently do. *(15)*

**A17.** Give the diagnostic for an anemic domain model and the fix. Define a bounded context and
give four practical ways to find boundaries. *(15)*

**A18.** Give five context-mapping patterns and say when each applies. Explain what an
anticorruption layer protects against, distinguish mild from strong CQRS and say which is a
refactoring and which is an architecture, and state when DDD does not apply — with the
core/supporting/generic distinction and what it is for. *(15)*

### Aggregates and consistency (L07)

**A19.** Give the four rules of aggregate design. State the aggregate-sizing trade-off and the
four-step procedure for resolving it. *(15)*

**A20.** Explain why the credit-limit example is the canonical aggregate-boundary problem, and what
asking the business actually reveals. Give three concurrency-control mechanisms with the cost of
each, and explain why read-committed does not prevent lost updates. *(15)*

**A21.** Describe the outbox pattern and state what it requires of consumers. Distinguish
choreography from orchestration and say when to prefer each. Then explain why compensation is not
rollback and what that implies for how compensating actions must be designed. *(15)*

### Evolutionary architecture (L08)

**A22.** Give the three mechanisms by which architecture decays. Define a fitness function and give
both classification axes — atomic/holistic and triggered/continuous — with an example in each
quadrant. *(15)*

**A23.** Give five design rules for a fitness function that survives contact with a team, and name
five fitness functions worth having in a layered Python application. *(15)*

**A24.** State Fowler's reframing of architecture — "the decisions that are hard to change" — and
its corollary for how design should proceed. Explain the last responsible moment and how it differs
from procrastination, why three is the practical maximum for optimised characteristics, and what
mechanism prevents a migration stalling at 60% complete. *(15)*

### Service boundaries (L09)

**A25.** State Waldo et al.'s claim and give three outcomes a remote call has that a local one does
not. Then give six costs of distribution. *(15)*

**A26.** State the honest reason most organisations distribute their systems, and what Conway's Law
implies about it. Explain what a modular monolith gives and does not give, and why it is the right
precursor to distribution rather than an alternative to it. *(15)*

**A27.** Give five criteria for where to cut a service boundary. State the sync/async rule and the
query/command default, name six resilience mechanisms and what each protects against, and give the
first three questions of the decision procedure for whether to split at all. *(15)*

### Legacy systems and ADRs (L10)

**A28.** Give Feathers's definition of legacy code and say what strategy it implies. Describe the
characterisation-test procedure and explain why it deliberately encodes existing bugs. *(15)*

**A29.** Name five seam types and the cost of each. Explain what "sprout method" and "sprout class"
are for, and the situation in which each is preferable to editing in place. *(15)*

**A30.** Give the five steps of a strangler fig migration and its three hard parts. Give the five
steps of branch by abstraction and explain why every step is independently mergeable. Then state
four elements that make an ADR worth reading three years later. *(15)*

---

## Section B — answer ONE

**B1. (40)** A team of forty engineers maintains a five-year-old Python monolith: 400k
lines, one Postgres database, a 52-minute test suite, weekly releases requiring a
coordinated freeze, and four teams that block each other constantly. Leadership has approved
"a move to microservices" and asked you to lead it.

Write the plan. It must include:

- Your diagnosis: which of the stated problems distribution actually solves, and which it
  does not.
- What you would measure in the first month, and what each measurement would tell you.
- The alternative you would present alongside the approved plan, with its cost.
- If splitting: how you would choose the first boundary, and the eight-step analysis you
  would run on it.
- The data problem, in detail — this is the largest cost and most plans omit it.
- What you would do about the 52-minute test suite and the release freeze *first*, and why
  the ordering matters.
- The organizational preconditions, and what happens if they are not met.
- How you would know, in six months, whether it was working — and what result would make you
  stop.

Marks are for diagnosis and sequencing. A plan that begins by splitting services scores
poorly.

**B2. (40)** Design the architecture for a system with a genuinely hard domain: a
subscription billing platform handling proration, mid-cycle plan changes, usage-based
charges, tax by jurisdiction, dunning and retries, refunds, and credits — for customers
across time zones.

Your answer must cover:

- The bounded contexts, with the language differences that justify each boundary, and which
  is the core domain.
- The value objects, with the rules each owns.
- The aggregates, with the invariants that determined their boundaries, and for each
  cross-aggregate invariant: instant or eventual, with the detection and correction.
- The consistency table (L07 §2.7) for at least six invariants.
- The transaction boundaries and where external calls happen relative to them.
- The layering level chosen (L05 §2.6) and why.
- Three fitness functions you would write on day one.
- Three decisions you would record as ADRs, with the options you would reject.
- What you would deliberately not build, and what you would buy.

Marks are for the boundary reasoning and for honesty about what is eventual.

**B3. (40)** Argue for or against:

> "Software architecture, as a distinct discipline, adds little. Systems survive or fail on
> the quality and continuity of the teams maintaining them; the structural choices that
> architects agonize over are either forced by the domain or are cheaply reversible. The
> observable correlation between 'good architecture' and successful systems is confounded by
> the fact that competent teams produce both."

Required: state the opposing position at its strongest; give at least four specific
mechanisms as evidence, drawn from this course, with concrete costs and benefits;
distinguish the claims that are empirically testable from those that are not, and say what
evidence exists for the testable ones (be honest that it is thin); address Conway's Law,
which cuts both ways; and conclude with a falsifiable claim about what would change your
mind.

Even-handedness is assessed. An answer that does not state the opposing case fairly cannot
score above 24.

---

**B4. (40)** You inherit a system described by its team as "microservices". It has eleven services;
all of them read and write the same PostgreSQL database; each has its own deployment pipeline;
changes to the shared schema require coordinating releases across an average of four services;
end-to-end tests are the only tests that catch integration defects and they take fifty minutes; and
the team reports that velocity has fallen every quarter for two years.

Write the assessment. Your answer must: name what this architecture actually is, precisely, and
explain which specific property of microservices it lacks; identify the coupling forms present using
the connascence taxonomy, ranked by strength; explain why the deployment independence they have
achieved is worthless without the property they are missing; state what you would do first and why
that first — being specific that the first move must be cheap, reversible, and diagnostic; give the
sequence after that, with the reasoning for the ordering; explain what you would do about the shared
database, addressing both the technical migration and the reason it will be resisted; and state the
outcome you would accept — including whether "back to a modular monolith" is on the table and under
what conditions you would recommend it.

Marks are for the diagnosis and the sequencing. A plan that begins with the database migration has
not understood the risk.

**B5. (40)** A team is about to begin a rewrite. The existing system is eight years old, has no
tests, has a domain nobody has written down, serves real customers continuously, and is understood
in parts by four people, two of whom are leaving. Leadership has approved eighteen months.

Write the plan you would actually execute — and, if you would not execute a rewrite, argue that
instead.

Your answer must: state the case for and against a rewrite fairly, using the specific properties of
this situation rather than general principle; describe how you would recover the domain knowledge
before it walks out of the door, with a concrete method and a timebox; describe the
characterisation-testing strategy and what you would do about the bugs it will encode; give the
migration architecture — strangler fig, branch by abstraction, or something else — with the reason
for the choice and the three hard parts of it in this specific case; state how the system keeps
serving customers throughout, including the data migration; identify the point of no return and what
you would want to be true before crossing it; and state the three fitness functions you would put in
place on day one to prevent the new system decaying into the old one's condition.

Then state what would make you abandon the plan at month nine, and what you would do instead.

---

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision and correct edge
cases; 3 for an example not drawn from the lessons; 2 for a stated limitation of your own
answer or a connection to another course.

Section B: a correct, complete answer is 24/40. The rest is judgement, sequencing, fairness
to the opposing case, and honesty about what you do not know.
