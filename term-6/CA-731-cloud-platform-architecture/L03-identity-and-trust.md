# CA-731 · Lesson 03 — Identity, Trust Boundaries, and Least Privilege

**Estimated study time:** 5 hours
**Prerequisites:** L01, L02; SE-521 L09

---

## 1. Orientation

In a datacentre, security architecture was largely topological: things inside the firewall were
trusted, things outside were not, and the perimeter was where you spent your effort. In cloud
systems that model does not survive contact with reality — the workload is spread across services
you do not run, the "inside" contains third-party code and other tenants, and the network location
of a request says almost nothing about whether it should be honoured.

What replaces it:

> **Identity is the primary security boundary.** Every request carries a verifiable claim about
> who is making it; every authorisation decision is made against that claim; and network position
> is at most a secondary signal.

This is the substance behind "zero trust", a term that has been so thoroughly marketed that its
technical content is worth restating plainly: **never trust a request because of where it came
from; authenticate and authorise every request, every time, from an identity that can be verified
cryptographically.**

The lesson covers the mechanisms (authentication, authorisation, credentials, delegation), the
discipline (least privilege, and why it is genuinely hard), and the specific failures that show up
in real incidents — because the failures are remarkably consistent and mostly avoidable.

## 2. Theory

### 2.1 Three separate questions

Constantly conflated, and each has its own mechanism:

- **Authentication (authn)**: *who is making this request?* Establishes identity.
- **Authorisation (authz)**: *is this identity permitted to do this?* Evaluates a policy.
- **Audit**: *what happened, and who did it?* Produces the record.

A system can authenticate perfectly and authorise terribly. A common real pattern: strong SSO at
the edge, and then every internal service trusting every other internal service completely, because
"they are inside the mesh". The perimeter moved; the trust model did not.

### 2.2 Workload identity: the mechanism that matters

Human identity is comparatively solved — SSO, MFA, a directory. **Workload identity** — how a
running process proves what it is — is where the interesting design lives, and it has a clear
hierarchy of quality:

1. **Long-lived static credentials** (an access key in an environment variable or, worse, in the
   repository). The worst option, and still the most common. They do not expire, they are copied,
   they leak into logs and images and CI output, and there is no way to tell who used one.
2. **Short-lived credentials from a metadata service.** The instance or container obtains
   temporary credentials from a local endpoint, scoped to a role, rotated automatically (IAM
   instance profiles, ECS task roles, GCP/Azure equivalents). Enormously better: nothing to leak
   permanently, automatic rotation, and identity tied to the workload rather than to a secret. The
   attack to know is **SSRF against the metadata endpoint** — a server-side request forgery that
   makes the application fetch its own credentials and return them — which is what IMDSv2's
   session-token requirement exists to prevent, and why hop limits and IMDSv1 deprecation matter.
3. **Federated identity / workload identity federation.** The workload proves its identity with a
   token from a trusted issuer (a Kubernetes service account token, a GitHub Actions OIDC token)
   and exchanges it for cloud credentials. This is what removes static keys from CI entirely, and
   it is the single highest-value change most organisations can make to their credential posture.
4. **Mutual TLS with short-lived certificates** (SPIFFE/SPIRE, service meshes). Each workload has a
   cryptographic identity with an automatically-rotated certificate; every connection authenticates
   both ends. The strongest general answer, and the most operational work.

The design rule: **no long-lived credentials anywhere, ever, if it can be avoided — and it almost
always can now.** Where one is unavoidable (a third party that supports nothing else), it is
inventoried, scoped as narrowly as possible, rotated on a schedule, and monitored for use from
unexpected sources.

### 2.3 Authorisation models

- **RBAC**: permissions attach to roles, roles to principals. Simple, auditable, and it explodes
  combinatorially when permissions need to depend on the resource ("this user, but only for their
  own team's projects") — producing hundreds of near-duplicate roles.
- **ABAC**: policies evaluate attributes of principal, resource, action and context ("allow if
  `principal.team == resource.team`"). Expressive, scales without role explosion, and is much
  harder to answer "who can access this?" about.
- **ReBAC** (relationship-based, Google Zanzibar's model): authorisation as a graph of
  relationships ("user is editor of document, document is in folder, user inherits from folder").
  Fits collaborative products naturally, and Zanzibar's paper is worth reading for the consistency
  design alone — it faces exactly DS-701 L04's problem, because a stale authorisation check is a
  security bug ("new enemy problem").
- **Policy as code** (OPA/Rego, Cedar): policies written and tested as code, evaluated by a
  dedicated engine, decoupled from application logic. The important property is not the language;
  it is that **policies become testable artifacts under version control**, which is what makes
  them reviewable.

Most real systems combine: coarse RBAC for the platform, ABAC or ReBAC within the application.
The design question is where each decision is made, and the answer that scales is: **authorisation
decisions are made in one place, by a component whose job that is, against a policy that is version
controlled and tested** — not scattered through handlers as `if user.is_admin` checks.

### 2.4 Least privilege, and why it is hard

The principle is trivial to state and hard to practise, for reasons worth naming rather than
moralising about:

- **Nobody knows what permissions are needed** until something fails, usually in production, at an
  inconvenient time. So the path of least resistance is a broad grant "to unblock the deploy",
  which is never narrowed.
- **Wildcards are seductive.** `s3:*` on `*` works immediately and is never revisited.
- **Permissions accumulate.** People and services gain permissions and never lose them; over years
  this produces principals with enormous, unexamined authority.
- **The blast radius is invisible** until an incident makes it concrete.

The practices that actually work, as opposed to the ones that are merely recommended:

1. **Start from deny, and derive the policy from observed use.** Run in a permissive mode with full
   logging, collect the actions actually performed, and generate the policy from that (AWS Access
   Analyzer does this from CloudTrail; the technique generalises). This inverts the problem from
   "guess what is needed" to "observe what is used".
2. **Continuously prune.** Alert on permissions unused for 90 days. Removal must be as routine as
   granting, or accumulation is guaranteed.
3. **Guardrails above grants.** Organisation-level policies (SCPs, Azure Policy, GCP org policies)
   that set a *maximum* boundary regardless of what individual grants say — "no principal in this
   account may disable logging, delete backups, or create resources outside these regions."
   Guardrails are more valuable than fine-grained grants because they hold even when someone gets
   the grant wrong.
4. **Separate the roles that grant permissions from the roles that use them.** A principal that can
   modify IAM has, in effect, all permissions — **privilege escalation via IAM is the most
   under-appreciated risk in cloud security**, and `iam:PassRole`, `iam:CreatePolicyVersion` and
   the ability to attach policies are effectively administrative permissions wearing a disguise.
5. **Break-glass with an audit trail.** Emergency access exists; make it explicit, time-bounded,
   loudly logged, and reviewed afterwards. A break-glass path that is inconvenient enough to avoid
   in a real incident will be routed around, and the workaround will be worse.
6. **Human access should be temporary by default.** Just-in-time elevation with approval, expiring
   automatically. Standing production access for humans is the anomaly, not the norm.

### 2.5 The trust boundary

Draw it explicitly. A **trust boundary** is a line across which data or requests move between
components with different trust levels, and at every such crossing you must state:

- What is authenticated, and how the credential is verified.
- What is authorised, and against what policy.
- What is validated (input validation belongs at trust boundaries — this is SE-521's boundary
  argument with a security consequence).
- What is logged.

Boundaries people forget, and which appear in incident reports:

- **Service to service inside the cluster.** "Internal" is not an authentication mechanism.
- **The CI/CD pipeline.** It usually has production credentials and executes code from pull
  requests. A pipeline that runs untrusted code with production access is a supply-chain incident
  waiting to be written up — and this specific pattern has produced several public ones.
- **Third-party dependencies.** Every package your build pulls executes code in your build
  environment.
- **The data plane of a managed service.** What can it reach on your behalf? A Lambda's role, a
  database's ability to make network calls, a CI runner's network position.
- **Between tenants** (L02) — the boundary you promised a customer exists.
- **Support and admin tooling**, which frequently has more authority than anything else and less
  review.

### 2.6 Secrets

Ordered by preference:

1. **No secret at all** — federated workload identity (§2.2). Always prefer this.
2. **A secret manager** with short-lived, dynamically-generated credentials (Vault's dynamic
   database credentials: a unique username and password per workload per hour, revoked
   automatically). Compromise is bounded in time and attributable.
3. **A secret manager** with static secrets, fetched at runtime, rotated on a schedule.
4. **Encrypted at rest in a config store**, decrypted at startup.
5. **Environment variables from a file on disk.** Widely used; leaks into process listings, crash
   dumps, logs and child processes.
6. **In the repository.** A breach with a delay. Note that removing a secret from a git repository
   does not remove it from history, from forks, or from anyone's clone — a leaked secret is
   rotated, not deleted.

Supporting practices: secret scanning in CI *and* on the git history; a documented rotation
procedure that has actually been executed at least once; alerting on secret access from unexpected
principals or locations; and — the one people skip — an *inventory*, because you cannot rotate what
you do not know exists.

### 2.7 What actually goes wrong

From published breach reports and postmortems, the recurring pattern in cloud security incidents is
not exotic:

1. **An over-permissioned credential** (usually a service account with far more access than its
   function requires).
2. **Obtained through a mundane path** — a leaked key in a repository or CI log, an SSRF reaching a
   metadata endpoint, a compromised dependency, a phished human.
3. **Lateral movement**, because internal services trust each other and the credential worked
   everywhere.
4. **Discovered late**, because the access looked like normal API traffic and nothing alerted.

Which tells you where effort belongs, and it is not where most of it goes: **eliminate long-lived
credentials, bound the blast radius of any single identity, and detect anomalous use.** Those three
address the actual pattern. Sophisticated attacks exist; they are not what is producing the
incidents.

## 3. Construction: identity, policy, and privilege

Build in `mpse/ca731/l03/`. Use your cloud account's free tier, or LocalStack for the IAM-shaped
exercises.

**Stage 1 — the credential audit.** For a system you have access to, inventory every credential:
what it is, where it lives, when it was created, when it was last rotated, and what it can do.
Categorise each by §2.2's hierarchy. Most audits find at least one long-lived key nobody
remembers creating — and finding it is the exercise working.

**Stage 2 — eliminate a static credential.** Take one CI pipeline using a long-lived key and
convert it to OIDC federation. Verify the old key can be deleted, then delete it. Document what
changed in the trust relationship — specifically, what the cloud provider is now trusting and what
would happen if that issuer were compromised.

**Stage 3 — the SSRF demonstration.** In an isolated environment, build a small service with a
deliberate SSRF vulnerability (a URL fetcher with no allowlist), and use it to retrieve instance
credentials from the metadata endpoint. Then enable IMDSv2 with a hop limit and show the attack
failing. Then fix the SSRF itself. Three layers, each demonstrated — and note which one you would
rely on if you could only have one.

**Stage 4 — policy as code.** Implement an authorisation service with OPA or a Cedar-style engine.
Write policies for a realistic scenario with resource ownership, team membership and role
inheritance. Write the *test suite* for the policies — including negative tests, which is where
policy bugs live. Then make a change that looks safe and is not, and show the tests catching it.

**Stage 5 — the three models.** Implement the same authorisation requirements three ways: RBAC,
ABAC, and a Zanzibar-style relationship graph. Compare: lines of policy, the effort to answer "who
can access resource X?", the effort to answer "what can principal Y access?", and the behaviour
when a new requirement arrives ("external collaborators can view but not comment"). Report which
questions each model makes easy and which it makes nearly impossible.

**Stage 6 — least privilege, derived.** Take an over-permissioned role (`*` on a service). Run the
workload with full API logging for a period. Generate a minimal policy from the observed calls.
Apply it and see what breaks — something will, because the log period missed a rare path. Write up
what that tells you about the technique's limits and what you would do about it (longer observation,
a permissive-with-alerting mode, staged tightening).

**Stage 7 — privilege escalation.** In an isolated account, construct a role that appears limited
but permits escalation to administrator — `iam:PassRole` with a permissive target, or the ability
to create a new policy version. Demonstrate the escalation. Then write the detection: what would
have to be logged and alerted to catch it. This exercise changes how you read IAM policies
permanently.

**Stage 8 — the trust boundary document.** For your L02 multi-tenant service, draw every trust
boundary and, for each crossing, state what is authenticated, authorised, validated and logged.
Then find the boundary you had not drawn — there is always one, and it is usually the CI pipeline
or the admin tooling.

**Stage 9 — guardrails.** Implement organisation-level guardrails that hold regardless of
individual grants: no disabling of audit logging, no deletion of backups, no resources outside
approved regions. Test each by attempting the action with an otherwise-administrative principal.

## 4. Failure modes

- **Long-lived static credentials.** The root cause of a large share of real breaches.
- **Trusting network position.** "It came from inside the VPC" authenticates nothing.
- **`*` in a policy.** Fast to write, permanent in practice.
- **IAM-modifying permissions treated as ordinary.** They are administrative permissions in
  disguise.
- **`iam:PassRole` granted without a resource constraint.** A direct escalation path.
- **Secrets in environment variables, then in logs.** Crash dumps and process listings leak them.
- **A rotation procedure never executed.** It does not work; you just have not found out yet.
- **Deleting a leaked secret from a repository instead of rotating it.** Git history is forever,
  and so are forks.
- **CI pipelines with production credentials running untrusted code.** A well-documented
  supply-chain pattern.
- **Authorisation decisions scattered through application code.** Unauditable and inconsistent.
- **No detection.** The credential worked, the API calls looked normal, and nobody was alerted.
- **Break-glass so inconvenient it is bypassed.** The workaround becomes the real path and is
  unaudited.

## 5. Exercises

### Warm-up (30 min)

1. Distinguish authentication, authorisation and audit, and give a system that does the first well
   and the second badly.
2. Give the four levels of workload identity in order, and say what each fixes about the one below.
3. Explain the SSRF-to-metadata attack and the three independent controls that stop it.

### Core (3.5 h)

4. Complete Stages 1–3, including the credential inventory and the three-layer SSRF demonstration.
5. Complete Stage 4: a policy engine with a test suite, including the safe-looking change caught by
   a negative test.
6. Complete Stage 6 and write up what the derived-policy technique cannot catch.
7. Complete Stage 8 and name the boundary you had not drawn.

### Challenge

8. Complete Stages 5, 7 and 9 — the three-model comparison, the escalation demonstration with its
   detection, and the guardrails with their tests.
9. Build a **blast radius analyser**: given an IAM policy set (or a Kubernetes RBAC configuration),
   compute for each principal the transitive set of actions it can reach — including escalation
   paths via `PassRole`, policy modification, role assumption chains, and any path that leads to
   modifying IAM itself. Output a ranked report of principals by reachable authority. Run it
   against a real configuration and report what you found. Then state honestly what your analyser
   cannot see: resource-based policies, condition keys you did not model, permissions granted
   outside IAM. That limitation list is the most valuable part of the exercise, because it is the
   reason automated analysis supplements review rather than replacing it.

## 6. Self-check

1. State the identity-as-boundary principle and the technical content of "zero trust".
2. Give the four levels of workload identity and the attack that motivates IMDSv2.
3. Compare RBAC, ABAC and ReBAC, giving a question each makes easy and one it makes hard.
4. Give four reasons least privilege is hard in practice.
5. What is a guardrail, and why is it more valuable than a fine-grained grant?
6. Why are IAM-modifying permissions effectively administrative?
7. Give the secrets hierarchy, and say why the top option is qualitatively different.
8. Why is deleting a leaked secret from a repository insufficient?
9. Give six trust boundaries teams commonly fail to draw.
10. Give the four-step pattern of a typical cloud breach and the three defences that address it.

## 7. Primary sources

- **Beyer et al., *Building Secure and Reliable Systems* (Google, 2020)** — free online; chapters
  on least privilege and design for understandability are the core reading.
- **Pang et al., "Zanzibar: Google's Consistent, Global Authorization System" (USENIX ATC 2019)** —
  the ReBAC model, and an authorisation system that takes consistency seriously.
- Google, "BeyondCorp" papers (2014–2018) — the original zero-trust deployment, described by the
  people who did it.
- NIST SP 800-207, *Zero Trust Architecture* — the sober reference behind the marketing.
- The SPIFFE/SPIRE specification and documentation.
- The Open Policy Agent documentation, and the AWS Cedar language documentation.
- AWS IAM documentation on `PassRole`, permission boundaries and SCPs; and the equivalent Azure
  and GCP organisation policy documents.
- Published cloud breach analyses — the Capital One (2019) SSRF incident and the several
  CI/CD-credential supply-chain incidents. Read the technical write-ups, not the news coverage.

---

**Previous:** [L02](L02-multi-tenancy-and-isolation.md) · **Next:**
[L04 — Network Architecture and the Perimeter That Isn't](L04-network-architecture.md)
