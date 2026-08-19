# Semantic Review

Review runs continuously and does not fix — `WORKFLOW.md` has the role. This is how it reads code.

**Read and report in this order.** Five lenses, weight descending — correctness and consequence first, preference last. A review that opens with naming buries everything after it.

| # | Lens | Question |
|---|---|---|
| 1 | Reversibility | What cannot be undone if it goes wrong |
| 2 | Model | Can the model represent something false, or fail to represent something true |
| 3 | Contract | Do both sides agree, and does disagreement make a sound |
| 4 | Structure | Which future change does this make silently wrong |
| 5 | Preference | Everything else |

Paired with `docs/QA.md`. QA asks whether the product lies to the user; this asks whether the **model is shaped to hold the truth**.

## Why it is separate

**Filling an empty field looks reasonable while it is being written.** "What goes here — nothing, I suppose" is problem-solving in the moment and only reads wrong later. **The author is the one person who cannot see it.**

And whether the contract still matches the code is nobody's job by default — each implementer looks at what they wrote, QA at behavior, the orchestrator at documents. Review is the space between them.

## Routing

| Found | Goes to |
|---|---|
| One side diverged from the contract | that implementer, directly |
| Two implementations of one contract disagree | both |
| **The contract itself is wrong, or silent** | Orchestrator |

**A model that contradicts the concept usually means the contract is wrong** — no code change fixes it.

## When

- **Right after the contract changes.** That is the day code and contract diverge most
- **Right after a change lands.** Handing back while the author still remembers is cheap
- **When nothing is happening.** Pick a clause nobody has checked yet. Not having work is not a reason to stop

## A pass

One pass is **one clause of the contract, or one landed change — and one report.**

Not one message per finding: a reviewer who sends each thing it notices becomes noise the owning agent learns to skip. Hold them, order them by lens, send once.

A pass ends when the clause has been followed through every place the code claims to uphold it. Close it by writing down **which clause you took and which one is next** — a continuous role that does not record where it stopped re-reads the same clause forever.

---

## 1. Reversibility

**Risk has one definition: irreversible and able to go wrong.** First locate the irreversible surface — effects that leave the system (outbound calls, notifications, transfers), values fixed at creation, data with no recovery path. Everything outside it can be redone, and does not need this lens.

**Check-then-act.** Two callers read the same condition, both find it true, and both write. By the time either write runs, the check describes the past. The shape to look for is `if (condition) { mutate }` where the condition came from shared state; it holds only when the check and the write are one step — a conditional write, a lock spanning both, or a constraint the store itself enforces. This is a TOCTOU race (CWE-367, under CWE-362). **Every sequential test passes, so until one runs concurrently this has not been checked.**

**Retry safety.** Whoever gets no response presses again. Is the idempotency check the **first** check, before validation? Reversed, a request that already succeeded returns a failure, because state is already the new one and a validation rule rejects it — **the system says something that happened did not happen.** Then: is the key **scoped to the caller or tenant** rather than globally unique, and does a replay return the **stored** response, including the stored failure, instead of re-executing?

This is where the two checks meet: **the idempotency lookup is itself a check-then-act.** Look up the key, find nothing, execute, store — two simultaneous retries of the same request both miss the lookup and both execute. The key needs a uniqueness constraint the store enforces, not a lookup the code trusts.

**Partial success.** Two writes that must both land, a write and the aggregate that summarizes it, a removal that must leave someone behind. What is left if it stops midway — one transaction, or an explicit compensating action? **Where no compensation exists, "we will correct it later" is not an answer.**

**Authority.** Every read, write, and delete of an owned resource verifies ownership **at the owner of the data**, not at the caller. A control not rendered is not a path closed; assume the API is called directly. This is the most common class in the field (OWASP A01, of which IDOR is the classic shape).

**Disclosure.** Where existence itself is sensitive, "forbidden" announces that the thing exists. It must answer the same as a typo. Where existence is public and only the operation is restricted, the honest refusal is fine — decide which one this resource is, and check that the code agrees.

## 2. Model

**Start from the types, not the logic.** Ask of each one: can it hold a value that is not true of anything, and can it fail to hold something that is? Logic defends a bad type forever; a better type ends the defense.

**A type whose central field is empty in half its cases is two types.** Count the consumers that branch on a discriminator immediately on receipt; if messages, labels, and layouts all split by it, it was never one type. And **empty fields get filled** — read how a consumer renders the optional field and you find invented words where there is no fact.

**Parse once, at the boundary, into a type that carries the guarantee** — rather than re-validating raw shapes at every use. Find it by grepping one validation — a regex, a range check, a non-empty test — and counting where else the same one appears: every repeat is a call site that can forget it, and a raw string or number carried deep is what makes the repeat necessary.

**Absence must mean one thing.** Grep the fallback operators (`??`, `||`, `getOrElse`, `unwrap_or`) and read everything after them: "unknown" is correct, **anything specific is a lie**. `?? ''` reads as "has no name", `?? 0` as "none", when the truth was "could not load". One value standing for both is the cheapest way for a system to state a falsehood.

**A past event carries the values it had when it happened.** Events are immutable facts; a historical record rendered from a live aggregate changes retroactively every time the aggregate does. Find it where a view of the past joins a table of the present. Attributes that never change are fine; **numbers are not**.

**Properties of the whole are answered by whoever holds the whole.** Uniqueness, permission, capacity, existence, decided inside the page a consumer happened to receive — anyone holding a partial list gets that check silently disabled. Find it by grepping the consumer for aggregation over a fetched collection (`filter`, `some`, `count`, `reduce`, set membership) and asking what the answer would be if the collection were one page shorter.

## 3. Contract

**Silent divergence is the only kind that survives.** A field the contract promises and the producer never sends does not error: absent is falsy, so behavior simply disappears — and usually only in the environment the tests do not run against. A rename one side did not follow is the same shape.

Check three things per clause:

1. **Who enforces it** — the owner of the data, or a consumer (lens 2)
2. **What is visible when it is not enforced** — an error, or silence? Silence is this lens
3. **Do all implementations agree** — a stub and the real one, a cache and its source, two callers. When they differ, tests pass and only users break

## 4. Structure that generates defects

Distinct from preference, and escalated as urgently as lenses 1 and 2. One question decides membership: **which future change does this structure make silently wrong?** If that can be answered it belongs here; if not, it is lens 5.

- **The same rule written by hand in two places.** One copy is missed and a value silently degrades into a wrong default. *Find it* by grepping a literal from the rule — an error code, a limit, a status name — and counting the files it lives in
- **Escape hatches in types.** A non-exhaustive branch with no declared return type, an unchecked cast, `any` at a boundary — the next added case compiles. *Find it* by grepping casts and untyped boundaries, and by adding a case to a union to see whether anything complains
- **Dead code holding tests hostage.** Code no caller uses, kept alive by its tests. *Find it* by listing exported symbols whose only references are test files. Deleting it exposes the bug those tests were not covering
- **Tests that cannot fail for the reason that matters.** *Find it* by grepping the suite for concurrent execution, injected failures, and boundary values. A suite with none of the three does not verify lens 1, whatever its coverage number says

## 5. Preference

Naming, file placement, plain duplication, function length, micro-performance (unless the number feeds an irreversible decision). **Always NON_BLOCKING, always one paragraph at the end of one report.** Never a separate message, never first.

---

## How to write a finding

State **who comes to believe what falsehood, or what becomes unrecoverable**. If neither can be written, it is lens 5.

The lens classifies; it does not decide. **Every finding carries exactly one decision — BLOCKING or NON_BLOCKING** (`WORKFLOW.md`). Lenses 1–3 are BLOCKING. Lens 4 is BLOCKING when you can name the change it makes silently wrong, and NON_BLOCKING when you cannot. Lens 5 is always NON_BLOCKING.

```
BLOCKING · lens 1 — retry re-executes instead of replaying
  What: idempotency lookup runs after validation
  Consequence: a caller who lost the response is told the write never happened
  Where: <file:line>
```

The owning agent fixes. If the contract must change, the Orchestrator writes it down and tells every side.
