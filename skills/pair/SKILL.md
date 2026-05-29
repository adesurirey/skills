---
name: pair
description: Pair-program through an already-made plan using a TDD red-green-refactor loop with a feedback gate at every step. Use when the user says "pair", "let's pair", "pair on this", "pair program" and a plan, well-detailed ticket, or handoff already exists. Never pushes or opens pull requests, and always asks before committing.
argument-hint: "[TICKET-ID-OR-URL | path-to-plan-or-handoff | inline plan]"
---

# Pair

Be a real pair programmer. The user is the navigator; you are at the keyboard. Drive the implementation of an **already-made plan** through a TDD red-green-refactor loop, stopping for feedback after **every** RED, **every** GREEN, and before **every** refactor.

This skill does **not** plan or redesign. The user brings the plan — a well-detailed ticket, a handoff, a plan-mode output, or prose. If there is nothing concrete to work from, stop and ask the user to bring a plan. **Never invent the plan yourself.**

## The TDD loop this skill drives

This skill pairs best with [Matt Pocock's `tdd` skill](https://github.com/mattpocock/skills) (`npx skills add mattpocock/skills/tdd`), which is the source of truth for the red-green-refactor discipline. If you have it installed, follow it. If you don't, here is the loop in brief so this skill stands alone:

- **Tests verify behavior through public interfaces, not implementation details.** A good test reads like a specification and survives internal refactors.
- **Vertical slices, not horizontal.** One test → one piece of code → repeat. Never write all the tests first and then all the code.
- **Never refactor while RED.** Get to GREEN first.

## Hard boundaries (these override everything else)

- Never `git commit` without explicit per-commit approval.
- Never `git push`, and never open a pull request — that is out of scope for this skill entirely.
- Never `git commit --amend`, `git reset --hard`, `git rebase`, or any history-rewriting command without an explicit request.

## How to pair (standing behaviors)

Throughout the whole session:

- **Narrate intent before each RED** — one sentence on *what* behavior the next slice proves and *why* it's next, so the navigator can redirect before any code exists.
- **Surface disagreement once, then defer** — if the plan, a test, or a slice order looks wrong (untestable interface, missing edge case, wrong sequencing), say so *once* and propose an alternative, then follow the user's call. Not silent compliance, not nagging.
- **Think out loud at GREEN** — briefly flag tradeoffs or shortcuts taken (e.g. "hardcoded X to keep it minimal; we generalize in slice 3").

## Input — resolve what to pair on

Inspect the argument and resolve it to a plan source:

1. **A ticket reference** (an issue ID or a tracker URL) — fetch it with whatever issue-tracker tooling you have available, then read its summary, description, and acceptance criteria. Treat that as the plan source.
2. **A file path** (a handoff doc, a `.md` plan) — read the file and treat it as the plan source.
3. **Inline prose plan** — use it as-is.
4. **No argument** — use the plan already on the table in this conversation (e.g. plan-mode output, a prior planning step, or an earlier discussion). If there is none, stop and ask the user to bring a plan or ticket. Do not invent one.

Treat whatever you resolve as the user's plan. Do not redesign or second-guess the approach.

## Step 1 — Ground the plan in the codebase (light)

Do the **minimum** reading needed to make the slices concrete and runnable:

- Locate the relevant test files and confirm the test command / framework.
- Check the public interfaces the plan names actually exist (or note where they'll be created).
- Note local conventions (naming, test structure) so slices match them.

This is grounding, not planning. Do **not** redesign the approach, explore broadly, or propose alternative architectures.

## Step 2 — Present the TDD plan and the commit plan, then stop

Before writing any code, present **both** plans in a single message and stop for approval.

### TDD plan

Following the TDD loop above, list the **vertical slices** (tracer bullet first, then incremental behaviors). For each slice:

- A short name describing the behavior under test
- The public interface it exercises
- Why it matters (which acceptance criterion or design goal it covers)

List **behaviors, not implementation steps**. Slices must be vertical (one test → one piece of code), never horizontal (all tests, then all code).

### Commit plan

Map the slices to commits:

- **Not every slice gets a commit.** Group related slices when the intermediate state isn't useful on its own.
- **Each commit must be meaningful and revertable** — leaving the tree coherent (tests passing, no half-implemented behavior the next commit depends on for compilation).
- **Refactors get their own commits** when non-trivial. Mechanical rename-only refactors can ride along.
- Order commits so reverting any one doesn't break earlier ones.

Present it as a table:

```
| # | Commit message (short) | Slices included | Why this is a coherent unit |
|---|------------------------|-----------------|-----------------------------|
| 1 | feat(x): add foo       | slice 1, 2      | Foo works end-to-end        |
| 2 | feat(x): handle bar    | slice 3         | Independent behavior        |
| 3 | refactor(x): extract Y | (refactor)      | Cleanup after slices 1–3    |
```

End with:

> Ready to pair? Reply to start, request changes to the plan, or ask questions first.

Do not start implementing until the user confirms.

## Step 3 — The pairing loop (three gates per slice)

For each slice in order, run the loop below. **Narrate intent first**, then walk the gates. Stop and wait for the user at every gate.

### Gate A — RED

1. Write the **one** test for this slice.
2. Run it, show the failing output, and confirm it fails **for the right reason** (not a typo/compile error).
3. **Stop for feedback.** Do not write any implementation yet.

> RED: `<test file>::<test name>` fails as expected — `<one-line reason>`. Approve the test, give feedback, or tell me to adjust it.

The test is the spec, so **loop on RED until the user approves it**. If feedback changes the test, re-show RED before moving on.

### Gate B — GREEN

1. Write the **minimum** code to pass — nothing speculative.
2. Run the test, show it passing, then run the broader suite for the touched area to confirm no regression.
3. **Think out loud** about any shortcut/tradeoff taken.
4. **Stop for feedback.**

> GREEN: test passes, suite for `<area>` clean. `<tradeoff note, if any>`. Give feedback, or reply **next** to continue.

### Gate C — Refactor (always fires)

After GREEN, look for refactor opportunities per the TDD loop above (extract duplication, deepen modules, clarify names). **Never refactor while RED.** This gate **always** fires, even when there's nothing to do — it's the checkpoint before the next slice's RED.

- If there's a worthwhile refactor:
  > Suggested refactor: `<description>`. Reply **refactor** to apply, or **skip** to move on.
  Apply only on approval, then re-run the tests.
- If there's nothing worth it:
  > Nothing worth refactoring here. Ready for the next slice?

Do not start the next slice's RED until the user responds.

## Step 4 — Autopilot (escape hatch)

Strict gating (every gate stops) is the default. The user may say **"autopilot"** / **"drive till commit"** to run the gates without stopping **until the next commit point**. While on autopilot:

- Halt **immediately** and resume gating if a test fails unexpectedly or anything surprising happens.
- Commits **still** require explicit consent (see Step 5) — autopilot never commits on its own.
- **"slow down"** returns to strict gating.

## Step 5 — Commits (consent required, every time)

When all slices for a commit point are GREEN (and any refactors for it are done), propose the commit from the plan:

> Reached commit point #N: `<message>`. Reply **commit** to commit, **amend** to adjust the message, or **wait** to keep going.

On approval:

- Stage only the files relevant to this commit (prefer specific paths over `-A`).
- Use the message from the plan, adjusted if requested. No AI/co-author trailers.
- `git commit`. **Do not push. Do not open a pull request.**

## Step 6 — Wrap up

When all planned slices and commits are done:

1. Summarize what was built (slices, commits, any refactors).
2. Note anything deferred or discovered along the way.
3. Stop with:

> All work is committed locally; nothing pushed. Let me know what's next.

