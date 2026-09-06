# SE-521 — Written Examination

**Time allowed: 3 hours. Closed book.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

---

## Section A — answer FOUR

**A1.** State Parnas's decomposition criterion and explain, using his KWIC example or one of
your own, why decomposition by processing step fails under change. Then define a module's
"secret" and give the diagnostic test that follows from it. *(15)*

**A2.** Name six things that are part of a module's interface beyond its declared
signatures. State Hyrum's Law and give the correct response to it. Then define deep and
shallow modules and give an example of each from code you know. *(15)*

**A3.** Give the coupling scale from worst to best with an example of each. Explain why
control coupling is worse than it appears and what the fix is. Then explain when stamp
coupling is acceptable and when it is harmful. *(15)*

**A4.** List the nine forms of connascence in order and state the three rules for using
them. Explain why dynamic connascence is strictly worse than static. Then name five
couplings that are invisible in the import graph. *(15)*

**A5.** State the Dependency Inversion Principle precisely and identify the word that does
the work. Explain where the interface must live and why the common mistake inverts nothing.
Then give four situations in which applying DIP makes a system worse. *(15)*

**A6.** Explain the fragile base class problem with a concrete example, and show why
composition cannot have an equivalent failure. Then state the inherit-versus-compose rule
and justify it with reference to Liskov substitutability. *(15)*

**A7.** State Norvig's critique of design patterns at its strongest, and give three patterns
where it clearly holds and two where it does not. Then take Observer and list four
consequences that are undefined by default in most implementations. *(15)*

**A8.** Draw the ports-and-adapters layering and state the dependency rule. Distinguish
driving from driven ports and say which requires an interface, with the reason. Then give
five honest costs of this architecture and the conditions under which it is not worth
paying. *(15)*

**A9.** Define the ubiquitous language and explain what an ambiguity in it indicates. Then
define a bounded context, give four ways to find boundaries, and explain what an
anticorruption layer protects against. *(15)*

**A10.** Give the four rules of aggregate design. State the sizing trade-off and the
four-step procedure for determining boundaries. Then work through the credit-limit example
and explain what asking the business reveals about the supposed invariant. *(15)*

**A11.** Explain the lost-update problem and why read-committed isolation does not prevent
it. Give three concurrency-control mechanisms with the costs of each. Then describe the
outbox pattern, what problem it solves, and what it requires of consumers. *(15)*

**A12.** State Waldo et al.'s claim about distributed computing and give three outcomes a
remote call has that a local one does not. Then give five criteria for where to cut a
service boundary, and describe the distributed monolith and its symptoms. *(15)*

**A13.** Define a fitness function and give five design rules for one that survives contact
with a team. Then state Fowler's reframing of architecture in terms of reversibility and its
corollary for design. *(15)*

**A14.** Give Feathers's definition of legacy code and the strategy it implies. Describe the
characterization-test procedure and explain why it deliberately encodes bugs. Then name five
seam types with their costs and say which is the crowbar and why it is temporary. *(15)*

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

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision and correct edge
cases; 3 for an example not drawn from the lessons; 2 for a stated limitation of your own
answer or a connection to another course.

Section B: a correct, complete answer is 24/40. The rest is judgement, sequencing, fairness
to the opposing case, and honesty about what you do not know.
