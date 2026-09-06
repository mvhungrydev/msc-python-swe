# SE-521 · Lesson 06 — Domain-Driven Design: Language, Model, Boundaries

**Estimated study time:** 4 hours
**Prerequisites:** L01–L05

---

## 1. Orientation

Domain-Driven Design is widely reduced to a set of building blocks — entity, value object,
aggregate, repository — which is the least important part of it. Evans's actual argument is
about **language and boundaries**: that the hardest part of software is not the code but
arriving at a model of the business that is precise enough to build on, and that the model
must be *bounded* because no single model can serve a whole organization.

The test of whether a team is doing DDD is not whether they have a `ValueObject` base class.
It is whether a domain expert can read the code's vocabulary and recognize their own
business, and whether the team can say where one model stops and another begins.

## 2. Theory

### 2.1 Ubiquitous language

**One language, used by developers and domain experts alike, in conversation, in
documentation, and in the code.** Not a translation layer between "business terms" and
"technical terms" — the same words.

The mechanism by which this pays off: ambiguity in language *is* ambiguity in the model.
When a business person says "customer" and means "the person who pays" while a developer's
`Customer` means "anyone with a login", every conversation between them silently succeeds
and every requirement is subtly misunderstood. Forcing one language surfaces the ambiguity
early, in conversation, where it is cheap.

Practical discipline:

- **Rename in code when the language changes.** If the business stops saying "ticket" and
  starts saying "case", rename the class the same week. Deferring is how the code and the
  conversation diverge permanently.
- **Refuse vague terms.** "Manager", "processor", "handler", "info", "data", "service" are
  not domain words. If the domain expert would not use it, it does not belong in a domain
  class name.
- **Notice when experts use the same word differently.** That is not sloppiness; it is a
  signal that there are two contexts (§2.4). "Order" in Sales (a request being negotiated)
  and "Order" in Fulfilment (a set of items to pick) are genuinely different concepts.
- **Write a glossary and keep it.** Twenty to fifty terms with one-line definitions,
  reviewed with domain experts. This is a cheap, high-value artifact that almost nobody
  produces.

### 2.2 Entities and value objects

**Value object** — defined entirely by its attributes. No identity. Immutable. Two value
objects with equal attributes are the same value. `Money`, `DateRange`, `Address`,
`EmailAddress`, `Coordinates`, `Quantity`.

**Entity** — has identity that persists through change. Two entities with identical
attributes are still different things. `Order`, `Customer`, `Shipment`.

The consequential advice: **make more value objects than you think you need.** Most codebases
are full of primitives that should be value objects, and the cost of the primitive is real:

```python
def transfer(from_: str, to: str, amount: float, when: str) -> None: ...
```

versus

```python
def transfer(from_: AccountId, to: AccountId, amount: Money, when: Instant) -> None: ...
```

The second cannot have its arguments swapped (the type checker catches it), cannot carry a
float rounding error, cannot be a naive datetime, and has a place for the rules
(`Money` knows currencies cannot be mixed; `AccountId` knows its format). Every rule that
lives in a value object is a rule that cannot be forgotten at a call site — this is L01's
information hiding applied at the smallest scale, and it is where most of the practical
value of DDD actually comes from.

The **primitive obsession** smell is the diagnostic: a domain function taking four `str`s
and two `float`s is one where the type system is doing nothing for you.

### 2.3 Domain services and the anemic model

Some behaviour does not belong to any single entity: a transfer between two accounts, a
pricing calculation involving a catalogue and a customer's contract. That is a **domain
service** — stateless, named in the domain language, living in the domain layer.

The failure to watch for is the **anemic domain model**: entities that are bags of
getters and setters, with all behaviour in "services". You then have a procedural program
with an object-shaped data layer, and none of the benefits: rules are not co-located with
the data they constrain, invariants can be violated by any service that forgets, and the
same rule gets reimplemented in three places.

The diagnostic: **can an entity be put into an invalid state by any caller?** If
`order.status = CANCELLED` is possible without checking whether cancellation is allowed, the
rule is not in the model. Make attributes read-only and expose intention-revealing methods
(`order.cancel(now, policy)`), so that invalid states are unrepresentable rather than merely
discouraged.

The counter-caution, since this is contested: not everything should be a rich object.
Reporting, bulk imports, and read-heavy queries do not benefit and often suffer. The
rich-model argument applies to the part of the system where *rules* live, and CQRS (§2.6) is
partly a recognition that the write side and read side want different models.

### 2.4 Bounded contexts

The most important idea in DDD, and the most ignored.

> **A model is valid only within a boundary. Different parts of the organization need
> different models of the "same" thing, and forcing one model on all of them fails.**

"Product" in a catalogue context is a description, images, and search terms. In inventory it
is a SKU with a count and a location. In pricing it is a set of price rules and margins. In
shipping it is dimensions and a hazard classification. A single `Product` class serving all
four accumulates thirty fields, four sets of rules that contradict each other, and a change
in any context breaks the others.

The move is to have **four models**, each valid in its context, each owning its own data,
with explicit translation between them. This costs duplication (four things called
"product") and buys independence (each can change without coordinating).

**How to find the boundaries.** In order of reliability:

1. **Language differences.** Where the same word means different things, or where different
   words mean the same thing, there is a boundary.
2. **Different rates and reasons for change.** Things that change together belong together
   (L02 §2.5's change coupling — and you can measure it).
3. **Organizational structure.** Conway's Law is not a warning, it is a description: your
   architecture will mirror your communication structure. Aligning contexts with teams is
   working with the grain (*Team Topologies*).
4. **Transactional needs.** Things that must be consistent together tend to belong together
   (L07).

What is *not* a good boundary: the technical layer (a "data context" and a "logic context"),
the database schema, or the org chart's reporting lines rather than its communication lines.

### 2.5 Context mapping

When contexts interact, the relationship has a *shape*, and naming it is useful because each
shape has different costs:

| Pattern | Meaning |
|---|---|
| **Shared Kernel** | Two contexts share a subset of the model. Requires tight coordination. Use sparingly. |
| **Customer/Supplier** | Downstream's needs influence upstream's plan. Requires a real relationship. |
| **Conformist** | Downstream accepts upstream's model as-is. Cheap, and couples you to their changes. |
| **Anticorruption Layer (ACL)** | Downstream translates upstream's model into its own. Costly, and the standard answer for integrating with legacy or third-party systems. |
| **Open Host Service** | Upstream publishes a well-defined protocol for many consumers. |
| **Published Language** | A shared, versioned interchange format (a schema, an event contract). |
| **Separate Ways** | No integration. Sometimes the right answer and rarely considered. |

The **anticorruption layer** is the one to internalize. When integrating with a system whose
model is wrong for you — a legacy database, a third-party API, another team's service — the
ACL is a translation layer that keeps their concepts out of your domain. It is exactly
SE-511 L06 §2.7's "validate at the boundary" and PY-502 L08's two-layer model rule, arrived
at from the domain side. Without it, their model leaks into yours and every change on their
side is a change in your core.

Draw the context map. Boxes for contexts, labelled arrows for relationships, and the team
that owns each. On one page. It is the single most useful architecture diagram a system can
have, and it takes an afternoon.

### 2.6 CQRS, carefully

Command Query Responsibility Segregation: separate the model used for *writes* (which needs
invariants, aggregates, and consistency) from the model used for *reads* (which needs
denormalized shapes, joins, and speed).

The mild version — and this is the one you should almost always use — is just: **queries
bypass the domain model.** A read that produces a dashboard does not need to load
aggregates; it can be a SQL query returning exactly the shape the view needs. This costs
nothing and removes an enormous amount of contorted mapping.

The strong version — separate write and read *stores*, kept in sync by events — is a serious
commitment: eventual consistency becomes visible to users, you need to handle out-of-order
and duplicate events, and debugging spans two systems. Justified when read and write loads
differ by orders of magnitude, and rarely otherwise. It is frequently adopted for reasons
that do not survive examination, and the honest framing is: *the mild version is a
refactoring; the strong version is a distributed system.*

### 2.7 When DDD does not apply

Evans is explicit about this and it gets dropped. DDD's investment pays when there is a
**complex domain with rules worth modelling**. It does not pay for:

- CRUD applications. The domain is "these fields, validated". Use a framework.
- Technical infrastructure — a proxy, a log shipper, a build tool. The domain is the
  technology.
- Systems whose complexity is in the *scale* rather than the *rules*.
- Anything you will throw away.

Applying the full apparatus to a simple domain produces ceremony and gives DDD its bad
reputation. Evans's own advice is to concentrate effort on the **core domain** — the part
that differentiates the business — and use the simplest thing that works for supporting and
generic subdomains (billing, notifications, auth: buy or use a library).

That distinction — core vs supporting vs generic — is the most practically useful piece of
DDD's strategic half, and it is a *business* judgement you should make explicitly and write
down.

## 3. Construction: modelling a domain

Take a domain you know that has real rules. Not a to-do list.

**Step 1 — the glossary.** Twenty to fifty terms with one-line definitions. Take it to
someone who knows the business and let them correct it. Record every correction — the
corrections are the finding, because each one is a place your model was wrong.

**Step 2 — find the ambiguities.** Every term that means two things, and every pair of terms
that mean one thing. These are your candidate context boundaries.

**Step 3 — event storming (a lightweight version).** On a wall or a document, list the
domain *events* in the business, in rough time order: "Order Placed", "Payment Authorized",
"Stock Reserved", "Shipment Dispatched". Then for each, what *command* caused it, what
*actor* issued the command, and what *data* had to be consulted. This is faster than
modelling nouns first and it surfaces the real workflow, including the parts nobody
mentioned.

**Step 4 — value objects first.** Before entities, find every concept that is defined by its
value: money, dates and ranges, identifiers, quantities, addresses, states. Implement them
with their rules (PY-502 L08). This step alone typically eliminates half the validation code
scattered through a system.

**Step 5 — entities and their invariants.** For each entity, write down the invariants that
must always hold. Then design the API so that they *cannot* be violated: read-only
attributes, intention-revealing methods, construction that validates, and no setters.

**Step 6 — the context map.** Draw it. Contexts, relationships (using §2.5's vocabulary),
and owners. Identify which context is your **core domain** and justify it in one sentence.

**Step 7 — the honest review.** Where did modelling help? Where was it ceremony? Which parts
of your system are generic subdomains you should not be building at all? A submission
without step 7 is incomplete.

## 4. Failure modes

- **Building blocks without language.** Entity/value-object base classes, and a model
  nobody in the business would recognize.
- **Anemic model.** §2.3.
- **One model for the whole organization.** §2.4. The thirty-field `Product`.
- **No context map**, so nobody knows where a model stops being valid.
- **Conformist by default** — accepting a third party's model into your core because writing
  an ACL felt like extra work.
- **Applying DDD to a CRUD app.**
- **Modelling the supporting domains as carefully as the core.** Effort in the wrong place.
- **Strong CQRS adopted for the mild version's benefits.**
- **Primitive obsession** surviving the whole exercise.
- **"Manager", "Service", "Processor", "Helper", "Info"** in domain class names.
- **A glossary written once and never revisited.**

## 5. Exercises

### Warm-up (25 min)

**W1.** List every domain class name in a codebase. Mark the ones a domain expert would
recognize. Report the proportion.

**W2.** Find three primitives that should be value objects. Implement one and report every
place a rule moved into it from.

**W3.** Find a term used with two meanings in your system. Describe the two contexts it
implies.

### Core (2.5 h)

**C1 — Model a domain.** Complete §3, all seven steps. Deliverable: the glossary with
corrections marked, the event list, the value objects and entities with invariants, the
context map, and the honest review. This is the Term 2 build artifact's foundation.

**C2 — De-anemify.** Take an anemic entity and its service. Move the rules into the entity
so that invalid states are unrepresentable. Report: how many places previously performed the
same check, and what the service is left doing.

**C3 — Build an anticorruption layer.** Take an integration where a third party's or legacy
system's model has leaked into yours. Build the ACL: their model in, your model out, in one
package. Report every place in your core that no longer mentions their concepts, and the
cost in translation code.

**C4 — Core/supporting/generic.** Classify every subdomain of a system you work on. For each
generic one, determine whether it should be bought, borrowed, or built, and estimate what
was spent building the ones that should have been bought. Present it as you would to a
manager — this is a business argument, and making it well is a skill.

### Challenge

**X1.** Read Evans, *Domain-Driven Design*, Parts I–II and ch. 14–15 (bounded contexts and
context mapping — the "strategic" half, which is the important half). Write 1,500 words on
the claim that DDD's tactical patterns are widely adopted and its strategic patterns widely
ignored, why that happened, and what it costs. Use examples from systems you know.

**X2.** Run a real event-storming session with at least two other people on a domain you all
know. Report: what the session surfaced that you did not know, where the disagreements were,
and what boundaries emerged. Then implement the core aggregate that came out of it. The
process finding is the deliverable, not the code.

## 6. Self-check

1. What is the ubiquitous language, and what does an ambiguity in it indicate?
2. Distinguish entity from value object and say why you should make more value objects.
3. What is the diagnostic for an anemic domain model, and the fix?
4. Define a bounded context and give four ways to find boundaries.
5. Give five context-mapping patterns and when each applies.
6. What is an anticorruption layer and what does it protect against?
7. Distinguish mild from strong CQRS and say which is a refactoring and which is a
   distributed system.
8. When does DDD not apply, and what is the core/supporting/generic distinction for?

## 7. Primary sources

- Evans, *Domain-Driven Design* (2003) — Parts I–II, and chs. 14–17 especially.
- Vernon, *Implementing Domain-Driven Design* (2013) — the practical companion.
- Brandolini, *Introducing EventStorming* — the workshop technique.
- Fowler, "BoundedContext" and "AnemicDomainModel" (bliki).
- Skelton & Pais, *Team Topologies* — Conway's Law taken seriously.

---

**Previous:** [L05](L05-ports-and-adapters.md) · **Next:**
[L07 — Aggregates, Transactions, and Consistency Boundaries](L07-aggregates-and-consistency.md)
