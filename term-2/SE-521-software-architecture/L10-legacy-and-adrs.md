# SE-521 · Lesson 10 — Legacy Systems, Seams, and Architectural Decision Records

**Estimated study time:** 3.5 hours
**Prerequisites:** all of SE-521

---

## 1. Orientation

Nine lessons of this course have been about designing systems. You will spend most of your
career on systems someone else designed, under constraints nobody recorded, containing
decisions whose reasons are lost.

Feathers's definition is the useful one: **legacy code is code without tests.** Not old
code, not bad code — code you cannot change safely, because nothing tells you when you have
broken it. That definition points directly at the strategy: get tests around it, then
change it.

The second half of this lesson is the counterpart: **writing down decisions so that the code
you are producing today does not become the legacy that defeats someone in 2031.** The
distinguishing feature of a system that ages well is not its architecture; it is that
somebody wrote down *why*.

## 2. Theory

### 2.1 The legacy dilemma and characterization tests

To change code safely you need tests. To write tests you need to change the code (to
introduce seams). That is the dilemma, and it is resolved by **characterization tests**:
tests that document what the code *currently does*, not what it should do.

The procedure:

1. Find the smallest piece you can invoke.
2. Write a test asserting something you know is wrong (`assert result == "TODO"`).
3. Run it; the failure message tells you the actual value.
4. Paste the actual value in.
5. Repeat until the behaviour is pinned.

You now have a safety net. It encodes bugs as well as features — deliberately. Your job at
this stage is not to fix anything; it is to be able to detect change. When you later fix a
bug, a characterization test will fail and you will *deliberately* update it, which is a
record of the fix.

Where the input space is large, **approval testing** scales this: capture the output of a
run on realistic input to a file, review it once, commit it, and thereafter diff. This works
well for report generators, serializers, formatters, and any batch process. `pytest`'s
`--snapshot-update`-style plugins do the mechanics.

Where a reference implementation exists (the old system, still running), **differential
testing** is stronger: run both on the same inputs and compare. This is the safest possible
way to replace a system, and it is what makes the strangler fig (§2.3) verifiable.

### 2.2 Seams

A **seam** is a place where you can change behaviour without editing that code. Feathers's
taxonomy, adapted for Python:

| Seam type | Mechanism | Cost |
|---|---|---|
| **Object seam** | Pass a collaborator in instead of constructing it | requires editing the constructor — but a small, safe edit |
| **Parameter seam** | Add a parameter with a default | smallest possible change |
| **Import seam** | `mock.patch("module.name")` | no edit at all; brittle (SE-511 L02 §2.5) |
| **Subclass seam** | Override a method in a test subclass | works, and couples the test to structure |
| **Environment seam** | Configuration or environment variable | crude, and it leaks into production |

The strategy for untestable code: use the *cheapest* seam that lets you write a
characterization test, get the test, then refactor toward a *better* seam (usually the
object seam), then delete the crude one.

`mock.patch` is the crowbar. It works on code you cannot otherwise touch, and it is
explicitly temporary: every `patch` in your test suite is a dependency that was not injected
(SE-511 L02 §2.5). Counting them over time is a useful measure of a legacy migration's
progress.

The specific hard cases and their standard resolutions:

- **A constructor that does I/O.** Extract the work into a method; test the method.
- **Global state / module-level singletons.** Introduce a parameter with the global as its
  default; migrate callers; remove the default.
- **A god class.** Sprout a new class for the new behaviour rather than adding to it
  ("sprout class"); do not attempt to decompose it before you have tests.
- **A long method you must change.** "Sprout method": write the new logic as a new, tested
  method and call it from one line in the untested one. The untested code grows by one line;
  the new logic is fully covered.
- **A dependency on the clock, randomness, or the network.** Parameter seam with a default.

Feathers's guidance, which is worth stating because it contradicts instinct: **do not
refactor before you have tests, and do not write tests that require large refactors.** Find
the smallest seam.

### 2.3 Strangler fig

Fowler's pattern, named for the vine that grows around a tree until the tree dies and the
vine stands on its own.

1. Put a **façade** (proxy, router, gateway) in front of the old system. All traffic goes
   through it; behaviour is unchanged.
2. Implement **one capability** in the new system.
3. **Route** that capability's traffic to the new implementation. Verify — ideally by
   running both and comparing (§2.1's differential testing, sometimes called "dark
   launching" or "parallel run").
4. Repeat, capability by capability.
5. When nothing routes to the old system, delete it.

Why it beats a rewrite: value is delivered continuously, risk is bounded per step, each step
is reversible by flipping a route, and there is never a big-bang cutover. The Netscape
rewrite and its many descendants are the argument against the alternative — Spolsky's
"Things You Should Never Do" is worth reading once, critically.

The hard parts, which are where these projects actually fail:

- **Shared data.** Both systems reading and writing the same tables. Options: the new system
  calls the old for data (slow, coupled); dual writes (consistency problems); or CDC/event
  streaming to keep a new store in sync (complex, and the standard answer at scale). Decide
  early; this is usually the largest cost.
- **The façade itself** becomes a critical, complex component.
- **The last 10%.** The remaining capabilities are the weird ones — the reason they were left
  until last. Projects stall here permanently, leaving two systems forever. The fix is a
  ratchet (L08 §2.6) and a *dated commitment to delete the old system*, decided at the start.
- **Behaviour parity is not always desirable.** Some old behaviour is a bug. Decide
  explicitly, per capability, whether you are replicating or correcting — and if correcting,
  who tells the users.

### 2.4 Branch by abstraction

For changes *inside* a system, where a strangler façade is too heavy:

1. Introduce an abstraction over the thing you want to replace.
2. Migrate all callers to the abstraction (behaviour unchanged).
3. Build the new implementation behind the abstraction.
4. Switch, one caller or one flag at a time.
5. Remove the old implementation and, usually, the abstraction.

Every step is mergeable to trunk. This is how you perform a months-long refactor without a
long-lived branch (SE-511 L10 §2.4), and it is the technique that makes trunk-based
development possible for large changes.

Step 5's "usually" matters: if the abstraction only ever had two implementations and one is
now gone, delete it (PY-502 L10 §2.6).

### 2.5 Architectural decision records

An ADR records **one decision**: the context, the options, the choice, and the consequences.
Short — one page. Immutable — you do not edit a decision, you supersede it.

```markdown
# ADR-014: Use optimistic concurrency for Order aggregates

- **Status:** Accepted (2026-03-11). Supersedes ADR-009.
- **Deciders:** M. Velasco, A. Okafor
- **Context:**
  Order updates are concurrent (customer + support agent + async payment webhook).
  We observed 14 lost updates in Q4, all in the payment-status field. Read-committed
  isolation does not prevent them. Peak contention is ~3 writes/sec on the hottest order.

- **Options considered:**
  1. **Pessimistic locking** (`SELECT FOR UPDATE`). Simple; serializes writers; risks
     deadlock with the existing shipment lock ordering; adds lock hold time across the
     payment gateway call, which we already want to remove.
  2. **Optimistic concurrency** with a version column. No locks; explicit conflict; callers
     must retry. Retry logic needed in 3 call sites.
  3. **Serializable isolation** for these transactions. Correct; measured 40% throughput
     reduction on the shared pool in our load test; affects unrelated transactions.

- **Decision:** Option 2. Contention is low enough that retry cost is negligible, and it
  removes lock hold time across external calls, which unblocks ADR-015 (outbox).

- **Consequences:**
  - Callers must handle `ConcurrentModification`. Added to the API error contract.
  - Fitness function: a test asserting a version conflict is raised under concurrent writes.
  - Under high contention this degrades (retry storms). Revisit if any single order exceeds
    ~20 writes/sec; alert added.
  - Batch update paths must be rewritten (3 of them, tracked in TICKET-882).
```

**What makes an ADR worth reading in three years:**

- **The options you rejected, with why.** This is 80% of the value. Without it, a future
  reader assumes you did not consider the obvious alternative and re-litigates it.
- **The context as it was.** Numbers, constraints, what you knew. Not "we needed
  scalability" — "peak was 3 writes/sec and growing 10% monthly."
- **The consequences, including the bad ones.** An ADR listing only benefits is marketing.
- **The trigger for revisiting.** "Revisit if X" turns a decision into something that can be
  re-examined on evidence rather than on vibes.
- **Dated and immutable.** Superseding creates a chain a reader can follow.

**What to record:** anything expensive to reverse (L08 §2.5). Datastore choice, service
boundaries, the consistency model, authentication, the public API contract, language and
framework choices, and any decision you had an argument about. Not: naming conventions,
library choices you could swap in a day, or anything already covered by a fitness function.

Keep them in the repository, in `docs/adr/`, numbered. Link fitness functions to them and
them to fitness functions.

### 2.6 Working on a system you did not design

A short procedure for the first month:

1. **Get it running locally**, and write down every undocumented step. That list is your
   first contribution.
2. **Read the composition root / entry points.** They tell you the shape.
3. **Read the schema.** In most business systems, the data model *is* the domain model,
   accurately or not.
4. **Run the change-coupling analysis** (L02 §2.5). It tells you where the real boundaries
   are, as opposed to the intended ones.
5. **Find the tests and run them.** Their absence, shape, and speed tell you what the team
   believed was risky.
6. **Trace one request end to end** and write the trace down. Count the files.
7. **Ask three people the same architecture question** and note the differences. The
   disagreements are where the model is unclear.
8. **Write the first ADR retroactively**, documenting a decision the system embodies but
   nobody recorded. Circulate it. This is the single most useful thing a new senior engineer
   can do, and it costs an afternoon.

Resist the urge to rewrite. Chesterton's fence applies with unusual force in software:
the weird code is often weird because of a production incident in 2019.

## 3. Construction: rescuing a module

Take the worst-tested, highest-churn module you can find (L02's change data will identify
it).

**Step 1 — characterize.** Write characterization tests for its current behaviour, using
approval testing where the output is large. Do not fix anything. Record the coverage and the
mutation score (SE-511 L03) — the second number is the honest one.

**Step 2 — find the seams.** List every hard dependency: globals, constructors doing I/O,
direct imports of clients, `datetime.now()`, `random`. For each, identify the cheapest seam.

**Step 3 — introduce one seam properly.** Parameter or object seam, not `patch`. Verify the
characterization tests still pass.

**Step 4 — sprout.** Implement one *new* behaviour as a new, fully tested unit, called from
one line in the old code.

**Step 5 — branch by abstraction on one dependency.** Introduce the abstraction, migrate
callers, add the new implementation, switch, remove the old. Every step a separate commit
that could be merged.

**Step 6 — count the patches.** How many `mock.patch` calls does the module's test suite
use, before and after? That number is your migration metric.

**Step 7 — write the ADR.** For a decision this module embodies that nobody recorded.
Include the options that were presumably considered, marked as reconstruction rather than
history — being explicit about that distinction is what makes the document honest.

## 4. Failure modes

- **Refactoring before testing.** The order is: characterize, seam, change.
- **Trying to write "proper" tests first.** Characterization tests encode bugs on purpose.
- **The big rewrite.** Two systems, no delivery, and the new one accumulates the same
  complexity for the same reasons.
- **A strangler migration with no deletion date.** Stalls at 90% forever.
- **Ignoring the shared-data problem** until the third capability is migrated.
- **`mock.patch` everywhere, permanently.** The crowbar left in the wall.
- **ADRs written after the fact as documentation theatre**, with no rejected options.
- **Editing ADRs instead of superseding them.** Destroys the history that is the point.
- **ADRs for everything.** Thirty a month, nobody reads them; record what is expensive to
  reverse.
- **No ADRs at all.** Every decision gets re-argued, and the constraints become folklore.
- **Removing a fence without finding out why it is there.**

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a characterization test for a function you do not understand, using the
assert-wrong-value technique. Report what you learned about its behaviour.

**W2.** Find a hard dependency and introduce a parameter seam with a default. Verify no
caller changed.

**W3.** Count `mock.patch` occurrences per test file in a real suite. Rank the files. The
top of that list is your legacy hot spot.

### Core (2.5 h)

**C1 — Rescue a module.** Complete §3, all seven steps. Deliverable: the characterization
suite with its mutation score, the seams introduced, the sprouted unit, the
branch-by-abstraction commit sequence, the patch-count metric, and the retroactive ADR.

**C2 — An ADR log.** Write eight ADRs for a system you work on: at least four retroactive
(decisions already embodied) and four for decisions you are making now. Each must have
rejected options with reasons, consequences including bad ones, and a revisit trigger. Have
someone who knows the system review them for accuracy — the corrections are informative.

**C3 — Strangler plan.** For a legacy component, write the full plan: the façade, the
capability ordering with justification, the shared-data strategy, the verification method
per step (differential? approval? manual?), the reversibility at each step, and the deletion
date. Estimate the calendar time honestly.

**C4 — Onboarding procedure.** Run §2.6's eight steps on an unfamiliar codebase (an
open-source project counts). Produce the artifacts: the setup gaps, the request trace with
file count, the change-coupling map, and the three-people disagreement list. This is a
deliverable you can reuse every time you join a team.

### Challenge

**X1.** Read Feathers, *Working Effectively with Legacy Code*, chs. 1–10 and 20–25, and
Fowler's "StranglerFigApplication". Then take a real legacy system and produce a full
migration proposal: current state with measurements, target state, the sequence, the
verification strategy, the cost, the risks, and the conditions under which you would abandon
the migration. Present it as you would to funders, including the option of doing nothing —
which is sometimes right and is almost never presented fairly.

**X2.** Build an ADR tooling setup: a template, a numbering scheme, a linter checking that
every ADR has all required sections and that superseded ADRs link both ways, a generated
index, and a check that every fitness function references an ADR. Then backfill your
project's ADRs and report what you discovered you could not explain.

## 6. Self-check

1. Give Feathers's definition of legacy code and say what strategy it implies.
2. Describe the characterization-test procedure and why it deliberately encodes bugs.
3. Name five seam types and their costs.
4. What are "sprout method" and "sprout class" for?
5. Give the five steps of a strangler fig migration and its three hard parts.
6. Give the five steps of branch by abstraction and why every step is mergeable.
7. What makes an ADR worth reading in three years? Name four elements.
8. Give four of the eight steps for joining an unfamiliar system.

## 7. Primary sources

- Feathers, *Working Effectively with Legacy Code* (2004).
- Fowler, "StranglerFigApplication", "BranchByAbstraction", "ParallelRun" (bliki).
- Nygaard, "Documenting Architecture Decisions" (2011) — the original ADR post.
- Spolsky, "Things You Should Never Do, Part I" (2000) — read critically.
- Tornhill, *Software Design X-Rays* — measuring legacy hot spots.

---

**Previous:** [L09](L09-service-boundaries.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), and
[Term 3](../../term-3/PY-601-concurrency/syllabus.md).
