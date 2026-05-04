# Failure Patterns

Use this file when the main skill identifies a likely upgrade regression and you need concrete examples of what to inspect.

Contribution rule:
- append new patterns or expand existing ones
- do not delete older patterns unless they are proven false
- preserve examples from other hosts even if the current host is healthy

## 1. Update channel drift

Symptom:
- host is intentionally on beta or a newer stable build
- `status --deep` says local version is newer than `npm latest`
- config still says `"update.channel": "stable"` after a beta install

What to inspect:
- `openclaw --version`
- `openclaw status --deep`
- `openclaw.json` update metadata

Why it matters:
- operators get misleading update advice
- maintainers should know when install/update channel state is not persisted

## 2. Stale config after upgrade

Symptom:
- `doctor` says provider or plugin is unknown
- runtime falls back to auto-detect or legacy behavior
- helper commands claim to fix config, but warnings remain

Common keys:
- `tools.web.search.provider`
- `plugins.allow`
- `plugins.entries.*`

Typical example:
- `tools.web.search.provider=brave` remains after host/plugin changes and becomes invalid

## 3. Bundled plugin vs global npm plugin shadowing

Symptom:
- bundled capability should work after host upgrade
- `plugins list` shows a global plugin path under `~/.openclaw/npm/node_modules`
- plugin version does not match host version

Example:
- host on `2026.5.3`
- global `@openclaw/discord` still at `2026.5.2`
- gateway warns about missing compiled runtime output because the global plugin is source-only

What to inspect:
- `openclaw plugins inspect <id>`
- plugin `source`
- plugin `origin`
- plugin `version`
- presence of `dist/`

## 4. Install records drift from disk reality

Symptom:
- config or install registry says a plugin is installed
- recorded path under `~/.openclaw/npm/node_modules/@openclaw/` or `~/.openclaw/extensions/` does not exist
- plugin not found / phantom allowlist warnings

What to inspect:
- `~/.openclaw/plugins/installs.json`
- actual install path on disk
- `openclaw plugins registry --refresh`

This is the case where reinstalling the plugin is often correct.

## 5. Third-party plugin runtime deps removed

Symptom:
- after `doctor --fix` or cleanup, a third-party plugin fails to load
- error looks like `Cannot find module ...`
- plugin root still exists, but plugin-side `node_modules` is gone

What to inspect:
- plugin package directory
- plugin `package.json`
- whether dependencies are externalized at build time

Why it matters:
- cleanup can be too aggressive for non-bundled plugins

## 6. Context engine not registered after restart

Symptom:
- logs say context engine falls back to legacy
- plugin may still be installed but failed to initialize

Look for:
- plugin load errors
- missing dependencies
- plugin contract warnings
- plugin registry metadata drift

## 7. Event loop degradation after update

Symptom:
- `channels status --deep` reports degraded event loop
- logs show lane wait exceeded, active-memory timeouts, or restart blocked by active tasks

Common culprits:
- stale running tasks
- active-memory timeout loops
- plugin load retries
- long-running approval followups

Check:
- `openclaw tasks audit`
- recent `gateway.err.log`
- recent `gateway.log`

## 8. Task ledger blocks clean restart

Symptom:
- restart or drain says blocked by active task runs
- `tasks audit` shows `stale_running`, `lost`, or repeated delivery failures

Useful commands:
- `openclaw tasks show <id>`
- `openclaw tasks cancel <id>`
- `openclaw tasks maintenance --apply`

Fix the ledger if it is obviously wrong; otherwise every later health check becomes noisy.

## 9. Command-path disagreement

Symptom:
- `status --deep` says a channel token is unavailable in this command path
- `channels status --deep` says channel is connected and healthy

Treat this as a reporting mismatch first, not a real outage.

Additional example:
- Discord can be healthy in the gateway service with `token:env`, while `doctor` still warns that `DISCORD_BOT_TOKEN` is absent in the doctor environment.
- Verify the LaunchAgent env file and `channels status`; do not treat the doctor shell-env warning alone as proof the live gateway is down.

## 10. What to hand maintainers

When the issue looks like an update regression, capture:

- host version before and after
- whether the capability was bundled, npm-installed, or ClawHub-installed
- the plugin source path actually loaded
- stale config keys still present after fix attempts
- exact `doctor` and `plugins doctor` messages
- startup log lines around the failure

## 11. Plugin updater follows stale install records

Symptom:
- core OpenClaw is updated, but `~/.openclaw/plugins/installs.json` still records older plugin specs
- `openclaw plugins update --all` tries to reinstall the older recorded versions instead of reconciling to the currently installed packages
- plugin directories under `~/.openclaw/npm/node_modules/@openclaw/` may disappear or config suddenly becomes invalid until the exact desired versions are reinstalled

Typical example:
- host core on `2026.5.3`
- install records still pinned to `2026.5.2-beta.1`
- running `openclaw plugins update --all` attempts the old beta plugin specs and leaves `brave`, `discord`, and `whatsapp` missing on disk until exact `2026.5.3` packages are reinstalled

What to inspect:
- `~/.openclaw/plugins/installs.json`
- actual package versions in `~/.openclaw/npm/node_modules/@openclaw/*/package.json`
- whether the plugin directories still exist after `plugins update --all`

Why it matters:
- the built-in updater can deepen an upgrade regression if install metadata drift is not corrected first
- prefer reconciling install records or reinstalling exact target versions before trusting `openclaw plugins update --all`

## 12. Control UI token mismatch after restart

Symptom:
- gateway is healthy and reachable
- dashboard page loads, but websocket auth fails
- `gateway.err.log` shows `[ws] unauthorized ... reason=token_mismatch`
- log text may say `unauthorized: gateway token mismatch (open the dashboard URL and paste the token in Control UI settings)`

What to inspect:
- recent `~/.openclaw/logs/gateway.err.log` websocket auth lines
- whether `gateway.auth.token` or its SecretRef source changed during reinstall/restart
- whether the browser-side Control UI is still holding an older token

Why it matters:
- this can look like a gateway outage even when the backend is healthy
- separate UI auth cache problems from real startup or channel failures before changing server-side config again

## 13. Channel SecretRef resolves but runtime account still cannot use it

Symptom:
- `openclaw config validate` passes
- `openclaw secrets audit` reports `unresolved=0`
- `openclaw channels status` says a channel is configured but stopped/disconnected with `secret unavailable in this command path`
- logs say a channel token is unavailable, for example Discord delivery says the bot token configured for account `default` is unavailable

What to inspect:
- the channel token config path, for example `channels.discord.token`
- the referenced secrets provider and backing file
- whether the gateway service env has a working fallback such as `DISCORD_BOT_TOKEN`
- whether the channel plugin prefers the broken config SecretRef over the env fallback

Observed workaround:
- add the token to the LaunchAgent service env from the existing local secret source
- remove the broken channel token config field so the channel falls through to the env-token path
- restart gateway and verify `channels status` reports `token:env` and connected

Why it matters:
- schema validation and secrets audit can both pass while the channel runtime still cannot consume the SecretRef
- this can leave a channel integration down after an update even though the secret exists

## 14. Gateway CLI start reports argument error but LaunchAgent recovers

Symptom:
- after update, `openclaw gateway start` prints `error: too many arguments for 'gateway'. Expected 0 arguments but got 1.`
- the command may still re-bootstrap the LaunchAgent afterward
- `launchctl` and `lsof` show the gateway running despite the CLI error

What to inspect:
- `launchctl print gui/$(id -u)/ai.openclaw.gateway`
- listener on the configured gateway port
- `openclaw status`
- gateway stdout/stderr logs

Why it matters:
- the CLI error is alarming but may not be the actual outage
- verify service reality before retrying installs or rolling back
