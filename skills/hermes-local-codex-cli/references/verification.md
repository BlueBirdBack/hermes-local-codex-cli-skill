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
```

## Notes

- `codex exec --help` exposed `--skip-git-repo-check` on the validated CLI version.
- The read-only probe succeeded, but the CLI showed reconnect attempts before returning `ok`, so network stability should still be treated as an environment variable rather than a guaranteed constant.
- This file is the small validation layer for the skill repo. Update it when Hermes or Codex behavior is re-verified.
