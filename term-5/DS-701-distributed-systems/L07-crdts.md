# DS-701 · Lesson 07 — CRDTs and Eventual Consistency

**Estimated study time:** 4 hours
**Prerequisites:** L02, L03, L04

---

## 1. Orientation

Consensus (L05) buys you strong consistency at the cost of coordination: a round trip to a
majority for every write, and no writes at all on the minority side of a partition.

CRDTs buy you something different: **replicas that always accept writes, never coordinate, and
provably converge.** No consensus, no leader, no quorum. Available under any partition,
including one where a replica is a phone that has been offline for a week.

The price is that you must express your data as a structure whose merge operation is
mathematically well-behaved. That is a real constraint, and the honest framing of this lesson
is: **CRDTs are a beautiful answer to a specific class of problem, and the skill is
recognizing whether you have that problem.**

## 2. Theory

### 2.1 Strong eventual consistency

Eventual consistency (L04 §2.5) promises only that replicas converge if updates stop.

**Strong eventual consistency** (SEC) promises more:

> **Replicas that have received the same set of updates are in the same state** — regardless of
> the order in which they received them, and with no conflict resolution required.

That is a much stronger and much more useful guarantee, and it is what CRDTs provide.

The mathematics that delivers it: the state forms a **join-semilattice** and the merge
operation is the **least upper bound**. The merge must be:

- **Commutative**: `merge(a, b) = merge(b, a)` — order does not matter.
- **Associative**: `merge(merge(a,b), c) = merge(a, merge(b,c))` — grouping does not matter.
- **Idempotent**: `merge(a, a) = a` — duplicates do not matter.

Those three properties are exactly what make the merge safe under an unreliable network: **you
can deliver messages out of order, duplicated, and repeatedly, and the result is the same.**
Which means no ordering guarantees are needed, no exactly-once delivery is needed (L01 §2.6),
and no coordination is needed.

Additionally, updates must be **monotonic** — the state only ever moves up the lattice, never
back down. That is why deletion is the hard case (§2.4), and it is the CALM theorem's
monotonicity condition (L01 §2.6) in concrete form.

### 2.2 State-based versus operation-based

**State-based (CvRDT)**: replicas exchange full state and merge with the lattice join.

- Simple, and robust: duplicated or reordered messages are harmless by construction.
- Expensive: shipping the whole state. **Delta CRDTs** fix this by shipping only the changed
  part, and are what production systems use.

**Operation-based (CmRDT)**: replicas broadcast operations, which must be commutative.

- Cheaper messages.
- Requires **exactly-once, causally-ordered delivery** — which pushes work back into the
  messaging layer, and which is precisely the hard problem CRDTs were avoiding. Read this
  carefully: op-based CRDTs need reliable causal broadcast, which is not free.

The two are equivalent in expressive power, and **state-based (with deltas) is the right
default** because its robustness assumptions are so much weaker.

### 2.3 The catalogue

**G-Counter (grow-only counter).** A vector of per-replica counts. Increment your own entry;
merge takes the element-wise maximum; the value is the sum.

```python
@dataclass
class GCounter:
    counts: dict[str, int]
    def inc(self, node: str, n: int = 1) -> None:
        self.counts[node] = self.counts.get(node, 0) + n
    def merge(self, other: "GCounter") -> "GCounter":
        return GCounter({k: max(self.counts.get(k,0), other.counts.get(k,0))
                         for k in self.counts.keys() | other.counts.keys()})
    def value(self) -> int:
        return sum(self.counts.values())
```

Element-wise max is commutative, associative, and idempotent. Done.

**PN-Counter.** Two G-Counters, one for increments and one for decrements; the value is the
difference. Supports decrement, at the cost of double the state.

**G-Set.** Add only. Union merges.

**2P-Set.** An add-set and a remove-set (tombstones). Once removed, an element can never be
re-added — which is often unacceptable.

**LWW-Element-Set.** Each element carries a timestamp; the latest wins. Depends on clocks
(L02 §2.1), with all the associated data loss.

**OR-Set (observed-remove set).** The one that behaves correctly. Each *add* generates a unique
tag; a *remove* removes the tags it has observed. Concurrent add and remove → the add wins,
because the remove did not observe that add's tag. This matches the intuition that a concurrent
add should not be silently swallowed.

**Registers.** LWW-Register (timestamp wins, loses data) or MV-Register (multi-value: keeps all
concurrent values and returns them as siblings, as Dynamo does — the application resolves).

**Sequence CRDTs** for collaborative text: RGA, LOGOOT, Treedoc, and the ones used in practice —
**Yjs** and **Automerge**. Each character has a unique identifier with a dense total order, so
concurrent insertions at the same position get a deterministic order without coordination. This
is what makes real-time collaborative editing work, and it is the CRDT application that
genuinely changed a product category.

**Maps** compose CRDTs: a map whose values are CRDTs, merged pointwise.

### 2.4 Why deletion is hard

The general problem: **deletion is non-monotonic.** The lattice only goes up; removing
information moves down.

Consequences:

- **Tombstones.** A removal must be recorded, because otherwise a replica that has not seen it
  will re-add the element on the next merge. So you store a record of every deletion, forever.
- **Tombstone growth.** A set with heavy churn accumulates unbounded metadata. This is the
  practical objection to CRDTs and it is real.
- **Garbage collection needs coordination.** You can only discard a tombstone when *every*
  replica has seen it — which requires knowing about every replica and hearing from all of
  them. That is a form of coordination, so the "no coordination" claim is true for the data
  path and not for the maintenance path.

Mitigations: causal stability (discard a tombstone once it is causally stable across all known
replicas), version vectors instead of per-element tags, and — the pragmatic one — bounded
retention with an accepted risk.

**The honest summary**: CRDTs eliminate coordination on the write path and reintroduce a
weaker form of it for garbage collection. That is still a large win, and it is not "no
coordination ever".

### 2.5 Where CRDTs fit, and where they do not

**They fit when:**

- Updates are naturally commutative (counters, sets, maps, collaborative text).
- Availability under partition matters more than a global invariant.
- Clients may be offline for a long time (mobile, local-first).
- Multiple writers per object are expected.

**They do not fit when:**

- **A global invariant must hold.** "The balance must never go negative" is not expressible:
  two replicas can each independently approve a withdrawal that is individually valid and
  jointly overdraws. **No CRDT solves this**, because it is exactly the case where coordination
  is provably required (CALM's non-monotonicity).
- **Uniqueness is required.** One seat, one username.
- **The merge is genuinely application-specific** and not expressible as a lattice join.
- **The metadata overhead is unacceptable** for the data volume.

**The escrow / reservation pattern** is the standard partial answer to the invariant case:
partition the budget in advance (each replica gets 100 of the 1,000 units to spend without
coordination) and coordinate only when a replica exhausts its allocation. This turns
"coordinate on every write" into "coordinate rarely", and it is used for inventory, rate
limits, and quotas. It is worth knowing because it is applicable far more often than CRDTs
themselves.

### 2.6 Local-first software

The design philosophy CRDTs enable, articulated by Kleppmann et al. (2019):

1. The application works fully offline.
2. Data lives on the user's device; the server is a synchronization aid, not the source of
   truth.
3. Collaboration is real-time when connected.
4. Data outlives the company.

CRDTs are what make (1) and (3) compatible: every device holds a full replica, edits locally
with no latency, and merges when connected.

Real systems: Automerge, Yjs, Figma's (custom, not strictly a CRDT), Linear's sync engine, and
a growing category of local-first tools.

The engineering costs are real and worth stating: metadata size, the complexity of the merge
implementation, and — the one people underestimate — **schema evolution**, since old clients
must merge with data written by new ones. That last is DI-721 L09's subject and it is harder
in a local-first system than anywhere else, because you cannot force a client to upgrade.

### 2.7 Choosing between CRDTs and consensus

The decision:

| Question | CRDT | Consensus |
|---|---|---|
| Available under partition? | **yes, always** | no, minority side stops |
| Write latency | local, ~0 | round trip to a majority |
| Global invariants | **no** | yes |
| Uniqueness constraints | **no** | yes |
| Offline clients | **yes** | no |
| Metadata overhead | grows with writers and churn | log, compacted |
| Ordering guarantee | none needed | total order |
| Implementation risk | merge correctness | protocol correctness |

**And the third option, which is usually right**: use both, for different data. Consensus for
the small set of things needing a global invariant (who owns what, the configuration, the
uniqueness constraints), CRDTs or plain eventual consistency for the bulk of the data. That
layering is what most successful systems do, and it is the same architectural move as L05
§2.5's "consensus for small critical decisions".

## 3. Construction: implementing CRDTs

Extend the L01 laboratory.

**Stage 1 — the basic set.** Implement G-Counter, PN-Counter, G-Set, 2P-Set, LWW-Register, and
OR-Set. For each, **property-test the three lattice laws** (SE-511 L04 §2.2, pattern 4):

```python
@given(states(), states(), states())
def test_merge_is_a_semilattice(a, b, c):
    assert merge(a, b) == merge(b, a)                      # commutative
    assert merge(merge(a, b), c) == merge(a, merge(b, c))  # associative
    assert merge(a, a) == a                                # idempotent
```

Then the property that matters most:

```python
@given(operation_sequences(), permutations_and_duplications())
def test_convergence(ops, delivery):
    """Replicas receiving the same ops in any order, with duplicates, converge."""
    replicas = [apply_in_order(ops, order) for order in delivery]
    assert all(r == replicas[0] for r in replicas)
```

**That convergence test is the CRDT's specification**, and running it over randomly permuted
and duplicated delivery orders is what actually verifies your implementation.

**Stage 2 — break one deliberately.** Implement a set with a *non-commutative* merge (say,
"the merge takes the left operand's version on conflict"). Verify the property test catches it,
and construct the concrete divergence: two replicas receiving the same operations in different
orders ending in different states.

**Stage 3 — the OR-Set, properly.** The interesting one. Verify the concurrent add/remove
semantics: replica A adds `x`, replica B (not having seen the add) removes `x`, they merge →
**`x` is present**, because B's remove did not observe A's add tag. Then contrast with a
2P-Set, where the removal wins permanently, and with an LWW-Set under clock skew.

Then measure the **tag growth**: run 10⁶ add/remove cycles on one element and report the
metadata size.

**Stage 4 — run them distributed.** Deploy across your five nodes with gossip-based
anti-entropy (each node periodically merges with a random peer). Under continuous partition
injection:

- Verify convergence after every heal, and **measure the convergence time** (p50, p99, max) as
  a function of gossip interval and partition duration.
- Verify that every replica accepted every write during the partition — no unavailability.
- Compare with your L05 Raft store under the same fault schedule: measure the *unavailability*
  of the consensus store during partitions, and the *divergence window* of the CRDT store.

That comparison table is the deliverable, and it is the concrete form of §2.7.

**Stage 5 — delta CRDTs.** Convert your state-based CRDTs to ship deltas rather than full
state. Measure the bandwidth reduction on a realistic update pattern. Then verify that the
convergence property still holds when deltas are lost, duplicated, and reordered.

**Stage 6 — the invariant that cannot hold.** Implement a PN-Counter representing an account
balance. Under a partition, have two replicas each approve a withdrawal that is individually
valid. Demonstrate the negative balance after merge.

Then implement the **escrow pattern**: partition the balance in advance, and coordinate only on
exhaustion. Measure: how often coordination is needed as a function of the escrow size and the
withdrawal distribution, and what the balance guarantee now is. **This is the exercise that
teaches the boundary of what CRDTs can do**, and the escrow pattern is directly reusable.

**Stage 7 — a sequence CRDT.** Implement a simple collaborative text CRDT (RGA is the most
tractable). Verify: concurrent insertions at the same position converge to the same order on
all replicas; deletions work; the result is what a user would expect. Then measure the metadata
per character and the memory for a 100 KB document with a long edit history.

Then compare with Yjs or Automerge on the same workload and report the gap. Their optimizations
(run-length encoding of contiguous insertions, garbage collection of tombstones) are what makes
this practical, and seeing the difference is the lesson.

**Stage 8 — the design note.** For a real feature you know, decide: CRDT, consensus, or
neither. Answer §2.7's table and justify.

## 4. Failure modes

- **A merge that is not commutative, associative, or idempotent.** Silent divergence. The
  property test is not optional.
- **LWW with wall clocks.** L02 §2.1's data loss.
- **2P-Set where re-adding is expected.**
- **Unbounded tombstone growth.** The practical objection, and it is real.
- **Assuming CRDTs need no coordination at all.** Garbage collection does.
- **Op-based CRDTs without reliable causal broadcast.** The requirement is easy to miss and it
  is what the state-based version avoids.
- **Trying to enforce a global invariant with a CRDT.** Provably impossible; use escrow or
  coordinate.
- **Not measuring the metadata overhead** before committing to the approach.
- **Ignoring schema evolution** in a local-first system, where you cannot force an upgrade.
- **Shipping full state instead of deltas** for a large structure.

## 5. Exercises

### Warm-up (30 min)

**W1.** Verify the three lattice laws for G-Counter by hand, and then as a property test.

**W2.** Construct the concurrent add/remove scenario and show what a 2P-Set, an LWW-Set, and an
OR-Set each produce.

**W3.** Show, with a concrete partition scenario, why a PN-Counter cannot enforce a
non-negative balance.

### Core (3 h)

**C1 — The catalogue.** Complete §3 stages 1–3. Deliverable: six CRDTs with lattice-law and
convergence property tests, the deliberately-broken one with its divergence demonstrated, the
OR-Set semantics verified against 2P-Set and LWW, and the tag-growth measurement.

**C2 — Distributed and compared.** Complete §3 stages 4–5. Deliverable: gossip-based
anti-entropy, convergence times under partition, the availability-versus-divergence comparison
with your Raft store under identical faults, and the delta bandwidth reduction.

**C3 — The invariant boundary.** Complete §3 stage 6. Deliverable: the negative balance
demonstrated, the escrow pattern implemented, the coordination frequency measured as a function
of escrow size, and a precise statement of the guarantee escrow provides.

**C4 — The decision.** Complete §3 stage 8 for two real features — one where you conclude CRDT
and one where you conclude consensus. Deliverable: both analyses against §2.7's table, with
the metadata and latency estimates that support each.

### Challenge

**X1.** Complete §3 stage 7: implement RGA, verify convergence under concurrent editing,
measure the metadata, and compare with Yjs or Automerge. Then read Kleppmann & Beresford's
"A Conflict-Free Replicated JSON Datatype" and write 800 words on what a JSON CRDT must handle
that a text CRDT does not (concurrent type changes, map/list nesting, and the "what does it
mean to concurrently set a key to an object and to a number" problem).

**X2.** Read Kleppmann et al., "Local-First Software" (Onward! 2019). Build a small local-first
application: a shared list or note-taking tool, working fully offline, syncing when connected,
with a CRDT backing store. Then handle the hard part — **schema evolution** — by adding a field
in a new version and verifying that old and new clients still merge correctly. Report what
broke and what you had to constrain. This is where local-first is genuinely difficult and it is
the part the papers gloss over.

## 6. Self-check

1. Define strong eventual consistency and the three merge properties that deliver it.
2. Why do those three properties mean the network needs no ordering or exactly-once delivery?
3. Distinguish state-based from operation-based, and say which is the right default and why.
4. Give the concurrent add/remove semantics of 2P-Set, LWW-Set, and OR-Set.
5. Why is deletion hard, and what does tombstone garbage collection require?
6. Give four situations where CRDTs fit and four where they do not.
7. Explain the escrow pattern and what guarantee it provides.
8. Give the CRDT-versus-consensus table and the third option that is usually right.

## 7. Primary sources

- **Shapiro, Preguiça, Baquero & Zawirski, "Conflict-Free Replicated Data Types" (SSS 2011)**
  and the accompanying tech report with the full catalogue.
- Kleppmann & Beresford, "A Conflict-Free Replicated JSON Datatype" (TPDS 2017).
- Kleppmann, Wiggins, van Hardenberg & McGranaghan, "Local-First Software" (Onward! 2019).
- Almeida, Shoker & Baquero, "Delta State Replicated Data Types" (2018).
- Hellerstein & Alvaro, "Keeping CALM: When Distributed Consistency is Easy" (CACM 2020).
- Bailis et al., "Coordination Avoidance in Database Systems" (VLDB 2014) — the escrow and
  invariant-confluence analysis.
- The Automerge and Yjs documentation and internals write-ups.

---

**Previous:** [L06](L06-partitioning-and-membership.md) · **Next:**
[L08 — Transactions, Sagas, and Idempotence](L08-transactions-and-idempotence.md)
