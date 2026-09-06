# SE-521 · Lesson 02 — Coupling, Cohesion, and the Cost of Change

**Estimated study time:** 3.5 hours
**Prerequisites:** L01

---

## 1. Orientation

"Low coupling, high cohesion" is the most repeated and least operationalized advice in
software engineering. Repeated because it is true; unoperationalized because almost nobody
can say, of a specific piece of code, *how much* coupling it has and to what.

This lesson makes both terms measurable. The measurement matters because it converts
architecture arguments from taste into evidence — and because the metric that actually
predicts pain is not either of them individually, but **the number of modules a typical
change touches.**

## 2. Theory

### 2.1 Coupling, graded

The classic scale (Stevens, Myers & Constantine, 1974), from worst to best. It is old and
it is still the sharpest tool available:

| Level | Description | Example |
|---|---|---|
| **Content** | One module reaches into another's internals | `other._items.append(x)`, monkey-patching |
| **Common** | Modules share global mutable state | a module-level config dict everyone mutates |
| **External** | Modules share an externally imposed format | both parse the same CSV layout independently |
| **Control** | One passes a flag telling the other *how* to behave | `process(data, mode="legacy")` |
| **Stamp** | Passing a whole structure when only part is needed | `f(user)` when `f` needs `user.email` |
| **Data** | Passing exactly the data needed | `f(email: str)` |
| **Message** | Communicating only through messages/events | publishing `OrderPlaced` |

Two clarifications that make this usable:

**Control coupling is the one people miss.** A boolean or enum parameter that selects
behaviour means the caller knows the callee's internal structure. It also means the callee
has two responsibilities. `render(data, as_html=True)` should be `render_html(data)` and
`render_text(data)`, or a strategy passed in. Each flag doubles the paths through the
function and the tests you owe.

**Stamp coupling is not always wrong.** Passing a whole `Order` to a function that needs
three fields is stamp coupling, and it may be the right call: it keeps the signature stable
as needs change, and it keeps a domain concept intact rather than smearing it into
primitives. The genuine harm is when it forces a *dependency*: a pure calculation function
that takes an ORM-mapped entity now depends on the ORM, cannot be tested without one, and
cannot be reused. The rule: pass the concept when the callee is in the same domain layer;
pass the data when it crosses a boundary.

### 2.2 Two dimensions people conflate

**Afferent coupling (Ca)** — how many modules depend *on* this one. High Ca means changes
here are expensive; it is a *stability requirement*, not a defect. Your `Money` type should
have high Ca and change rarely.

**Efferent coupling (Ce)** — how many modules this one depends on. High Ce means this module
breaks often; it is *fragile*.

Martin's derived metrics:

- **Instability** `I = Ce / (Ca + Ce)` — 0 is maximally stable (everyone depends on it, it
  depends on nothing), 1 is maximally unstable.
- **Abstractness** `A` = abstract types ÷ total types.
- **The Stable Abstractions Principle**: `A + I ≈ 1`. A module everyone depends on should be
  abstract; a module that depends on everything should be concrete. Distance from that line
  is `D = |A + I − 1|`.

Treat these as *diagnostics, not targets* (SE-511 L03's Goodhart warning applies). Their
value is in the outliers: a module with high Ca and low A is a **concrete thing everyone
depends on** — the classic "changing this breaks the world" module, and worth finding.

You can compute all of this from the import graph in an afternoon. `pydeps`, `import-linter`,
or forty lines of `ast` walking (SE-511 L08).

### 2.3 Cohesion, graded

From worst to best (same lineage):

| Level | Description |
|---|---|
| **Coincidental** | Grouped arbitrarily — `utils.py` |
| **Logical** | Grouped by category, selected by a flag — "all the validators, pick one" |
| **Temporal** | Grouped by when they run — `startup.py` |
| **Procedural** | Grouped by order of execution |
| **Communicational** | Operate on the same data |
| **Sequential** | Output of one is input to the next |
| **Functional** | All contribute to one well-defined task |

The practical test: **can you describe what this module does in one sentence, without
"and"?** If the sentence needs "and", you have at least two modules.

A subtlety worth stating: **low coupling and low cohesion together is still bad**, and it is
a common state. A module of unrelated functions that happens to import nothing scores well
on coupling and is still a bucket — nobody can find anything in it, and every change adds
another unrelated function. Coupling is a property of relationships; cohesion is a property
of purpose. Both must hold.

### 2.4 Connascence: a finer instrument

Meilir Page-Jones's refinement, which is more useful than the coupling scale for
*day-to-day* decisions. Two pieces of code are **connascent** if changing one requires
changing the other. The forms, from weakest to strongest:

**Static** (visible in the source):

1. **Name** — both use the same name. Weakest, unavoidable, fine.
2. **Type** — both agree on a type.
3. **Meaning** — both agree what a value means. `status == 3` — a magic number. Fix with an
   enum, which converts it to connascence of name.
4. **Position** — both agree on order. `f(1, 2, 3)`. Fix with keyword arguments.
5. **Algorithm** — both must use the same algorithm. Two services independently computing
   the same hash or checksum. Fix by sharing, or by making one authoritative.

**Dynamic** (only visible at runtime — strictly worse, because it cannot be found by
reading):

6. **Execution order** — `a.open()` must precede `a.read()`. Fix by making it impossible:
   a context manager, or a factory that returns an already-open object.
7. **Timing** — a race; correctness depends on relative speed.
8. **Value** — two values must change together (a `start` and `end` that must stay ordered;
   a denormalized count and the rows it counts).
9. **Identity** — two references must be to the *same* object.

Three rules that make this actionable:

- **Minimize overall connascence** by decomposing into encapsulated modules.
- **Minimize the strength** of connascence that crosses boundaries. Inside one small
  function, connascence of position is fine. Across a service boundary, it is a bug waiting
  for a deploy-order mismatch.
- **Maximize connascence within** a module: things that must change together should live
  together. This is cohesion, restated with a mechanism.

The reason to prefer connascence over the 1974 scale: it tells you *what to do*. "This is
connascence of meaning across a module boundary" implies "introduce a named type". "This is
control coupling" implies less.

### 2.5 The metric that actually matters

All of the above are proxies. The thing you actually care about is:

> **How many modules does a typical change touch?**

Ousterhout calls the failure "change amplification": a small conceptual change requires
edits in many places. You can measure it directly, from git:

```bash
git log --format=%H --since="1 year ago" | while read c; do
  git show --name-only --format= "$c" | xargs -r dirname | sort -u | wc -l
done | sort -n | uniq -c
```

That gives the distribution of "modules touched per commit". The tail — commits touching
eight or more modules — is where your architecture is failing, and reading those commits
tells you *which* boundary is wrong far more reliably than any static metric.

The companion measure is **temporal coupling**: pairs of files that change together. Files
that always change together belong together (a cohesion failure); files in the same module
that never change together may be two modules.

```bash
# pairs of files co-occurring in commits, most frequent first
git log --format="%H" --name-only | awk 'BEGIN{RS=""} {print}' | ...
```

Adam Tornhill's *Your Code as a Crime Scene* is the systematic treatment; the technique is
called *change coupling analysis* and it is the single most useful thing you can do with
your git history.

### 2.6 Coupling you cannot see in the import graph

The import graph misses several real couplings, and these are the ones that hurt:

- **Database schema.** Two services reading the same table are coupled as tightly as if they
  shared a global variable — *common coupling*, the second-worst level. This is the single
  most common hidden coupling in service architectures.
- **Message formats.** A producer and consumer agreeing on a JSON shape, with no schema, is
  external coupling.
- **Deployment order.** If A must be deployed before B, they are coupled in time.
- **Shared libraries with behaviour.** A common library that both services must upgrade in
  lockstep couples their release cycles.
- **Configuration.** A feature flag that two services both read and interpret.
- **Semantic coupling.** Two modules that independently implement the same business rule.
  Nothing links them; both must change when the rule changes; nothing will remind you.

For each of these, the mitigation is the same shape: *make the coupling explicit and
versioned*. A schema registry, a published contract, an API version, an ADR. An implicit
coupling is one that will be violated by someone who did not know it existed.

## 3. Construction: measuring a real system

**Step 1 — the import graph.** Build it with `ast` (SE-511 L08) or `pydeps`. Compute Ca, Ce,
I, and A per package. Plot A against I and mark the main sequence.

**Step 2 — the outliers.** Identify: modules with high Ca and low A (concrete and depended
upon — the fragile core); modules with high Ce (fragile); and cycles (a cycle means the
modules in it are, for change purposes, one module — say so out loud).

**Step 3 — the change data.** From git: distribution of modules-touched-per-commit, and the
top twenty co-changing file pairs.

**Step 4 — reconcile.** The interesting result is where the static and historical measures
*disagree*:

- Static coupling low, change coupling high → **hidden coupling** (§2.6). Find it. This is
  the most valuable finding available and it is invisible without doing both analyses.
- Static coupling high, change coupling low → the dependency is stable and probably fine.
  Do not "fix" it.

**Step 5 — connascence audit on the worst boundary.** Take the pair of modules with the
highest change coupling. Enumerate every connascence between them by form. Classify each as
static or dynamic. Propose a change that reduces the strongest one.

**Step 6 — write it up** with a recommendation and a cost. One boundary, one change, one
number for the expected improvement, and how you would verify it in six months.

## 4. Failure modes

- **Treating coupling metrics as targets.** Refactoring to improve `I` without changing what
  a change costs is theatre.
- **Missing control coupling.** Boolean parameters everywhere, unremarked.
- **Ignoring hidden coupling.** §2.6. Shared database tables especially.
- **Optimizing coupling while ignoring cohesion.** A well-decoupled bucket is still a
  bucket.
- **Cyclic dependencies tolerated.** A cycle is a single module wearing several file names,
  and it defeats every layering rule you write.
- **"Low coupling" used to justify indirection.** An interface with one implementation
  reduces the metric and not the change cost.
- **Never looking at git history.** All the static analysis in this lesson is a proxy for
  something you can measure directly.
- **Confusing stable with rigid.** High Ca is *fine* for a module that should not change; it
  is a problem only for one that must.

## 5. Exercises

### Warm-up (25 min)

**W1.** Classify ten function signatures from your codebase on the coupling scale. Report
how many are control-coupled.

**W2.** Find an instance of each connascence form 3–8 in a real codebase. Some will take
effort; the dynamic ones are the interesting ones.

**W3.** Run the modules-touched-per-commit distribution on a repository. Read the three
commits that touched the most modules and say what boundary each reveals.

### Core (2 h)

**C1 — Full measurement.** Complete §3, all six steps. Deliverable: the A/I plot, the outlier
list, the change-coupling data, the reconciliation table (this is the assessed part), the
connascence audit of the worst boundary, and the one-change recommendation with a verification
plan.

**C2 — Remove control coupling.** Find the function in your codebase with the most behaviour
flags. Split it. Report: total lines before/after, number of distinct code paths before/after,
number of test cases needed for full path coverage before/after. The last number is usually
the persuasive one.

**C3 — Hidden coupling hunt.** For a system with more than one deployable, enumerate every
coupling *not* visible in the import graph: shared tables, message formats, deployment
ordering, shared libraries, flags, duplicated business rules. For each, propose how to make
it explicit and versioned. Estimate the cost.

**C4 — Break a cycle.** Find a dependency cycle. Break it, using one of: dependency inversion
(L03), extracting a shared module, moving a function, or merging the modules honestly.
Report which you chose and why, and add a fitness function (L08) preventing its return.

### Challenge

**X1.** Read Page-Jones on connascence and Tornhill's *Your Code as a Crime Scene*, chs. 4–7.
Build a tool that computes change coupling from git history and cross-references it with
static import coupling, producing a ranked list of "suspicious" pairs. Run it on a large
open-source project. Report the top ten and hand-analyse three.

**X2.** Take a system with two or more services sharing a database. Design the migration to
eliminate the shared-table coupling: who owns which table, what API replaces the direct
reads, how the migration proceeds without downtime, and how you prevent regression. Estimate
the effort honestly. Then argue the case for *not* doing it — there is one, and stating it
fairly is worth as much as the plan.

## 6. Self-check

1. Give the coupling scale from worst to best, with an example of each.
2. Why is control coupling worse than it looks, and what is the fix?
3. When is stamp coupling acceptable and when is it harmful?
4. Distinguish afferent from efferent coupling, and say which one high values are fine for.
5. Give the connascence forms in order, and the three rules for using them.
6. Why is dynamic connascence worse than static?
7. What is the metric that all the others are proxies for, and how do you measure it?
8. Name five couplings invisible in the import graph.

## 7. Primary sources

- Stevens, Myers & Constantine, "Structured Design" (IBM Systems Journal, 1974) — the origin
  of the coupling and cohesion scales.
- Page-Jones, *What Every Programmer Should Know About Object-Oriented Design* (1995) —
  the connascence chapters.
- Martin, "Design Principles and Design Patterns" (2000) — the Ca/Ce/I/A metrics.
- Tornhill, *Your Code as a Crime Scene*, 2nd ed. — change coupling from version history.
- Ousterhout, ch. 2 (the nature of complexity: change amplification, cognitive load,
  unknown unknowns).

---

**Previous:** [L01](L01-modularity-and-information-hiding.md) · **Next:**
[L03 — Dependency Inversion, Composition, and Their Limits](L03-dependency-inversion.md)
