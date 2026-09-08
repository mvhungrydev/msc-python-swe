# Appendix — Errata

Nothing in this programme is infallible. It was written in one pass, without peer review, and parts
of it were already dating as they were written.

**Recording an error you found, with evidence, is itself a passing grade on that lesson.** Finding
something wrong and being able to prove it is the skill the whole programme is trying to build — it
is the same skill as reading a paper adversarially (see `paper-index.md`) or refusing to accept an
abstraction at face value (CA-731 L01). Use this file.

---

## How to use this file

When you find something wrong, add an entry:

```markdown
### [Course] [Lesson] §[section] — one-line summary

**Claimed:** what the material says.
**Actual:** what is true, with evidence — a doc link, a measurement, a paper, a version number.
**Checked on:** date, and the environment (Python version, OS, library versions, provider).
**Severity:** typo / imprecise / wrong / dangerously wrong.
**Fix:** what it should say.
```

The "checked on" line matters more than it looks. A claim that was true on Python 3.12 and false on
3.15 is not an error in the material; it is an error in the material's *scope*, and the fix is a
version qualifier rather than a correction.

---

## What is most likely to be wrong

Ranked by how quickly it will date. Check these first, and treat them with suspicion by default.

### 1. Python version behaviour — very high risk

The programme's baseline is Python 3.12+, written in 2026. Specific hazards:

- **Free-threading (PEP 703)** — status, performance characteristics and per-object locking overhead
  are all moving. PY-601 L02's claims are hedged for this reason; verify against your build.
- **The JIT (PEP 744)** and the specialising interpreter's behaviour — PY-501 L08's bytecode
  examples are explicitly illustrative and *will* differ from what you see.
- **Subinterpreters (PEP 554/734)** — the API and its ergonomics are still settling.
- **`dis` output** for anything. It changes between minor versions. The reasoning is durable; the
  exact opcodes are not.
- **Typing features** — the typing ecosystem moves fast, and CS-641 L08 and SE-511 L06 will date.

**Always check `dis` output and free-threading claims against your own interpreter before believing
them.**

### 2. LLM material — highest risk of all

**ML-741 L09 is the most perishable content in the programme.** Model capabilities, context limits,
prices, tooling, and best practice change on a timescale of months. The lesson is written to
emphasise the durable engineering — evaluation, cost, latency, containment — and flags its specifics
as provisional. Assume every concrete claim in it needs checking.

### 3. Cloud provider details — high risk

CA-731 uses AWS as its worked example. Service names, limits, quotas, pricing, default settings and
even architectural facts change. Every number in that course should be verified against current
documentation before you rely on it, and pricing especially.

### 4. Tool behaviour — moderate risk

Terraform, Kubernetes, TLA+ tooling, Hypothesis, Z3, Postgres defaults. APIs and defaults shift.
Postgres in particular changes its planner and vacuum defaults between major versions.

### 5. Hardware constants — moderate risk

The memory hierarchy latencies in DI-721 L01 §2.2 are approximate and drifting. **The ratios are
what matter and they are more stable than the absolutes** — but measure your own (DI-721 L01 Stage
5) rather than trusting the table.

### 6. Research claims — low risk, high value when found

Where the material states an empirical claim from a paper — bloom filter false-positive rates,
compaction amplification figures, the effect size of an ML technique — the original may have been
narrower than the summary suggests. **These are the most valuable errata to find**, because finding
one means you read the paper properly.

### 7. Cross-references — low risk, easily fixed

The programme cross-references heavily. If a lesson points at "DS-701 L07 §2.4" and that section
covers something else, record it.

---

## Known limitations of the material

Stated up front, because acknowledging them is more useful than letting you discover them as
surprises.

**Written in one pass, unreviewed.** No second reader checked the arguments. Expect occasional
imprecision, and treat confident-sounding claims with the same scepticism the programme asks you to
apply to vendor documentation.

**Bibliographic details may be imperfect.** Author lists, years and venues are given from memory of
the literature. Verify a citation before using it in your own written work.

**Some numbers are illustrative.** Where the material gives a figure without a source — "typically
5–20× compression", "usually under 15 minutes" — it is a rough practitioner's figure, not a measured
result. Measure your own; several construction exercises exist specifically to make you do so.

**The exercises are unvalidated.** Nobody has completed them. Some will be harder than their time
estimate suggests, and a few may have flaws or ambiguities. Record those here too — an exercise
whose intent is unclear is an error in the material.

**Coverage choices are opinionated.** Some things a traditional MSc would cover are absent:
compilers beyond CS-641's scope, operating systems, networking below CA-731's level, security
beyond CA-731 L03, HCI, and most of numerical computing. If your work needs one, this programme
will not supply it.

**Positions are taken deliberately.** The material argues against feature stores as reuse
infrastructure, against over-adoption of Kubernetes, against most agent architectures, and in favour
of lightweight formal methods over heavyweight ones. Each argument is given so that you can
disagree with it on the evidence — and where you do, an entry here saying why is a better outcome
than agreement.

---

## Errata

*Add entries below. Newest first.*

<!--
### DS-701 L05 §2.3 — example

**Claimed:** Raft requires `votedFor` to be persisted before responding to a RequestVote.
**Actual:** [what you found]
**Checked on:** 2026-09-08, against the Raft dissertation §3.4 and the TLA+ spec.
**Severity:** imprecise
**Fix:** [what it should say]
-->

*(none recorded yet)*

---

## The programme review

When you finish — or abandon — the programme, write the honest account here. What worked, what did
not, what you skipped, what was missing, what you would tell someone starting.

That account is the most valuable thing you can add to this file. The material was written without
knowing you, your background or your goals; your account is the only evidence that exists about
whether it works.

Suggested headings:

- **What I actually completed**, versus what I started.
- **Where the difficulty estimates were wrong**, in both directions.
- **What was missing** that I needed.
- **What I could have skipped** without loss.
- **What changed in my work** as a result — concretely.
- **What I would tell someone starting this**, in one paragraph.
