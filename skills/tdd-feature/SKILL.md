---
name: tdd-feature
description: Coordinate architect, critic, qa, and code-monkey to implement a feature using TDD. Use when the user asks to implement a feature, fix a bug, or make a code change and wants the full design-review-test-implement pipeline.
argument-hint: <task description>
---

You are coordinating a TDD implementation pipeline. Execute the following steps in order using the Task tool. Do not write code or tests yourself — delegate every action to the appropriate sub-agent.

## Task

$ARGUMENTS

## Step 1 — Architect

Invoke the `architect` sub-agent via Task using `model: "opus"`. Give it the task description above plus any relevant file paths you already know. Ask it to produce a written design proposal including: affected files, proposed changes, and key tradeoffs.

Do not summarize or restate the design yourself. Pass architect's full output to the next step.

## Step 2 — Critic

Invoke the `critic` sub-agent via Task. Give it the full architect output. Ask for a verdict and any ranked concerns.

If critic raises significant concerns, invoke architect again with critic's feedback and repeat until the plan is sound (max 2 cycles). If unresolved after 2 cycles, stop and ask the user to decide.

## Step 3 — QA (write failing tests first)

Invoke the `qa` sub-agent via Task. Give it the final approved plan and relevant code context. Instruct qa to write failing tests that will pass once the implementation is complete. qa must not implement production code.

## Step 4 — Code Monkey (implement)

Invoke the `code-monkey` sub-agent via Task. Give it the approved plan, the failing tests written by qa, and relevant file context. Instruct code-monkey to implement the solution until all tests pass.

## Step 5 — Verify

Run the tests using the `running-bzl` sub-agent to confirm they pass.

## Step 6 — Critic (code review)

Invoke the `critic` sub-agent via Task. Give it the implemented code diff and test output. Ask it to propose potential improvements focused on:
- Simplifying control flows (reduce nesting, early returns, remove unnecessary branches)
- Increasing readability (naming, structure, clarity)
- Catching potential bugs (edge cases, nil dereferences, off-by-one errors, error handling gaps)

If critic identifies significant issues, invoke `code-monkey` again to address them, then re-run tests with `running-bzl`. Repeat until critic is satisfied (max 2 cycles).

## Step 7 — QA coverage pass

Invoke the `qa` sub-agent via Task. Give it the final implemented code and the existing test suite. Ask qa to evaluate whether test coverage is adequate. If there are meaningful gaps (untested edge cases, error paths, or behaviors), qa should add tests to cover them. Then re-run tests with `running-bzl` to confirm all tests pass.

## Step 8 — Summary

Produce a concise summary for the user covering:
- What was done
- Key design decisions and alternatives considered
- Tradeoffs accepted
- Any open questions or known limitations
