---
name: asana-task-orchestrator
description: Use when continuing an initialized Asana task that contains agent_meta and must be routed to its next delivery stage.
---

# Asana Task Orchestrator

Route one initialized Asana task through its persisted delivery state machine. The task notes are the source of truth; local checkout state must never substitute for them.

## Contract

- Input: required `task_gid`.
- Persisted state: the YAML `agent_meta` block between `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->` in task notes.
- Asana tools: `asana_get_task`, `asana_update_tasks`, and `asana_add_comment`.
- Write status values, `agent_meta`, and comments in English.

## State machine

`Needs estimation → Estimating → Ready for work → In progress → Ready for review → In review → Approved → Deploying → Ready for testing → Testing → Ready for release → Completed`

## Worktree isolation

- Each Asana task owns an isolated Git worktree.
- Never switch, reset, clean, stash, or otherwise mutate another task's current checkout.
- A dirty checkout, different branch, or concurrent task is not a blocker. Resolve the repository and branch stored in `agent_meta` in the task's worktree.
- Pass the same `task_gid` and persisted `agent_meta` to downstream stages. Do not use local checkout state as a cross-task lock.
- If the required worktree is missing, safely locate or create it from the saved source branch. Block only when the repository or ref cannot be resolved.

## Routing procedure

1. Read the task using `asana_get_task(task_gid)`.
   - If MCP read fails, returns a non-empty `failed`, cannot find the task, or does not expose `notes`, do not create or overwrite notes and do not persist `last_error`.
   - Add a failure comment only when a confirmed task ID is available; otherwise stop.
2. Parse the bounded `agent_meta` YAML block. It must include `current_status`.
   - If the block is missing, malformed, non-YAML, or lacks `current_status`, preserve all notes unchanged. Add an English comment with `recovery_required: true` only if the task ID is available, explaining that a new task must first be initialized via `asana-task-plan`. Stop without guessing a state or invoking a child skill.
3. If `last_error` is populated and the matching stage attempt count exceeds 3, add an English comment and stop.
4. Route by `current_status`:

| Status | Next skill |
|---|---|
| `Needs estimation`, `Estimating` | `asana-task-plan` |
| `Ready for work`, `In progress` | `asana-task-work` |
| `Ready for review`, `In review` | `asana-task-review` |
| `Approved`, `Deploying` | `asana-task-deploy` |
| `Ready for testing`, `Testing` | `asana-task-test` |
| `Ready for release` | `asana-task-mr` |
| `Completed` | Set Asana `completed: true` only when it is not already true; otherwise do nothing. Never rerun MR/merge. |

5. For an unknown status, set `last_error.stage: orchestrator` and `last_error.message: "Unknown status: <current_status>"`, update notes, add an English comment, then stop.
6. After a child skill runs, do not change state yourself. The child owns metadata, history, comments, and transitions.
7. After successful read and parsing, treat any later non-empty Asana MCP `failed` result as an error: persist an orchestrator `last_error`, update notes first, then add an English comment and stop.

## Description updates

For every metadata write, follow the description-preserving fresh-read, metadata-only patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve rich HTML, unknown metadata, and concurrent human edits. On a reported or uncertain task-update failure, reconcile by reading back and stop this invocation before any handoff; do not restore an old full description. Outcome-comment failures do not invalidate a verified transition.
