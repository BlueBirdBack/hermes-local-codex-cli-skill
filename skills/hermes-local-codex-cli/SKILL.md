---
name: hermes-local-codex-cli
description: Use Hermes to orchestrate the local Codex CLI as an external worker, with separate Codex auth and optional separate proxy/Clash routing from Hermes itself.
version: 1.0.0
author: Ash
license: MIT
metadata:
  hermes:
    category: autonomous-ai-agents
    tags: [hermes, codex, orchestration, proxy, clash, oauth, subprocess]
    related_skills: [codex, hermes-agent]
---

# Hermes Orchestrating Local Codex CLI

Use this when the user wants **Hermes to run the installed `codex` binary** on the same machine, rather than using Hermes's built-in `openai-codex` provider.

For a fuller user-facing guide, load `skill_view("hermes-local-codex-cli", "references/orchestration-guide.md")` if the installed copy includes it.
For exact versions and validation commands, load `skill_view("hermes-local-codex-cli", "references/verification.md")`.

## Source roles

- `README.md` in the standalone repo: install-focused
- `SKILL.md`: canonical operational instructions for the agent
- `references/orchestration-guide.md`: fuller user-facing explanation and examples
- `references/verification.md`: last validated versions and commands checked

Keep the operational rules in `SKILL.md` first; references should extend or verify them, not redefine them.

## When this matters

Choose this path when the user wants one or more of:

- a **different Codex account** than Hermes's main provider/account
- a **different IP / proxy route** for Codex than for Hermes
- **Clash-only routing** for the Codex subprocess
- Hermes to stay the planner/supervisor while Codex CLI acts as a focused local worker

## Core distinction

There are two different Codex paths:

1. **Hermes built-in provider**
   - provider name: `openai-codex`
   - auth lives in `~/.hermes/auth.json`
   - network path is Hermes's own process/runtime

2. **Local Codex CLI subprocess**
   - command: `codex ...`
   - auth lives in `~/.codex/auth.json`
   - network path is whatever env/proxy the subprocess receives

Do not conflate them.

## Rule Zero

Keep the **main Hermes session responsive**.

- Main session: conversation, one quick probe, task framing, launching Codex, monitoring, reporting
- Codex worker path: coding, multi-file edits, tests/builds, repo work likely to branch, long-running execution

If the task needs writes, builds/tests, multi-file coding, more than one quick probe, or more than ~30 seconds of sustained execution, hand it off to Codex instead of letting Hermes drift inline.

**Spawn → Monitor → Report.** Launching Codex is not enough — tell the user what is running and what event will trigger the next update.

## Task Shape Gate

Before Hermes calls local Codex CLI, check:

- Any write / build / deploy / repo work likely to branch? → use Codex
- Coding task at all? → prefer Codex over inline drift
- Already at tool call #3 inline? → stop and hand off
- Need another real probe? → stop and hand off
- No real timed delivery loop? → promise event-based updates only

## Prerequisite checks

Before promising anything, verify all of these with tools:

```bash
command -v codex
codex --version
codex login status
```

If repo-local work is requested, also verify the target directory:

```bash
git -C /path/to/repo rev-parse --is-inside-work-tree
```

## 60-second smoke test

For a first-run validation, use a fast read-only check before any real repo work:

```bash
command -v codex
codex --version
codex login status
codex exec -C /path/to/repo -s read-only "Reply with exactly: LOCAL_CODEX_OK plus the current working directory."
```

Expected signals:

- `codex login status` succeeds
- Codex runs in the requested directory
- the final reply proves Hermes used the local Codex CLI path, not just its built-in provider path

For a proxy-routed validation, use the **same env on both auth and execution**, but replace the placeholders with the actual Clash/Mihomo proxy URLs on the machine:

```bash
export LOCAL_HTTP_PROXY=http://127.0.0.1:<http-port>
export LOCAL_SOCKS5_PROXY=socks5://127.0.0.1:<socks-port>

HTTPS_PROXY="$LOCAL_HTTP_PROXY" \
HTTP_PROXY="$LOCAL_HTTP_PROXY" \
ALL_PROXY="$LOCAL_SOCKS5_PROXY" \
NO_PROXY=localhost,127.0.0.1 \
codex login status

HTTPS_PROXY="$LOCAL_HTTP_PROXY" \
HTTP_PROXY="$LOCAL_HTTP_PROXY" \
ALL_PROXY="$LOCAL_SOCKS5_PROXY" \
NO_PROXY=localhost,127.0.0.1 \
codex exec -C /path/to/repo -s read-only "Reply with exactly: PROXY_OK plus the current working directory."
```

## Execution pattern

Run Codex through Hermes terminal calls with `pty=true`.

### Safe read-only probe

```bash
codex exec -s read-only "Summarize the auth flow in this repo"
```

### Review mode

```bash
codex review --base origin/main
```

### Implementation mode

```bash
codex exec --full-auto "Implement X. Do not commit or push. Run relevant tests."
```

Prefer read-only first if the user is asking whether it works.

## Choosing Codex effort

Use the smallest effort that matches the task:

- `medium` → deterministic reads, grep, git log, config lookup, read-only repo summaries
- `high` → normal bug fixes, feature work, code review, standard repo operations
- `xhigh` → proving a bug, deep refactors, tricky multi-file reasoning

**Default:** `high`

**Do not use `xhigh` for reads.** If Hermes is only asking Codex to inspect files or summarize code, prefer `medium`.

Examples:

```bash
codex exec -c model_reasoning_effort="medium" -s read-only "Summarize the config flow"
codex exec -c model_reasoning_effort="high" --full-auto "Implement the fix and run the targeted tests"
codex exec -c model_reasoning_effort="xhigh" --full-auto "Prove whether this race condition is real, then fix it"
```

## Separate account handling

If the user wants Codex under a different account than Hermes:

- do **not** switch Hermes provider config
- re-auth the local Codex CLI instead:

```bash
codex logout
codex login
codex login status
```

Be explicit in user-facing wording:

- “Use the local Codex CLI, not Hermes's built-in Codex provider.”

## Separate proxy / Clash routing

To route only Codex through Clash or a different egress path, set proxy env vars **on the Codex subprocess only**.

Do **not** copy a port blindly. Use the actual local proxy URLs configured on that machine.

Common pattern:

- `HTTP_PROXY` / `HTTPS_PROXY` → `http://127.0.0.1:<http-port>`
- `ALL_PROXY` → `socks5://127.0.0.1:<socks-port>`

Example with placeholders:

```bash
export LOCAL_HTTP_PROXY=http://127.0.0.1:<http-port>
export LOCAL_SOCKS5_PROXY=socks5://127.0.0.1:<socks-port>

HTTPS_PROXY="$LOCAL_HTTP_PROXY" \
HTTP_PROXY="$LOCAL_HTTP_PROXY" \
ALL_PROXY="$LOCAL_SOCKS5_PROXY" \
NO_PROXY=localhost,127.0.0.1 \
codex exec -s read-only "Summarize this repo"
```

Important finding:

- prefer `socks5://`, not `socks://`
- Hermes code normalizes generic proxy env vars to canonical forms because Python HTTP stacks often reject bare `socks://`

So when documenting or running Clash-style proxy commands, use the actual machine-specific proxy URLs, and prefer `socks5://` for the SOCKS endpoint.

## Current CLI reality observed

Verified on `codex-cli 0.118.0`:

- `codex login status` exists
- `codex exec --help` shows `--skip-git-repo-check`
- a git repo is still preferred for repo work, but not always mandatory

## Failure mode to recognize immediately

If Codex starts but fails with errors like:

- `refresh_token_reused`
- `token_expired`
- repeated `401 Unauthorized` from `chatgpt.com/backend-api/codex`

then the local Codex CLI login is stale. Fix with:

```bash
codex logout
codex login
codex login status
```

Do not waste turns debugging the repo before fixing auth.

## Recommended Hermes prompt patterns

### Read-only validation

“Use the local Codex CLI in /path/to/repo. Run `codex login status`, then `codex exec -s read-only ...`, and return the result only.”

### Separate-IP Codex execution

“Use the local Codex CLI in /path/to/repo. Route only the Codex subprocess through the machine's actual Clash/Mihomo proxy URLs — `HTTP_PROXY`/`HTTPS_PROXY` as `http://127.0.0.1:<http-port>` and `ALL_PROXY` as `socks5://127.0.0.1:<socks-port>`. Keep Hermes on its current provider/network path.”

### Separate-account PR review

“Use the local Codex CLI, not Hermes's built-in Codex provider. Run `codex login status`, then `codex review --base origin/main`, and summarize findings.”

## Reporting guidance

Do not promise timed progress updates unless a real delivery loop exists. For this skill, the safe default is event-based reporting:

- done
- blocked
- or a named checkpoint

A good supervisor message shape is:

- what Codex is running
- why it was handed off
- what event will trigger the next update

When successful, report:

- that Hermes used the **local Codex CLI**
- working directory used
- whether run was read-only / review / full-auto
- whether a separate proxy/env path was used

When blocked, report the exact auth or proxy failure, not a vague “Codex failed.”
