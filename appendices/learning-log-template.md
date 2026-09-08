# Appendix — Learning Log Template

A log is not a diary of what you did. It is a record of **what you understood, what you got wrong,
and what you decided** — and its value is entirely in being re-read.

Three reasons to keep one, in ascending order of importance:

1. **Retrieval.** Writing what you learned, from memory, is itself the most effective study
   technique there is. The log entry is the retrieval practice.
2. **Error patterns.** Nobody notices their own recurring mistakes without a record. Over a term you
   will find you make the same two or three errors repeatedly, and that is worth more than any
   individual lesson.
3. **The capstone.** Twelve weekly entries are the raw material of the defence document, and they
   capture reasoning you will otherwise have forgotten by week 10.

Keep it in your study repository, in version control, one file per month. Ten minutes per session,
twenty at the end of a lesson.

---

## Per-session entry

```markdown
## 2026-09-08 · PY-501 L03 (Descriptors) · 2.0 h

**What I did**
Construction stages 1–4. Read §2.1–2.4.

**What I understood that I didn't before**
Properties are descriptors. `property` isn't special syntax — it's a class implementing
`__get__`/`__set__`, and everything I thought was magic about attribute access is one lookup rule
plus this protocol. That also explains why `self.x = 1` in `__init__` can silently do nothing when
`x` is a data descriptor.

**What I got wrong**
Assumed non-data descriptors take precedence over instance `__dict__`. They don't — data
descriptors do, and non-data descriptors don't. I had it exactly backwards, which is why my
stage-3 test failed for twenty minutes.

**Open question**
Why do `__slots__` use descriptors rather than a dict-free fast path in the C layer? Come back
after L09 (memory).

**Time check**
2.0 h for four stages. On track for the lesson's 4 h estimate.
```

Five headings. If an entry takes more than ten minutes you are writing too much.

---

## The headings, and why each

**What I did.** One line. Purely so you can reconstruct the sequence later.

**What I understood that I didn't before.** The core of the entry. Write it from memory, without the
lesson open — that constraint is the point. If you cannot state it without looking, you have not
learned it yet, and discovering that now is exactly what the log is for.

**What I got wrong.** The most valuable heading, and the one people quietly drop. Record the actual
misconception, not "I made a mistake in stage 3". Over a term these accumulate into a pattern, and
the pattern is usually more instructive than any single entry.

**Open question.** What you did not resolve, with a note of where it might be answered. Re-read
these at term end; several will have been answered and you will not have noticed.

**Time check.** Actual hours against the lesson's estimate. Two purposes: it keeps your plan honest,
and a consistent large overrun usually means a prerequisite is weak rather than that you are slow.

---

## Weekly entry

At the end of each week, ten minutes:

```markdown
## Week of 2026-09-07

**Hours**: 11.5 (target 12)
**Lessons completed**: PY-501 L03, L04
**Core exercises done**: L03 yes, L04 partial (4, 5 done; 6, 7 outstanding)

**The one thing worth remembering from this week**
The MRO is not "search left to right" — it's C3 linearisation, and it can *fail*. A hierarchy can
be unbuildable. That reframes multiple inheritance from "confusing" to "constrained", which is a
much more useful way to hold it.

**What I'm avoiding**
Exercise 7 on L04 (implementing C3 by hand). I keep not starting it because it looks tedious. It
is probably the exercise I most need.

**Used in real work this week**
Explained a `super()` bug in a colleague's PR by drawing the MRO. First time the programme has
paid off directly.

**Next week**
L05, L06, and the L04 exercise 7 I've been avoiding.
```

The "what I'm avoiding" line is deliberately uncomfortable and it is the most useful line in the
template. Self-directed study fails by attrition at the hard exercises, not by dramatic
abandonment, and naming the avoidance is usually enough to end it.

---

## Term-end entry

Longer — an hour, once a term. Answer the six questions from the progress tracker:

1. What can I now do that I could not at the start of this term? *Be concrete: name a thing you
   built or a problem you solved.*
2. Which lesson's exercises did I skip, and why? Is that a real gap?
3. What did I get wrong on the exam, and what does the pattern say?
4. Which idea from this term have I already used in real work?
5. What hours per week did I actually sustain, and is the plan still realistic?
6. What did I find wrong in the material? → `appendices/errata.md`

Then read your "what I got wrong" entries for the whole term in one sitting and write one paragraph
on the pattern. This is the single highest-value hour in the term. Most people find they have one or
two systematic blind spots — off-by-one reasoning, confusing safety with liveness, assuming
independence where things are correlated — and naming them is what fixes them.

---

## Capstone log

During the capstone, weekly, with two extra headings:

```markdown
## Capstone week 6

**Progress**: [what moved]

**Decisions made this week**
Switched from levelled to size-tiered compaction as the default for the write-heavy arm.
Reason: the write amplification difference was 4× larger than the read amplification cost at this
workload's read ratio. Alternative considered: hybrid, rejected because it doubles the
configuration space I have to evaluate and I don't have the weeks.

**Surprises**
The measurement harness's own overhead is 8% of throughput at high load. Have to subtract it or
report it. Chose to report it, since subtracting requires assuming it's uniform and I haven't
checked that.

**Risks**
Week 9 evaluation depends on getting 30 GB of realistic data. Still haven't confirmed I can. →
Confirm by Friday or switch to the synthetic generator and state it as a threat to validity.

**Next week**: [plan]
```

**Decisions made this week** is the heading that writes your defence document's design chapter, and
it must include the alternative you rejected and why. Six months later you will not remember, and
"why did you do it that way?" is the first question in the oral.

---

## Common failures of log-keeping

- **Writing what you did instead of what you understood.** A list of activities teaches nothing on
  re-reading.
- **Omitting the errors.** They are the most valuable content and the least comfortable to write.
- **Writing with the lesson open.** Then it is transcription, not retrieval, and the study benefit
  is lost.
- **Never re-reading.** A log that is only written is a diary. Read the week's entries at week end
  and the term's at term end; that is where the value is realised.
- **Too long.** Ten minutes. An hour-long entry will not be written next time.
- **Stopping when it gets hard.** The weeks you least want to write an entry are the weeks whose
  entries are most informative later.

---

## A minimal alternative

If the template feels heavy, the irreducible version is three lines per session:

```
2026-09-08 · PY-501 L03 · 2.0h
Learned: properties are just descriptors; data descriptors beat instance __dict__.
Wrong: had the data/non-data precedence backwards.
```

That is enough. Consistency beats completeness, and a three-line log kept for two years is worth
more than a detailed one abandoned in month three.
