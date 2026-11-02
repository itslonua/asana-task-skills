# Asana Task Skills

A collection of Codex skills for planning and delivering Asana tasks using persisted task metadata and isolated Git worktrees.

## Skills

| Skill | Purpose |
| --- | --- |
| [asana-task-plan](skills/asana-task-plan/SKILL.md) | Initialize task metadata and save a repository-grounded implementation plan. |
| [asana-task-orchestrator](skills/asana-task-orchestrator/SKILL.md) | Route an initialized task to its next delivery stage. |
| [asana-task-work](skills/asana-task-work/SKILL.md) | Implement and verify the plan in the task worktree. |
| [asana-task-review](skills/asana-task-review/SKILL.md) | Review task changes and acceptance criteria before approval. |
| [asana-task-deploy](skills/asana-task-deploy/SKILL.md) | Deploy the approved commit to a configured verification environment. |
| [asana-task-test](skills/asana-task-test/SKILL.md) | Run task-scoped QA and record its outcome. |
| [asana-task-mr](skills/asana-task-mr/SKILL.md) | Manage the merge request, CI, merge, and task completion. |

## Requirements

- Codex with skill support and an Asana connector providing task reads, updates, and comments. Discover actual tool names and schemas in your environment.
- Access to the target Git repository, its configured Git provider, and the project's verification tools.
- A project-configured verification environment and deployment procedure for the deployment and QA stages. Production access and infrastructure changes require their own applicable authorization.

## Installation

Clone this repository, then copy the individual skill directories into your Codex skills directory. Run from the repository root:

```sh
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
for skill in skills/asana-task-*; do
  target="${CODEX_HOME:-$HOME/.codex}/skills/$(basename "$skill")"
  if [ -e "$target" ]; then
    echo "Already exists; skipped: $target"
  else
    cp -R "$skill" "$target"
  fi
done
```

Existing installed skills are skipped. Review and update them explicitly when upgrading.

## Usage

Ask Codex to use `asana-task-plan` with an Asana task ID to initialize or plan a task. For a task with valid existing `agent_meta`, ask it to use `asana-task-orchestrator` with the same task ID.

Example:

```text
Use asana-task-plan to plan Asana task <task_gid> in <repository_path>.
```

Task notes hold a bounded YAML `agent_meta` block. The skills preserve the human description, use English for persisted metadata and Asana comments, and isolate repository work per task.

## Workflow

```text
Needs estimation → Estimating → Ready for work → In progress
→ Ready for review → In review → Approved → Deploying
→ Ready for testing → Testing → Ready for release → Completed
```

The collection includes the five original stages plus review and deployment. QA and merge checks require evidence for the same reviewed, deployed, and tested commit. This repository packages instructions; it does not include an Asana integration server or an executable workflow runner. No live Asana task was changed while packaging it.
