# RubricAdjudicator

A reusable adjudication primitive for GenLayer Intelligent Contracts.

It turns a **rubric** — an ordered list of yes/no criteria, some of them
blocking — into an on-chain adjudication process with a deterministic verdict,
an integrity-checked rationale, and a bounded appeal loop.

---

## The problem this solves

A lot of real decisions are neither pure computation nor pure opinion. They are
**structured judgement**: a fixed set of questions, applied to a messy piece of
evidence, producing an outcome that follows a stated rule.

I work at the front desk of a rehabilitation clinic in Korea. Every day I watch
insurance claims get decided this way. There is a rubric — was the treatment
medically indicated, is the documentation complete, is the item covered. A human
reads the chart, answers each question, and the outcome follows from the answers
by policy, not by mood. When the claimant disagrees, they don't get a different
mood; they get a stricter re-review.

The same shape shows up in freelance deliverable QA, grant screening, bounty
review, and content moderation. Solidity cannot express it because the questions
require reading. A plain LLM call cannot express it either, because then the
model owns the outcome and nobody can audit the rule.

## The design

**The model never decides.** It answers one boolean per criterion. That is all
it is permitted to emit. The verdict comes from `_derive_verdict`, twelve lines
of deterministic Python:

| Condition | Verdict |
| --- | --- |
| any criterion marked `mandatory` failed | `REJECTED` |
| every criterion passed | `APPROVED` |
| otherwise | `PARTIAL` |

Swap the underlying model and the decision rule is unchanged. That is the
difference between an adjudication primitive and an "AI decides X" demo.

### Two equivalence principles, chosen by output type

This is the part worth reviewing.

**1. Criterion scoring — `gl.eq_principle.strict_eq`**

The scoring block returns a fixed-length bit vector positionally aligned to the
rubric, e.g. `"1011"`. It is discrete and order-fixed, so exact agreement is
both achievable and meaningful. If validators disagree on a single criterion,
consensus fails outright rather than quietly averaging the disagreement away.
The part that carries the decision gets the strictest principle available.

The scoring block also **fails closed**: `raw.get(key) is True` is an identity
check, not a truthiness check. A missing key, a `null`, or the string `"yes"`
all score as `0`. Absence of evidence is never treated as satisfaction.

**2. Rationale — `gl.eq_principle.prompt_non_comparative`**

Prose will never match word-for-word across validators, so strict equality is
the wrong tool. Validators instead check the rationale against integrity
criteria: at most three sentences, states the same verdict it was given, refers
only to listed criteria, contradicts no PASS/FAIL result, and introduces no fact
absent from its input.

The rationale writer is deliberately given a **closed fact set** — the rubric,
the scores, and the verdict — and never sees the raw subject or evidence. It
cannot smuggle unscored details into the explanation, because it does not have
them.

So prose tolerance exists in the system, but it is walled off from the outcome.

### Appeals

Mirroring GenLayer's own escalation idea: an adjudicated case can be appealed by
recording a counter-argument on-chain. Re-adjudication then runs with the
counter-argument in scope and an explicitly stricter instruction. Rounds are
capped at `MAX_ROUNDS = 3`, so the loop terminates.

State machine:

```
OPEN ──adjudicate──> ADJUDICATED ──finalize──> FINAL
                          │  ▲
                       appeal │ adjudicate
                          ▼  │
                       APPEALED
```

### Prompt injection

Untrusted text is length-capped, fenced with explicit `[BEGIN]/[END]` markers,
and the instruction to treat it as inert data is placed **after** the fenced
block — so a claimant cannot close the section early and append their own rules.

---

## Usage

### Deploy

Two constructor arguments:

| Argument | Type | Meaning |
| --- | --- | --- |
| `domain` | `str` | Short label for what is being adjudicated. Used as prompt context. |
| `criteria_json` | `str` | JSON array of `{"key", "question", "mandatory"}`. Max 10. |

```json
[
  {"key": "medically_indicated", "question": "Does the record show the treatment was medically indicated rather than elective maintenance?", "mandatory": true},
  {"key": "documentation_complete", "question": "Is a dated practitioner record present describing the treatment?", "mandatory": true},
  {"key": "within_policy_scope", "question": "Does the treatment fall within the stated policy scope?", "mandatory": false},
  {"key": "amount_consistent", "question": "Is the claimed amount consistent with the described treatment?", "mandatory": false}
]
```

See `examples/` for ready-to-paste rubrics in three different domains — the
point being that none of them require touching the contract code.

### Call sequence

```
open_case(case_id, subject, evidence)   # register
adjudicate(case_id)                     # score + verdict + rationale
get_case(case_id)                       # read full breakdown
appeal(case_id, counter_argument)       # contest -> stricter re-review
adjudicate(case_id)                     # round 2
finalize(case_id)                       # close
```

### Methods

| Method | Kind | Notes |
| --- | --- | --- |
| `open_case` | write | Registers a case in `OPEN`. Does not adjudicate. |
| `adjudicate` | write | Valid from `OPEN` or `APPEALED`. Increments round. |
| `appeal` | write | Valid from `ADJUDICATED` only. Requires a counter-argument. |
| `finalize` | write | Valid from `ADJUDICATED` only. Terminal. |
| `get_rubric` | view | Domain plus criteria list. |
| `get_case` | view | Full state including per-criterion `breakdown`. |
| `get_verdict` | view | Verdict string only. |
| `list_cases` | view | All case ids with status and verdict. |

---

## Storage notes

- `list` / `dict` are replaced by `DynArray` / `TreeMap` as GenVM requires.
- Custom types are decorated with `@allow_storage`; all fields are scalars, so
  no `gl.storage.inmem_allocate` is needed.
- `rounds` uses `bigint` for portability across SDK versions rather than a
  sized integer. A counter capped at 3 would fit in `u32`.
- Storage views are copied into plain memory before entering non-deterministic
  blocks, since nondet blocks cannot read storage directly.

## Testing

`TESTPLAN.md` contains a reproducible walkthrough with exact inputs and
expected outputs, including the mandatory-failure path and the appeal path.
The deterministic decision rule is separately verified against all six
meaningful flag patterns.

## Korean documentation

`docs/README.ko.md` — full Korean translation. GenLayer material in Korean is
close to nonexistent, and the Korean-speaking builder community is currently
underserved.

## License

MIT
