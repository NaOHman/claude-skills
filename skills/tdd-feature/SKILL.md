---
name: tdd-feature
description: Coordinate architect, critic, qa, and code-monkey to implement a feature using TDD. Use when the user asks to implement a feature, fix a bug, or make a code change and wants the full design-review-test-implement pipeline.
argument-hint: [--interactive] [--no-coverage] <task description>
---

You are coordinating a TDD implementation pipeline. Execute the following steps in order using the Task tool. Do not write code or tests yourself — delegate every action to the appropriate sub-agent.

The `agents/` directory alongside this skill contains the definitions for architect, critic, qa, and code-monkey. If those agent types are not yet installed, copy each file to `~/.claude/agents/` before running.

## Flags

Before starting, scan `$ARGUMENTS` for the following flags and strip them from the task description before passing it to any sub-agent.

- `--interactive` — Pause after the architect and critic agree on a plan (end of Step 2) and present it to the user for approval before implementation begins.
- `--no-coverage` — Skip the QA coverage pass in Step 6. The critic code review still runs; only the QA agent is omitted.

## Model selection

Before invoking the architect, assess the complexity of the task and pick a model. The architect is the most expensive agent in this pipeline — using sonnet when the task doesn't need opus saves meaningful time and cost without sacrificing quality.

Use **`sonnet`** when the task is tightly scoped and the design space is narrow:
- The description names specific files, methods, fields, or endpoints being changed
- There is essentially one reasonable way to implement it — no significant tradeoffs to weigh
- No new abstractions are likely needed; it fits into existing patterns
- Small blast radius: 1–3 files, no cross-component coordination
- *Examples: "make field X optional", "add a flag to endpoint Y", "rename Z", "add a config toggle"*

Use **`opus`** when the task requires deeper judgment:
- Vague or under-specified — needs exploration to scope properly
- Multiple valid approaches with real tradeoffs between them
- New abstractions, interfaces, or cross-cutting patterns are likely needed
- Spans multiple components or subsystems
- Involves security, performance, concurrency, or API contract changes
- *Examples: "refactor the auth system", "add real-time updates", "make the service multi-tenant"*

When uncertain, choose opus — a weak architect output costs more to recover from than the price of a stronger model.

## Task

$ARGUMENTS

## Step 1 — Architect

Invoke the `architect` sub-agent via Task using the model selected above. Give it the task description and any relevant file paths. Prompt it to:

- Map the relevant directory structure and key files before proposing anything
- Identify existing abstractions, interfaces, and patterns that could be reused or extended
- Trace the current coupling between components in the affected area
- Find prior art: how have similar problems been solved elsewhere in this codebase?

The design must reuse existing abstractions rather than inventing new ones wherever possible, minimize new coupling, and explicitly name which existing patterns it builds on — not leave this implicit.

Ask it to produce a written design proposal including: affected files, proposed changes, a summary of existing patterns being leveraged, and key tradeoffs.

Do not summarize or restate the design. Pass the architect's full output — including its exploration findings — to all subsequent steps.

## Step 2 — Critic (plan review)

Invoke the `critic` sub-agent via Task. Give it the full architect output and the original task description. Ask it to pay particular attention to:
- Whether the design appropriately reuses existing abstractions or invents unnecessarily
- Whether any new coupling is justified or could be avoided
- Whether the approach fits established patterns in the codebase

If the critic raises concerns, invoke architect again with the full critic feedback and ask it to revise the plan. Repeat until the plan is sound (max 2 cycles). After 2 unresolved cycles, surface the disagreement to the user and stop.

**If `--interactive` was passed:** Once the plan is approved, pause and present it to the user before continuing to implementation. Summarize:
- Affected files and what changes in each
- Key tradeoffs the architect identified
- Open questions or assumptions baked into the plan

Use `AskUserQuestion` to ask whether they want to proceed or request changes.

- **Approve**: continue to Step 3.
- **Request significant changes**: feed their feedback back to the architect as a new revision request and run one more critic cycle, then proceed to Step 3 regardless (to avoid an unbounded loop). If unresolved concerns remain, carry them forward to the Step 8 summary.
- **Minor clarifications**: incorporate them into the plan context passed to subsequent steps, then continue.

## Step 3 — QA (write failing tests)

Invoke the `qa` sub-agent via Task. Give it the approved plan and relevant code context. Instruct it to write failing tests that will pass once implementation is complete. It must not write production code.

## Step 4 — Code Monkey (implement)

Invoke the `code-monkey` sub-agent via Task. Give it:
- The approved design plan, **including the architect's exploration findings about existing patterns and abstractions**
- The failing tests from Step 3
- Relevant file context

Instruct it to:
- Study the existing code in the affected area before writing anything
- Implement with the grain of the codebase — use existing helpers, types, abstractions, and idioms
- Reach for existing abstractions first; only introduce new ones when the plan explicitly calls for them

## Step 5 — Verify (run tests)

Check the available sub-agent types in your Task tool description. If `running-bzl` is listed, use it to run the tests for the affected packages. Otherwise, spawn a `general-purpose` sub-agent and ask it to:
1. Identify the build/test system from the codebase (BUILD files → Bazel with `bzl test`; `go.mod` only → `go test`; `pyproject.toml` or `pytest.ini` → `pytest`)
2. Run the appropriate test command scoped to the affected packages
3. Report the full output

If tests fail, return to Step 4 with the failure details.

## Step 6 — Parallel Review (Critic + QA)

In a **single message**, spawn both agents concurrently via two Task tool calls:

1. **Critic (code review)**: Give it the implemented code and test output. Ask it to review for tight coupling, missed opportunities to use existing abstractions, control-flow complexity, readability, and potential bugs. Ask it to **explicitly label each concern** as either `[correctness/security]` or `[quality]` (quality = comments, naming, missing tests, documentation).

2. **QA (coverage pass)** — *skip if `--no-coverage` was passed*: Give it the final code and the existing test suite. Ask it to identify meaningful gaps — untested edge cases, error paths, or behaviors — and **write the missing tests**. It must not write production code.

Wait for all spawned agents to complete before proceeding. If `--no-coverage` was passed, wait only for the critic.

## Step 7 — Address All Findings

Invoke `code-monkey` once with the **combined** findings from both agents:
- The critic's full list of concerns (with their labels)
- The QA's new tests (which may be failing and need production code to pass)

Require a point-by-point response for each critic concern explaining what was changed and why.

After code-monkey finishes, re-run tests using the Step 5 approach.

**Conditional verification:** Check whether the critic labeled any concerns as `[correctness/security]`.

- **If yes**: invoke `critic` again with the revised code and code-monkey's point-by-point responses, asking it to verify those specific concerns were resolved. Re-run tests if further corrections were made. Repeat up to 1 more cycle.
- **If no** (all concerns were `[quality]` only): skip re-verification. The passing test run is sufficient — quality concerns like comments, naming, and test coverage don't need a second critic pass to confirm.

## Step 8 — Summary

Write a concise summary covering:
- What was implemented
- Key design decisions and alternatives considered
- Tradeoffs accepted
- Open questions or known limitations

---

> **Sharing this skill:** The `agents/` directory contains the definitions for all agents this skill depends on. To use this skill in a new environment, copy each file from `agents/` into `~/.claude/agents/`.
