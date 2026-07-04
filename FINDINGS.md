# FINDINGS: v1 phased-skill schema vs. two real processes

Schema-gap report for sh/sage#475 (feeding PRD sh/sage#473). Findings were
collected while porting the `tdd` and `issue-implementation` workflows into
the v1 frontmatter schema (Tasks 4-5), consolidated and deduplicated here.

## Schema gaps found while authoring

### From the tdd port

**1. Cyclic workflow vs linear phase list.** TDD is a loop: after refactor,
the next behavior starts a new red phase. The v1 schema has no "loops back"
concept — `phases:` is a linear list — so the cycle is expressed only in
prose via a backward transition ("transition backward to red with a reason
naming the next test"). If the engine doesn't surface backward transitions
well, multi-cycle TDD can't be represented; the frontmatter describes one
cycle, not the workflow.

**2. Pre-phase content with its own approval.** The source's Planning step
carries a hard requirement ("Get user approval on the plan"), but planning
is not one of the mandated phases (red/green/refactor). It now lives in the
skill body's intro ("Before the First Red: Planning"), and its approval
requirement cannot attach a gate to non-phase content. A
`planning: { gate: human }` phase is the likely fix, but adding a phase was
outside the mandated frontmatter for this port.

**3. Tracer bullet doesn't map to a phase.** The source treats the tracer
bullet as a distinct first step; it is not a phase and has no schema slot,
so it was folded into prose as "the first cycle" of the ordinary loop.

### From the issue-implementation port

**4. No subagent capability constraints in frontmatter.** The scout must be
read-only, but there is no schema key to declare a subagent capability
constraint; the restriction lives in prose only ("dispatch a
pre-implementation scout subagent (read-only)"). The repo owner has
confirmed read-only is the intended constraint.

**5. Repeating micro-workflow inside a phase.** The implement phase's serial
per-task loop (worker → review → fix → re-review → commit, once per task) is
itself a small repeating workflow, invisible to the flat phase list — from
the schema's view, `implement` is a single opaque phase.

**6. No cross-phase/global directives concept.** The subagent dispatch
discipline (async-only, complete initial prompts, worker-initiated
escalation) applies to every phase that dispatches subagents, but the schema
has no place for workflow-wide directives. It had to be housed in the
`# Scout` section with explicit "this and every subagent in this process"
scoping — correct, but positionally arbitrary.

**7. Agent-review vs human-review split.** The post-implementation agent
review (spec adherence, code quality/DRY lenses) is a review activity, but
the `review` phase is the human gate, so the agent review had to ride inside
the `pr` phase. There is no way to mark a phase as carrying an agent-review
obligation distinct from a human gate.

**8. No invariant/guard concept.** "If any required step is skipped, STOP
immediately, report it, and ask the human" is a workflow-wide invariant, not
a review-phase rule, yet it lives in the `# Review` section because v1 has
no schema slot for guards that hold across all phases.

**9. Gate semantics inside subagents are undefined.** Gates are a
session-level primitive — they pause the loop and prompt the human — but
phased skills are exactly what a worker subagent may be told to follow
(e.g. implementation workers following `tdd` inside `issue-implementation`'s
implement phase). A non-interactive subagent has no gate surface, so
`refactor: { gate: human }` is unreachable when tdd guides a worker rather
than the main session. This is also why tdd did not gain a
`planning: { gate: human }` phase despite gap 2: the owner ruled that adding
gates to a skill likely to run in subagent contexts bakes in a primitive
that may not exist there. Candidate resolutions for the PRD: gates degrade
to an escalation return (NEEDS_CONTEXT-style) in child contexts; gates
auto-waive in child contexts; or phased skills declare whether they are
session-level.

## `triggers:` verdict

During authoring, both phased skills' `description` fields carry the
when-to-use signal — "Use when asked to grab, take, or work on a tracked
issue" and "Use when building features or fixing bugs test-first" — which is
exactly how non-phased skills already get discovered. Explicit `triggers:`
patterns were NOT missed during authoring. Verdict: `triggers:` can stay
deferred. One caveat: issue-implementation's activation phrases ("grab
#123") are load-bearing in its description — if descriptions ever get
truncated or rewritten by an installer or catalog, that skill loses its
activation surface first.

## Installer-layout notes

The pack uses the standard `skills/<name>/SKILL.md` layout; supporting files
(tests.md, mocking.md, deep-modules.md, interface-design.md, refactoring.md,
implementer-prompt.md, task-reviewer-prompt.md,
plan-document-reviewer-prompt.md) ride inside their skill
folders and travel with the skill under any standard installer.
Sage-specific frontmatter keys (`phases:` and its `gate`/`skip_when`/`skill`
entries) are inert extra YAML for other consumers — a non-Sage agent reads
the same file as plain markdown guidance.

Observed risk: the phase entries use YAML inline maps
(`grill: { gate: human, skip_when: criteria_clear }`). A frontmatter parser
that only handles simple `key: value` scalars would fail on these entries
and could reject the whole frontmatter block; in that case the skill
degrades to plain markdown — which is the designed fallback, so the failure
mode is graceful, but consumers that key discovery off frontmatter
`name`/`description` would then not discover the skill at all.

## Overall verdict

The v1 schema survived contact with both real processes — as a
guidance-first mechanism. Every gap above had a workable prose fallback,
and none blocked authoring: cycles, pre-phase approvals, capability
constraints, per-task loops, global directives, and invariants all landed
cleanly in the skill bodies, with the frontmatter carrying the phase
skeleton and gates. The gaps are candidates for future schema keys (loop
transitions, non-terminal gates on arbitrary phases, capability
declarations, workflow invariants), not v1 blockers.
