# hermes-local-codex-cli-skill

A Hermes skill for using **Hermes as the orchestrator** and the **local `codex` CLI as the worker**.

This skill is for the case where you want Hermes to run the installed `codex` binary on the same machine, instead of using Hermes's built-in `openai-codex` provider path.

## What it covers

- separate auth stores: `~/.hermes/auth.json` vs `~/.codex/auth.json`
- separate egress / Clash routing for the Codex subprocess
- keeping the main Hermes session responsive
- task-shape gating: when Hermes should hand work off to Codex
- Codex effort selection: `medium` vs `high` vs `xhigh`
- auth failure recognition (`refresh_token_reused`, `token_expired`, `401`)

## Install directly from GitHub

```bash
hermes skills install BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli
```

## Add as a custom tap

```bash
hermes skills tap add BlueBirdBack/hermes-local-codex-cli-skill
```

Then browse or install from the GitHub source.

## Skill path in this repo

```text
skills/hermes-local-codex-cli/SKILL.md
```

## Typical use

Tell Hermes explicitly:

```text
Use the local Codex CLI, not Hermes's built-in Codex provider.
Use /path/to/repo as the working directory.
Route only the Codex subprocess through Clash at socks5://127.0.0.1:7897.
Run a read-only probe first, then report back.
```

## License

MIT
