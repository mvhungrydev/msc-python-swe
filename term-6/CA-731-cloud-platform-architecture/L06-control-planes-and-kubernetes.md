# CA-731 · Lesson 06 — Control Planes, Reconciliation, and Kubernetes

**Estimated study time:** 5 hours
**Prerequisites:** L01, L05; DS-701 L05, DS-701 L09; PY-601 L06

---

## 1. Orientation

Kubernetes is usually taught as a set of object types to memorise. That is the least interesting
thing about it. What is worth learning is the **pattern** it implements, because that pattern is
how every modern control plane works — cloud providers' own control planes, Terraform (L05),
autoscalers, certificate managers, and any platform you build yourself.

The pattern:

> **Store declared desired state. Observe actual state. Run a loop that continuously computes the
> difference and acts to reduce it. Never assume the action succeeded; observe again.**

This is a control system in the engineering sense — a feedback loop with a setpoint — and its
properties are what make it robust: it is **level-triggered** rather than edge-triggered (it acts on
the current difference, not on the event that caused it), so a missed event, a duplicated event, a
restart, or an out-of-order delivery are all self-correcting. That single property is why
reconciliation beats imperative orchestration in an unreliable world, and it is the same argument
as DS-701 L07's idempotent-merge and L08's idempotence, arriving from a third direction.

The lesson's aim is that by the end you can (a) explain what happens between `kubectl apply` and a
running pod, in enough detail to debug it, and (b) write a controller of your own — which is what
converts Kubernetes from a system you operate into a pattern you can apply.

## 2. Theory

### 2.1 The reconciliation loop

```
for ever:
    desired = read_spec()          # what the user declared
    actual  = observe_world()      # what is really there
    if actual != desired:
        act_to_converge()          # take one step toward desired
    report_status()                # what the controller believes
```

Properties, each of which matters operationally:

- **Level-triggered, not edge-triggered.** The loop reads current state; it does not depend on
  having seen the transition. Lost events do not cause permanent divergence.
- **Idempotent.** Running it twice is running it once. Restarts are safe.
- **Eventually consistent.** Convergence is not instantaneous, and the system may pass through
  states that satisfy neither the old nor the new spec.
- **Self-healing.** Someone deletes a pod; the loop notices the difference and recreates it. Nobody
  had to detect the deletion as an event.

And the properties that bite:

- **A reconciler that cannot converge loops forever.** An unsatisfiable spec (a pod requesting more
  memory than any node has) produces an infinite retry, and unbounded retries are a load problem
  (DS-701 L09). Hence rate limiting and exponential backoff in every serious controller.
- **Two controllers with overlapping authority fight.** Each undoes the other's work, forever. This
  is the most common serious bug in custom controllers, and it manifests as resource thrashing with
  high API server load.
- **Status is a belief, not a fact.** The `status` field is what a controller last observed;
  treating it as truth without checking `observedGeneration` is how you act on stale information.

### 2.2 The Kubernetes architecture, as an instance

- **API server**: the only component that talks to etcd; provides a RESTful, versioned,
  validated, authenticated and authorised interface with *watch* semantics. Everything else is a
  client. This is a strong design decision — one gate, one place for policy — and it is why
  admission control (§2.4) is possible at all.
- **etcd**: a Raft-based (DS-701 L05) consistent key-value store. Linearizable reads and writes,
  and the availability characteristics of a majority-quorum system: lose the majority and the
  cluster becomes read-only at best. etcd's performance is disk-latency-bound, which is why
  Kubernetes control planes are sensitive to disk performance in a way that surprises people.
- **Controller manager**: a collection of reconcilers — deployment, replicaset, node, endpoint,
  and others. Each watches the resources it cares about and drives convergence.
- **Scheduler**: assigns pods to nodes. Also a reconciler, over unscheduled pods.
- **Kubelet**: on each node, reconciles "the pods assigned to me" against "the containers actually
  running".
- **Controllers everywhere else**: ingress controllers, cert-manager, autoscalers, operators. All
  the same pattern.

The **watch** mechanism is worth understanding because it is what makes this efficient: clients
open a long-lived connection and receive changes since a `resourceVersion`, rather than polling.
The client-side machinery — an informer with a local cache, a work queue with rate limiting and
deduplication, and a periodic full resync to correct any missed event — is the standard controller
runtime, and the periodic resync exists precisely because watches can miss things and
level-triggering makes that recoverable.

### 2.3 What actually happens on `kubectl apply`

Being able to narrate this is the debugging skill:

1. `kubectl` sends the manifest to the API server.
2. **Authentication** (certificate, token, OIDC), then **authorisation** (RBAC), then **admission**:
   mutating webhooks and defaulting, then validation, then validating webhooks, then quota
   enforcement.
3. The object is persisted to etcd. `kubectl` returns. **At this point nothing is running.**
4. The deployment controller sees a new Deployment and creates a ReplicaSet.
5. The replicaset controller sees a ReplicaSet with zero pods and creates Pod objects — with no
   node assigned.
6. The scheduler sees unscheduled pods, filters nodes by feasibility (resources, taints, affinity,
   topology constraints), scores the feasible ones, and writes a binding.
7. The kubelet on that node sees a pod bound to it, pulls images, calls the container runtime,
   configures networking via CNI, mounts volumes via CSI, and starts containers.
8. Probes determine readiness; the endpoints controller adds ready pods to the Service's endpoint
   slice; kube-proxy (or a CNI dataplane) programs the node's routing.

**Every arrow is asynchronous and every step can fail.** Which is why the debugging procedure
follows the same path: is the object in etcd (`kubectl get`)? Was it admitted (events)? Did the
controller create the child (`kubectl get rs`)? Was it scheduled (`kubectl describe pod` — the
scheduler writes its reason)? Did the image pull? Did the container start? Did the probes pass? Is
it in the endpoints? Each question has a specific command and each isolates one step. Learn the
sequence, not the flags.

### 2.4 Extending the control plane

- **Custom Resource Definitions** add new object types to the API server, with an OpenAPI schema,
  validation, versions with conversion, and subresources (`status`, `scale`). You get the
  authentication, authorisation, admission, watch, and audit machinery for free — which is the
  reason to build on Kubernetes rather than beside it.
- **Operators** are CRDs plus a controller encoding operational knowledge: how to provision this
  database, how to take a backup, how to perform a version upgrade safely. The valuable ones encode
  genuinely difficult procedures; the valueless ones wrap a Helm chart in Go.
- **Admission webhooks** intercept requests: *mutating* (inject a sidecar, set defaults) and
  *validating* (enforce policy). The operational hazards are real and worth stating: a webhook that
  is down blocks every affected API request if `failurePolicy: Fail`, and a webhook that
  accidentally applies to `kube-system` can prevent the cluster from recovering. **A misconfigured
  webhook is one of the few ways to make a Kubernetes cluster unrecoverable**, so scope
  `namespaceSelector` and `objectSelector` tightly, exclude system namespaces, and set a low
  timeout.
- **Policy engines** (Kyverno, OPA Gatekeeper) are validating webhooks with a policy language, and
  they are how organisational guardrails (L03 §2.4) reach the cluster.
- **Scheduler extensions and custom schedulers** when placement is genuinely special.

### 2.5 Writing a controller correctly

The rules, each of which corresponds to a bug you will otherwise write:

1. **Reconcile from observed state, not from the event.** The event tells you *to look*; it does
   not tell you what to do. A controller that acts on the event's contents is edge-triggered and
   will diverge.
2. **Be idempotent.** `Reconcile` will be called repeatedly for the same object, including
   immediately after you finish.
3. **Own a clear set of resources.** Use owner references so children are garbage-collected with
   the parent, and never write to resources another controller owns.
4. **Report status honestly**, including `observedGeneration` so consumers can tell whether the
   status reflects the current spec. Use standard conditions (`Ready`, `Progressing`) with
   reasons; this is what makes a resource debuggable by someone who has never seen your controller.
5. **Rate limit and back off.** The work queue's exponential backoff exists because an
   unsatisfiable spec otherwise becomes a denial of service against the API server.
6. **Handle deletion with finalizers** when external cleanup is required — and handle the case
   where cleanup *cannot* succeed, because **a finalizer that never completes makes an object
   undeletable and blocks namespace deletion forever.** This is a common, embarrassing production
   problem.
7. **Assume concurrent modification.** Use optimistic concurrency (resource version conflicts) and
   requeue on conflict rather than retrying blindly (DI-721 L04's CAS, again).
8. **Leader-elect** if you run multiple replicas, or two instances will fight (§2.1).

### 2.6 Kubernetes as a platform substrate — and when not to

The honest case *for* building your platform on Kubernetes: you get a consistent declarative API,
authentication and authorisation, admission control, an extension model, a large ecosystem, and —
most valuable — a *uniform* way to express "here is desired state, converge to it" across everything
you operate. That uniformity is worth a great deal at organisational scale.

The honest case *against*: the complexity is substantial and irreducible; the number of things that
can be misconfigured is enormous; the failure modes are subtle and often require deep knowledge to
diagnose; upgrades are ongoing work forever; and for a small number of services, a managed
container service or plain VMs with L05's IaC is simpler and adequate.

The rule of thumb worth stating plainly: **Kubernetes pays off when you have enough services,
enough teams, and enough platform-shaped work that a uniform control plane saves more than it
costs.** Below that threshold it is a large fixed cost — and running your own control plane rather
than a managed one raises the threshold considerably. Very few organisations should operate their
own etcd.

The transferable insight regardless of your choice: **the control-plane pattern is the valuable
part, and it is not exclusive to Kubernetes.** L05's Terraform provider, an internal service
reconciling desired state from a database, and a cloud provider's own control plane are all the
same idea. Being able to build one is worth more than knowing Kubernetes' object types.

## 3. Construction: use the pattern, then implement it

Build in `mpse/ca731/l06/`. `kind` or `k3d` locally is sufficient for everything here; no cloud
account required.

**Stage 1 — narrate the path.** Deploy a service and trace every step of §2.3 with commands and
their output. Then break it at five distinct steps — an invalid image, insufficient resources so it
cannot be scheduled, a failing readiness probe, a missing secret, and a mutating webhook that
rejects it — and for each record the symptom and the single command that localises the failure.
Build the symptom-to-command table.

**Stage 2 — watch the loop.** Delete a pod owned by a Deployment and watch the reconciliation with
`kubectl get -w` and the controller manager's logs. Then delete the Deployment's ReplicaSet and
watch the deployment controller recreate it. Then, in a scratch cluster, stop the controller
manager, make changes, restart it, and observe convergence from an arbitrary state. That last
experiment is the level-triggering property, demonstrated.

**Stage 3 — a CRD.** Define a `TenantEnvironment` resource for your L02 multi-tenant platform: a
schema with validation, a `status` subresource, and printer columns. Apply it and confirm the API
server validates, stores and serves it with no controller present at all — which is the point about
getting the machinery for free.

**Stage 4 — the controller.** Implement a reconciler for `TenantEnvironment` (Kopf for Python, or
controller-runtime if you prefer Go — and building it in Python is more instructive here because
you write the loop rather than inheriting it). It should create a namespace, a resource quota, a
network policy and a deployment, with owner references. Verify: deleting the custom resource
garbage-collects everything; deleting a child recreates it; and the controller converges after
being stopped and restarted.

**Stage 5 — status and conditions.** Add `observedGeneration` and standard conditions with reasons.
Then construct the case that motivates `observedGeneration`: change the spec, and show that a naive
consumer reading `status.ready` acts on a stale belief. Fix the consumer.

**Stage 6 — the failure modes, deliberately.** Produce each of these and record the symptom:
(a) an unsatisfiable spec causing a hot reconcile loop — then fix it with backoff and measure the
API server request rate before and after; (b) two controllers with overlapping ownership fighting;
(c) a finalizer that cannot complete, making an object undeletable — then implement the escape
hatch and document the manual recovery.

**Stage 7 — an admission webhook.** Write a validating webhook enforcing an organisational policy
(every workload must have an owner label, resource limits, and a non-root security context). Then
demonstrate the hazard: set `failurePolicy: Fail`, take the webhook down, and observe that affected
API requests now fail. Then scope it properly with selectors excluding system namespaces, and set
a timeout. Write the operational runbook for "the webhook is down".

**Stage 8 — the same pattern, off Kubernetes.** Implement the identical `TenantEnvironment`
reconciliation as a standalone service: desired state in a database, a reconcile loop, leader
election (use your DS-701 Raft, or a database lease with a fencing token), status reporting, and
backoff. Compare the two implementations: lines of code, what you had to build yourself, what you
got for free, and what each is better at. This comparison is the lesson's real deliverable, because
it separates the pattern from the product.

## 4. Failure modes

- **Edge-triggered controllers.** Act on the event's payload; diverge permanently on a missed
  event.
- **Two controllers owning the same resource.** Endless thrashing and API server load.
- **No rate limiting.** An unsatisfiable spec becomes a self-inflicted denial of service.
- **A finalizer that cannot complete.** The object and its namespace become undeletable.
- **Status treated as truth.** Without `observedGeneration`, consumers act on stale beliefs.
- **A webhook with `failurePolicy: Fail` and no selector scoping.** Can render a cluster
  unrecoverable.
- **Ignoring resource version conflicts.** Blind retries overwrite concurrent changes.
- **No leader election with multiple replicas.** Duplicate work and conflicting actions.
- **Operating your own etcd without understanding quorum.** Losing the majority means read-only,
  and restoring from a snapshot is a procedure you must have practised.
- **Adopting Kubernetes for three services.** A large fixed cost against a small benefit.
- **Confusing liveness and readiness probes** (L04 §2.4) — a failing dependency triggers restart
  loops.

## 5. Exercises

### Warm-up (30 min)

1. Write the reconciliation loop in pseudocode and state its four properties.
2. Explain level-triggered versus edge-triggered and give the concrete failure of the latter.
3. Narrate the eight steps from `kubectl apply` to a serving pod, marking every asynchronous
   boundary.

### Core (3.5 h)

4. Complete Stages 1–2 and deliver the symptom-to-command table plus the convergence-from-arbitrary
   -state demonstration.
5. Complete Stages 3–5: the CRD, the controller, and the `observedGeneration` demonstration.
6. Complete Stage 6 with all three failure modes produced and their symptoms recorded, including
   the API server request-rate measurement.
7. Complete Stage 7, including the outage demonstration and the runbook.

### Challenge

8. Complete Stage 8 and deliver the comparison — this is the required deliverable.
9. Extend your controller into a **genuine operator**: it must handle a version upgrade of the
   managed workload safely (drain, upgrade, verify, roll back on failure), take and restore
   backups, and expose a `status` rich enough that an operator can diagnose a stuck upgrade without
   reading logs. Then write the chaos experiments (DS-701 L10) that validate it: kill the
   controller mid-upgrade, kill it mid-backup, make the upgrade fail at each stage, and partition
   the controller from the API server. Report which failures your controller handled correctly and
   which required a fix — and note that the ones requiring a fix are exactly the ones a
   Helm-chart-wrapper operator would never have surfaced.

## 6. Self-check

1. State the reconciliation pattern and its four properties.
2. Why is level-triggering more robust than edge-triggering in an unreliable network?
3. Why does every serious controller need rate limiting and backoff?
4. What is the API server's role, and what does routing everything through it make possible?
5. Give the eight steps of `kubectl apply` and the command that localises a failure at each.
6. What is a watch, and why is a periodic full resync still necessary?
7. Give the eight rules for writing a controller and the bug each prevents.
8. How can an admission webhook make a cluster unrecoverable, and what prevents it?
9. What does a finalizer that cannot complete do, and how do you recover?
10. State the honest case for and against Kubernetes as a platform substrate, and the threshold
    rule.

## 7. Primary sources

- **Burns, Grant, Oppenheimer, Brewer & Wilkes, "Borg, Omega, and Kubernetes" (ACM Queue, 2016)** —
  the design lineage and the reasoning; short and essential.
- **Verma et al., "Large-scale cluster management at Google with Borg" (EuroSys 2015).**
- **The Kubernetes API conventions document** — the actual specification of the pattern, including
  the rules for `status`, conditions and `observedGeneration`. Most controller bugs are violations
  of this document.
- The controller-runtime and client-go documentation on informers, work queues and leader election.
- Dobies & Wood, *Kubernetes Operators*; and the Operator SDK's capability model.
- Kyverno and OPA Gatekeeper documentation, read as policy-as-code for clusters (with L03).
- Hightower, Burns & Beda, *Kubernetes: Up and Running* — for the object model, if you need it.
- Published Kubernetes postmortems and the `kubernetes/kubernetes` issue tracker for webhook and
  finalizer incidents — the failure modes in §4 all have public examples.

---

**Previous:** [L05](L05-infrastructure-as-code.md) · **Next:**
[L07 — Reliability Engineering: SLOs, Error Budgets, and Architecture](L07-reliability-engineering.md)
