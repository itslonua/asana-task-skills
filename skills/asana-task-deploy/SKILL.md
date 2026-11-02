---
name: asana-task-deploy
description: Use when an initialized Asana task is Approved or Deploying and needs deployment of its reviewed commit to a configured verification environment.
---

# Asana Task Deploy

Deploy the reviewed task commit to the project's configured verification environment and hand its verified deployment to QA. This stage does not merge the task branch or mark the task completed.

## Persisted state and isolation

- Required input: `task_gid`. Read the task with its notes, rich description, completion flag, and relevant comments. Discover actual Asana tool schemas; do not assume argument names.
- Parse exactly one YAML `agent_meta` block bounded by `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->`. On failed reads or malformed metadata, preserve the description and stop; report `recovery_required: true` when the task is addressable.
- Follow the description-preserving read, patch, and read-back procedure in [asana-task-plan](../asana-task-plan/SKILL.md). Preserve unknown metadata, history, human text, rich content, and concurrent edits. Treat `isError` and every non-empty `failed` as failure. Reconcile uncertain writes before retrying or handing off.
- Use only the task-owned worktree and persisted branch. Never switch, reset, clean, stash, or commit another checkout. A dirty caller checkout is not a blocker.
- Write metadata, statuses, and Asana comments in English. Use UTC history entries with `at`, `from`, `to`, and `stage`.
- Increment the stage attempt counter once per invocation that enters stage execution after its prerequisites are resolved. If the new count exceeds 3, persist a stage diagnostic and stop. A verified transition must not be undone or repeated solely because a reporting comment failed. A failed outcome comment does not block a handoff after a verified state transition; description-update failures always require reconciliation and stop this invocation.

## Procedure

1. Continue only from `Approved` or `Deploying`. For other states, report the persisted state and stop. Never reopen a completed task.
2. Resolve the reviewed commit, saved branch, repository deployment instructions, target environment, existing deployment evidence, and required authorization. The source commit must match the approved review evidence. Changed code returns to `Ready for review` with a diagnostic and invokes `asana-task-review` after a verified write.
3. Use the deployment method and environment configured by the project or explicitly selected by the user. Missing target configuration or authorization keeps the current state with `last_error.stage: deploy`; state the exact missing input. End-to-end delivery does not by itself select production or authorize new infrastructure, destructive migrations, new access, or paid resources.
4. Increment `attempts.deploy`, persist `Deploying`, and append history before external mutations. Reuse a running or successful deployment for the same commit and target. For uncertain prior outcomes, inspect the provider before retrying; avoid duplicate releases.
5. Build and deploy the approved commit using the documented project commands or provider workflow. Scope temporary containers and local resources to this task. Clean up only temporary resources in unconditional cleanup; retain the deployed verification target for QA. Do not stop or overwrite another task's environment.
6. Wait for the provider's terminal result and run the configured health/smoke checks. A submitted command or successful build alone is not deployment success. Record `deployment` evidence: commit, environment, provider run ID/URL or local endpoint, commands/checks and outcomes, timestamp, and cleanup/rollback notes where applicable. Keep credentials out of notes and comments.
7. On confirmed deployment or smoke-check failure, record `last_error.stage: deploy` and the evidence. If implementation changes are required, transition to `Ready for work`, verify the write, comment, and invoke `asana-task-work`. For infrastructure, permissions, or uncertain provider outcomes, retain `Deploying` and stop with the specific diagnostic; do not create an automatic implementation retry loop. Use a rollback only when the project has a documented, authorized procedure for this deployment.
8. On verified success, clear only resolved deployment errors, transition to `Ready for testing`, append history, and verify the saved state and deployment commit. Add an English outcome comment and invoke `asana-task-test` with the same task ID and verification endpoint. QA must test this deployed commit; a newer branch head requires another review and deployment.
