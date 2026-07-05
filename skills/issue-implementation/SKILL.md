---
name: issue-implementation
description: Use when asked to grab, take, or work on a tracked implementation issue.
phases:
  - intake
  - grill:     { gate: human, skip_when: criteria_clear }
  - scout
  - plan:      { skill: writing-plans }
  - mode:      { gate: human, options: [hitl, afk] }
  - implement: { skill: subagent-driven-development }
  - pr
  - review:    { gate: human }
---

# Issue Implementation

When the human asks to grab, take, or work on a tracked implementation issue, the primary assistant is an orchestrator, not the direct implementer. The orchestrator coordinates subagents through the phases below and never does the implementation work itself. Work through the phases in order, top to bottom.

# Intake

Establish exactly what the issue requires before anything else happens.

- MUST fetch and read the issue.
- MUST check the acceptance criteria.
- If the issue is part of a PRD, MUST fetch and check the acceptance criteria of unresolved future blocked issues, so the current slice does not paint them into a corner.

# Grill

If the acceptance criteria do not make the edge cases and approach clear, MUST run a grill session with the human to confirm edge cases and approach before proceeding.

If the acceptance criteria are already unambiguous, propose skipping this phase (condition: criteria_clear) — the human confirms the skip at the gate. Do not skip silently; the human decides.

# Scout

MUST dispatch a pre-implementation scout subagent (read-only) to inspect relevant code, existing patterns, risks, and likely validation BEFORE implementation starts. The scout's findings feed the plan.

Dispatch discipline for this and every subagent in this process:

- ALWAYS use async subagents.
- Assume async subagents are non-interactive while they are actively working; do not rely on follow-up messages reaching a running worker.
- Put all known scope, review feedback, stop rules, and validation requirements in the initial subagent prompt.
- If requirements change while a worker is running, prefer waiting for completion and then launching a focused follow-up worker. Use interrupt/resume only when the running work must stop or the change would make the current work harmful.
- Escalation is worker-initiated: workers escalate to the orchestrator; the orchestrator does not steer a worker mid-run except as the previous rule allows.

# Plan

MUST create an implementation plan in a temporary folder, following the writing-plans skill (mentioned by name so any reader can find it — it carries the how: task decomposition, file structure, testing strategy).

MUST create/use an isolated workspace (worktree) by default. Do not ask before using one unless the human explicitly asks not to. Use the VCS the project already uses — never assume one. If a VCS-specific skill is available (e.g. a jujutsu skill for jj repositories), read it before running VCS commands.

# Mode

Before starting the task loop, MUST determine the execution mode:

- If the skill invocation includes a mode argument, use it. Supported forms are case-insensitive `HITL` or `AFK`, for example `/issue-implementation 123 AFK` or `/skill:issue-implementation 123 HITL`.
- If no mode argument was provided, ask the human to choose one mode.

Modes:

- **HITL mode**: human-in-the-loop. Keep per-task human feedback gates during implementation.
- **AFK mode**: autonomous task execution. Ignore human gates during task implementation and review/fix loops until every task is complete and all agent review issues are resolved. The final PR-readiness gate still applies: the human MUST approve moving the draft pull request to ready for review.

Do not infer the mode silently from labels, issue content, or vague phrasing. Only an explicit `HITL`/`AFK` invocation argument or explicit human statement selects a mode.

Mode only changes human gates inside the task loop. All subagent delegation, task-sized worker dispatch, agent reviews, commits, pushes, draft pull request behavior, and stop rules still apply.

# Implement

MUST delegate implementation of the agreed slice to worker subagents in the isolated workspace, following the subagent-driven-development skill's cadence.

- MUST delegate implementation in task-sized units, not as one broad worker.
- If the implementation plan has more than one task, the orchestrator MUST run this serial loop:
  1. Dispatch exactly one async worker for Task N.
  2. Wait for Task N implementation completion.
  3. Ensure the Task N implementation is committed on the working branch. Follow the project's commit-message convention if one is evident from instructions or the VCS history (default to Conventional Commits otherwise) and keep each task commit scoped to that task.
  4. If N is 1, create a draft pull request immediately after the Task 1 implementation commit and BEFORE Task 1 review.
  5. Push the Task N implementation commit to the draft pull request branch.
  6. Dispatch at least one async reviewer for Task N.
  7. After the Task N review returns, handle human feedback according to the selected mode:
     - In HITL mode, always ask the human whether they have any comments, concerns, or additional issues before dispatching fixes or proceeding. This is a synchronous gate: do not start a fix worker and do not proceed to Task N+1 until the human responds.
     - In AFK mode, do not stop for human feedback. Continue through reviewer findings, fix workers, re-review, commits, pushes, and subsequent tasks until every task is complete and all agent review issues are resolved.
  8. If the Task N review finds blocking issues, or the human adds issues, fix them with a focused async worker.
  9. Commit and push any Task N fix work to the draft pull request branch.
  10. Re-review until Task N is approved or blocked.
  11. Only after Task N is approved and all Task N implementation and fix commits are pushed, proceed according to the selected mode: in HITL mode, the human feedback gate must also have passed; in AFK mode, proceed directly to Task N+1.
- A single broad implementation worker is forbidden unless:
  - the plan contains exactly one implementation task; or
  - the human explicitly approves collapsing the plan into one worker.
- Never treat a worker's self-review, acceptance contract, or final report as the required Task N review.
- If a pre-existing baseline failure must be fixed, treat it as Task 0 and review it before feature implementation.
- After Task N is approved, MUST ensure all Task N implementation and fix commits have been pushed to the draft pull request branch before dispatching Task N+1.
- The orchestrator MUST NOT edit production code directly in the primary session unless the human explicitly says to skip delegation and implement directly.

# PR

MUST create a draft pull request after Task 1 implementation is committed and pushed, before Task 1 review. The pull request MUST remain draft while task implementation and agent review/fix loops continue.

MUST push commits to the draft pull request branch after each task implementation commit and after each review-fix commit.

After all tasks are complete and all agent review issues are resolved, ask the human to approve moving the draft pull request to ready for review. This gate applies in both HITL and AFK mode. Only after the human approves the result, mark the pull request as ready for review.

Then MUST request post-implementation review from agent subagents covering all of these lenses:

- spec/acceptance-criteria adherence;
- code quality/maintainability/DRY/KISS/YAGNI/campfire principle.

The DRY review MUST look wider than the changed modules: inspect the changed files plus one dependency/reference step out (depth 1) — direct callers, direct imports, sibling modules in the same abstraction, and existing helpers that could reduce duplication without obscuring provider- or domain-specific differences.

# Review

MUST request human review after all tasks are complete and all agent review issues are resolved. MUST mark the draft pull request ready only after human approval. MUST merge only after review approval.

If any required step is skipped or the orchestrator starts implementing directly by mistake, STOP immediately, report the skipped step, and ask the human how to proceed.

Unless explicitly stated in the acceptance criteria: DO NOT enforce backwards compatibility.

A `ready-for-agent` label means the issue is ready for this kickoff-and-implementation flow; it does not by itself select AFK mode or mean completely autonomous fire-and-forget work.
