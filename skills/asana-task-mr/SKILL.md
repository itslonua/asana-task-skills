---
name: asana-task-mr
description: Use when an initialized Asana task is Ready for release and needs a task-scoped merge request, CI confirmation, and merge lifecycle.
---

# Asana Task MR

Create or complete the merge-request lifecycle for one initialized Asana task. Persist all state transitions and comments in English.

## Contract

- Input: required `task_gid`; optional `branch`, otherwise use `agent_meta.branch`.
- Success: `agent_meta.current_status: Completed` and Asana `completed: true`.
- Failure: `agent_meta.current_status: Ready for work`, then invoke `asana-task-work`.
- Metadata: YAML `agent_meta` between `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->` in task notes.
- Required Asana tools: `asana_get_task`, `asana_update_tasks`, and `asana_add_comment`.

## State machine

`Needs estimation → Estimating → Ready for work → In progress → Ready for review → In review → Approved → Deploying → Ready for testing → Testing → Ready for release → Completed`

## Worktree isolation

- Create and inspect the MR only from the task's dedicated worktree and saved branch.
- Never switch, reset, clean, stash, or mutate another checkout. A dirty checkout, different branch, or concurrent task is not a blocker.
- Coordinate using persisted Asana metadata, never local checkout ownership.

## Procedure

1. Read the task and parse valid `agent_meta`. If it is missing or malformed, preserve notes, add an English `recovery_required: true` diagnostic comment, and stop.
2. If status is `Completed`, set Asana `completed: true` only if needed, then stop without creating, waiting for, or merging an MR. If status is not `Ready for release`, comment in English and stop.
3. Resolve and persist `source_branch` from input or metadata. If absent, set `last_error.stage: mr`, transition to `Ready for work`, update notes, comment in English, and invoke `asana-task-work`.
4. Before creating anything, inspect the existing MR for `source_branch` through the repository's configured Git provider workflow.
   - If merged, clear `last_error`, transition to `Completed`, persist notes and `completed: true`, comment in English, and stop.
   - If open, reuse it.
   - If absent, create one from `source_branch` into the repository's integration branch with summary, acceptance criteria, and verification evidence. Use the configured assignee/reviewer policy.
5. Verify that the MR source head equals the commit recorded in successful `review`, `deployment`, and `testing` evidence. If evidence is absent, retain `Ready for release` with a diagnostic and stop. If the source changed, transition to `Ready for review`, verify the update, and invoke `asana-task-review`; do not merge unreviewed changes. Wait for the MR's CI pipeline for that exact commit using the configured provider workflow.
   - On CI failure, persist `last_error.stage: mr`, transition to `Ready for work`, update notes, add an English comment, and invoke `asana-task-work`.
6. On CI success, re-read the MR source head and merge only the verified commit, using the provider's expected-head SHA guard. If the provider cannot guard the merge against concurrent source changes, stop with a diagnostic. If the source has changed, return to review as in step 5. Merge the MR if it is not already merged. Clear `last_error`, transition to `Completed`, append history, persist notes and `completed: true`, verify the Asana update has no non-empty `failed`, and add an English completion comment.

## Description updates

For every metadata write, follow the description-preserving fresh-read, metadata-only patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve rich HTML, unknown metadata, and concurrent human edits. On a reported or uncertain task-update failure, reconcile by reading back and stop this invocation before any handoff; do not restore an old full description. Outcome-comment failures do not invalidate a verified transition.
