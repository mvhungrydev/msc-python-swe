# SE-521 · Lesson 01 — Modularity and Information Hiding

**Estimated study time:** 3.5 hours
**Prerequisites:** none
**Required reading before this lesson:** Parnas (1972). Five pages. Read it first.

---

## 1. Orientation

Parnas's 1972 paper is the founding document of software architecture, and its argument is
still routinely ignored fifty-four years later. He takes one small program — a KWIC index —
and decomposes it two ways.

**Decomposition 1**, by processing step: an input module, a circular-shift module, an
alphabetizer, an output module. This is the obvious decomposition. It matches the flowchart.
It is what most people draw.

**Decomposition 2**, by *secret*: each module hides one design decision. A line-storage
module hides how characters are stored. A circular-shift module hides whether shifts are
precomputed or computed on demand. And so on.

Then he asks: what happens when a requirement changes?

- Change the input format → decomposition 1 changes one module; decomposition 2 changes one
  module. Tie.
- Change from storing lines in arrays to storing them packed in one string → decomposition 1
  changes **every** module, because every module knows the storage format. Decomposition 2
  changes one.
- Change from precomputing all shifts to computing them lazily → decomposition 1 changes the
  shift module *and* everything downstream. Decomposition 2 changes one.

Same functionality, same modules-count, wildly different change cost. The difference is
what each module *knows*.

## 2. Theory

### 2.1 The criterion

> **A module should hide a design decision that is likely to change.**

Not "a step". Not "a noun in the requirements". A *decision*. And the decisions that matter
are the ones that are likely to change and that would otherwise be known in many places.

The practical procedure, and it is a procedure you can actually run:

1. List the decisions your system embodies. Storage format. Wire protocol. Pricing rule.
   Authentication mechanism. Retry policy. Ordering guarantee. Time zone handling.
2. For each, estimate: how likely is this to change, and if it changes, how many places
   currently know about it?
3. The high-likelihood, high-spread decisions are your module boundaries. Draw the module
   *around* the decision, so that it is known in exactly one place.

This produces boundaries that do not look like the flowchart, and that is the point.

### 2.2 "Secret" is the useful word

A module's **secret** is what it knows that nobody else does. Stating it out loud is a
sharp diagnostic:

- "The `PricingService` hides how discounts are calculated." Good — one decision, likely to
  change.
- "The `UserManager` hides... user stuff." No secret. This is a bucket, not a module.
- "The `Utils` module hides..." nothing. It is a namespace for homeless functions.
- "The `DatabaseHelper` hides which SQL we run." Plausible. But if callers must know that
  `get_user_with_orders` exists to avoid N+1, the secret leaks.

**Every module should have a one-sentence secret.** If you cannot write it, the module is
not a module; it is a folder. Run this test on ten modules in your codebase and the results
will be uncomfortable and useful.

### 2.3 Interface as contract, not as surface

A module's interface is not "its public methods". It is **everything a caller may rely on**,
which is more than you declared:

- The declared signatures and types.
- The exceptions it raises.
- Its performance characteristics. If a call is O(1) and callers depend on that, the
  complexity is part of the interface. Making it O(n) later is a breaking change even though
  the signature is unchanged.
- Its concurrency properties. Thread-safe or not; reentrant or not.
- Its side effects and their ordering.
- Its failure modes: does it retry? is it idempotent? is it atomic?
- Observable defaults: the order of results, the encoding of output, precision of numbers.

**Hyrum's Law**: with enough users, every observable behaviour of your system will be
depended upon by somebody, regardless of what you documented. That is empirical, not
normative, and the correct response is to *deliberately narrow* what is observable —
randomize iteration order in a set-like API, add jitter to a timestamp, or actively test the
undefined behaviour to keep it undefined.

### 2.4 Deep modules

Ousterhout's formulation, and the single most useful heuristic in this course:

> **Module quality = functionality provided ÷ interface complexity.**

A **deep** module has a small interface hiding substantial implementation. Unix file I/O —
`open`, `read`, `write`, `close`, `lseek` — hides device drivers, buffering, permissions,
filesystems, and journalling behind five calls. That is the exemplar.

A **shallow** module has an interface nearly as complex as what it hides. Common examples:

```python
class UserRepository:
    def get_by_id(self, id): return db.query("SELECT * FROM users WHERE id=%s", id)
    def get_by_email(self, e): return db.query("SELECT * FROM users WHERE email=%s", e)
    def save(self, u): ...
```

Every method is a thin pass-through. The interface has the same complexity as the SQL, adds
a hop, and hides nothing. It might still be worth having — as a *seam* for testing (SE-511
L02), or as the place where the mapping to domain objects lives — but you should be able to
say which, and "it's good practice" is not an answer.

The related anti-pattern is the **pass-through method**: a method that does nothing but call
another method with the same signature. Each one adds interface surface for zero
functionality. When you find a chain of them, ask which layer should own the behaviour and
delete the others.

### 2.5 Different layer, different abstraction

A corollary: if two adjacent layers have the same abstraction, one of them is probably
unnecessary. Symptoms:

- A `Service` whose methods are one-to-one with a `Repository`'s methods.
- A DTO that has exactly the same fields as the entity.
- A "façade" that forwards every call.

Each layer should *change the vocabulary*. `OrderService.place(order)` speaking in orders,
over `Repository.save(row)` speaking in rows, is a real layer. `OrderService.save_order`
over `OrderRepository.save_order` is not.

This is where genuine judgement is required, because the counter-argument is real: a
pass-through today may be the seam you need tomorrow. Ousterhout's answer, which is right:
add it *tomorrow*. The cost of introducing a layer later is usually low and localized; the
cost of maintaining an unnecessary one is paid continuously.

### 2.6 Errors are part of the interface

Ousterhout's most contrarian claim, and it holds up: **define errors out of existence where
you can.**

Examples where an API removed a failure mode rather than reporting it:

- Python's `str.split()` on a string with no separator returns `[s]`, not an error.
- `dict.get(k, default)` instead of a `KeyError` the caller must handle.
- Unix `unlink` on an open file succeeds; the file disappears when the last handle closes,
  so there is no "file in use" error to handle.
- A `delete(id)` that is idempotent: deleting a non-existent thing is success, not
  `NotFound`. The caller wanted it gone; it is gone.

Every exception a module raises is a branch every caller must consider. Reducing the number
of distinct failure modes reduces the interface's complexity more than reducing the number
of methods does.

The counter-discipline: **do not define errors out of existence by ignoring them.** Silently
returning a default for a genuinely exceptional condition is how data corruption happens.
The test is whether the "non-error" outcome is *correct and useful* for the caller, or
merely convenient for you.

### 2.7 Information hiding versus encapsulation

They are not the same and conflating them is why "make the fields private and add getters"
became a habit that provides nothing.

- **Encapsulation** is a language mechanism: restricting access to internals.
- **Information hiding** is a design property: callers do not *know* the internals, so the
  internals can change.

A class with private fields and a getter and setter for every one is fully encapsulated and
hides nothing: the field structure is public in all but name, and changing it breaks callers
just the same. Python, with no real access control, makes this obvious — a leading
underscore is a convention, and the design work is what matters, not the enforcement.

## 3. Construction: decomposing a system two ways

Take a system you know: a report generator, an import pipeline, a notification service.

**Step 1 — decomposition by processing step.** Draw it. It will come easily; this is how
everyone naturally thinks.

**Step 2 — list the design decisions.** Aim for fifteen. Force yourself past the obvious.
For a notification service:

| Decision | Likely to change? | Places that know it now |
|---|---|---|
| Which channels exist (email, SMS, push) | yes | 6 |
| Template format and engine | yes | 3 |
| How user preferences are stored | yes | 4 |
| Retry policy per channel | yes | 5 |
| Rate limiting per user | yes | 2 |
| Time-zone handling for quiet hours | yes | 7 |
| Delivery receipt tracking | maybe | 3 |
| Whether sends are batched | yes | 8 |
| The idempotency key scheme | no | 2 |
| Vendor for each channel | yes | 1 (good) |

The "places that know it now" column is the finding. A decision known in eight places is a
change that touches eight files, and that is a module boundary you failed to draw.

**Step 3 — decomposition by secret.** Draw modules around the high-likelihood, high-spread
decisions. Write each module's one-sentence secret.

**Step 4 — run the change experiment.** Take five plausible requirement changes and count
the modules each touches, in both decompositions. This is Parnas's method and it converts
the argument from aesthetics into arithmetic.

| Change | Decomposition 1 | Decomposition 2 |
|---|---|---|
| Add a Slack channel | 6 modules | 1 |
| Switch template engines | 3 | 1 |
| Respect quiet hours per timezone | 7 | 1 |
| Batch sends per user per hour | 8 | 2 |
| Change email vendor | 1 | 1 |

**Step 5 — be honest about the cost.** Decomposition 2 is not free. It typically has more
indirection, more interfaces, and worse locality for a reader tracing one request. Count
that too: for each decomposition, how many files must be opened to answer "what happens when
a notification is sent?" (L10 of PY-502's reader trace).

The right answer is usually a hybrid, and the deliverable is the *reasoning*, not a
doctrinal choice.

## 4. Failure modes

- **Decomposing by processing step** because it matches the flowchart. §1.
- **Decomposing by data-table** — one module per database table. This makes the schema the
  architecture, and every schema change an architectural one.
- **Modules with no secret.** `utils`, `helpers`, `common`, `core`, `managers`. If it cannot
  be named by what it hides, it is a bucket.
- **Shallow modules and pass-through methods.** §2.4.
- **Adjacent layers with the same abstraction.** §2.5.
- **Leaky interfaces**: a caller must know the implementation to use it correctly (must
  call `prefetch()` first to avoid N+1; must not call it concurrently; must handle three
  exception types from a lower layer).
- **Interface as "the public methods".** §2.3 — performance, concurrency, and failure
  behaviour are part of it.
- **Encapsulation mistaken for information hiding.** §2.7.
- **Hiding something that never changes.** An abstraction over the `math` module buys
  nothing.
- **Hiding something that must vary per call site.** If every caller passes a different
  strategy, the "hidden" decision was actually the caller's.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write the one-sentence secret for ten modules in a codebase you work on. Report how
many you could not write.

**W2.** Find a pass-through method chain of length ≥2. Delete the middle. Report what broke.

**W3.** Find a leaky interface — one where a caller must know an implementation detail to
use it correctly. Write down the detail and how a caller would discover it.

### Core (2 h)

**C1 — The two decompositions.** Complete §3, all five steps, for a real system.
Deliverable: both diagrams, the fifteen-decision table with the "places that know it"
column, the module secrets, the five-change comparison table, and the reader-trace cost of
each. Recommend a decomposition — possibly a hybrid — and defend it in 600 words.

**C2 — Depth ratio.** For eight modules, compute lines-of-interface (signatures, docstrings,
exported names) against lines-of-implementation-hidden. Rank them. For the three shallowest,
decide: delete, merge, or deepen. Do one of them and report the diff.

**C3 — Define an error out of existence.** Find an API in your system that raises an
exception callers routinely catch and handle the same way. Redesign so the error does not
exist. Then argue the counter-case: what is now silently accepted that should not be? Decide,
and say why.

**C4 — Hyrum's Law in practice.** Take a module with users you do not control. Identify
three observable behaviours that are not part of the documented interface but that callers
could depend on. For each: would changing it break someone? Propose a way to make it
*unobservable* (randomization, explicit tests asserting non-determinism, deprecation
telemetry).

### Challenge

**X1.** Re-do Parnas's KWIC exercise in full, in Python: implement both decompositions,
implement all four of his change scenarios in each, and report the actual diffs (files
touched, lines changed). Then extend it with a change he did not consider and see whether
decomposition 2 still wins. Report honestly if it does not.

**X2.** Read Parnas (1972), Parnas (1979) "Designing Software for Ease of Extension and
Contraction", and Ousterhout chs. 4–6. Write 1,500 words on the relationship between
Parnas's "likely to change" criterion and Ousterhout's "deep module" criterion: are they the
same claim in different vocabulary, do they ever disagree, and which is more useful as a
design procedure? Give an example where they point in different directions.

## 6. Self-check

1. State Parnas's criterion. Why does decomposition by processing step fail?
2. What is a module's "secret", and what is the diagnostic that follows?
3. Name six things that are part of an interface beyond the signatures.
4. State Hyrum's Law and the correct response to it.
5. Define deep and shallow modules with an example of each.
6. What does "different layer, different abstraction" rule out?
7. What does it mean to define an error out of existence, and when is it wrong?
8. Distinguish encapsulation from information hiding.

## 7. Primary sources

- **Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM, 1972).**
  If you read one paper in this program, read this one.
- Parnas, "Designing Software for Ease of Extension and Contraction" (1979).
- Ousterhout, *A Philosophy of Software Design*, chs. 4–6, 10.
- Winters et al., *Software Engineering at Google*, ch. 1 (for Hyrum's Law, which originates
  there).

---

**Next:** [L02 — Coupling, Cohesion, and the Cost of Change](L02-coupling-cohesion-change.md)
