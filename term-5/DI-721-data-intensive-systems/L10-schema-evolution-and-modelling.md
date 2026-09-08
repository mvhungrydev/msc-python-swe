# DI-721 · Lesson 10 — Schema Evolution, Contracts, and Data Modelling

**Estimated study time:** 4.5 hours
**Prerequisites:** L06, L08, L09; SE-521 L09 (service boundaries), SE-511 L09 (packaging and
versioning)

---

## 1. Orientation

The course's driving question was *how does data keep its meaning as it moves?* This lesson is the
answer, and it has two halves.

The first is **evolution**: data outlives the code that wrote it. An event emitted today will be
read in five years by a service that does not exist yet, written by someone who has never met you,
using a schema version you have not designed. A row written today will be migrated, exported,
joined and aggregated by people who will never read your code. Every one of those readers is
relying on a contract, and the only question is whether the contract is written down.

The second is **modelling**: the same facts can be organised many ways, and the organisation
encodes assumptions about the questions that will be asked. Normalised or dimensional, wide or
narrow, one big table or a constellation — these are not stylistic choices, they are bets, and this
lesson is about making them explicitly.

The connective claim:

> **A schema is an interface, and a dataset is an API.** Everything you know about interface design
> — information hiding, backward compatibility, versioning, deprecation — applies, and the fact
> that data interfaces are usually undocumented and unversioned is a defect of practice, not a
> property of data.

## 2. Theory

### 2.1 Compatibility, defined precisely

Three terms, constantly confused, and each defines who can be upgraded first:

- **Backward compatible**: **new code can read old data.** Upgrade readers first. This is what you
  need when consumers are upgraded before producers, and it is the common requirement for stored
  data and event logs.
- **Forward compatible**: **old code can read new data.** Upgrade producers first. Required when
  you cannot upgrade all consumers — which, in any organisation with more than a few teams, is
  always.
- **Full compatibility**: both. The safe default for anything published outside your team.

Whether a change is compatible depends on the encoding, which is why the encoding choice is a
governance decision as much as a technical one:

| Change | Avro | Protobuf | JSON (unvalidated) |
|---|---|---|---|
| Add optional field with default | full | full | usually full |
| Add required field | backward only | not recommended | breaks old readers' assumptions |
| Remove optional field | forward only | forward only | forward only |
| Rename field | breaking (use aliases) | safe if tag unchanged | breaking |
| Change type | mostly breaking (some widening allowed) | mostly breaking | breaking, silently |
| Reorder fields | safe (name-based) | safe (tag-based) | safe |

The lesson in that table: **Protobuf identifies fields by tag number and Avro by name with a
writer's schema, so both can evolve; JSON without a schema identifies fields by name with no
recorded contract, so nothing is checked and breakage is discovered by the consumer, at runtime, in
production.** JSON is fine as a *wire format*; JSON without a schema registry is not a contract.

Two rules that prevent most incidents:

- **Never reuse a field tag or name.** Deleted fields are *reserved*, permanently. Protobuf has
  explicit `reserved` syntax for this, and it exists because reuse is a data-corruption bug that
  passes every test.
- **Every new field is optional with a default.** If a field is genuinely required, it is a new
  message type, not a new version of the old one.

### 2.2 Schema registries and CI enforcement

A registry stores schemas, assigns versions, and — the part that matters — **rejects incompatible
changes**. Producers register the schema and embed a schema ID in each message; consumers fetch the
writer's schema by ID and resolve it against their own reader's schema.

The property that makes this work: **compatibility is checked at registration time, in CI, not at
read time in production.** An incompatible change fails the build, in the pull request, in front of
the person who made it. Without that enforcement point, a compatibility policy is a wiki page.

Set the compatibility mode deliberately per subject:

- `BACKWARD` (the usual default): new schema can read data written by the previous one.
- `FORWARD`: previous schema can read data written by the new one.
- `FULL`: both.
- `*_TRANSITIVE`: checked against *all* previous versions, not just the last. This is what you
  actually want for an event log you intend to replay, because replay reads every version ever
  written — and the non-transitive default silently allows a sequence of individually-compatible
  changes that is collectively incompatible.

### 2.3 Data contracts

The idea, and it is mostly organisational rather than technical:

> A **data contract** is an explicit, versioned, enforced agreement between a data producer and its
> consumers, covering schema, semantics, quality guarantees, and change policy.

The problem it solves is specific and familiar: a team refactors a table, an analytics dashboard
breaks, and nobody knew there was a dependency because the coupling was a `SELECT` written eighteen
months ago by someone who has left. The producer never agreed to anything; the consumer assumed
everything.

A contract states: **schema** (with an enforced compatibility mode); **semantics** — what each
field *means*, including units, timezone, nullability semantics, and whether "no row" and "null"
differ; **quality guarantees** — freshness, completeness, expected volume, uniqueness of the key;
**an SLA** with someone accountable; and a **change policy** with a deprecation window.

The engineering practices that make it real, as opposed to a document:

- **Compatibility enforced in CI**, per §2.2.
- **Quality checks that run continuously** (Great Expectations, dbt tests, Soda) and alert the
  *producer*, not only the consumer. A contract the producer never sees fail is not enforced.
- **A public interface distinct from internal tables.** This is SE-521's information hiding applied
  to data: publish a view or an event, not your physical schema, so you retain the freedom to
  refactor.
- **Lineage** so that "who depends on this column?" is answerable before the change, not after.

### 2.4 Migrations that do not require downtime

The standard technique, and it is worth knowing exactly because the alternative is an outage:
**expand, migrate, contract**.

1. **Expand.** Add the new structure (column, table, field) alongside the old. Nothing reads it yet.
   The schema now supports both shapes.
2. **Dual-write.** Write to both old and new. Deploy. Both are current for new data.
3. **Backfill.** Copy historical data into the new structure, in batches, throttled to protect
   production load, and idempotently so it can be resumed after a failure.
4. **Verify.** Compare old and new for consistency, on real data, and do not skip this.
5. **Migrate readers.** Switch consumers to the new structure, one at a time, with the ability to
   revert.
6. **Contract.** Stop writing the old structure; after a soak period, drop it.

Each step is individually reversible, which is the entire point. Additional practical rules:

- **Never rename in one step.** Add, dual-write, backfill, migrate, drop.
- **`NOT NULL` with a default on a large table** rewrites the table in older engines — check your
  version, and use the add-nullable-then-backfill-then-constrain sequence when in doubt.
- **Add indexes concurrently** (L03), or take a write lock for the duration.
- **Long-running migrations hold transactions**, which pin the MVCC horizon (L05 §2.5) and bloat
  the database. Batch and commit.
- **Test the migration on production-scale data.** A migration that takes 2 seconds on 10,000 rows
  may take 9 hours on 400 million, and finding that out during the maintenance window is the
  classic failure.

For **event logs**, migration is different because you cannot rewrite history. The options are
**upcasting** (transform old event versions into the current shape at read time, permanently),
**versioned event types** (`OrderPlaced.v2` as a distinct type, with both handled), or a **new
stream** with a one-time migration and a switchover. Upcasting is the usual answer, and the
important consequence is that upcasters accumulate and must be tested against real archived events
— a test fixture of genuine old events is one of the most valuable artifacts an event-sourced
system has.

### 2.5 Modelling for analytics

**Normalisation** (3NF) minimises redundancy and update anomalies. It is right for OLTP, where
writes are frequent and single-row, and where an update should touch one place.

**Dimensional modelling** (Kimball) organises data into **facts** — the measurements, one row per
event, narrow and numerous — and **dimensions** — the descriptive context, wider and fewer. A
**star schema** is one fact table joined to its dimensions; a **snowflake** normalises the
dimensions further, which saves space nobody needs and adds joins everybody pays for.

Why denormalise for analytics: fewer joins, and a shape humans can query without a data dictionary.
The classic objection — update anomalies — largely does not apply, because analytical data is
append-mostly and derived from a source of truth that *is* normalised.

**Slowly changing dimensions** are the subtle part, and the thing most often got wrong:

- **Type 1**: overwrite. History is lost. A customer moves from Texas to Oregon, and every
  historical sale now appears to have been to an Oregon customer. Sometimes that is what you want;
  usually it is not, and nobody notices for a year.
- **Type 2**: add a new row with validity dates and a current flag. History preserved; facts join
  to the dimension row valid *at the time of the fact*. This is the correct default for anything
  where historical accuracy matters, and it is the same temporal-correctness issue as L08 §2.6's
  stream–table join, in a different vocabulary.
- **Type 3**: keep a "previous value" column. Limited, occasionally exactly right.

**Grain** is the first decision and the one that determines everything else: *what does one row of
this fact table represent?* One order? One line item? One order per day? State it explicitly, in
writing, before designing anything. Mixed grain in one table is the single most common modelling
defect, and it produces double-counted aggregates that look plausible.

Modern variations worth knowing: **Data Vault** (hubs, links, satellites) optimises for auditable
integration from many sources at the cost of query complexity, and suits regulated environments
with many changing sources. **One Big Table** — fully denormalised and wide — exploits columnar
storage's projection and compression (L06) to make joins unnecessary; excellent for a well-
understood query pattern, poor when the pattern changes. **Medallion** (bronze raw / silver cleaned
/ gold aggregated) is a layering convention rather than a modelling technique, and it is useful
mainly because it names where each transformation belongs.

### 2.6 The organisational layer

Because the failures here are usually organisational, three ideas you will encounter:

- **Data mesh** (Dehghani): domain teams own their data as a product, with contracts, SLAs and
  discoverability, supported by a self-serve platform. The valuable core is *ownership and
  contracts*; the term has been attached to a great deal of vendor material, so read the original.
- **Data products**: a dataset with an owner, documentation, a contract, quality guarantees and a
  deprecation policy — the same standard you would apply to a public API.
- **Semantic layer**: metric definitions in one governed place, so "revenue" means one thing.
  Unglamorous, and it eliminates a large category of disagreement that otherwise consumes
  executive meetings.

The engineering point behind all three: **most data quality problems are contract problems, and
most contract problems are ownership problems.** A pipeline with no owner degrades; a schema with
no contract breaks; a metric with no definition means whatever the last person to compute it
assumed.

## 3. Construction: contracts, migrations, and a model

Build in `mpse/di721/l10/`, on top of the L09 pipeline.

**Stage 1 — schemas and a registry.** Define your L09 domain events in Avro and in Protobuf.
Stand up a schema registry (Confluent's, or a minimal one you write). Register the schemas and set
`FULL_TRANSITIVE` compatibility.

**Stage 2 — the compatibility matrix.** For each change type in §2.1's table, attempt it in both
encodings and record what the registry says. Then, for each *accepted* change, verify the actual
behaviour: write with the new schema, read with the old, and check what the old reader sees.
Produce the matrix. Where the registry accepted something whose read behaviour surprised you, write
it up — that gap is where real incidents come from.

**Stage 3 — CI enforcement.** Wire compatibility checking into a build. Make a pull request with a
breaking change and demonstrate the build failing with a message that says what broke and why. Then
demonstrate the non-transitive trap: three individually-compatible changes that together break a
replay of the oldest data.

**Stage 4 — a data contract.** Write a real one for one dataset: schema, per-field semantics
(units, timezone, null meaning), freshness and completeness guarantees, key uniqueness, owner, and
change policy with a deprecation window. Then implement the quality checks that enforce the
guarantees, run them continuously, and route failures to the producer.

**Stage 5 — expand/migrate/contract.** Perform a real column split (a `name` column into
`given_name` and `family_name`) on a live table with concurrent traffic, following all six steps.
Measure downtime (target: zero), verify consistency at the verification step, and demonstrate that
you can revert at each of the first four steps.

**Stage 6 — the migration that hurts.** Take a table of several million rows and run a naive
migration: a single transaction adding a `NOT NULL` column with a default, or a `UPDATE` over every
row. Measure lock duration, bloat and the effect on concurrent queries. Then do it properly in
batches and compare. This is the experiment that makes the rules in §2.4 stick.

**Stage 7 — event log evolution.** Change an event's schema three times. Implement upcasters so
that a projection rebuild from the very beginning produces correct results. Then write the test
that would have caught a broken upcaster: replay a fixture of genuine archived events of every
version and assert the resulting state. Note how much this test is worth relative to its size.

**Stage 8 — the dimensional model.** Design a star schema over your order data: state the grain in
one sentence, build the fact table and three dimensions, and implement one dimension as Type 1 and
the same one as Type 2. Then run the analytical query that distinguishes them ("revenue by customer
region, historically accurate") and show the two answers differing. Explain which is correct and
why, and state the condition under which the other would be.

**Stage 9 — the layouts compared.** Load the same data three ways — normalised, star schema, and
one big table — into your columnar store from L06. Run five analytical queries against each and
report storage, query time and the effort to add a sixth, previously unanticipated query. That last
column is the one that decides the architecture in practice, and it is the one benchmarks never
measure.

## 4. Failure modes

- **JSON with no schema registry as an inter-team contract.** Nothing is checked; breakage is
  discovered by consumers in production.
- **Reusing a field tag or name.** Silent data corruption that passes every test.
- **Non-transitive compatibility on a replayable log.** Individually-safe changes that collectively
  break replay.
- **Renaming in one step.** Guaranteed breakage for anyone mid-deploy.
- **Migrations untested at production scale.** The 2-second migration that takes 9 hours.
- **A long-running migration in one transaction.** Locks, bloat, and a pinned MVCC horizon.
- **Publishing physical tables as the interface.** You can never refactor again.
- **Undocumented semantics.** The field is `amount` — currency? cents? gross or net? Which
  timezone? Every one of these has caused a real, expensive incident somewhere.
- **Type 1 dimensions where history matters.** Silently rewrites the past, and is usually
  discovered a year later.
- **Mixed grain in a fact table.** Double-counted aggregates that look plausible.
- **No owner.** The pipeline degrades and no one is accountable for it.
- **Quality checks that alert only consumers.** The producer never learns they broke something.

## 5. Exercises

### Warm-up (30 min)

1. Define backward, forward and full compatibility, and for each say who can be upgraded first.
2. Give five schema changes and classify each under Avro, Protobuf and unvalidated JSON.
3. Explain why field tags must never be reused, and what mechanism enforces it in Protobuf.

### Core (3.5 h)

4. Complete Stages 1–3 and deliver the compatibility matrix plus the non-transitive trap
   demonstration.
5. Complete Stage 4: a real data contract with working, continuously-running quality checks.
6. Complete Stages 5–6 and deliver the zero-downtime migration write-up and the naive-versus-batched
   comparison.
7. Take a dataset you work with and write its contract as it *should* be — then find out how many
   of the guarantees are currently true. The gap is the finding.

### Challenge

8. Complete Stages 7–9, including the Type 1 versus Type 2 divergence and the three-layout
   comparison with the "add a sixth query" effort column.
9. Build a **breaking-change detector** for a data warehouse: parse the SQL of all downstream
   consumers (dbt models, reports, notebooks), build a column-level lineage graph, and given a
   proposed schema change, report exactly which downstream artifacts break and which are unaffected.
   Then run it against a real change and check its answer by making the change in a copy. Report its
   false positives and false negatives honestly — the interesting result is the class of dependency
   it *cannot* see (dynamic SQL, string-built queries, consumers outside the warehouse), because
   that class is what a technical solution alone can never cover, and it is the argument for the
   organisational half of §2.6.

## 6. Self-check

1. Define the three compatibility directions and the upgrade order each permits.
2. Why can Protobuf and Avro evolve where unvalidated JSON cannot?
3. What does a schema registry enforce, and at what moment?
4. What is `FULL_TRANSITIVE` and why does a replayable log need it?
5. Give the five components of a data contract.
6. Give the six steps of expand/migrate/contract and say why each is reversible.
7. Why can event logs not be migrated in place, and what are the three options?
8. Distinguish Type 1, 2 and 3 slowly changing dimensions, and give the failure of Type 1.
9. What is grain, and what does mixing it produce?
10. State the claim connecting data quality, contracts and ownership.

## 7. Primary sources

- **Kleppmann, *Designing Data-Intensive Applications*, ch. 4** — the best treatment of encoding
  and evolution anywhere; read it twice.
- The Avro specification (schema resolution rules) and the Protobuf language guide (field numbers,
  `reserved`, and the compatibility section).
- Confluent's schema registry documentation on compatibility types — particularly the transitive
  variants.
- **Kimball & Ross, *The Data Warehouse Toolkit*, 3rd ed.** — dated infrastructure, correct
  modelling; chapters 1–3 and the SCD material are the essential parts.
- Inmon, *Building the Data Warehouse* — the other side of the historical Inmon/Kimball argument;
  read enough to understand what the disagreement was actually about.
- Dehghani, *Data Mesh* (and the original 2019 articles) — read the original before the commentary.
- Linstedt & Olschimke, *Building a Scalable Data Warehouse with Data Vault 2.0* — if you work in a
  regulated multi-source environment.
- Great Expectations and dbt test documentation — the practical tooling for §2.3.

---

**Previous:** [L09](L09-event-sourcing-and-cdc.md) ·
**Next:** [Problem sets](problem-sets.md) · [Exam](exam.md)

*This completes the DI-721 lesson sequence — and, with DS-701, Term 5. The two courses answer
complementary questions: DS-701 asks what you can guarantee when components fail independently;
DI-721 asks what data means once it has crossed several systems. The term artifact requires both
answers at once, which is the point.*
