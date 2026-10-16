---
description: Delegate investigation, an explicit fix request, or follow-up rescue work to Codex as a Claude background task
argument-hint: "[--resume|--fresh] [--read-only] [--sandbox <read-only|workspace-write|danger-full-access>] [--model <model|spark>] [--effort <none|minimal|low|medium|high|xhigh>] [--output <file>] [--no-preamble] [what Codex should investigate, solve, or continue]"
allowed-tools: Bash(node:*), AskUserQuestion
---

Hand the request to Codex by running the companion `task` command yourself as a Claude Code background Bash task. Do not use the `Agent` tool and do not call `Skill(codex:rescue)` (that re-enters this command).

Raw user request:
$ARGUMENTS

Why a background Bash task:
- The `task` process runs until Codex finishes its turn, then exits and prints Codex's final response.
- Claude Code notifies this session when the process exits, with that response. Nobody has to poll.
- Stopping the background task (`TaskStop`) stops Codex and every command it started, because Codex runs inside that process tree.

Thread choice:
- If the request includes `--resume` or `--fresh`, do not ask. The user already chose.
- Otherwise run:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task-resume-candidate --json
```

- If it reports `available: true`, use `AskUserQuestion` exactly once with these two choices:
  - `Continue current Codex thread`
  - `Start a new Codex thread`
- Put `Continue current Codex thread (Recommended)` first when the user is clearly following up ("continue", "keep going", "apply the top fix", "dig deeper"); otherwise put `Start a new Codex thread (Recommended)` first.
- Continue means add `--resume-last`. A new thread means no `--resume-last`.
- If it reports `available: false`, do not ask.

Building the command:
- Add `--write` unless the user asks for read-only behavior (`--read-only`) or only wants review, diagnosis, or research without edits. In that case leave out `--write` and start the task text with "Do not modify any files." (the sandbox follows the user's Codex config, so this is what keeps the run read-only).
- Pass `--sandbox`, `--model`, `--effort`, `--output <file>`, and `--no-preamble` through only when the user gave them. Map `spark` to `gpt-5.3-codex-spark`. Without `--sandbox`, Codex uses the sandbox from the user's Codex config.
- Strip `--resume`, `--fresh`, `--read-only`, and the flags above from the task text. Keep the rest of the user's text as-is.
- Always send the task text on stdin with a quoted heredoc so quoting cannot break it and the process never waits on an open stdin.

Launch exactly one background Bash call:

```typescript
Bash({
  command: `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --write [flags] <<'CODEX_TASK'
<task text>
CODEX_TASK`,
  description: "Codex: <short task label>",
  run_in_background: true
})
```

After launching:
- Tell the user in one line that Codex is running in the background and give the background task id. Do not wait, poll, or read its output in this turn.
- When the completion notification arrives, read the task output and return Codex's final response verbatim. Do not paraphrase it or redo the work yourself.
- If the run failed or Codex could not be invoked, report the actionable error and stop. If it says setup or authentication is required, point the user to `/codex:setup`.
- To stop the run, use `TaskStop` on that background task (or `/codex:cancel <job-id>`).
- If the user did not supply a request, ask what Codex should investigate or fix.
