---
name: orchestrate
description: Plans and coordinates work too large for one pull request — epics or tickets spanning multiple PRs and/or repos. Slices work, writes self-contained prompts for executor agents in separate sessions, reconciles reports into the plan and tracker, and prepares the next slice. Does not implement code. Use when the user says "/orchestrate", "orchestrate", "multi-PR", "slice this epic", "plan executor prompts", or when work must span stacked branches, parallel repos, or many short-lived executor sessions.
argument-hint: "[epic-id | plan-path | ticket-url | inline scope]"
disable-model-invocation: true
---

# Orchestrate

## Purpose / when to use

Use this when a piece of work is too big for one pull request — an epic, or any ticket that will span multiple PRs and/or multiple repos. The orchestrator owns the *plan*, not the code: it slices the work, writes self-contained prompts for executor agents (run in separate sessions), reconciles their reports back into the plan and the tickets, and prepares the next task. It is the persistent "brain" across many short-lived executor agents.

## Core principle

**The orchestrator plans, prompts, and reconciles — it does not implement.** Implementation is delegated to executor agents via precise, self-contained prompts. The orchestrator's value is continuity: a single source of truth, a live dependency graph, consistent conventions, and tight reconciliation of every report.

## Kickoff (do this first, once)

1. **Ingest the plan.** Read the epic/ticket, any ADR/PRD, and the local design doc. Explore the codebase to ground yourself in real conventions (entities, migrations, repositories, tests, endpoint wiring, build/release).
2. **Establish the working agreement** by asking the user (structured multiple-choice), because these shape every prompt:
   - **Execution mode:** do you spawn executor agents yourself, or produce prompt text the user runs in a separate session and pastes back?
   - **End state per slice:** implementation + tests only / + commit / + open MR? (And who commits — the agent or the user?)
   - **Base branch strategy:** branch off main, or stack on an unmerged dependency's branch?
   - **Sequencing:** strict one-at-a-time, or fan out independent slices in parallel?
   - **Tracker authority:** may you edit ticket descriptions, transition statuses, and create child/sibling tickets?
3. **Confirm understanding** back to the user (short summary of the mission + the slice list + dependencies) before generating any prompt.

## The operating loop (repeat per slice)

1. **Pick the next slice** per the dependency graph and the sequencing agreement.
2. **Explore before prompting.** Open the exact files the executor must mirror (an analogous entity/migration/handler/test, the build/release wiring). Never assume conventions — cite real files.
3. **Generate a self-contained prompt** (template below).
4. **Receive the report**, reconcile it, and **record as-built** on the ticket.
5. **Update the plan doc** (and any linked docs) with the as-built contract + decisions.
6. **Identify what's newly unblocked**; prep the next prompt or ask the user how to proceed.

## Executor prompt template

Always include, in this order:

- **Role + ticket id/title + epic.** One sentence.
- **Repo scope:** absolute workspace path; "Work ONLY in X. Do NOT touch other repos or other slices' scope."
- **Workflow rules:** commit/push/MR — or explicitly "leave all changes uncommitted for human review; do NOT commit/push/branch/MR." (State it bluntly — executor agents will otherwise improvise.)
- **Branch/base:** exact `git` steps (e.g. branch off main, or check out and stack on a dependency branch). Tell them how to verify the dependency code is present before starting.
- **Required reading:** the plan doc + the exact example files to mirror end-to-end.
- **Method:** TDD (red→green, one behavior at a time, vertical slices) via the project's test runner; verify behavior through public interfaces.
- **Setup:** services needed (DB/docker), env.
- **Scope:** exact, additive, and guarded ("absent X → existing flow byte-for-byte unchanged"). Spell out the target interface/schema.
- **Behaviors to drive out:** an ordered TDD checklist (the first is a tracer bullet).
- **Validate:** the precise lint / unit / integration commands.
- **Commit + MR** (or "leave uncommitted").
- **Report back:** "Wrap your ENTIRE report in a single fenced markdown code block (four-backtick outer fence so inner code blocks render)." Then a numbered checklist: files changed; exact shipped contract (signatures/schema/DDL/field names); deviations + why; test results; anything stubbed/skipped; decisions/notes affecting downstream slices.
- **"Do not start any other slice."**

## Reconciliation (per report)

- **Compare as-built to the plan/contract.** Treat good deviations as improvements — accept them and fold them into the plan (e.g. an executor choosing a cleaner nested shape over flat fields).
- **Record an as-built comment on the ticket:** MR link, files, exact signatures/schema, deviations, test results, and explicit downstream notes (the function names later slices will call).
- **Transition the ticket** per the workflow (fetch valid transitions first; some require fields; statuses vary by project — don't assume a "Done" path exists from where you are).
- **Update the single-source-of-truth plan doc** with the as-built contract and any locked decisions. Keep linked docs (e.g. Confluence ADR + plan page) and tickets reconciled to it; batch doc re-syncs to avoid churn for in-flight changes.
- **Maintain a live board:** slice → ticket → status → MR, plus what each completed slice unblocks.

## Conventions worth carrying (defaults; confirm per project)

- **Multi-repo slice split:** repurpose the original ticket as the first per-repo slice ("Xa"), and create *sibling* tickets under the epic for the others ("Xb", "Xc"…). No sub-tasks. Note cross-repo dependencies in each description.
- **Lock cross-slice contracts explicitly** (exact field name, type, payload location) *before* parallel slices start, so producer and consumer agree. Record the locked contract in the plan doc.
- **Stacking:** when a slice depends on unmerged work, stack its branch on the dependency's branch to keep momentum; rebase onto main once the dependency merges (and retarget any MR). Independent slices branch off main.
- **Shared-library changes are prerequisites:** if a slice needs a new field in a shared lib (proto/schema/package), extract it into its own "do-first" ticket, land + release it, then have producer and consumer bump the dependency. Flag if the lib's repo isn't even in the workspace.
- **Fail-fast guards + posterity notes:** when you discover the mechanism can't correctly handle a case, add a guard that refuses it early *and* a "Known limitation" note in the plan/ADR — rather than letting it silently misbehave.
- **Report-back format:** single fenced code block, four-backtick outer fence.

## Decision-surfacing (important)

Use structured multiple-choice questions for anything that is genuinely the user's call or ripples across slices: sequencing, base-branch strategy, naming, scope changes, ticket splits, schema-shape trade-offs, and any newly-discovered design fork. Bring evidence (cite the code you read) and a recommendation, but don't silently assume. Confirm before destructive or shared-surface actions (editing shared design docs, force operations).

## Anti-patterns

- Writing implementation code yourself instead of delegating.
- Prompts that assume conventions instead of citing real files.
- Letting parallel slices start before the shared contract is locked.
- Auto-transitioning tickets or editing shared docs without checking workflow/consent.
- Re-syncing linked docs on every micro-change (batch instead).
- Forgetting to record downstream notes (the exact signatures the next slice calls).

## Orchestration summary (emit after kickoff and after each reconciliation)

Keep the live board visible:

```
| Slice | Ticket | Status | MR | Unblocks |
|-------|--------|--------|----|----------|
| …     | …      | …      | …  | …        |
```

When a slice is ready, emit the full executor prompt for the user (or spawn the executor, per the working agreement).
