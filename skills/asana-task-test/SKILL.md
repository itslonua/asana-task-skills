---
name: asana-task-test
description: Use when an initialized Asana task is in Ready for testing or Testing and requires task-scoped QA verification.
---

# Asana Task Test

Execute QA verification for one initialized Asana task. Preserve the task notes as the source of truth and record every state transition in English.

## Contract

- Input: required `task_gid`.
- Success: set `agent_meta.current_status` to `Ready for release`, then invoke `asana-task-mr`.
- Failure: set it to `Ready for work`, then invoke `asana-task-work`.
- Metadata is the YAML `agent_meta` block between `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->` in task notes.
- Required Asana tools: `asana_get_task`, `asana_update_tasks`, and `asana_add_comment`.

## State machine

`Needs estimation → Estimating → Ready for work → In progress → Ready for review → In review → Approved → Deploying → Ready for testing → Testing → Ready for release → Completed`

## Isolation

- Run checks only from the task's dedicated worktree. Never switch, reset, clean, stash, or mutate another task's checkout.
- A dirty checkout, a different branch, or concurrent work is not a blocker.
- Use task-scoped Docker/container resources. Never stop or overwrite resources owned by another task.

## Procedure

1. Read the task with `asana_get_task(task_gid)` and parse valid `agent_meta` from notes.
   - If the metadata block is missing or malformed, preserve notes, add an English diagnostic comment containing `recovery_required: true`, and stop. Do not reconstruct metadata or invoke `asana-task-work`.
2. Continue only when status is `Ready for testing` or `Testing`. Otherwise add an English comment and stop without running tests.
3. Increment `attempts.test`. If it exceeds 3, persist `last_error` with `stage: test`, add an English escalation comment mentioning the configured owner when available, and stop.
4. Persist `Testing` and append a status-history entry before commenting or starting checks.
5. Treat every non-empty `failed` result from `asana_update_tasks` as an error. Persist the diagnostic in `last_error` when notes are still writable, then stop.
6. Require verified `deployment` evidence and confirm that the deployed commit and source branch head match the approved `review` commit. If they differ, persist `Ready for review` with a diagnostic and invoke `asana-task-review` after a verified write; do not test a different revision. Missing deployment evidence keeps `Testing` with a diagnostic. Run the task's relevant automated and/or QA checks from its worktree against the recorded verification endpoint. Record `testing` evidence with the tested commit, endpoint, checks, and outcomes. After testing starts, use unconditional cleanup to stop only temporary task-scoped Docker resources created for these checks before any handoff, whether tests or Asana calls succeed or fail. Keep the deployed verification environment available unless its documented lifecycle explicitly requires teardown.
7. On failed checks, persist the failure in `last_error`, transition to `Ready for work`, append history, update notes, comment in English, then invoke `asana-task-work`.
8. On success, clear `last_error`, transition to `Ready for release`, append history, update notes, comment in English with the test outcome and next stage, then invoke `asana-task-mr`.

## Description updates

For every metadata write, follow the description-preserving fresh-read, metadata-only patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve rich HTML, unknown metadata, and concurrent human edits. On a reported or uncertain task-update failure, reconcile by reading back and stop this invocation before any handoff; do not restore an old full description. Outcome-comment failures do not invalidate a verified transition.
