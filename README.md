# hermes-local-codex-cli-skill

Hermes orchestrates. The local `codex` CLI does the work.

Use this when you want Hermes to run the installed `codex` binary on the same machine instead of Hermes's built-in `openai-codex` provider.

## Install

```bash
hermes skills inspect BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli
hermes skills install BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli --force --category autonomous-ai-agents
```

Use `--category autonomous-ai-agents` so the installed skill stays grouped correctly in `hermes skills list`.

## Optional tap

```bash
hermes skills tap add BlueBirdBack/hermes-local-codex-cli-skill
hermes skills inspect BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli
```

## 60-second smoke test

```bash
codex login status
codex exec -C /path/to/repo -s read-only "Reply with exactly: LOCAL_CODEX_OK plus the current working directory."
```

Expected:
- `codex login status` succeeds
- output contains `LOCAL_CODEX_OK`
- output shows the repo path you passed with `-C`

## Proxy note

Use the real local proxy URLs on that machine. Do not hardcode a port from an example.

- HTTP: `http://127.0.0.1:<http-port>`
- SOCKS5: `socks5://127.0.0.1:<socks-port>`

## Files

- `skills/hermes-local-codex-cli/SKILL.md` — canonical instructions
- `skills/hermes-local-codex-cli/references/orchestration-guide.md` — fuller guide
- `skills/hermes-local-codex-cli/references/verification.md` — last verified commands

## License

MIT
