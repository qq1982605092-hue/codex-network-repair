---
name: codex-network-repair
description: Use when Codex CLI or the OpenAI VS Code extension has network or transport problems, especially WebSocket fallback warnings, proxy issues, timed out requests, broken VSIX downloads, or AppTranslocation/incorrect app path problems on macOS.
---

# Codex Network Repair

Use this skill when Codex works intermittently, prints transport warnings, or an OpenAI/Codex install in VS Code fails to update or download cleanly.

## Symptoms this skill covers

- Codex CLI prints `Falling back from WebSockets to HTTPS transport`
- `Operation timed out (os error 60)`
- VS Code logs `Failed downloading vsix`
- `End of central directory record signature not found`
- `ERR_TIMED_OUT`
- VS Code runs from `AppTranslocation/.../Visual Studio Code.app`
- `code` points to a temporary AppTranslocation path

## Core diagnosis

Two patterns are common:

1. CLI path:
   Codex is using an unstable proxy path for streaming transport. The most reliable fix is usually to keep ChatGPT login, disable `ALL_PROXY` for `codex`, and provide stable `http_proxy` / `https_proxy`.

2. VS Code path:
   VS Code or an extension is running from an AppTranslocation path, or VS Code is re-downloading a truncated `.vsix` because the network path is unstable.

## CLI workflow

1. Inspect:
   - `~/.codex/config.toml`
   - `~/.zshrc`
   - `codex login status`
2. Prefer ChatGPT login if already present. Do not switch the default provider to `openai-http` unless the user actually has `OPENAI_API_KEY`.
3. In `~/.zshrc`, set:
   - `http_proxy=http://127.0.0.1:7890`
   - `https_proxy=http://127.0.0.1:7890`
   - uppercase variants too
4. Wrap `codex` so it drops `all_proxy` / `ALL_PROXY` but keeps HTTP(S) proxy:

```zsh
codex() {
  env -u all_proxy -u ALL_PROXY command codex "$@"
}
```

5. In `~/.codex/config.toml`, keep:
   - `responses_websockets = false`
   - `responses_websockets_v2 = false`
6. Validate with a real command from a login shell:

```bash
zsh -lic 'codex exec --skip-git-repo-check "你好"'
```

Success looks like:
- `provider: openai`
- normal response returns
- no WebSocket fallback warning during the run

## VS Code workflow

1. Check whether VS Code is running from `/Applications` or from `AppTranslocation`.
   - Use `ps aux | rg 'Visual Studio Code|Code Helper'`
2. Find installed app copies:

```bash
find /Applications ~/Applications ~/Downloads -maxdepth 2 -name 'Visual Studio Code.app'
```

3. If the running process uses `AppTranslocation`, quit VS Code and launch `/Applications/Visual Studio Code.app`.
4. Remove quarantine from the `/Applications` app copy:

```bash
xattr -dr com.apple.quarantine /Applications/Visual\ Studio\ Code.app
```

5. Fix the `code` symlink if it points to AppTranslocation. Prefer `/opt/homebrew/bin/code` when writable.
6. Set VS Code proxy in `~/Library/Application Support/Code/User/settings.json`:
   - `"http.proxy": "http://127.0.0.1:7890"`
   - `"http.proxySupport": "override"`
7. If VSIX downloads are failing, stop auto-update churn:
   - `"extensions.autoCheckUpdates": false`
   - `"extensions.autoUpdate": false`
8. Clear extension cache safely by renaming and recreating `CachedExtensionVSIXs` instead of using destructive deletion when possible.
9. Restart VS Code and inspect the latest `sharedprocess.log`.

Success looks like:
- no running process under `AppTranslocation`
- `code` resolves to `/Applications/Visual Studio Code.app/.../bin/code`
- latest logs no longer show:
  - `Failed downloading vsix`
  - `ERR_TIMED_OUT`
  - `End of central directory record signature not found`

## Important guardrails

- Do not force `model_provider = "openai-http"` for CLI if the user logs in with ChatGPT and does not have `OPENAI_API_KEY`.
- Do not start by deleting installed extensions. First stop bad update loops and fix transport/proxy pathing.
- For VS Code caches, prefer rename-and-recreate over `rm -rf` when a safer option works.
- Verify by re-running the failing workflow, not just by editing config files.

## Deliverable

When done, report:

- root cause
- exact files changed
- whether CLI is stable
- whether VS Code is now running from `/Applications`
- whether the latest logs still show transport or VSIX corruption errors
