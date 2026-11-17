# Phase 0 findings (Claude Code 2.1.317, captured 2026-11-03)

Raw samples are in `payloads/` (paths anonymised, long text trimmed).

## What differs from the spec

1. **The task tool is `TaskCreate` / `TaskUpdate`, not `TodoWrite`.** Calls are incremental.
   - `TaskCreate` input: `{ subject, description, activeForm }`. **No id in the input.** The id only
     appears in the `PostToolUse` response: `tool_response.task.id` (`"1"`, `"2"`, ...).
     So the task map must be built from `PostToolUse`, never `PreToolUse`.
   - `TaskUpdate` input: `{ taskId, status }` (also `subject`/`activeForm` when renamed, `status: "deleted"`
     to remove). Response: `{ success, taskId, updatedFields, statusChange: { from, to } }`.
   - The parser still accepts `TodoWrite` (`todos: [{ content, status, activeForm }]`, full list) for older versions.
2. **Scope baseline must be provisional.** Tasks arrive one `TaskCreate` at a time, so locking the baseline on
   the first task showed "+9 tasks" while Claude was still writing its plan. The baseline now follows the total
   until the first status change or the first non-task tool call.
3. Extra common fields: `prompt_id`, `permission_mode`, `effort`, `scratchpad_dir`. `PostToolUse` has `duration_ms`.
   `Stop` has `stop_hook_active`, `last_assistant_message`, `background_tasks`. `SessionEnd` has `reason`.
4. A new phase does not clear Claude's task list (old completed tasks stay in the session). On a phase change the
   hook hides tasks that were already completed, so Phase 2 does not open at "6/11".

## Notification (captured 2026-11-17 from an interactive session)

```json
{ "hook_event_name": "Notification", "message": "Claude needs your permission", "notification_type": "permission_prompt" }
```

- `notification_type` exists, so the hook uses it first (`permission_prompt`, `idle_prompt`, `elicitation_dialog`)
  and only falls back to matching text in `message` when it is missing.
- **The message does not name the tool** (the spec expected "...permission to use Bash"). The widget takes the
  tool name from `runningTool`, which `PreToolUse` set just before the prompt.
- The event arrived about 6 s after `PreToolUse`: Claude Code waits a moment before it notifies. The hook
  itself adds about 40 ms, so the widget turns amber well within 1 s of the notification.

## Token cost of the meter

Hooks cost zero tokens: they run outside the model and print nothing (stdout from `SessionStart` and
`UserPromptSubmit` hooks is injected into Claude's context, which is why the hook must stay silent).
The only token cost is Claude's own task calls plus the CLAUDE.md snippet:

| Run | Task tool calls per task |
|---|---|
| Phase 0, no guidance (create, in_progress, completed) | 3 |
| With Needle's snippet (create, completed) | 2 |

The hook infers "current task" from the next pending one, so `in_progress` calls are unnecessary.
