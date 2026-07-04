---
name: tdd
description: Test-driven development with a red-green-refactor loop. Use when building features or fixing bugs test-first.
phases:
  - red
  - green
  - refactor: { gate: human }
---

# Test-Driven Development

## Philosophy

**Core principle**: Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_ it does it. A good test reads like a specification - "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means (like querying a database directly instead of using the interface). The warning sign: your test breaks when you refactor, but behavior hasn't changed. If you rename an internal function and tests fail, those tests were testing implementation, not behavior.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" - treating RED as "write all tests" and GREEN as "write all code."

This produces **crap tests**:

- Tests written in bulk test _imagined_ behavior, not _actual_ behavior
- You end up testing the _shape_ of things (data structures, function signatures) rather than user-facing behavior
- Tests become insensitive to real changes - they pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat. Each test responds to what you learned from the previous cycle. Because you just wrote the code, you know exactly what behavior matters and how to verify it.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Before the First Red: Planning

When exploring the codebase, use the project's domain glossary so that test names and interface vocabulary match the project's language, and respect ADRs in the area you're touching.

Before writing any code:

- [ ] Confirm with user what interface changes are needed
- [ ] Confirm with user which behaviors to test (prioritize)
- [ ] Identify opportunities for [deep modules](deep-modules.md) (small interface, deep implementation)
- [ ] Design interfaces for [testability](interface-design.md)
- [ ] List the behaviors to test (not implementation steps)
- [ ] Get user approval on the plan

Ask: "What should the public interface look like? Which behaviors are most important to test?"

**You can't test everything.** Confirm with the user exactly which behaviors matter most. Focus testing effort on critical paths and complex logic, not every possible edge case.

Your very first cycle is a **tracer bullet**: write ONE test that confirms ONE thing about the system, then make it pass with minimal code. This proves the path works end-to-end. Every cycle after that follows the same red-green-refactor loop below, one behavior at a time.

# Red

Write exactly ONE failing test for the next behavior:

```
RED: Write test for next behavior → test fails
```

Rules:

- One test at a time
- Verify behavior through public interfaces, not implementation details
- Keep tests focused on observable behavior
- The test should describe WHAT the system does, not HOW

See [tests.md](tests.md) for what good tests look like, and [mocking.md](mocking.md) for when mocking is appropriate (mock only at system boundaries — never your own modules or internal collaborators).

Run the test suite and confirm the new test fails for the expected reason. Only then report the transition to green. Use the project's own test command (some, like `zig build test`, exit silently on success — a failing test must produce visible failure output).

# Green

Write the minimal implementation to make the current test pass:

```
GREEN: Minimal code to pass → test passes
```

Rules:

- Only enough code to pass the current test
- Don't anticipate future tests
- No speculative features — each future behavior gets its own red-green cycle

Run the suite; all tests pass before transitioning to refactor. Never refactor while RED.

# Refactor

With all tests green, improve the structure. Look for [refactor candidates](refactoring.md):

- **Duplication** → Extract function/class
- **Long methods** → Break into private helpers (keep tests on public interface)
- **Shallow modules** → Combine or deepen
- **Feature envy** → Move logic to where data lives
- **Primitive obsession** → Introduce value objects
- **Existing code** the new code reveals as problematic

Also:

- [ ] Deepen modules (move complexity behind simple interfaces) — see [deep-modules.md](deep-modules.md)
- [ ] Apply SOLID principles where natural
- [ ] Consider what new code reveals about existing code
- [ ] Run tests after each refactor step

For design guidance, see [deep-modules.md](deep-modules.md) (small interface, deep implementation) and [interface-design.md](interface-design.md) (designing for testability).

## Checklist Per Cycle

Before signing off a cycle, verify:

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```

## Committing

Use the VCS the project already uses — never assume one. If a VCS-specific skill is available (e.g. a jujutsu skill for jj repositories), read it before running VCS commands. Commit each completed cycle as a coherent unit, following the project's commit-message convention if one is evident from instructions or the VCS history; default to Conventional Commits otherwise (e.g. `feat: ...`, `fix: ...`, `test: ...`). Note that an enclosing workflow may own commits — if so, leave the working copy clean of unrelated changes and let it commit.

This workflow is one cycle. For the next behavior, transition backward to red with a reason naming the next test. The human gate on leaving refactor is the cycle's sign-off.
