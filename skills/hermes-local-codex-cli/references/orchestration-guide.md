---
sidebar_position: 16
title: "Hermes Orchestrating Local Codex CLI"
description: "Use Hermes as the conductor and the local Codex CLI as a worker — with separate auth, separate proxy/Clash routing, and repo-local execution."
---

# Hermes Orchestrating Local Codex CLI

Hermes can orchestrate the **local `codex` CLI** as an external worker.

This is **not** the same thing as setting Hermes's own provider to `openai-codex`.

| Path | What it is | Auth state | Network path |
|------|------------|------------|--------------|
| Hermes provider = `openai-codex` | Hermes talks to Codex through its built-in provider runtime | `~/.hermes/auth.json` | Hermes process |
| Hermes shells out to local `codex` CLI | Hermes uses the installed `codex` binary as a subprocess | `~/.codex/auth.json` | Whatever env/proxy the `codex` subprocess gets |

The built-in provider path is documented in [Providers](/docs/integrations/providers). This guide is about the **second path**: Hermes as conductor, local Codex CLI as worker.

## Source roles

- `SKILL.md` is the canonical operational source for the installed skill
- this guide is the fuller user-facing explanation and example set
- `references/verification.md` records the last validated Hermes/Codex versions and commands checked

If guidance ever diverges, update `SKILL.md` first, then bring this guide back into sync.

---

## When to Use This Pattern

Use local Codex CLI orchestration when you want one or more of these:

- **A different account** from Hermes's main model/provider
- **A different IP / egress path** for Codex than for Hermes itself
- **Clash or proxy-based routing** only for the Codex subprocess
- **Repo-local worker behavior** with `codex exec` or `codex review`
- Hermes to stay conversational, memoryful, and multi-tool while Codex handles focused code work

If you just want Hermes itself to use Codex as its main model, use `hermes model` and choose `openai-codex` instead.

---

## Prerequisites

- Hermes installed
- Terminal tool enabled in Hermes
- Local Codex CLI installed
- Local Codex CLI authenticated
- Target repo or working directory available

Verify the local CLI outside Hermes:

```bash
codex --version
codex login status
```

Typical success output:

```text
codex-cli 0.118.0
Logged in using ChatGPT
```

## 60-second smoke test

Before real repo work, run a fast read-only validation:

```bash
command -v codex
codex --version
codex login status
codex exec -C /path/to/repo -s read-only "Reply with exactly: LOCAL_CODEX_OK plus the current working directory."
```

Expected signals:

- `codex login status` succeeds
- Codex clearly runs in the requested directory
- the reply confirms the local CLI path is active for the worker

If you need a proxy-routed validation, use the **same proxy env on both auth and execution**, but replace the placeholders below with the actual local Clash/Mihomo proxy URLs configured on that machine:

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

---

## Mental Model

- **Hermes** decides what to do, gathers context, and talks to you
- **Codex CLI** does a focused coding/review task in a local working directory
- Hermes then reads the Codex result and reports back

Think of it as:

```text
You -> Hermes -> local codex CLI -> repo/files/tests -> Hermes -> You
```

---

## Rule Zero: Keep Hermes Responsive

When Hermes orchestrates local Codex CLI work, the **main Hermes session stays conversational and responsive**.

- **Main session:** conversation, one quick probe, task framing, launching Codex, monitoring, reporting back
- **Codex worker path:** coding, multi-file edits, builds/tests, repo work likely to branch, remote debugging, or SSH-style ops around a repo

Move work out of the main session immediately when it needs any of these:

- file writes / edits / commits / pushes
- builds, tests, installs, deploys
- multi-file coding work
- git beyond a quick read or status check
- more than one quick probe
- more than ~30 seconds of sustained execution
- the thought: **"one more quick check"**

**Spawn → Monitor → Report.** Launching Codex is not enough — Hermes should also tell the user what is running and what event will trigger the next update.

## Task Shape Gate

Before Hermes uses local Codex CLI, check whether inline work is still allowed:

- **Any write / build / deploy / repo work likely to branch?** → hand off to local Codex CLI
- **Coding task at all?** → prefer local Codex CLI over inline tool drift
- **Already at tool call #3 inline?** → stop and hand off
- **Need another real probe?** → stop and hand off
- **No real progress-update path?** → promise event-based updates only

For this pattern, a good default is: **Hermes frames; Codex executes; Hermes reports.**

## Delivery Gate

Do not promise timed updates unless a real delivery loop exists.

If Hermes is just supervising a local Codex subprocess, the safe promise is usually:

- done
- blocked
- or a named checkpoint

For example: **"I'll report back when Codex finishes, fails auth, or reaches the test step."**

---

## Basic Usage

Inside Hermes, ask for the local CLI explicitly:

```text
Use the local Codex CLI in /opt/work/hermes-agent.

1. Run `codex login status`
2. Run `codex exec -s read-only "Summarize how delegation works in this repo"`
3. Return the result only
```

Common local Codex commands Hermes can orchestrate:

```bash
codex exec -s read-only "Summarize the auth flow"
codex review --base origin/main
codex exec --full-auto "Implement X and run the relevant tests"
```

For long tasks, tell Hermes to keep Codex in the background and monitor it until done.

## Choosing Codex Effort

The orchestration notes you shared are right on the important point: **don't over-think with Codex when the answer is deterministic.**

| Task shape | Effort | Example |
|---|---|---|
| File reads, grep, git log, config lookup, deterministic checks | `medium` | read-only architecture probe |
| Normal bug fix, feature work, code review, standard repo operations | `high` | default worker mode |
| Prove a bug, deep refactor, tricky multi-file reasoning | `xhigh` | only when the task truly needs it |

**Default:** `high`

**Do not use `xhigh` for reads.** If Hermes is just asking Codex to inspect files or summarize code, prefer `medium`.

Examples:

```bash
codex exec -c model_reasoning_effort="medium" -s read-only "Summarize the config flow"
codex exec -c model_reasoning_effort="high" --full-auto "Implement the fix and run the targeted tests"
codex exec -c model_reasoning_effort="xhigh" --full-auto "Prove whether this race condition is real, then fix it"
```

---

## Separate Account from Hermes

This is the main reason to use the external CLI path.

### Important distinction

- Hermes provider auth for `openai-codex` lives in **`~/.hermes/auth.json`**
- Local Codex CLI auth lives in **`~/.codex/auth.json`**

That means:

- Hermes can use one provider/account for the main session
- the local Codex CLI can use a different login entirely
- changing one does **not** automatically switch the other

### Practical rule

If you want Hermes to orchestrate Codex under a different account, authenticate the **local Codex CLI** with that account:

```bash
codex logout
codex login
codex login status
```

Then tell Hermes to use the local CLI, not the built-in `openai-codex` provider.

:::tip
A good explicit instruction is: **"Use the local Codex CLI, not Hermes's built-in Codex provider."**
:::

### Multiple Codex identities on one machine

If you need more than one Codex identity at the same time, don't share one `~/.codex/` directory across all of them. In practice, use separate OS users, containers, or isolated home directories for each Codex identity.

---

## Route Only Codex Through Clash / a Different IP

Because Codex is launched as a subprocess, Hermes can give that subprocess a different proxy environment than the main Hermes process.

### The simplest pattern

Set proxy env vars **only on the Codex command**.

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

When Hermes orchestrates it, ask for the same thing explicitly:

```text
Use the local Codex CLI in /opt/work/hermes-agent.
Route only the Codex subprocess through the machine's actual Clash/Mihomo proxy URLs.
Keep Hermes itself on its current provider and current network path.
Run:
- `codex login status`
- `codex exec -s read-only "Summarize the delegation flow"`
Return the final result only.
```

### Clash note: prefer `socks5://`

Hermes normalizes generic proxy env vars like `HTTP_PROXY`, `HTTPS_PROXY`, and `ALL_PROXY` internally. In `utils.py`, Hermes rewrites `socks://127.0.0.1:PORT` to `socks5://127.0.0.1:PORT` because mixed Python HTTP stacks commonly require the explicit `socks5://` scheme.

So if your Clash setup exports `socks://127.0.0.1:<socks-port>`, prefer using the corresponding `socks5://...` URL in docs, scripts, and inline commands.

That is the safest form when Hermes and other tools share the same shell environment.

---

## Recommended Prompt Patterns

### 1. Read-only architecture check

```text
Use the local Codex CLI in /path/to/repo.
Run a read-only probe first:
- `codex login status`
- `codex exec -s read-only "Explain the CLI entrypoint and config flow"`
Do not modify files.
Return the result and any blockers.
```

### 2. Focused implementation via Codex, Hermes supervising

```text
Use the local Codex CLI as a worker in /path/to/repo.
Route only Codex through the machine's actual Clash/Mihomo proxy URLs.
Run:
- `codex exec --full-auto "Implement a docs page for X. Do not commit or push. Run the relevant tests if any."`
Then inspect the changed files and summarize exactly what Codex changed.
```

### 3. PR review under a separate Codex account / IP

```text
Use the local Codex CLI, not Hermes's built-in Codex provider.
Run in /path/to/repo:
- `codex login status`
- `codex review --base origin/main`
Route only Codex through the machine's actual Clash/Mihomo proxy URLs.
Return a concise review summary with concrete findings.
```

---

## Troubleshooting

### Codex auth errors

If Codex fails with errors like:

- `refresh_token_reused`
- `token_expired`
- repeated `401 Unauthorized`

re-auth the local CLI:

```bash
codex logout
codex login
codex login status
```

### Repo check failures

Prefer running in the actual target repo. Current Codex CLI builds also expose `--skip-git-repo-check` when you intentionally want scratch mode.

### Proxy weirdness with Clash

If traffic fails and your environment uses `socks://...`, switch to `socks5://...` explicitly.

### Wrong account being used

Remember the split:

- Hermes built-in provider auth: `~/.hermes/auth.json`
- local Codex CLI auth: `~/.codex/auth.json`

If the wrong Codex identity is active, fix the **local** CLI login, not just Hermes provider config.

---

## Summary

Use this pattern when you want:

- Hermes as the orchestrator
- local Codex CLI as the worker
- a different Codex account than Hermes's main provider
- a different IP path for Codex via Clash or another proxy

The key is to be explicit:

1. say **local Codex CLI**
2. name the **working directory**
3. state whether Codex should be **read-only** or allowed to edit
4. set **proxy env only for the Codex subprocess** when you want separate egress
5. keep Hermes and Codex auth/network assumptions separate
