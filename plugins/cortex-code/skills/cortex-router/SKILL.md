---
name: cortex-router
description: "Auto-routing skill loaded by the prompt filter hook. Routes Snowflake-related operations to Cortex Code CLI. Not for direct invocation — use $cortex-run instead."
license: Proprietary. See LICENSE-SKILLS.md for complete terms
user-invocable: false
context: fork
model: haiku
effort: low
metadata:
  author: Snowflake Integration Team
  version: 3.5.0
  compatibility: Requires Cortex Code CLI installed and configured
---

# Cortex Code Router

You run in a forked subagent. Route one request to Cortex Code CLI and return what Cortex returned.

The request:

$ARGUMENTS

It may start with tags:
- `[RW]`: the user asked for a change (DDL or DML)
- `[resume]`: it continues the previous Cortex turn
- `[connection=NAME]`: the user named a Snowflake connection

Strip the tags. The rest is `USER_PROMPT`; pass it shell-escaped in both commands below.

If either command fails, reply with the error and stop. Never bypass a configuration or policy error. If the error says `cortex` is not found, tell the caller to load the `snowflake-cortex-code:cortex-setup` skill.

## Step 1: Confirm routing

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/run_python.sh" "${CLAUDE_PLUGIN_ROOT}/scripts/router/route_request.py" --prompt "USER_PROMPT"
```

- **route: claude** → reply with exactly `route: claude` and stop. The caller handles the request.
- **route: cortex** → go to Step 2.

## Step 2: Execute via Cortex Code

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/run_python.sh" "${CLAUDE_PLUGIN_ROOT}/scripts/router/execute_cortex.py" \
  --prompt "USER_PROMPT" \
  --envelope "RO"
```

- **Envelope**: `RO` unless the request has `[RW]`, then `--envelope "RW"`. RO refuses DDL and DML, so a change the user did not tag fails safe and the caller can re-ask with `[RW]`. RESEARCH and DEPLOY stay available through `$cortex-run`.
- **`[resume]`** → add `--resume-last`, so Cortex resumes its last session.
- **`[connection=NAME]`** → add `--connection NAME`.
- **Timeout**: the command takes 30–90 seconds. Past 2 minutes it has likely hung: kill it and reply that the user can try `$cortex-run`.

Reply with Cortex's result: SQL results as tables, any generated artifacts, and one line saying whether it succeeded. Report only what Cortex returned.
