---
name: codex-cli-runtime
description: Delegate a substantial coding, debugging, or investigation task to Codex as a Claude background task that notifies this session with Codex's response when Codex finishes. Use when Claude Code is stuck, wants a second implementation or diagnosis pass, or should hand off long-running work.
user-invocable: false
---

# Codex Runtime

Run Codex through the companion `task` command as a Claude Code background Bash task, from the main thread. Do not wrap it in a subagent.

```typescript
Bash({
  command: `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task --write [--resume-last] [--sandbox <mode>] [--model <model>] [--effort <effort>] <<'CODEX_TASK'
<task text>
CODEX_TASK`,
  description: "Codex: <short task label>",
  run_in_background: true
})
```

How it behaves:
- The process exits when Codex ends its turn and prints Codex's final response. Claude Code then notifies this session; read the task output at that point.
- `TaskStop` on the background task stops Codex and every command Codex started. `/codex:cancel <job-id>` does the same from anywhere.
- Progress while it runs: `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" status <job-id>`. Only check when the user asks; do not poll.
- Several tasks can run at once; each owns its own Codex process.

Rules:
- Do not pass `--background` to `task`. That detaches a worker Claude Code cannot see, so no completion notification arrives.
- Send the task text on stdin with a quoted heredoc, always. Without stdin input the process can wait on an open stdin forever.
- Add `--write` unless the work must not edit files; then leave it out and say "Do not modify any files." in the task text.
- Leave `--sandbox` unset unless the user asks; Codex then uses the sandbox from the user's Codex config.
- Leave `--model` and `--effort` unset unless the user asks. Map `spark` to `--model gpt-5.3-codex-spark`. `--effort` accepts `none`, `minimal`, `low`, `medium`, `high`, `xhigh`.
- If the repository root has `.codex-delegate.md`, its text is prepended to every new task prompt automatically; do not repeat those rules in the task text. `--no-preamble` skips it.
- `--output <file>` also saves Codex's final answer alone to that file (like `codex exec -o`). Pick a path no other agent writes to.
- Use `--resume-last` to continue the latest Codex task thread in this repository ("keep going", "apply the top fix", "dig deeper").
- You may use the `gpt-5-4-prompting` skill to tighten the task text. Do not solve the task yourself first.
- When the result arrives, return Codex's final response without rewriting it. If the run failed, report the actionable error instead of redoing the work on the Claude side.
