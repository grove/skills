# `/fix-workflow` Skill Design

## Status

Proposed

## Summary

`/fix-workflow` diagnoses and repairs failed GitHub workflows for pull requests, then verifies that the PR branch satisfies its required checks.

The skill is outcome-oriented rather than check-oriented.

A workflow is not fixed merely because the YAML parses or a rerun turns green. The skill must determine:

1. why the workflow failed,
2. whether the failure represents a real code defect, CI defect, environment problem, or transient failure,
3. what the smallest correct repair is,
4. whether the PR's required checks pass on the repaired commit.

---

# Motivation

A common agent workflow looks like:

```text
GitHub issue
    ↓
/implement
    ↓
/code-review
    ↓
/prove
    ↓
PR opened
    ↓
GitHub Actions fails
```

At this point the implementation may be correct, but the work is not finished.

Developers often end up manually:

* opening the failed workflow,
* finding the failing job,
* reading several hundred lines of logs,
* determining whether the failure is real or flaky,
* reproducing it locally,
* fixing the problem,
* rerunning CI,
* and checking that the operation actually completed.

`/fix-workflow` exists to carry the PR workflow through to completion.

---

# Primary Objective

> **Restore the workflow's intended outcome by identifying and fixing the underlying failure without weakening the verification that exposed it.**

The skill optimizes for the successful operation, not merely a green workflow badge.

---

# Typical Invocation

For a pull request:

```text
/fix-workflow #123
```

or:

```text
/fix-workflow https://github.com/acme/project/pull/123
```

For a failed workflow run:

```text
/fix-workflow <workflow-run-url>
```

The skill should infer the relevant workflow from the supplied PR or run whenever possible.

---

# Core Principle

The skill should work from:

```text
Failed PR workflow
    ↓
Failed job
    ↓
Failed step
    ↓
Underlying cause
    ↓
Repair and verification
```

A green rerun without a relevant change is not evidence of a durable fix.

---

# Operating Model

Use one repair loop for PR and CI workflows:

> Identify the failed check, repair its cause, preserve its guarantee, and verify the repaired PR commit.

# Non-Goals

## Not “make CI green at any cost”

The skill must not solve failures by weakening verification unless the verification itself is demonstrably incorrect.

Strongly resist:

* deleting failing tests,
* skipping tests,
* disabling lint rules,
* lowering coverage requirements,
* marking required checks optional,
* adding unconditional `continue-on-error`,
* swallowing command failures,
* adding arbitrary retries,
* commenting out workflow steps.

If the check itself is incorrect, modifying it can be the correct repair, but the skill must demonstrate why.

---

## Not a general refactoring pass

Changes should remain tightly related to the workflow failure.

Do not use a CI failure as an excuse to clean up unrelated code.

---

## Not a workflow redesign skill

If the workflow has a clear defect, repair it.

Do not redesign the entire CI architecture unless the failure cannot be resolved safely within the existing design.

---

## Not a blind rerun command

A rerun is diagnostic evidence, not automatically a fix.

If a failing test passes on rerun, investigate whether the failure was genuinely transient or whether the system contains nondeterminism.

---

# Failure Classification

Before changing code, classify the primary cause:

* **Product:** the branch, test target, or generated output is wrong.
* **Workflow:** GitHub Actions configuration, permissions, or job wiring is wrong.
* **External:** a dependency, runner, credential, or service is unavailable.

Flakiness is a confidence modifier, not a separate cause: a rerun without a relevant change is `WORKFLOW GREEN — FLAKE NOT RESOLVED` until nondeterminism is understood.

---

# Workflow

## 1. Establish the Intended Operation

Determine what the failed workflow was supposed to accomplish. For example:

```text
Validate branch before merge
Run required checks for commit abc123
```

The intended operation defines success.

## 2. Diagnose the Failure

Inspect the run, failed jobs and steps, logs, workflow YAML, and relevant changes. Start with the earliest causal failure, then check for independent failures in parallel jobs. Capture the workflow, job, step, command, and observed error.

Reproduce the failing command locally where practical. State the failure mechanism and classify its primary cause before editing.

## 3. Apply the Smallest Correct Repair

Repair the product, test, workflow, or external dependency issue while preserving the guarantee that exposed it. Do not bypass checks, add arbitrary retries, or broaden the change.

## 4. Verify the Repair

Run the narrowest reproduction first, then the affected package or workflow checks. Rerun or observe the GitHub workflow and verify all required checks, not only the original failing job.

## 5. Verify the Intended Outcome

Confirm the repaired commit is pushed, required PR checks are green, and no verification was weakened.

# Flaky Workflow Handling

When a workflow passes after rerun without code changes:

```text
Do not immediately report FIXED.
```

Instead classify it as:

```text
Transient / suspected flaky
```

Then inspect whether there is enough evidence to identify a cause.

A valid final result may be:

```text
WORKFLOW GREEN — FLAKE NOT RESOLVED
```

if the rerun passes but the underlying nondeterminism remains unexplained.

The skill should not claim to have fixed something it merely failed to reproduce.

---

# Final Outcomes

The skill should finish with one of two outcomes.

## FIXED

Use when the underlying failure is understood, a durable repair was applied, focused verification passes, the repaired commit is pushed, and all required PR checks are green.

## NOT FIXED

Use when the failure remains unresolved, the required capability is unavailable, the worktree is unsafe to modify, or the workflow is green only because of an unexplained rerun.

---

# Authorization

Invoking `/fix-workflow` authorizes automatic inspection, scoped edits, local verification, commit, push, and workflow reruns for the target PR.

Before editing, require a clean worktree apart from the target branch's known changes. Stop with `NOT FIXED` when unrelated edits, missing permissions, unavailable credentials, or an unsafe branch state would make the repair ambiguous.

Never weaken required checks, suppress failures, or continue past a failed verification merely to obtain a green status. Cap repair attempts and stop when the cause remains uncertain.

---

# Relationship to Other Skills

A typical workflow becomes:

```text
/interrogate #123
    ↓
/implement #123
    ↓
/code-review main
    ↓
/prove #123
    ↓
PR workflow fails
    ↓
/fix-workflow <PR or run>
```

The responsibilities remain distinct:

```text
/interrogate
Is the proposed implementation approach sound?

/implement
Build it.

/code-review
Does the diff meet the spec and repository standards?

/prove
Can we demonstrate that the issue is actually resolved?

/fix-workflow
Why did automation fail, and can we restore the operation safely?
```

If `/fix-workflow` materially changes production code or tests, rerun `/code-review` and consider whether `/prove` should also be rerun for the affected issue.

---

# Suggested Frontmatter

```yaml
---
name: fix-workflow
description: Diagnose and repair failed GitHub Actions workflows for pull requests without weakening required checks, then verify the repaired commit.
disable-model-invocation: true
---
```

Keep the skill user-invoked initially while its automatic commit and push behavior is established.

---

# Initial Version

The first version should support:

1. locating a failed PR workflow run,
2. reading logs and identifying the earliest causal failure,
3. classifying and reproducing the failure where practical,
4. applying a narrow repair without weakening checks,
5. running focused verification,
6. committing, pushing, and rerunning the workflow,
7. reporting `FIXED` or `NOT FIXED`.

Avoid workflow redesign, release or deployment orchestration, generalized retry frameworks, and cross-repository support. The skill's value should come from disciplined diagnosis and completion.

---

# One-Sentence Definition

> **`/fix-workflow` traces a failed pull-request workflow to its underlying cause, repairs that cause without weakening verification, and verifies the repaired commit until the required checks are green or the repair is reported `NOT FIXED`.**


