# CA-731 · Lesson 05 — Infrastructure as Code and Provider Design

**Estimated study time:** 5 hours
**Prerequisites:** L01–L04; SE-511 L09 (packaging and versioning), SE-521 L02 (coupling)

---

## 1. Orientation

Infrastructure as code is usually taught as a tool tutorial. It is better understood as a
*distributed systems* problem wearing a configuration-management costume, because that is what the
hard parts are:

> A Terraform run reads a **state** file, queries the **actual world** through an API, computes a
> diff against **declared intent**, and applies changes to a system that is concurrently modified
> by other actors, whose API calls are not transactional, and where a failure halfway through
> leaves the world in a state neither the code nor the state file describes.

Every genuinely difficult IaC problem — state locking, drift, partially-applied changes, imports,
refactors that want to destroy production — is a consequence of that paragraph. Once you see it,
the tooling's quirks stop being arbitrary.

The second half of the lesson is about IaC as *software*: modules are libraries, they have
interfaces, those interfaces have consumers who cannot easily be changed, and everything SE-521
says about coupling and information hiding applies. Most organisations' IaC is bad in exactly the
way their code would be bad if nobody applied software engineering to it — because nobody did.

Mike's context makes this the most immediately applicable lesson in the course; the exercises are
written so the artifacts are usable in real work.

## 2. Theory

### 2.1 Declarative, and what "converge" means

The model: you declare desired state; the tool computes the difference from actual state and makes
the minimum change. This is the same reconciliation pattern as Kubernetes (L06), and it is worth
naming the properties it gives you:

- **Idempotence**: applying twice equals applying once (DS-701 L08's property, again).
- **Convergence**: repeated application moves toward the declared state.
- **A reviewable diff before the change**: the plan. This is IaC's single largest safety property
  and it is routinely wasted by not reading it.

Where the model leaks, and these are the important cases:

- **Not everything is declarative.** Data migrations, one-off imports, and anything with an
  ordering requirement do not fit. Trying to force them in produces worse results than keeping them
  out.
- **Ordering is inferred from references**, not stated. If A must exist before B and B does not
  reference A, the tool does not know. Implicit dependencies produce intermittent, timing-dependent
  failures — the hardest class of IaC bug.
- **The provider API is not transactional.** A plan of forty changes that fails at change
  twenty-three leaves twenty-two applied. There is no rollback; there is only re-running.
- **Some changes are destructive.** Changing an attribute that the API cannot update in place
  produces a destroy-and-recreate, and the plan says so — in the middle of a long output nobody
  read. **`# forces replacement` on a database is the single most expensive line in IaC**, and
  reading for it is a skill.

### 2.2 State

The state file maps declared resources to real ones. It exists because the tool must know that
`aws_instance.web` corresponds to `i-0abc123`, and because it caches attributes for planning.

State is the source of most operational pain, for reasons that are all distributed-systems
reasons:

- **It must be shared** among everyone who applies, hence remote state.
- **It must be locked**, or two concurrent applies corrupt it. (This is DS-701's mutual exclusion
  problem, with the same requirement for a fencing mechanism.)
- **It contains secrets** in plaintext — database passwords, generated keys. State is a sensitive
  artifact and must be encrypted and access-controlled accordingly. Many organisations do not treat
  it that way.
- **It drifts** from reality when anyone changes anything outside the tool (§2.3).
- **It can be lost or corrupted**, and recovery means importing every resource by hand. Version the
  state bucket; this is your undo.

Two structural decisions follow:

**State splitting.** One state file for everything means every plan is slow, every apply locks
everything, and the blast radius of a mistake is total. Split by blast radius and change frequency:
networking (changes rarely, breaks everything) separate from applications (changes constantly,
breaks one thing). Cross-state references are then either remote state data sources (a coupling,
and a read at plan time) or explicit inputs (looser, and requires plumbing). Prefer explicit inputs
at important boundaries — a remote state data source is a compile-time dependency on someone else's
internals.

**Workspaces versus directories.** Workspaces share code with different state; separate directories
duplicate code with different variables. Workspaces tempt you into environment-conditional logic
(`count = var.env == "prod" ? 3 : 1`) scattered through the code, which is the IaC equivalent of
`if (isProd)` and rots the same way. Prefer separate root modules per environment with a shared
module library, and keep environment differences in variable files where they can be diffed.

### 2.3 Drift

**Drift** is divergence between declared, recorded and actual state. Causes: a console change
during an incident, another tool managing the same resource, provider-side defaults changing, or an
apply that failed partway.

Drift is not merely untidy; it is dangerous, because the next apply "corrects" it — reverting the
emergency fix somebody made at 2 a.m. to keep the system up, without anyone noticing until it
breaks again.

Management:

1. **Detect it**: scheduled `plan` runs that alert on non-empty diffs. Cheap and high value.
2. **Reduce the causes**: no console write access in production for humans (L03's just-in-time
   elevation), and a documented emergency path that *includes* reconciling the change back into
   code afterwards.
3. **Decide per drift**: adopt it (import into code) or revert it (apply). Both are legitimate; the
   failure is not deciding.
4. **Accept some**: tags added by other systems, autoscaling-managed capacity, provider-managed
   attributes. Use `ignore_changes` deliberately and document why — an undocumented lifecycle
   ignore is a landmine.

### 2.4 Modules as libraries

A module is a library, and SE-511 and SE-521 apply directly.

**Interface design.** The variables are the public API. Every variable is a commitment; adding one
is easy and removing one is a breaking change for every consumer. Design for the common case with
sensible defaults, expose only what genuinely varies, and resist the pressure to add a variable for
every attribute — a module with sixty variables is a thin, leaky wrapper that provides abstraction
in name only, and the honest response is often to delete it and use the resource directly.

**Information hiding** (SE-521 L01): a module should hide a decision. `vpc` hiding subnet layout,
routing and NAT is a real abstraction; `ec2_instance_wrapper` passing through every argument hides
nothing. The test is whether you could change the implementation without changing consumers.

**Versioning.** Modules are versioned artifacts, consumed by reference to a tag. Semantic
versioning applies, and — the point people miss — *the state is part of the contract*. A refactor
that renames a resource inside the module destroys and recreates it for every consumer unless a
`moved` block is provided. **A module upgrade that silently recreates a database is a production
incident delivered as a patch version**, so treat state-affecting refactors as breaking changes and
ship `moved` blocks.

**Testing.** IaC is code, so:

- *Static*: `validate`, `fmt`, and policy checks (`tflint`, `checkov`, `tfsec`) in CI.
- *Plan-based*: assert properties of the plan without applying — no public ingress, encryption
  enabled, tags present. Fast, and catches most misconfigurations.
- *Integration*: apply into a throwaway environment, assert the real resources behave, destroy
  (Terratest, or `terraform test`). Slow, and the only thing that catches "the code applies but the
  thing does not work".
- *Policy as code*: OPA/Sentinel against the plan, enforcing organisational rules independently of
  the module author's intent (L03's guardrail idea, applied to infrastructure).

### 2.5 Writing a provider

Writing a provider — for an internal service, or to extend an existing one — is the exercise that
makes the whole model legible, and it is directly relevant to anyone doing platform work.

A provider implements, per resource type:

- **Schema**: attributes, types, whether each is required, optional or computed, and — critically —
  which force replacement when changed.
- **Create**: call the API, set the resource ID, save computed attributes to state.
- **Read**: fetch current state from the API and update state. **This is where drift detection
  happens**, and the correctness rule is: if the resource no longer exists, remove it from state
  rather than erroring, so the next plan recreates it.
- **Update**: apply a diff in place where the API supports it.
- **Delete**: remove it, and handle already-deleted gracefully (idempotence again).
- **Import**: adopt an existing resource by ID.

The subtleties you will meet, which are the same subtleties every provider bug comes from:

- **Eventual consistency.** A create returns before the resource is fully available, so `Read`
  immediately after may 404. Providers wait and retry — this is DS-701 L09's material inside a
  provider.
- **Computed attributes** unknown until apply, which is why plans contain `(known after apply)`.
- **Drift semantics**: what constitutes a difference. Normalisation matters — is `Name` equal to
  `name`? Is an unordered list of tags different when reordered? Getting this wrong produces
  perpetual diffs, which train people to ignore plans, which is how a destroy gets approved.
- **Partial failure**: if create succeeds but a subsequent configuration call fails, the resource
  exists and must be saved to state anyway, or it is orphaned forever.

Anyone who has written a provider reads plans differently, permanently. That is the point of
Stage 6.

### 2.6 Pipelines and blast radius

The workflow that holds up:

1. Change proposed in a pull request.
2. CI runs `fmt`, `validate`, lint and policy checks.
3. CI runs `plan` and posts the output on the PR.
4. **A human reads the plan** — specifically for replacements, deletions, and anything touching
   data. This is the control, and every other step supports it.
5. Merge triggers `apply` with the *saved* plan from the PR, so what was reviewed is what runs.
6. Post-apply verification: the drift check should now be clean, and health checks should pass.

Practices that reduce the damage when this fails:

- **`prevent_destroy`** on stateful resources. Cheap, and it has saved many databases.
- **Separate states per environment and per blast radius** (§2.2).
- **Staged rollout across environments** — dev, then staging, then production, with a soak.
- **Deletion protection at the provider level**, independent of the IaC tool.
- **Backups verified by restore**, because "we have backups" and "we can restore" are different
  claims.

The cultural point, which is where organisations actually fail: **a plan nobody reads is not a
control.** Long, noisy plans full of perpetual diffs are the mechanism by which review dies. Keep
plans small, fix perpetual diffs rather than tolerating them, and make destructive changes visually
unmissable in the pipeline.

## 3. Construction: build the platform's infrastructure properly

Build in `mpse/ca731/l05/`. Terraform (or OpenTofu) is assumed; the concepts transfer to Pulumi and
CDK, and the provider exercise is Terraform-specific because that is where the model is clearest.

**Stage 1 — the module library.** Write three modules — network (your L04 three-tier VPC), a
service module (compute, load balancer, security groups, IAM role), and a data module (database
with backups and encryption). Design each interface deliberately: write down, for each variable,
why it exists and what would happen if it did not. Delete any variable you cannot justify.

**Stage 2 — root modules per environment.** Compose the library into dev, staging and production
root modules with variable files. No environment conditionals inside the modules. Split state by
blast radius: network separate from services.

**Stage 3 — the pipeline.** CI with fmt, validate, lint, policy checks, and a plan posted to the
PR. Merge applies the saved plan. Then make a change that forces replacement of the database and
verify that the pipeline makes it visually obvious — if it does not, fix the pipeline, not the
reviewer.

**Stage 4 — testing at three levels.** Plan-based assertions (no `0.0.0.0/0` ingress except on
443, encryption enabled, required tags present); a policy-as-code rule enforced independently of
module code; and one integration test that applies to a throwaway environment, asserts the service
responds, and destroys. Measure how long each level takes and where each catches a bug the others
miss.

**Stage 5 — drift.** Change something in the console. Detect it with a scheduled plan. Then handle
it three ways — revert, adopt via import, and accept via a documented `ignore_changes` — and write
the runbook stating how to decide which.

**Stage 6 — write a provider.** Build a small HTTP service with a CRUD API (a tenant registry for
your L02 service is ideal), then write a Terraform provider for it: schema with at least one
force-new attribute, full CRUD, import, and drift detection that removes deleted resources from
state. Then deliberately introduce three provider bugs and observe their symptoms: a `Read` that
errors instead of removing a deleted resource; a normalisation bug producing a perpetual diff; and
a create that returns before the resource is consistent. Write up each symptom as an operator would
see it.

**Stage 7 — the dangerous refactor.** Rename a resource inside a published module and consume the
new version. Observe the destroy-and-recreate. Then do it properly with `moved` blocks and show a
clean plan. Then write the module's versioning policy: what constitutes a breaking change, given
that state is part of the contract.

**Stage 8 — recovery.** Delete the state file (from a copy of a real environment, not production).
Recover by importing every resource. Time it. Then implement the preventive controls — state bucket
versioning, replication, and access restrictions — and write the recovery runbook. This exercise
converts "we should version the state bucket" from advice into a number.

**Stage 9 — blast radius documentation.** For each module, document: what a mistaken apply could
destroy, what is protected by `prevent_destroy` or provider-level deletion protection, what the
recovery procedure is, and what the recovery time would be. This is a required deliverable of the
term artifact, and it is the document that makes a platform trustworthy.

## 4. Failure modes

- **One state file for everything.** Slow plans, global locks, total blast radius.
- **State not encrypted or access-controlled.** It contains secrets.
- **No state versioning.** No undo, and recovery is a manual import of everything.
- **Environment conditionals inside modules.** Untestable, and production behaves differently from
  everything you tested.
- **Modules with sixty pass-through variables.** No abstraction, and a breaking change every time
  the underlying resource changes.
- **Renaming resources in a module without `moved` blocks.** Silent destroy-and-recreate for every
  consumer.
- **Perpetual diffs tolerated.** They train reviewers to stop reading plans, which disables the
  only real control.
- **Applying without reading the plan**, especially the replacement lines.
- **Console changes with no reconciliation.** The next apply reverts the emergency fix.
- **No `prevent_destroy` on stateful resources.**
- **Backups never restored.** An untested restore is a hypothesis.
- **A provider whose `Read` errors on a deleted resource.** The resource is stuck in state forever
  and every plan fails.

## 5. Exercises

### Warm-up (30 min)

1. Explain why IaC is a distributed systems problem, naming the four properties of the world that
   make it one.
2. Give three reasons state must be locked, encrypted and versioned.
3. Explain why state is part of a module's public contract, with the concrete failure.

### Core (3.5 h)

4. Complete Stages 1–3, including the forced-replacement visibility test.
5. Complete Stage 4 and report where each testing level caught something the others did not, with
   timings.
6. Complete Stage 5 and deliver the drift decision runbook.
7. Complete Stage 7 and deliver the module versioning policy.

### Challenge

8. Complete Stage 6 — the provider, with all three deliberate bugs and their operator-visible
   symptoms — and Stage 8 with the measured recovery time.
9. Build a **blast radius analyser for IaC**: parse a plan (the JSON output), classify each change
   as create / update-in-place / replace / destroy, cross-reference against a resource criticality
   annotation, and emit a risk score plus a human-readable summary that leads with the destructive
   changes. Then wire it into the pipeline so a high-risk plan requires a second approver. Run it
   against ten real plans and report its false positives — a checker that cries wolf is disabled
   within a month, so calibration is the actual engineering here.

## 6. Self-check

1. State the four properties of the world that make IaC a distributed systems problem.
2. What are the three properties the declarative model gives you, and where does each leak?
3. Give the two structural decisions about state and the reasoning for each.
4. Why is drift dangerous rather than merely untidy?
5. Give the four ways to handle a detected drift and the rule for choosing.
6. What makes a module a real abstraction rather than a wrapper?
7. What does a provider's `Read` do that is essential to the whole model, and what is the
   correctness rule for a deleted resource?
8. Name three provider subtleties and the operator-visible symptom of getting each wrong.
9. Give the six-step pipeline and identify which step is the actual control.
10. Why do perpetual diffs constitute a safety problem rather than a cosmetic one?

## 7. Primary sources

- **The Terraform documentation on state, `moved` blocks, and provider development** — read the
  state pages as distributed systems documents.
- **Morris, *Infrastructure as Code*, 2nd ed.** — the design-level treatment; the chapters on
  module interfaces and blast radius are the relevant ones.
- The Terraform Plugin Framework documentation and a small open-source provider's source — read one
  end to end; they are shorter than you expect.
- HashiCorp's module composition and registry publishing guidance.
- Open Policy Agent's Terraform tutorial, and the `checkov`/`tfsec` rule sets read as a catalogue of
  known misconfigurations.
- Google, *Site Reliability Engineering*, chapters 8 and 27 (release engineering, and reliable
  product launches).
- Humble & Farley, *Continuous Delivery* — the pipeline argument, which predates IaC and still
  governs it.

---

**Previous:** [L04](L04-network-architecture.md) · **Next:**
[L06 — Control Planes, Reconciliation, and Kubernetes](L06-control-planes-and-kubernetes.md)
