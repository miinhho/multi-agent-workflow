# How QA Looks

QA verifies behavior independently — see `WORKFLOW.md` for the role. This document is how it looks, every round, regardless of what was requested.

## Order of a round

1. Use the product like a person
2. Run the six checks below, in order
3. **Then** open the request and verify its items
4. Close the round: record what you did not cover

**Do not read the request before step 3, and do not close the round without doing step 3.** The report follows the same order — **what nobody asked about goes on top.**

A request is made of what the orchestrator already suspects. Answering only that can **confirm or deny a suspicion, but never surprise anyone.** And what gets missed is usually **what is absent** — the missing hover, the missing skeleton, the missing failure state, the missing hierarchy. **A checklist can only point at what exists.**

---

## Every round — independent of the request

### 1. Find what is missing

Every view has four states. **With only three, the user meets the fourth first.**

| State | Ask |
|---|---|
| Loading | **Is the placeholder the same shape as the content** (measured, see below) |
| Failed | Does it say it failed. Is there a way to retry |
| Empty | **Is it distinguishable from failure** |
| Overflowing | What happens at a hundred items |

If the last two look alike, that alone is a defect: **"could not load" read as "there is none" is the cheapest way for a product to lie.**

**Do not judge a placeholder present/absent. Measure it by overlay.**

1. Throttle the network and capture the loading state
2. Capture the same area after content arrives
3. Compare the two

Two ways to do step 3, and they answer different questions. A pixel diff of the two captures shows **how much** moved; the browser tells you **which element** moved:

```js
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    if (e.hadRecentInput) continue;           // user-initiated shifts do not count
    console.log(e.value, e.sources?.map((s) => s.node));
  }
}).observe({ type: 'layout-shift', buffered: true });
```

**If elements move, it failed.** A placeholder's only job is to reserve the position of what is coming; if things moved, it did not do that job — it was a spinner drawn as gray rectangles.

Look especially at **whether the largest element has a reserved place.** Evenly spaced gray bars fit any view and therefore fit none. **When what the eye lands on first has no place reserved, the whole layout rearranges the moment it arrives.**

### 2. Break it on purpose

The happy path has already been seen a hundred times by whoever built it. **What QA sees first is the broken path.**

- Throttle the network. What does it say while waiting
- Disconnect and press — especially **the irreversible actions**
- Force reads to fail
- Enter as an account with nothing in it, and as one created a moment ago
- Every input at its maximum length
- 0, 1, and the maximum
- Hammer back — both system back and in-app back
- Press the same control twice quickly — especially where it writes

### 3. Are the four interaction states different

For everything pressable.

| | How does it read |
|---|---|
| Rest | Does it look pressable |
| Hover | Does it change under the cursor |
| Active | Does it change while held |
| Focus | Can the keyboard tell where it is |

Also measure the target: **24×24 CSS pixels is the floor**, or enough spacing that neighbouring targets do not collide.

**Four states in one color is the same as having none.** Even where pointer input is secondary, absent and present-but-weak are different.

### 4. Count the hierarchy

This is countable, not a feeling.

- How many text styles in one view. **More than five is noise, not hierarchy** — nobody distinguishes five levels
- Is the largest text the most important thing
- With several controls, **does size follow importance**? If the irreversible one is largest, that is a trap, not a hierarchy
- Can the purpose of the view be said in one sentence? If not, it is two views
- Look for three seconds and close your eyes. **What remains** — is it the most important thing
- **What sits in the fixed, easiest-to-reach area?** If it is not this view's main action it does not belong there, and **an irreversible action living there is a defect**
- If every control is full-width, size says nothing. Count them

### 5. Did the fix create something else

Re-examine how last round's findings were fixed. **A narrow fix breaks its neighbor.**

Do not close on "fixed" — look at **what the fix cost**. A layout complaint answered by pinning the control to the easiest-to-reach position trades a small defect for a larger one.

### 6. Record what you did not cover

One line each at the end of the round — **what was not opened, which states were not produced, which roles were not used.** The next round starts there.

Without it, the same places get looked at every time.

---

## Rotate roles

The same view fails differently depending on who is looking. Pick **at least two** each round. Each role has a **question that must be answered there**. If the answer is not present, that is a defect.

| Role | Situation | Question that must be answered |
|---|---|---|
| First-time user | one item, no history | What is this. Where did it come from. Can I use it |
| About to make a mistake | ambiguous or near-identical targets | Is this the one I mean |
| Judging something unfamiliar | first encounter with it | Can I trust this. What could still change |
| Excluded or revoked | access removed | Why can't I. How do I get back |
| Owner or operator | privileged capability | What can I do now. What happens if I change it |
| Creating something irreversible | first time through | Which option do I pick. Can I undo it |

When looking as a role, **forget the view's name.** Not "does the settings page work" but "does this person get their answer here".

---

## How to write it

### Observe, do not adjudicate

Record it even when you cannot judge it. **"Something is off and I can't tell what" is a valid observation** — that is how users report. Adjudication belongs to the orchestrator.

### Use the user's words

"Text is unreadable" comes before "contrast below 3:1". The number goes on the next line. Users do not complain in measurements.

### Severity by what the user is made to believe

Severity classifies; it does not decide. **Every finding carries exactly one decision — BLOCKING or NON_BLOCKING** (`WORKFLOW.md`), and severity 1–2 is BLOCKING, 4–5 is NON_BLOCKING, 3 is BLOCKING when the view exists to answer that question.

Looking bad and lying are different.

1. **The user believes something untrue** — a success read as a failure, one target read as another, a load failure read as emptiness. Worst where there is no undo
2. **Something irreversible looks reversible** (or the reverse)
3. **They came for an answer and there is none** — it works, the purpose is not served
4. **Hard to read, hard to reach, unresponsive**
5. Looks bad

### Format

```
What I did        — reproduction path: account, location, action
What I saw        — facts only
What I expected
Why it matters    — which role comes to believe what falsehood
Severity          — one of the five above
Decision          — BLOCKING or NON_BLOCKING
Environment-bound — if it splits by build, device, or browser, say which
```

---

## What the words say

The six checks above look at what the product does. These four look at what it **says** — the same defect class, found by reading rather than pressing. Reversibility and access are checked by pressing in §2 and §4; here, only the wording.

- **Words that promise a reversal the system does not implement.** "Cancel", "Undo", "Restore", "Pending" on a control that does none of those. Dismissing a form and undoing an event must not use the same word
- **The message where existence is concealed.** It must be the same message a typo produces. Whether the server actually conceals it is `docs/REVIEW.md` lens 1; whether the two messages differ is visible here
- **What the producer said versus what the consumer shows.** The API answered "not allowed" and the UI offers the action — or the reverse
- **Register.** User-facing text in the product's voice. Flag document voice, debug strings, raw identifiers, and untranslated keys

---

## What QA does not do

- **Does not fix code.** Things to fix are reported to the owning agent
- **Does not decide the contract.** If the contract seems wrong, write that down and hand it off
- **Does not decide in advance that it cannot be checked.** Whether a real device or a specific environment is needed is decided by trying — perceived speed, mis-taps, contrast, targets, keyboard order are often settled without one. Request it, and treat "cannot" as the only reason to wait
- **Does not stop to wait.** Record it and move on

**Record which environment every observation came from** — stub or real, which build. They drift, some behavior exists in only one, and an observation without that note cannot be reproduced by the next person.
