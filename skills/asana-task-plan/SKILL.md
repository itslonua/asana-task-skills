---
name: asana-task-plan
description: Use when an Asana task needs initial agent_meta or is in Needs estimation or Estimating and needs a repository-grounded implementation plan before work begins.
---

# Asana Task Plan

Initialize and plan one user-selected Asana task. Task notes are the persisted source of truth. Write metadata, status values, and Asana comments in English; communicate with the user in their language.


## Contract

- Required input: `task_gid`. Optional inputs: repository, task branch, base branch, integration branch. Explicit user choices take precedence over defaults.
- Tools: `asana_get_task`, `asana_update_tasks`, `asana_add_comment` (discover their actual names and argument schemas; task reads currently use `task_id`, updates use `tasks[].task`).
- Planning transitions: `Needs estimation → Estimating → Ready for work`. Do not implement, deploy, merge, or mark the task completed during planning.
- A request to initialize, start, or process a specific task authorizes adding missing metadata. It does not authorize replacing malformed metadata or discarding the human description.
- If the user requested end-to-end delivery, invoke installed `asana-task-work` after verified planning success. For a planning-only request, return the saved plan. Creating/installing this skill alone does not invoke it against a live task.

## Preserve the description on every write

1. Read `gid,name,notes,html_notes,modified_at,completed,custom_fields` and relevant comments, dependencies, and subtasks. Save the exact original `notes` and `html_notes` locally or in session storage before writing. A failed read, non-empty `failed`, absent task, or missing `notes` means no description write and no invented `last_error`.
2. Parse exactly one bounded YAML block between literal `<!-- AGENT_META_START -->` and `<!-- AGENT_META_END -->` in plain `notes`. Canonical YAML has an `agent_meta` mapping. Accept an existing flat mapping with `current_status` and preserve its shape; do not add a second block. Reject duplicate/mismatched markers, invalid YAML, or missing status in an existing block. Preserve notes and report `recovery_required: true` with the concrete problem.
3. For rich descriptions, update only `html_notes`. Keep the existing HTML outside the metadata region unchanged, including links, Asana attributes, and embedded media objects. Do not regenerate it from plain `notes` or send both description fields.
4. For first initialization, append a `<pre>` containing XML-escaped marker lines and YAML immediately before `</body>`. Escaped markers must become visible literal marker text in the returned plain `notes`; actual HTML comments can disappear. For subsequent writes replace only the uniquely identified metadata region, preserving its surrounding HTML. If a safe region cannot be located, stop instead of rewriting the description. If `html_notes` is unavailable, obtain it before proceeding when rich content may exist; use `notes` only for confirmed plain text tasks.
5. Re-read immediately before each write. If notes/HTML changed, rebuild the patch from the latest description, preserving others' changes; if metadata changed, re-evaluate status and ownership before proceeding. There is no assumed atomic compare-and-swap in MCP.
6. Check `isError` and every non-empty `failed` result, even when some updates succeeded. On partial or uncertain failure, read back and reconcile this task before any retry; do not increment attempts again or hand off. Stop the invocation on a reported failure, recording the error safely when possible; a later invocation resumes from verified persisted state. Re-read after writing; compare the description outside the metadata region with the latest pre-write snapshot and validate the saved YAML/status. Do not advance on a mismatch. Report a discrepancy without blindly restoring an old full description over concurrent edits.

## Metadata

Preserve unknown keys and existing history. Use actual resolved values, not the example values below. Record branch/worktree facts only after verification; omit unknown values rather than presenting guesses as facts.

```yaml
<!-- AGENT_META_START -->
agent_meta:
  task_gid: "123456789"
  current_status: "Needs estimation"
  repository: "/absolute/path/to/repository"
  branch: "feature/123456789-short-description"
  base_branch: "develop"
  integration_branch: "develop"
  attempts:
    plan: 0
  last_error: null
  history: []
<!-- AGENT_META_END -->
```

The contents between the marker lines are YAML; the marker lines themselves are not YAML. `branch` is the task's implementation branch, `base_branch` is its starting ref, and `integration_branch` is the user's merge target. They need not match. Do not treat Asana's unrelated Branch text field as authoritative over explicit user input or persisted metadata.

## Planning procedure

1. After a successful read, initialize absent metadata with `Needs estimation` when authorized. Never reopen a completed task. For existing metadata continue only from `Needs estimation` or `Estimating`; return other statuses to the orchestrator without resetting progress.
2. Increment `attempts.plan` once per stage invocation. If the increment exceeds 3, persist a plan-stage diagnostic and stop; do not retry indefinitely. Otherwise persist `Estimating` with a UTC history entry (`at`, `from`, `to`, `stage: plan`) before repository investigation. On resumption retain previous findings; do not duplicate initialization.
3. Resolve the repository and refs from explicit input, metadata, then verified repository configuration. Read applicable `AGENTS.md`. Use read-only inspection or the task-owned worktree. Never switch, reset, clean, stash, or edit another task's checkout. If checkout is needed, locate or safely create the isolated worktree from the saved base ref; persist its verified absolute path. A dirty caller checkout is not a blocker. Honor an explicit branch prefix choice; otherwise follow project conventions. Do not invent a missing ref or silently fall back to a different base.
4. Read the ticket's requirements and relevant repository code/tests. Identify current versus desired behavior, affected components, dependencies, and risks. Distinguish verified facts from assumptions. Ask only questions whose answers materially change scope or correctness, continuing independent investigation while waiting. Missing material requirements keep the task in `Estimating`.
5. Persist a `plan` mapping containing `summary`, `scope`, `steps`, `acceptance_criteria`, `verification`, `risks`, and `open_questions`. Steps must identify concrete components/files and dependencies. Acceptance criteria describe observable outcomes; verification names relevant commands or manual checks and expected outcomes. Include an effort estimate only if requested or required by the project, with its assumptions. Do not claim planning-time checks prove unimplemented behavior.
6. When the plan is actionable and material open questions are resolved, clear plan-stage `last_error`, transition to `Ready for work`, append history, and verify the saved update. Do not erase unrelated unresolved stage errors; investigate them before transition. If requested, hand off the same `task_gid`, branch, and persisted metadata to `asana-task-work`. Do not call unavailable downstream skills or invent their behavior.

## Errors and reporting

- After valid metadata has been loaded, a planning failure keeps the current stage status and records `last_error: {stage: plan, message: ...}` only if a fresh read and safe metadata-only write remain possible. Never create notes to report a read failure.
- Add an English diagnostic or planning-context comment when permitted by the connector. Use comments for findings, assumptions, questions, or recovery context, not duplicate automatic field/status logs. Comment failure must not roll back or rerun a verified successful transition.
- Do not infer Asana custom status-field IDs/options from status names. `agent_meta.current_status` controls this workflow; update custom fields only when their mapping is verified and authorized.
- Before handoff verify: exactly one parseable metadata block, human text and rich content preserved, repository/refs resolved, actionable plan and checks recorded, correct target branch retained, and no claim that implementation or merge already happened.
