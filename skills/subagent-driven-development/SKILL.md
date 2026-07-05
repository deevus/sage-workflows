---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

Execute a plan by dispatching a fresh implementer subagent per task, a task review (spec compliance + code quality) after each task, and a broad whole-branch review at the end.

**Why subagents:** You delegate tasks to specialized agents with isolated context. By precisely crafting their instructions and context, you ensure they stay focused and succeed at their task. They should never inherit your session's context or history — you construct exactly what they need. This also preserves your own context for coordination work.

**Core principle:** Fresh subagent per task + task review (spec + quality) + broad final review = high quality, fast iteration

**Narration:** between tool calls, narrate at most one short line — the ledger and the tool results carry the record.

**Pacing:** Pacing is owned by the enclosing workflow. When run inside Sage's issue-implementation workflow, follow the selected HITL/AFK mode there; per-task commits always apply, while human gates may be deferred in AFK mode until final PR-readiness approval. Independent of pacing, these situations are escalation triggers — raise them with your human partner when they occur: a BLOCKED status you cannot resolve, ambiguity that genuinely prevents progress, or all tasks complete.

## When to Use

Work through these questions in order:

1. **Do you have a written implementation plan?** If not, stop — write one first (use the `writing-plans` skill) or brainstorm the design before executing anything.
2. **Are the tasks mostly independent?** If they are tightly coupled — each task's design depends on discoveries made in the previous one — execute manually instead; per-task subagents would thrash.
3. **Will execution happen in this session?** If yes, use this skill. If the work must happen in a separate parallel session instead, this skill does not apply — coordinate that handoff outside it.

**What this buys you:**
- Fresh subagent per task (no context pollution)
- Review after each task (spec compliance + code quality), broad review at the end
- Your own context stays free for coordination

## The Process

**Setup (once):**

1. Use the VCS the project already uses — never assume one. If a VCS-specific skill is available (e.g. a jujutsu skill for jj repositories), read it before running VCS commands. Ensure you are in an isolated workspace, not working directly on the main branch/bookmark (create one first if needed with the project's VCS — git: `git worktree add`; jj: `jj workspace add`).
2. Read the plan file once. Note its context section and global constraints.
3. Record the branch base — the revision the branch's work sits ON TOP OF (the parent of the branch's first commit, not the working copy itself) — you need it for the final whole-branch diff. After the branch is done, the BRANCH_BASE..tip diff must include every branch commit; a suspiciously small or empty diff means the base was recorded wrong.
4. Initialize the progress ledger file in the working folder the orchestrator supplies (e.g. alongside the plan file): one line per task, none marked complete. Run the Pre-Flight Plan Review (below) before dispatching Task 1.

**Per task:**

1. Record BASE as the revision the task's work will sit on top of — the revision BELOW the task's first commit. After the task, the BASE..tip diff must include every task commit; a suspiciously small or empty diff means BASE was recorded wrong. (git: `git rev-parse HEAD` before dispatch; jj: the working-copy parent, `jj --no-pager log -r @-`, because `jj commit` keeps the working copy's change ID on the completed commit — if the working copy already contains task work, record its parent, not @.)
2. Write the task-brief file: copy that task's full section from the plan — heading and everything under it, nothing else — into its own file (e.g. `<workdir>/task-N-brief.md`).
3. Dispatch an implementer subagent using [implementer-prompt.md](implementer-prompt.md).
4. If the implementer asks questions, answer them completely — with more context if needed — before letting it proceed.
5. The implementer implements, tests, commits with the project's VCS (using the project's commit-message convention if one is evident from instructions or the VCS history; default to Conventional Commits otherwise), self-reviews, and reports one of four statuses. Handle the status per "Handling Implementer Status" below.
6. On DONE: write the review diff file — a unified diff from BASE to the current tip with ~10 lines of context (git: `git diff -U10 BASE..HEAD`; jj: `jj diff --git --context 10 --from BASE --to @`) — redirected to a uniquely named file, and dispatch a task reviewer subagent using [task-reviewer-prompt.md](task-reviewer-prompt.md).
7. If the reviewer reports Critical or Important findings (or a spec ❌), dispatch a fix subagent with the findings, regenerate the diff file, and re-review. Repeat until spec ✅ and quality approved. Record Minor findings in the progress ledger.
8. Mark the task complete in the progress ledger.
9. More tasks remain? Return to step 1 for the next task.

**After all tasks:**

1. Dispatch a final whole-branch reviewer on the most capable available model. Hand it one diff file covering the whole branch (a unified diff from BRANCH_BASE to the tip with ~10 lines of context — git: `git diff -U10 BRANCH_BASE..HEAD`; jj: `jj diff --git --context 10 --from BRANCH_BASE --to @` — plus the commit list: git: `git log --oneline BRANCH_BASE..HEAD`; jj: `jj --no-pager log -r 'BRANCH_BASE..@'`), the plan's requirements and global constraints, and the ledger's accumulated Minor-findings list so it can triage which must be fixed before merge. Ask for the same Critical/Important/Minor severity categories and an overall verdict.
2. If the final review returns findings, dispatch ONE fix subagent with the complete findings list — not one fixer per finding — then re-review.
3. Report completion to the enclosing workflow (or your human partner): tasks done, commits created, review outcomes, open Minor items.

## Dispatching Subagents

Dispatch every worker with the `subagent` tool. Its fields:

- `task` — the full prompt (composed from the templates in this skill)
- `label` — a short name for logs, e.g. `task-3-implementer`, `task-3-reviewer`
- `system_prompt` — optional; use only when the role needs standing rules the task text shouldn't carry
- `child.tools` — the child's tool policy
- `child.model` / `child.provider` — model overrides

The `subagent` tool is blocking and defaults to a read-only tool policy; implementer and reviewer children need an explicit `child.tools` policy that permits editing. Since dispatch is blocking, subagents run one at a time — which is what you want; parallel implementers conflict.

**Async vs. blocking dispatch:** "async dispatch" (as issue-implementation mandates) means fire-with-a-complete-prompt and non-interactive execution — the worker cannot converse with you mid-run. When the available `subagent` tool is blocking, the same discipline applies: compose a complete initial prompt and do no mid-run steering. Questions from a worker arrive only as a NEEDS_CONTEXT return; answer them by re-dispatching with the answer added to the prompt.

## Pre-Flight Plan Review

Before dispatching Task 1, scan the plan once for conflicts:

- tasks that contradict each other or the plan's Global Constraints
- anything the plan explicitly mandates that the review rubric treats as a defect (a test that asserts nothing, verbatim duplication of a logic block)

Present everything you find to your human partner as one batched question — each finding beside the plan text that mandates it, asking which governs — before execution begins, not one interrupt per discovery mid-plan. If the scan is clean, proceed without comment. The review loop remains the net for conflicts that only emerge from implementation.

## Model Selection

Use the least powerful model that can handle each role to conserve cost and increase speed. Select it with the `child.model` (and, where relevant, `child.provider`) override on the dispatch.

**Mechanical implementation tasks** (isolated functions, clear specs, 1-2 files): use a fast, cheap tier. Most implementation tasks are mechanical when the plan is well-specified.

**Integration and judgment tasks** (multi-file coordination, pattern matching, debugging): use a standard tier.

**Architecture and design tasks**: use the most capable available tier. The final whole-branch review is one of these — dispatch it on the most capable available model, not the session default.

**Review tasks**: choose the model with the same judgment, scaled to the diff's size, complexity, and risk. A small mechanical diff does not need the most capable model; a subtle concurrency change does.

**Always set `child.model` explicitly when dispatching a subagent.** An omitted model inherits your session's model — often the most capable and most expensive — which silently defeats this section.

**Turn count beats token price.** Wall-clock and context cost scale with how many turns a subagent takes, and the cheapest models routinely take 2-3x the turns on multi-step work — costing more overall. Use a mid-tier model as the floor for reviewers and for implementers working from prose descriptions. When the task's plan text contains the complete code to write, the implementation is transcription plus testing: use the cheapest tier for that implementer. Single-file mechanical fixes also take the cheapest tier.

**Task complexity signals (implementation tasks):**
- Touches 1-2 files with a complete spec → cheap tier
- Touches multiple files with integration concerns → standard tier
- Requires design judgment or broad codebase understanding → most capable tier

## Handling Implementer Status

Implementer subagents report one of four statuses. Handle each appropriately:

**DONE:** Write the review diff file — a unified diff from BASE to the current tip with ~10 lines of context (git: `git diff -U10 BASE..HEAD`; jj: `jj diff --git --context 10 --from BASE --to @`) redirected to a uniquely named file (BASE is the revision you recorded before dispatching the implementer — never assume the task produced a single commit; deriving the diff from only the last commit silently drops all but the last commit of a multi-commit task). Then dispatch the task reviewer with that file path.

**DONE_WITH_CONCERNS:** The implementer completed the work but flagged doubts. Read the concerns before proceeding. If the concerns are about correctness or scope, address them before review. If they're observations (e.g., "this file is getting large"), note them and proceed to review.

**NEEDS_CONTEXT:** The implementer needs information that wasn't provided. Provide the missing context and re-dispatch.

**BLOCKED:** The implementer cannot complete the task. Assess the blocker:
1. If it's a context problem, provide more context and re-dispatch with the same model
2. If the task requires more reasoning, re-dispatch with a more capable model
3. If the task is too large, break it into smaller pieces
4. If the plan itself is wrong, escalate to the human

**Never** ignore an escalation or force the same model to retry without changes. If the implementer said it's stuck, something needs to change.

## Handling Reviewer ⚠️ Items

The task reviewer may report "⚠️ Cannot verify from diff" items — requirements that live in unchanged code or span tasks. These do not block the rest of the review, but you must resolve each one yourself before marking the task complete: you hold the plan and cross-task context the reviewer lacks. If you confirm an item is a real gap, treat it as a failed spec review — send it back to the implementer and re-review.

## Constructing Reviewer Prompts

Per-task reviews are task-scoped gates. The broad review happens once, at the final whole-branch review. When you fill a reviewer template:

- Do not add open-ended directives like "check all uses" or "run race tests if useful" without a concrete, task-specific reason
- Do not ask a reviewer to re-run tests the implementer already ran on the same code — the implementer's report carries the test evidence
- Do not pre-judge findings for the reviewer — never instruct a reviewer to ignore or not flag a specific issue. If you believe a finding would be a false positive, let the reviewer raise it and adjudicate it in the review loop. If the prompt you are writing contains "do not flag," "don't treat X as a defect," "at most Minor," or "the plan chose" — stop: you are pre-judging, usually to spare yourself a review loop.
- The global-constraints block you hand the reviewer is its attention lens. Copy the binding requirements verbatim from the plan's Global Constraints section or the spec: exact values, exact formats, and the stated relationships between components ("same layout as X", "matches Y"). The reviewer's template already carries the process rules (YAGNI, test hygiene, review method) — the constraints block is for what THIS project's spec demands.
- Hand the reviewer its diff as a file: hand each reviewer a unified diff file produced with the project's VCS plus the task-brief. Build the file by redirecting the commit list in BASE..tip (git: `git log --oneline BASE..HEAD`; jj: `jj --no-pager log -r 'BASE..@'`), a stat summary (git: `git diff --stat BASE..HEAD`; jj: `jj diff --stat --from BASE --to @`), and the full diff with ~10 lines of context (git: `git diff -U10 BASE..HEAD`; jj: `jj diff --git --context 10 --from BASE --to @`) into one uniquely named file per review (a re-review after fixes gets a fresh, distinctly named file). The output never enters your own context, and the reviewer sees the commit list, stat summary, and full diff in one Read call. Use the BASE you recorded before dispatching the implementer — never just the parent of the latest change, which silently truncates multi-commit tasks.
- A dispatch prompt describes one task, not the session's history. Do not paste accumulated prior-task summaries ("state after Tasks 1-3") into later dispatches — a real session's dispatch hit 42k chars of which 99% was pasted history. A fresh subagent needs its task, the interfaces it touches, and the global constraints. Nothing else.
- Dispatch fix subagents for Critical and Important findings. Record Minor findings in the progress ledger as you go, and point the final whole-branch review at that list so it can triage which must be fixed before merge. A roll-up nobody reads is a silent discard.
- A finding labeled plan-mandated — or any finding that conflicts with what the plan's text requires — is the human's decision, like any plan contradiction: present the finding and the plan text, ask which governs. Do not dismiss the finding because the plan mandates it, and do not dispatch a fix that contradicts the plan without asking.
- The final whole-branch review gets a diff file too: build it the same way from the branch base you recorded at setup (the change the branch started from) to `@`, and name the path in the final review dispatch, so the final reviewer reads one file instead of re-deriving the branch diff itself.
- Every fix dispatch carries the implementer contract: the fix subagent re-runs the tests covering its change and reports the results. Name the covering test files in the dispatch — a one-line fix does not need the whole suite. Before re-dispatching the reviewer, confirm the fix report contains the covering tests, the command run, and the output; dispatch the re-review once all three are present.
- If the final whole-branch review returns findings, dispatch ONE fix subagent with the complete findings list — not one fixer per finding. Per-finding fixers each rebuild context and re-run suites; a real session's final-review fix wave cost more than all its tasks combined.

## File Handoffs

Everything you paste into a dispatch prompt — and everything a subagent prints back — stays resident in your context for the rest of the session and is re-read on every later turn. Hand artifacts over as files:

- **Task brief:** before dispatching an implementer, hand each worker a task-brief file containing only its own task section from the plan. Extract the task's full text — its heading and everything under it, up to the next task heading — into a uniquely named file (e.g. `<workdir>/task-N-brief.md`) without routing the text through your own context (a small shell extraction or a cheap subagent can do the copy). Compose the dispatch so the brief stays the single source of requirements. Your dispatch should contain: (1) one line on where this task fits in the project; (2) the brief path, introduced as "read this first — it is your requirements, with the exact values to use verbatim"; (3) interfaces and decisions from earlier tasks that the brief cannot know; (4) your resolution of any ambiguity you noticed in the brief; (5) the report-file path and report contract. Exact values (numbers, magic strings, signatures, test cases) appear only in the brief.
- **Report file:** name the implementer's report file after the brief (brief `…/task-N-brief.md` → report `…/task-N-report.md`) and put it in the dispatch prompt. The implementer writes the full report there and returns only status, commits, a one-line test summary, and concerns.
- **Reviewer inputs:** the task reviewer gets three paths — the same brief file, the report file, and the diff file — plus the global constraints that bind the task.
- Fix dispatches append their fix report (with test results) to the same report file and return a short summary; re-reviews read the updated file.

## Durable Progress

Conversation memory does not survive compaction. In real sessions, controllers that lost their place have re-dispatched entire completed task sequences — the single most expensive failure observed. Track progress in a ledger file, not only in conversation.

- The ledger lives in the working folder the orchestrator supplies (e.g. alongside the plan file): `<workdir>/progress.md`.
- At skill start, check for an existing ledger. Tasks listed there as complete are DONE — do not re-dispatch them; resume at the first task not marked complete.
- When a task's review comes back clean, append one line to the ledger in the same message as your other bookkeeping: `Task N: complete (changes <base>..<head>, review clean)`.
- The ledger is your recovery map: the changes it names exist in the repo even when your context no longer remembers creating them. After compaction, trust the ledger and the VCS log (git: `git log --oneline`; jj: `jj --no-pager log`) over your own recollection.

## Prompt Templates

- [implementer-prompt.md](implementer-prompt.md) - Dispatch implementer subagent
- [task-reviewer-prompt.md](task-reviewer-prompt.md) - Dispatch task reviewer subagent (spec compliance + code quality)
- Final whole-branch review: adapt the task-reviewer template to branch scope — whole-branch diff file, the plan's requirements as the spec, and the ledger's Minor-findings list to triage.

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Initialize the progress ledger with all tasks]

Task 1: Hook installation script

[Record BASE; write task-1 brief file; dispatch implementer with brief + report paths + context]

Implementer: Status: NEEDS_CONTEXT — should the hook be installed at user or system level?

[Re-dispatch with the answer added to the prompt: "Install at user level (~/.config/hooks/)"]

Implementer:
  - Implemented install-hook command
  - Added tests, 5/5 passing
  - Self-review: Found I missed --force flag, added it
  - Committed (project convention)

[Write diff file from BASE..@; dispatch task reviewer with its path]
Task reviewer: Spec ✅ - all requirements met, nothing extra.
  Strengths: Good test coverage, clean. Issues: None. Task quality: Approved.

[Mark Task 1 complete in the ledger]

Task 2: Recovery modes

[Record BASE; write task-2 brief file; dispatch implementer with brief + report paths + context]

Implementer: [No questions, proceeds]
Implementer:
  - Added verify/repair modes
  - 8/8 tests passing
  - Self-review: All good
  - Committed

[Write diff file; dispatch task reviewer with its path]
Task reviewer: Spec ❌:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)
  Issues (Important): Magic number (100)

[Dispatch fix subagent with all findings]
Fixer: Removed --json flag, added progress reporting, extracted PROGRESS_INTERVAL constant

[Regenerate diff file; task reviewer reviews again]
Task reviewer: Spec ✅. Task quality: Approved.

[Mark Task 2 complete in the ledger]

...

[After all tasks]
[Dispatch final whole-branch reviewer with the branch diff file]
Final reviewer: All requirements met, ready to merge

Done — report completion to the enclosing workflow.
```

## Advantages

**vs. Manual execution:**
- Subagents follow TDD naturally
- Fresh context per task (no confusion)
- Subagent can ask questions — they arrive as NEEDS_CONTEXT returns (answered by re-dispatch), not mid-run conversation

**Efficiency gains:**
- Controller curates exactly what context is needed; bulk artifacts move as files, not pasted text
- Subagent gets complete information upfront
- Questions surfaced before work begins (not after)

**Quality gates:**
- Self-review catches issues before handoff
- Task review carries two verdicts: spec compliance and code quality
- Review loops ensure fixes actually work
- Spec compliance prevents over/under-building
- Code quality ensures implementation is well-built

**Cost:**
- More subagent invocations (implementer + reviewer per task)
- Controller does more prep work (extracting all tasks upfront)
- Review loops add iterations
- But catches issues early (cheaper than debugging later)

## Red Flags

**Never:**
- Start implementation on the main branch/bookmark without explicit user consent — create an isolated workspace first with the project's VCS (git: `git worktree add`; jj: `jj workspace add`)
- Skip task review, or accept a report missing either verdict (spec compliance AND task quality are both required)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make a subagent read the whole plan file (hand it its task-brief file instead)
- Skip scene-setting context (subagent needs to understand where task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance (reviewer found spec issues = not done)
- Skip review loops (reviewer found issues = implementer fixes = review again)
- Let implementer self-review replace actual review (both are needed)
- Tell a reviewer what not to flag, or pre-rate a finding's severity in the dispatch prompt ("treat it as Minor at most") — the plan's example code is a starting point, not evidence that its weaknesses were chosen
- Dispatch a task reviewer without a diff file — write it first (git: `git diff -U10 BASE..HEAD`; jj: `jj diff --git --context 10 --from BASE --to @`) and name the path in the prompt
- Move to next task while the review has open Critical/Important issues
- Re-dispatch a task the progress ledger already marks complete — check the ledger (and the VCS log) after any compaction or resume
- Skip a human gate or per-task commit required by the enclosing workflow

**If subagent asks questions:**
- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

**If reviewer finds issues:**
- Implementer (same subagent) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

**If subagent fails task:**
- Dispatch fix subagent with specific instructions
- Don't try to fix manually (context pollution)

## Integration

**Related skills in this pack:**
- **writing-plans** - Creates the plan this skill executes
- **tdd** - Subagents follow test-driven development for each task
- **issue-implementation** - The enclosing workflow this skill typically runs inside; its HITL/AFK mode and per-task commit rules govern pacing

**Before starting:** work in an isolated workspace, not directly on the main branch/bookmark — create one with the project's VCS (git: `git worktree add`; jj: `jj workspace add`).

**After the final review passes:** report completion to the enclosing workflow or your human partner; integration (merge, PR, cleanup) is their call.
