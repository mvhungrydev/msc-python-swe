# CA-731 — Cloud & Platform Architecture at Scale

**Term:** 6 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** DS-701, SE-521
**Co-requisites:** ML-741, FM-751

---

## Driving question

> What is the platform's actual contract with its users?

Not the marketing contract, and not the SLA document. The *actual* one: what a service really
guarantees, what it silently assumes about your behaviour, what it does when you exceed a limit
you did not know existed, what it charges you for in ways the pricing page does not make obvious,
and what happens to your workload when one of its dependencies has a bad afternoon.

This course treats the cloud as what it is — **a distributed system (DS-701) with a billing model
and a control plane** — and treats an internal platform as what it should be: a product with
users, a contract, and a support obligation. The two halves are the same discipline at different
scales, which is why they are one course.

The professional judgement being trained here is specific: most cloud architecture material is
either vendor documentation (accurate, unbalanced) or conference talks (aspirational,
unfalsifiable). What is missing is the ability to look at a managed service and ask *what
distributed systems problem did they solve, which of DS-701's trade-offs did they choose, and what
does that choice cost me?* Once you can do that, the specific provider matters much less than it
appears to.

## Learning outcomes

On completion you will be able to:

1. **Read** a managed service's documentation and identify the DS-701 trade-off it embodies, the
   failure modes it inherits, and the guarantee it actually provides.
2. **Design** for multi-tenancy, stating the isolation model and its failure boundary.
3. **Reason** about identity and the trust boundary as the primary security architecture, and
   apply least privilege in a system where it is genuinely inconvenient.
4. **Design** cloud network architecture — addressing, segmentation, egress, service-to-service
   authentication — and say what each control actually prevents.
5. **Build** infrastructure as code with a tested, versioned module interface, and reason about
   state, drift and blast radius.
6. **Explain** Kubernetes as an instance of the control-plane pattern, and design a controller of
   your own.
7. **Operate** to an SLO with an error budget, and connect availability targets to architecture
   decisions with arithmetic.
8. **Model** capacity and cost, and identify the architectural choices that dominate the bill.
9. **Design** a platform as a product: golden paths, self-service, versioned interfaces, and a
   support model.
10. **Assess** a design against failure, cost, security and operability together, rather than one
    at a time.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | The Cloud as a Distributed System with a Bill | 4 |
| L02 | Multi-Tenancy and Isolation | 4.5 |
| L03 | Identity, Trust Boundaries, and Least Privilege | 5 |
| L04 | Network Architecture and the Perimeter That Isn't | 4.5 |
| L05 | Infrastructure as Code and Provider Design | 5 |
| L06 | Control Planes, Reconciliation, and Kubernetes | 5 |
| L07 | Reliability Engineering: SLOs, Error Budgets, and Architecture | 4.5 |
| L08 | Capacity, Cost, and the Economics of Architecture | 4.5 |
| L09 | Platform Engineering as a Product Discipline | 4 |
| L10 | Reviewing a Design: Failure, Cost, Security, Operability | 4 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a multi-tenant service with a stated isolation and trust model | 20% |
| Problem set 2 (L05–L07): infrastructure as code, a controller, and an SLO with teeth | 20% |
| Problem set 3 (L08–L10): a costed, reviewed platform design | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 6 build artifact (with ML-741 and FM-751)

**A small internal platform**, built and operated as a product:

- A **multi-tenant service** with a stated isolation model and a demonstrated noisy-neighbour
  boundary.
- **Infrastructure as code** with a module interface, tests, and a documented blast radius per
  change.
- A **custom controller** implementing the reconciliation pattern against a declared desired
  state, with the failure cases handled.
- **SLOs with error budgets** and multi-burn-rate alerting (DS-701 L10), connected to at least one
  architectural decision you changed because of the arithmetic.
- A **cost model** for the platform with a per-tenant unit cost, and an identified architectural
  choice that dominates it.
- A **golden path**: a documented, self-service way for a user team to get from nothing to running,
  with the interface versioned and the support model stated.
- A **design review document** (L10) covering failure, cost, security and operability, written to
  the standard a staff engineer would expect.

ML-741 supplies a workload for this platform to run; FM-751 supplies a specification for one of
its protocols. The three courses are deliberately entangled — a platform with nothing on it, and
no proof its design is right, is not a platform.

## A note on providers

The lessons are provider-neutral in their concepts and use **AWS as the worked example** where a
concrete system is needed, because a course that names no specifics teaches nothing operational.
Everything transfers: the equivalent Azure and GCP services are named at each point, and the
exercises are written so they can be completed on any of the three, or locally with LocalStack and
kind for those without an account.

Where the course criticises a provider's design, the criticism is technical and specific, and you
should extend the same scepticism to this course's claims. Prices and limits change; verify
current numbers before relying on them, and log any that have moved in
`appendices/errata.md`.

## Required reading

- **Google, *Site Reliability Engineering*** and ***The SRE Workbook*** — free online; chapters 3–6,
  and Workbook chapters 2 and 5, are core.
- **Burns, Grant, Oppenheimer, Brewer & Wilkes, "Borg, Omega, and Kubernetes" (ACM Queue, 2016)** —
  the design lineage, from the people who built all three.
- **Verma et al., "Large-scale cluster management at Google with Borg" (EuroSys 2015).**
- Hashimoto et al. / HashiCorp, the Terraform documentation on state and providers — read the
  state section as a distributed systems document, because it is one.
- Brooker, "Reliability, Constant Work, and a Good Cup of Coffee" and the AWS Builders' Library
  generally — the most honest public writing about operating cloud services.
- Skelton & Pais, *Team Topologies* — for L09; the platform-as-product argument.
- Newman, *Building Microservices*, 2nd ed., chapters 8–11 — deployment, security, and the
  organisational reality.
- The **AWS Well-Architected Framework** — read it critically, as a checklist with a vendor's
  incentives, not as scripture.

## Recommended

- Beyer et al., *Building Secure and Reliable Systems* (Google) — free; the security half of L03
  and L04.
- Hightower, Burns & Beda, *Kubernetes: Up and Running*; and the Kubernetes API conventions
  documentation, which is the real specification of the control-plane pattern.
- Morris, *Infrastructure as Code*, 2nd ed.
- Vogels, "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database
  Service" (USENIX ATC 2022) — a managed service explained by its builders; the model for L01's
  reading skill.
- Cloud provider post-incident reports — the AWS, Azure and GCP public postmortems. Read six.
  They teach more about cloud architecture than any book.
