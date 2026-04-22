# Verification

Last updated: 2026-04-22

## Validated against

| Item | Value |
|---|---|
| Hermes | `v0.10.0 (2026.4.16)` |
| Codex CLI | `0.118.0` |
| Local Codex auth check | `codex login status` → `Logged in using ChatGPT` |

## Commands used

```bash
hermes --version
codex --version
codex login status
codex exec --help
codex exec -s read-only "Reply with exactly: ok"

# Hub/install path checks
export HERMES_HOME=$(mktemp -d)
hermes skills inspect BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli
hermes skills install BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli --force --category autonomous-ai-agents
hermes skills tap add BlueBirdBack/hermes-local-codex-cli-skill
hermes skills inspect BlueBirdBack/hermes-local-codex-cli-skill/skills/hermes-local-codex-cli
hermes skills list
rm -rf "$HERMES_HOME"
```

## Notes

- `codex exec --help` exposed `--skip-git-repo-check` on the validated CLI version.
- Direct GitHub install, tap add, and inspect were all re-verified.
- The install path was re-verified with `--force --category autonomous-ai-agents`; `--force` is required here because community GitHub skills with caution-level findings are blocked by default until explicitly overridden after inspection.
- The read-only probe succeeded, but the CLI showed reconnect attempts before returning `ok`, so network stability should still be treated as an environment variable rather than a guaranteed constant.
- Proxy examples in this repo intentionally use placeholders like `http://127.0.0.1:<http-port>` and `socks5://127.0.0.1:<socks-port>`; use the actual local Clash/Mihomo URLs on the machine being configured.
- This file is the small validation layer for the skill repo. Update it when Hermes or Codex behavior is re-verified.
