---
name: asana-task-work
description: Use when an initialized Asana task is in Ready for work or In progress and requires implementation in its task-scoped Git worktree.
---

# Asana Task Work

Implement one initialized Asana task in its isolated worktree. Task notes and their `agent_meta` are authoritative; write all metadata, status values, and comments in English.

## Contract

- Input: required `task_gid`; optional `branch`, otherwise use `agent_meta.branch`.
- Success: commit changes on the persisted branch, transition to `Ready for review`, and invoke `asana-task-review`.
- Metadata: YAML `agent_meta` between `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->` in task notes.
- Required Asana tools: `asana_get_task`, `asana_update_tasks`, and `asana_add_comment`.

## State machine

`Needs estimation → Estimating → Ready for work → In progress → Ready for review → In review → Approved → Deploying → Ready for testing → Testing → Ready for release → Completed`

## Worktree isolation

- Do all Git operations in the task's dedicated worktree for its saved source branch.
- Never switch, reset, clean, stash, or commit in the caller's checkout. Its dirty state or current branch is not a blocker.
- Concurrent tasks use separate worktrees and task-scoped branches; checkout ownership is never a state-machine lock.
- If the worktree is missing, safely locate or create it from the saved branch. Block only when repository or requested ref cannot be resolved.

## Procedure

1. Read the task and parse valid `agent_meta`.
   - If task read fails, or metadata is missing/malformed, preserve notes unchanged. Add an English diagnostic comment with `recovery_required: true` only when the task can be addressed, then stop. Do not reconstruct metadata.
2. Continue only for `Ready for work` or `In progress`; otherwise comment in English and stop before Git operations.
3. Increment `attempts.work`. If it exceeds 3, persist `last_error.stage: work`, add an English escalation comment, and stop.
4. Persist `In progress` and a status-history entry before commentary or repository work. Treat every non-empty `failed` response from `asana_update_tasks` as an error.
5. Resolve `source_branch` from input `branch`, otherwise `agent_meta.branch`; if none exists, create `agent/<task-name>`. Persist the chosen branch before Git preparation or downstream actions.
6. In the task worktree, safely synchronize with the repository's integration branch, implement the persisted plan and acceptance criteria, and verify proportionately.
7. Commit only task-scoped source changes to `agent_meta.branch`.
8. On success, describe changed files and verification in metadata, clear `last_error`, transition to `Ready for review`, append history, update notes, then add an English result comment. A failed result comment is reporting-only: do not undo the saved successful transition or retry the work attempt solely for that failure.
9. Invoke `asana-task-review` only after the successful state update.
10. For an execution error after valid metadata was read but before final success, persist `last_error.stage: work`, keep `In progress`, update notes, add an English comment, and stop without review.

## Description updates

For every metadata write, follow the description-preserving fresh-read, metadata-only patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve rich HTML, unknown metadata, and concurrent human edits. On a reported or uncertain task-update failure, reconcile by reading back and stop this invocation before any handoff; do not restore an old full description. Outcome-comment failures do not invalidate a verified transition.
