# Workflow State & Resume Guidelines

This repository now supports resumable workflows for `/dev`, `/bmad-pilot`, and `/requirements-pilot`. State is persisted per feature to allow long-running development to continue across sessions.

## State Location
- Base directory: `.claude/state/`
- Per feature and workflow: `.claude/state/{feature_slug}/{workflow}.json`
  - `feature_slug`: kebab-case of the user request (lowercase, spaces/punctuation → `-`, collapse repeats, trim edges)
  - `workflow`: `dev`, `bmad`, or `requirements`

## State Schema (superset)
```json
{
  "workflow": "dev|bmad|requirements",
  "feature": "user-login",
  "phase": "reqs|analysis|plan|dev|review|test|qa|done",
  "step": "in_progress|waiting_user|blocked|done",
  "options": {"skip-tests": false, "direct-dev": false, "skip-scan": false},
  "artifacts": [
    {"name": "00-repo-scan", "path": ".claude/specs/user-login/00-repo-scan.md", "status": "done"}
  ],
  "tasks": [
    {
      "id": "task-backend-api",
      "agent": "codex",
      "status": "in_progress",
      "codex_session": "019a...",
      "scope": "backend/api",
      "test_cmd": "npm test -- login",
      "coverage": null,
      "error": null
    }
  ],
  "created_at": "...",
  "updated_at": "..."
}
```
- Workflows may omit unused fields; add fields as needed.

## Save Rules
- Save after each meaningful transition: end of a phase, entering a user approval gate, launching or finishing a Codex task, writing artifacts, or finishing the workflow.
- For question/approval gates, set `step=waiting_user` and only advance after a user response.
- For Codex tasks, store returned `SESSION_ID` as `codex_session` to allow resume with `codex-wrapper resume`.
- Include coverage or errors on task completion/failure.
- Use atomic writes: write to a temp file in the same directory then rename to `{workflow}.json`.

## Resume Rules
- CLI options add `--resume <feature_slug>` to load state; if missing, list available slugs under `.claude/state/`.
- Validate that referenced artifacts exist; if missing, set `step=blocked` and ask whether to regenerate or stop.
- Jump directly to the recorded `phase`/`step`, skipping completed work.
- When resuming Codex tasks with a stored `codex_session`, call `codex-wrapper resume <session_id> ...`; if a session fails twice, start a new session and record the fallback reason.
- Do not overwrite existing state on resume until new progress is made.

## Cleanup
- Keep state files for audit after completion (`phase=done`).
- Optional flags like `--reset` may remove/refresh state (only on explicit user confirmation).
