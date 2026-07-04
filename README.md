# sage-workflows

`sage-workflows` is a reference pack of agent skills in the standard
`skills/<name>/SKILL.md` layout. It contains two phased Sage workflows
(`issue-implementation`, `tdd`) and the two supporting skills they reference
(`writing-plans`, `subagent-driven-development`).

Install with the standard skills installer:

```sh
npx skills add deevus/sage-workflows
```

In Sage, phased skills activate workflow mode; in any other agent the same
files read as plain markdown guidance.

## Skills

### issue-implementation

Human-in-the-loop implementation of tracked issues. The primary assistant acts
as an orchestrator — never the direct implementer — coordinating read-only
scouts, implementer workers, and reviewer subagents from issue intake through
merge, with the human holding the gates. It is phased:
intake → grill (human gate, skippable when criteria are clear) → scout →
plan → implement → pr → review (human gate). The plan phase follows the
`writing-plans` skill and the implement phase follows the
`subagent-driven-development` skill — both referenced in its frontmatter.
Invoke in Sage with `/skill:issue-implementation`, or just ask to grab, take,
or work on a tracked issue.

### tdd

Test-driven development as a red-green-refactor loop, built on the principle
that tests verify behavior through public interfaces, not implementation
details. It insists on vertical slices — one test, one implementation, repeat,
starting with a tracer bullet — and explicitly forbids writing all tests up
front. It is phased: red → green → refactor (human gate); each pass through
the phases is one cycle, and the workflow loops back from refactor to red for
the next behavior, with the human gate on refactor serving as the cycle's
sign-off. Supporting references on test quality, mocking, deep modules,
interface design, and refactoring ride alongside the skill; it references no
other pack skills. Invoke in Sage with `/skill:tdd`.

### writing-plans

Not phased — a guidance skill for turning a spec into a comprehensive
implementation plan before touching code. Plans assume an engineer with zero
codebase context: exact file paths, complete code in every step, exact
commands with expected output, bite-sized 2-5 minute steps, and no
placeholders. It covers scope checks, file-structure decisions, task
right-sizing, a self-review checklist, and hands off execution to the
`subagent-driven-development` skill. Referenced by `issue-implementation`'s
plan phase. Invoke in Sage with `/skill:writing-plans`.

### subagent-driven-development

Not phased — a guidance skill for executing an implementation plan by
dispatching a fresh implementer subagent per task, reviewing each task
(spec compliance + code quality), and running a broad whole-branch review at
the end. It covers model selection per role, file-based handoffs (task
briefs, report files, diff files), a durable progress ledger that survives
compaction, and handling of implementer statuses and reviewer findings. It
executes plans produced by `writing-plans` and typically runs inside
`issue-implementation`, whose human gates and per-task commit rules govern
pacing. Referenced by `issue-implementation`'s implement phase. Invoke in
Sage with `/skill:subagent-driven-development`.

## Acknowledgements

Several skills in this pack are adaptations of existing work:

- `writing-plans` and `subagent-driven-development` are adapted from
  [obra/superpowers](https://github.com/obra/superpowers) by Jesse Vincent.
- `tdd` is adapted from
  [mattpocock/skills](https://github.com/mattpocock/skills) by Matt Pocock.

The adaptations make them VCS-agnostic, phrase subagent dispatch against
Sage's blocking `subagent` tool, and defer pacing to enclosing human-gated
workflows. `issue-implementation` derives from the Sage project's required
issue workflow.
