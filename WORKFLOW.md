# Engineering Workflow

All engineering tasks follow this workflow.

## Roles

* **Orchestrator** — plans, assesses risk, routes work, and owns completion.
* **Backend** — implements backend changes.
* **Frontend** — implements frontend changes.
* **QA** — independently verifies behavior using browser, accessibility, usability, performance, and regression testing as relevant. Follows `docs/QA.md`.
* **Review** — continuously checks **risk first, then logic**: what could go irreversibly wrong, whether the model matches the concept, and whether the code matches the contract. Code quality is in scope but reported last, except where a structure generates defects. Follows `docs/REVIEW.md`. Reports; does not fix.

**Self-review does not count as review.** A decision that fills an empty field looks reasonable while it is being written and only reads wrong later — the author is the one person who cannot see it. Model-level defects survive author checks by construction.

**Review does not write code.** Two agents writing in one working tree overwrite each other, and a reviewer who fixes loses the independence that is the entire point. Review sends findings **straight to the owning agent**, not through the Orchestrator; only contract changes go to the Orchestrator.

**Agents sharing one workspace share more than files.** Each agent writes only its own area, and never issues a broadly matched command — killing processes by name pattern takes down what another agent started, in a mode nobody can reconstruct. Know the identifier of what you started and stop only that.

**A rule the harness can enforce is not a paragraph.** Review can be given no editing tools, the author check can run as a hook, workspace-wide commands can be denied before they run. `HARNESS.md` maps the rules above to the mechanisms that hold them.

---

## Workflow

```text id="jyxtur"
Request
  ↓
Orchestrator
  ├─ Define acceptance criteria
  ├─ Identify affected areas
  ├─ Assess risk
  └─ Define verification
  ↓
Implementation
  ├─ Backend if affected
  └─ Frontend if affected
  ↓
Author check
  ↓
Cross-review if FE/BE boundary changed
  ↓
Relevant QA if needed
  ↓
HIGH risk → Review deep pass
  ↓
Final gate (Orchestrator)

Review ─── runs continuously, outside this flow ───→ findings to owning agent
```

Use the minimum workflow necessary for the identified risk.

Do not invoke agents or verification steps that do not add meaningful confidence.

**Review is not a gate except at HIGH risk.** It runs continuously and its findings arrive asynchronously as new work. Making it a per-task gate turns the one agent that never stops looking into the one thing everybody waits on.

**Author check is not review.** It is the author reading their own diff before handing off — useful, and not independent.

---

## Risk

### LOW

Isolated, well-understood changes with limited impact and easy recovery.

Examples:

* styling/copy
* isolated UI changes
* straightforward bug fixes
* behavior-preserving refactors

Typical flow:

```text id="w8qqgi"
Implement → Author check → Relevant checks → Done
```

### MEDIUM

Changes with meaningful behavioral, contract, or regression risk.

Examples:

* API changes
* validation changes
* new user flows
* state-management changes
* schema changes
* cross-component behavior

Typical flow:

```text id="7l8fp8"
Implement → Author check → Cross-review if needed → QA → Done
```

### HIGH

Changes with significant security, data, operational, or user impact.

Examples:

* auth/authz
* payments
* sensitive data
* destructive operations
* risky migrations
* data integrity
* concurrency/idempotency
* critical infrastructure or flows

Typical flow:

```text id="ryv96a"
Implement
→ Author check
→ Cross-review if needed
→ Relevant QA
→ Review deep pass
→ Orchestrator final gate
→ Done
```

Risk determines **verification depth**, not which agents must participate.

---

## Implementation

Implementation agents must:

1. Understand the acceptance criteria and existing behavior.
2. Implement only the assigned scope.
3. Run relevant tests, type checks, and lint checks.
4. Read the resulting diff. This is an author check, not review.
5. Report unresolved concerns.

Passing tests alone does not imply completion. Neither does an author check.

---

## Cross-Review

Cross-review is required when Frontend and Backend share a **changed contract or assumption**.

It is not a general second code review. It is also not a substitute for **Review** — cross-review is bounded to one change; Review runs continuously and owns the question "does the contract still describe what the code does".

**Frontend reviewing Backend:**

* API contracts
* response/error behavior
* auth behavior
* null/optional states
* frontend-visible failure behavior

**Backend reviewing Frontend:**

* API usage
* validation/auth assumptions
* retries and duplicate requests
* state transitions
* concurrency/idempotency assumptions
* unintended backend side effects

If the FE/BE boundary did not meaningfully change, skip cross-review.

---

## QA

QA provides independent behavioral verification when it adds meaningful confidence.

Select only relevant checks:

| Change                         | Verification       |
| ------------------------------ | ------------------ |
| User-visible behavior          | Functional/browser |
| UI structure or interaction    | Accessibility      |
| New/changed user flow          | Usability          |
| Performance-sensitive behavior | Performance        |
| Meaningful regression risk     | Regression         |

QA is not mandatory for every LOW-risk task.

For bug fixes, reproduce the original failure when practical.

---

## HIGH-Risk Review

For HIGH-risk changes, **Review** performs the focused pass and the Orchestrator owns the final gate. The Orchestrator does not repeat the pass — it decides whether the evidence is enough to complete.

Review the risk-bearing parts of the change, especially:

* security/auth boundaries
* data integrity
* migration/destructive behavior
* concurrency/idempotency
* failure and recovery behavior
* unresolved review/QA findings

Do not repeat a general code review.

---

## Reports state what was read

Agents share one working tree and fix concurrently. **The target moves while it is being read.**

Every report about another agent's work opens with one line:

```text id="rd4tsq"
> Read: <sha> · <what was opened>
```

That line stops two failures that look alike and are not.

**Speaking from a stale read.** With the sha, the author answers "that was fixed after it" at once. Without it both sides guess, and withdrawing a block costs a round trip — the reviewer re-reads, the author re-explains, and nothing in the code changed.

**Speaking wider than what was read.** Naming what was opened makes the writer catch themselves: "nobody here has ever exercised that path" cannot sit directly below "opened one controller". To widen a claim, open the wider place. Inferring what a server does from what a screen showed is the same move.

**The Orchestrator opens the evidence before relaying.** When passing along another agent's judgment would change **what a third agent works on next**, read the basis directly. This is not redoing the judgment — only checking its extent, and it is usually one line. An Orchestrator that relays without this is an amplifier: one agent's overreach becomes another agent's work order.

A report missing the line goes back. **A format with no rule for its absence reads as a suggestion.**

---

## Findings and Rework

Findings are either:

* **BLOCKING** — must be resolved before completion.
* **NON_BLOCKING** — may be recorded separately.

**Every finding carries exactly one of these, and nothing else decides.** `docs/REVIEW.md` lenses and `docs/QA.md` severities classify a finding so it can be read and grouped; they never replace the decision, so that findings arriving from different roles land on one list.

Findings come from QA, cross-review, or Review. **Review sends them straight to the owning agent**; the Orchestrator is involved only when the contract itself is wrong.

Blocking findings return to the owning agent:

```text id="ngqt5s"
Finding
→ Fix
→ Author check
→ Re-run affected verification
```

Only repeat verification invalidated by the fix.

If implementation reveals new risk, unexpected scope, or a changed system boundary, stop expanding scope and return to the Orchestrator for replanning.

---

## Completion

The Orchestrator marks the task complete when:

* acceptance criteria are satisfied
* implementation and the author check are complete
* required checks pass
* required cross-review is complete
* required QA is complete
* blocking findings are resolved
* the HIGH-risk Review pass is complete when applicable

Completion requires evidence, not agent confidence.

---

## Decision Rules

```text id="k73sxy"
Backend affected?
→ Backend

Frontend affected?
→ Frontend

FE/BE contract or shared assumption changed?
→ Cross-review

Behavior needs independent verification?
→ Relevant QA

HIGH risk?
→ Review deep pass, then Orchestrator final gate

Model, contract, or irreversible-path concern at any risk level?
→ Review (continuous — no need to route it)
```

> **Risk determines verification depth.
> Affected behavior determines verification type.
> Changed boundaries determine cross-review.**
