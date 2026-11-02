---
name: asana-task-review
description: Use when an initialized Asana task is Ready for review or In review and needs a task-scoped code review before deployment.
---

# Asana Task Review

Review the implemented task against its saved plan and acceptance criteria. Success advances to `Approved`; actionable findings return the task to `Ready for work`.

## Persisted state and isolation

- Required input: `task_gid`. Read the task with its notes, rich description, completion flag, and relevant comments. Discover actual Asana tool schemas; do not assume argument names.
- Parse exactly one YAML `agent_meta` block bounded by `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->`. On failed reads or malformed metadata, preserve the description and stop; report `recovery_required: true` when the task is addressable.
- Follow the description-preserving read, patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve unknown metadata, history, human text, rich content, and concurrent edits. Treat `isError` and every non-empty `failed` as failure. Reconcile uncertain writes before retrying or handing off.
- Use only the task-owned worktree and persisted branch. Never switch, reset, clean, stash, or commit another checkout. A dirty caller checkout is not a blocker.
- Write metadata, statuses, and Asana comments in English. Use UTC history entries with `at`, `from`, `to`, and `stage`.
- Increment the stage attempt counter once per invocation that enters stage execution after its prerequisites are resolved. If the new count exceeds 3, persist a stage diagnostic and stop. A verified transition must not be undone or repeated solely because a reporting comment failed. A failed outcome comment does not block a handoff after a verified state transition; description-update failures always require reconciliation and stop this invocation.

## Procedure

1. Continue only from `Ready for review` or `In review`. For any other state, report the persisted state and stop without changing it. Never reopen a completed task.
2. Resolve the repository, worktree, source branch, integration ref, and implementation commit. Missing or ambiguous refs keep the task in its current stage with `last_error.stage: review`; do not invent a base or review unrelated changes.
3. Increment `attempts.review`, persist `In review`, and append history before review starts.
4. Inspect the complete task diff against the verified integration ref, project instructions, affected callers, and relevant tests. Check acceptance criteria, correctness, domain boundaries, invalid input, authorization where applicable, compatibility, and unnecessary complexity. Use installed project review guidance where available. Run proportionate checks from the task worktree; distinguish executed checks from unverified claims.
5. Record `review` evidence: inspected commit and base, findings with severity and file/line, acceptance-criteria outcome, checks and results, and material limitations. Do not approve solely because CI is green. If checks are unavailable or correctness remains unresolved, retain `In review` and record the concrete blocker.
6. For actionable failures, persist findings and `last_error: {stage: review, message: ...}`, transition to `Ready for work`, append history, verify the saved state, then comment and invoke `asana-task-work` with the same task ID. Do not fix unrelated issues in the review stage.
7. When no blocking findings remain, recheck that the source commit and integration ref still match the reviewed versions. If either changed, review the updated diff before approving. Clear only resolved review errors, persist `Approved` and history, and read back the result. Add an English review outcome comment and invoke `asana-task-deploy` only after the verified transition and when delivery scope authorizes continuation.

A review error after valid metadata was read retains `In review` and records `last_error.stage: review` when a safe metadata-only write remains possible. Do not claim approval after an uncertain update.
