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

Refinement (observed 2026-05-05 on host upgrading 2026.5.3 → 2026.5.4):
- `openclaw plugins registry --refresh` does NOT rewrite the install record's `spec` field. It refreshes `hostContractVersion` and compatibility data only.
- After a refresh, install records can still carry pinned specs like `@martian-engineering/lossless-claw@0.9.2` even when the disk version is `0.9.3`. `plugins update --all` will then **downgrade** the on-disk plugin to match the pinned spec.
- Correct sequence to actually move a third-party plugin forward:
  1. `openclaw plugins update <id> @<scope>/<pkg>@latest` (note: `update` accepts an explicit spec; this rewrites the install record's spec).
  2. Or `openclaw plugins install <pkg>@latest --force` to drop the pin.
- `--all` is safe only after every install record's `spec` already points at `@latest` or the desired version — never trust it after a host upgrade without spot-checking install records first.

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

Confirmed regression scope (as of 2026-05-05):
- Reproduced cleanly on `@openclaw/discord` versions 2026.5.2, 2026.5.3, and 2026.5.4 with a `file:filemain:/discord_token` SecretRef pointing at a valid `secrets.json` entry.
- `openclaw secrets audit` reports `unresolved=0`, `openclaw secrets reload` says "Secrets reloaded.", but the plugin's `normalizeDiscordToken` (`@openclaw/discord/src/token.ts`) still throws `unresolved SecretRef ... Resolve this command against an active gateway runtime snapshot before reading it.` at startup.
- Sibling plugins using the same SecretRef shape (e.g. brave's `/brave_api_key`) resolve fine — the bug is plugin-side, not in the secrets layer.
- Pragmatic workaround (when env fallback isn't available): inline the literal token into `channels.discord.token`. This adds one entry to `secrets audit --plaintext` findings but restores Discord. Plan to revert once upstream `@openclaw/discord` ships a fix that resolves SecretRefs against the runtime snapshot.

Addendum — `token:config` in `channels status` is ambiguous:
- the same status row appears whether the token came from a successfully resolved SecretRef OR from an inline literal (workaround applied); it is not a signal that the upstream bug is fixed.
- To disambiguate, inspect the actual config field directly:
  - `node -e 'const c=JSON.parse(require("fs").readFileSync("/Users/<user>/.openclaw/openclaw.json","utf8")); console.log(typeof c.channels?.discord?.token, c.channels?.discord?.token)'`
  - `string` value → inline literal (workaround in place)
  - `object` value → SecretRef (relies on plugin runtime resolution)
- An operator running this runbook on an inherited host should not assume `token:config` means the regression is gone; verify the field shape before claiming the workaround is no longer needed.

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

## 15. `plugins uninstall` is destructive of every config trace, not just the install record

Symptom:
- after `openclaw plugins uninstall <id> --force`, the plugin disappears from `plugins list` as expected
- but the plugin then shows as `disabled` once you try to re-enable a sibling install (e.g., a copy under `~/.openclaw/extensions/`)
- exclusive slots (e.g., `plugins.slots.contextEngine`) silently revert to `legacy`

What `uninstall` actually removes (observed 2026.5.4):
- the install record in `~/.openclaw/plugins/installs.json`
- the on-disk install directory
- `plugins.entries.<id>` from `openclaw.json`
- the `<id>` entry from `plugins.allow`
- exclusive slot assignments where `<id>` was the holder

Recovery after rolling back to a different copy of the same plugin:
- `openclaw plugins enable <id>` re-adds the entry, allowlist row, and slot assignment
- restart gateway

CLI flag note:
- `plugins uninstall` does not accept `--yes`, `-y`, or `--non-interactive`. Use `--force` to skip the confirmation prompt over a non-interactive shell.

Why it matters:
- treat `uninstall` as "wipe all traces", not as "remove just the install record"
- if you only wanted to swap install paths (npm → extensions or vice versa), prefer manual relocation + `plugins registry --refresh` over `uninstall` + reinstall

## 16. Third-party plugin declares optional peer dependency but compiled bundle imports it unconditionally

Symptom:
- after upgrading a third-party plugin, `plugins doctor` reports `lossless-claw [load]: Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@mariozechner/pi-coding-agent'` (or similar)
- the plugin's `package.json` lists the missing package under `peerDependenciesMeta` with `optional: true`, suggesting it should be skippable

Root cause:
- the published `dist/index.js` was emitted with an unconditional `import` of an "optional" peer dependency
- Node's ESM resolver cannot satisfy the import, so the plugin fails to load even though `package.json` says the dep is optional

Typical example (observed 2026-05-05):
- `@martian-engineering/lossless-claw@0.9.4` declared `@mariozechner/pi-coding-agent` as `peerDependenciesMeta.<dep>.optional: true`
- the plugin still failed to load because `dist/index.js` imported it unconditionally
- the previous version (`0.9.3`) shipped a bundled `node_modules/` next to the plugin and worked fine
- `plugins update --all` on a host with a stale install record (pattern #11) would have downgraded to `0.9.2` instead, masking this as a different failure mode

What to inspect:
- `package.json` `peerDependencies`, `peerDependenciesMeta`, `dependencies`, `devDependencies`
- the actual import sites in `dist/index.js` (`grep -E "from '@.*pi-" dist/index.js`)
- whether a previous version's bundled deps are still on disk (e.g., `~/.openclaw/extensions/<id>.stale-*` or `.backup-*` directories)

Recovery:
- roll back to the last known-good plugin version, ideally one that bundled its deps
- prefer renaming the broken install dir (e.g., to `<dir>.broken-<date>`) over deleting it, so the failure can still be reproduced for an upstream report
- file an upstream issue with: declared optional deps in `package.json` vs unconditional imports in `dist`

Why two hosts on the same plugin version can show different results (observed 2026-05-05):
- the bug only surfaces when Node's ESM resolver cannot find the "optional" peer from the plugin's location
- on hosts where the peer is **hoisted** at `~/.openclaw/npm/node_modules/<scope>/<peer>` (sibling to the plugin), the import succeeds silently and `plugins doctor` reports clean
- the peer can also be satisfied by a copy under `/opt/homebrew/lib/node_modules/openclaw/node_modules/<scope>/<peer>` (bundled with the host package), or by leftover `~/.openclaw/plugin-backups/<id>.*/node_modules/<scope>/<peer>` directories from a prior disabled install
- a host that recently ran a clean reinstall (or `npm prune`, or a `doctor --fix` cleanup that removed disabled plugin backups) is more likely to hit the failure than a host that has accumulated multiple historical copies of the peer
- if you reproduce the bug, also enumerate every on-disk copy of the peer before rolling back, so you can explain the divergence to upstream:
  ```
  find ~/.openclaw /opt/homebrew/lib/node_modules -maxdepth 6 -type d -name "<peer-package-name>"
  ```

Inspection note:
- `dist/index.js` is typically a single esbuild-bundled minified line; `grep` will appear to match the entire file. Use `grep -oE "from'@[^']+'" dist/index.js` (or similar token-level patterns) to enumerate actual import specifiers without dumping the bundle.

Why it matters:
- this is not a missing-dep on the operator's side — it is a packaging defect
- avoid the temptation to manually `npm install` the missing peer into the plugin dir, because the next `plugins update` will overwrite the directory and the fix will silently disappear
- the resolver-luck variance is itself the bug: a plugin that "works on my host" but breaks for the next operator is the same defect, not a host configuration difference; do not dismiss the upstream report because your host happens to satisfy the import

## 17. "Duplicate plugin id detected" warning text wraps in a self-referential way

Symptom:
- `plugins doctor` and `openclaw doctor` warn: `plugin <id>: duplicate plugin id detected; global plugin will be overridden by global plugin (/path/A)`
- only one path is visible at a glance; the second path is wrapped to a later line and easily missed
- on a narrow terminal the warning can look self-referential ("global plugin will be overridden by global plugin (X)") and is easy to dismiss as a UI bug

Reality:
- the warning is real — there are two on-disk plugin manifests for the same id
- the conflict is almost always between `~/.openclaw/extensions/<id>/` and `~/.openclaw/npm/node_modules/<scope>/<id>/` (or two copies under the same root)
- the npm path generally wins, but the extensions path still triggers the warning every restart

What to inspect:
- `find ~/.openclaw -maxdepth 4 -type d \( -name "<id>" -o -name "@*<id>*" \)`
- whether `plugins.load.paths` in `openclaw.json` is empty or pointing at an extra root
- whether a previous `plugins update --all` left a `.backup-*` directory next to the new install (those are typically ignored, but a renamed-not-deleted manual copy can be picked up)

Recovery:
- pick the canonical install (npm-tracked is preferred for plugins managed via `openclaw plugins install`)
- rename the unwanted copy to `<dir>.stale-<date>` (safer than `rm -rf` mid-runbook)
- restart gateway and confirm `plugins doctor` reports zero errors and the warning is gone

Why it matters:
- operators often dismiss this as cosmetic; it is not — the second path keeps generating doctor noise that masks new regressions
- the warning text formatter wraps poorly; always re-read the full multi-line warning before deciding the conflict is benign

True false-positive variant (observed 2026-05-05 on `@martian-engineering/lossless-claw@0.9.4`):
- after archiving every redundant on-disk copy and confirming `find ~/.openclaw -maxdepth 6 -name 'openclaw.plugin.json' | xargs grep -l '"<id>"'` returns only the canonical install path, the warning can still persist
- `openclaw plugins inspect <id>` then shows the warning's path field is **identical** to the loaded plugin's `Source` path — i.e., the warning is comparing the manifest against itself
- this looks like an OpenClaw bug where the same manifest is being matched twice (once via the `installs.json` install record, once via filesystem scan) and both lookups are tagged `Origin: global`, generating a phantom duplicate
- distinguishing genuine #17 (two real manifests on disk) from this false-positive: run `plugins inspect <id>` and compare the `Source` line to the path inside the WARN line. Same path = false positive. Different paths = genuine duplicate, keep hunting.
- when it is the false positive, leave it alone; do not delete the canonical install in an attempt to silence it

## 18. Bundled provider discovery mode change after host upgrade (2026.5.4+)

Symptom:
- after upgrading the host package, `openclaw doctor` adds a new warning:
  `plugins.allow is restrictive, but bundled provider discovery is still in legacy compatibility mode. Bundled provider plugins can ... set plugins.bundledDiscovery to "allowlist" after confirming omitted providers.`
- previously absent config key is now expected: `plugins.bundledDiscovery`

Background:
- `plugins.allow` historically gated only third-party plugins; bundled provider plugins (anthropic, openai, gemini, etc.) were always discoverable.
- 2026.5.4 introduced `plugins.bundledDiscovery` with two modes:
  - `"compat"` — preserves legacy behavior; bundled providers stay discoverable regardless of `plugins.allow`
  - `"allowlist"` — bundled providers must also appear in `plugins.allow`
- Hosts upgraded from 2026.5.3 inherit the legacy behavior implicitly but doctor flags it until the key is set explicitly.

What to do:
- if `plugins.allow` is restrictive and you intentionally rely on bundled providers, set `plugins.bundledDiscovery: "compat"` to lock in current behavior — note that this **does not silence the doctor warning**, it only pins the mode against a future default flip (see refinement below)
- if you want strict allowlisting end-to-end and want the warning gone, audit which bundled providers your agent fallback chains require, add them to `plugins.allow`, then set `plugins.bundledDiscovery: "allowlist"`

Refinement (observed 2026-05-05 on a 2026.5.4 host):
- Some 2026.5.x point releases auto-migrate `plugins.bundledDiscovery` to `"compat"` during the host upgrade, so the key may already be set even on hosts that never had it explicitly. Always re-read the live config before assuming the warning means the key is unset.
- Even with `"compat"` explicitly set, doctor continues to print: `plugins.allow is restrictive, but bundled provider discovery is still in legacy compatibility mode ... set plugins.bundledDiscovery to "allowlist" after confirming omitted bundled providers are intentionally blocked`. The warning is the doctor's nudge to migrate forward, not a "key missing" warning. Two paths to silence:
  1. Migrate to `"allowlist"` (recommended): enumerate the bundled providers your agents actually need by walking `c.agents.defaults.model.{primary,fallbacks}` and any agent-level overrides; the model strings are typically `provider/model` shaped (e.g., `anthropic/claude-opus-4-7`, `openai-codex/gpt-5.5`). Map each `provider/` prefix to its bundled plugin id (`openai-codex` → `openai`, since the openai plugin owns both `openai` and `openai-codex` provider ids). Add the corresponding plugin ids to `plugins.allow`, set `plugins.bundledDiscovery: "allowlist"`, restart, and re-run doctor.
  2. Accept the persistent warning and rely on `"compat"` — fine for now, but re-audit after every minor bump in case a future version changes the warning into an error.
- When migrating to `"allowlist"`, also confirm the corresponding API-key env vars are present in the LaunchAgent service-env (e.g., `ANTHROPIC_API_KEY` for the `anthropic` plugin); plugins added to `plugins.allow` without credentials will load but fail at first use, which is harder to diagnose than a discovery warning.

Why it matters:
- this is a config-shape change introduced silently by a minor version bump; treat it as a host-upgrade follow-up, not a one-off doctor warning
- ignoring it doesn't break anything today, but a future minor that flips the default to `"allowlist"` will instantly regress provider discovery on every host that hasn't pinned the mode

## 19. CLI uninstall confirmation prompt blocks non-interactive runbooks

Symptom:
- `openclaw plugins uninstall <id>` prints `Uninstall plugin "<id>"? [y/N]` and then exits without doing anything in a non-interactive shell (e.g., a single ssh command with no stdin).
- stderr may include an unrelated `Detected unsettled top-level await` warning that obscures the real reason (no input piped to the prompt).

What to do:
- always pass `--force` for non-interactive uninstalls
- `--yes` and `-y` are NOT accepted as of 2026.5.4; only `--force` skips the prompt
- if you also want a preview, run `--dry-run` first

Why it matters:
- a runbook that pipes a single `ssh` command without a TTY will silently no-op the uninstall, then proceed to "verify" steps that report the plugin still present and confuse the operator into deeper changes

## 20. Multi-step SSH update command disconnects mid-run while the box keeps working

Symptom:
- operator runs a single multi-step `ssh user@host '... stop ... npm install ... reinstall plugins ... start ...'` command
- the SSH session appears hung or returns no output to the operator's terminal
- reconnecting with a fresh ssh shows the box has actually completed most or all of the work — versions bumped, gateway running, plugins on disk

What's happening (observed 2026-05-05):
- when one of the inner steps restarts launchd or replaces the wrapper script the gateway plist sources, the parent shell association can break and the local ssh client stops receiving stdout, even though the remote `zsh -c '...'` keeps running detached and finishes the script.
- the remote orphan can persist as a `zsh -c` process for minutes after the parent ssh exits.

What to inspect:
- on the remote host: `pgrep -fl "openclaw/dist/index.js gateway"` (current gateway PID and command line)
- `pgrep -fl "zsh -c"` for orphan wrapper processes from the disconnected session
- on-disk plugin versions vs `npm view @openclaw/<id> version`
- `~/.openclaw/logs/gateway.log` for the latest `http server listening` line (confirms a fresh restart actually happened)

Recovery:
- kill orphan wrapper zsh processes (`kill <pid>`)
- re-run the verification suite (`openclaw --version`, `openclaw plugins doctor`, `openclaw channels status`, `openclaw tasks audit`) from a fresh ssh session
- do NOT re-run the update script blindly; it may have completed successfully and a second run can re-pin install records to versions that were just bumped

Practical advice:
- prefer breaking the update into separate ssh invocations per phase: stop → host update → plugin reinstalls → start → verify. A disconnect then loses only the current phase, not the whole sequence.
- where a single transactional run is unavoidable, redirect the script's output to a remote file (`> /tmp/openclaw-update.log 2>&1`) and tail it from a second ssh session, so the parent disconnect does not lose the audit trail.

Why it matters:
- treating an apparent hang as failure and rerunning can corrupt install records mid-flight
- the runbook's "verify" step must rely on freshly inspected box state, not on the success path of the update command's stdout

## 21. Version drift between operator sessions on hosts with autopilot agents

Symptom:
- operator returns to a host they audited recently and finds a different `openclaw --version` than what they last left it on
- no explicit operator-initiated update happened in the interim
- `update.auto.enabled` may be `false` in `openclaw.json`, but the host still moved versions

Background:
- some hosts run autopilot or scheduled cron agents (e.g., `gbrain`, `com.gbrain.autopilot.plist`, scheduled openclaw cron jobs) that may bump `openclaw` or its plugins out of band, ignoring the host-level update channel/auto flag
- the `meta.lastTouchedVersion` field in `openclaw.json` only reflects the last writer, not the last installer

What to do:
- always re-snapshot the live state at session start, even within hours of the previous session:
  - `openclaw --version`
  - per-plugin disk versions: `for d in ~/.openclaw/npm/node_modules/@*/*/; do node -e 'process.stdout.write(JSON.parse(require("fs").readFileSync(process.argv[1])).name+" "+JSON.parse(require("fs").readFileSync(process.argv[1])).version+"\n")' "$d/package.json"; done`
  - any service-env or config edits applied by previous workarounds (`grep -l DISCORD_BOT_TOKEN ~/.openclaw/service-env/*.env`)
- do not rely on prior-session memory for current state; treat every session as a fresh audit

Why it matters:
- a stale mental model leads to the wrong fix path — e.g., applying an env-var workaround when a literal-inline workaround is already in place, or "rolling back" an update the operator never made
- two parallel operator sessions (or an operator + a long-running autopilot) can converge on contradictory workarounds if neither re-snapshots first

## 22. Cohort version snapshot before host update

Practice:
- before stopping the gateway, capture every plugin's installed version with a single command:
  ```
  for d in ~/.openclaw/npm/node_modules/@*/*/; do
    node -e 'const p=JSON.parse(require("fs").readFileSync(process.argv[1])); process.stdout.write(p.name+"@"+p.version+"\n")' "$d/package.json"
  done | sort > /tmp/openclaw-pre-upgrade-plugins.txt
  ```
- after the upgrade, re-run the same command into `/tmp/openclaw-post-upgrade-plugins.txt` and `diff` them.
- a clean cohort upgrade should show every plugin version moving in the diff; any plugin that did NOT move is a candidate for shadow drift (Pattern #3) once the host moves further.

Why it matters:
- a `plugins install <pkg>@latest --pin --force` no-op (e.g., from a network or registry hiccup, or because npm latest temporarily lagged ClawHub) is invisible until you hit a sub-feature that depends on the new version.
- without the snapshot/diff, the operator cannot prove the cohort actually moved — only that the host did.
- the snapshot also documents what to roll back to if the new cohort surfaces a packaging defect (Pattern #16).

## 23. Service-env writer corrupts string secrets with literal double-quote wrapping (2026.5.4)

Symptom:
- a previously working channel (e.g., Discord) returns auth failure (`401 Unauthorized` from the upstream API) immediately after a host upgrade, even though `channels status` reports `token:env` and the channel was healthy before the upgrade
- `secrets audit` reports `unresolved=0` and the underlying value in `secrets.json` is unchanged
- the channel reconnects fine if you manually re-paste the token into the env file

Root cause:
- the 2026.5.4 service-env writer JSON-encodes string values from `secrets.json` (wrapping them in `"`) and **then** shell-single-quotes the result for the env file
- the resulting line looks like `export DISCORD_BOT_TOKEN='"MTQ3...3RI"'` — the outer single quotes are correct shell quoting, but the inner literal `"` characters become part of the value when the env file is sourced
- the upstream API receives a token with stray leading and trailing `"` chars and rejects it
- the bug only surfaces the next time the env file is regenerated (a host upgrade, certain `doctor --fix` runs, plugin reinstalls), so it presents as "the upgrade broke Discord" rather than a config drift

What to inspect:
- the raw bytes of the env line, not just the masked output:
  ```
  node -e 'const l=require("fs").readFileSync(process.env.HOME+"/.openclaw/service-env/ai.openclaw.gateway.env","utf8").split("\n").find(l=>l.startsWith("export DISCORD_BOT_TOKEN=")); console.log("first5="+JSON.stringify(l.slice(25,30)),"last5="+JSON.stringify(l.slice(-5)))'
  ```
- a clean line: `first5="'MTQ3"` and `last5="63RI'"` (single quotes only)
- a corrupted line: `first5="'\"MTQ"` and `last5="3RI\"'"` (literal `"` baked inside)
- check every `*_TOKEN` / `*_API_KEY` line in the env file the same way; the same writer emits all of them

Recovery:
- back up the env file: `cp <env> <env>.bak-token-fix-<date>`
- rewrite the affected lines using the value from `secrets.json` (which is the canonical clean value), shell-single-quoted with no inner JSON wrapping; only safe if the secret itself contains no single quotes (almost always the case for API tokens)
- `launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway` to restart
- re-run `openclaw channels status --deep` and confirm the channel reconnects

Why it matters:
- this is a packaging defect in the env-file writer, not operator drift; the local fix is fragile because the next regeneration will re-corrupt the file
- file an upstream issue with the exact line bytes, the source `secrets.json` value type (string), and the affected host version
- until upstream ships a fix, treat any operation that may rewrite `service-env/*.env` (host updates, plugin updates, `doctor --fix` involving secrets) as a Discord/Telegram outage risk and re-verify channel auth immediately after

Workflow addendum:
- when post-update channel health shows auth failure, **inspect the env file's raw bytes for quote corruption before assuming the upstream credential was rotated**. The wrong diagnosis path leads to credential rotation and operator confusion; the right diagnosis takes 30 seconds.

## 24. Non-interactive SSH hides the OpenClaw binary on macOS

Symptom:
- `ssh host 'openclaw --version'` returns `command not found`
- the same host has a working LaunchAgent and `openclaw status` works in an interactive shell
- `launchctl print` may also look wrong if the operator guesses an old service label

What to inspect:
- `echo "$PATH"` inside the non-interactive SSH command
- common binary locations: `/opt/homebrew/bin/openclaw`, `~/.local/bin/openclaw`
- the actual LaunchAgent label under `~/Library/LaunchAgents`
- the gateway command and port inside the plist or `openclaw status --deep`

Observed example:
- non-interactive SSH inherited only `/usr/bin:/bin:/usr/sbin:/sbin`
- OpenClaw was installed under `/opt/homebrew/bin`
- the gateway LaunchAgent label was `ai.openclaw.gateway`, not an older guessed label
- the live gateway port was `18789`, so probing an old hard-coded health port falsely reported an outage

Why it matters:
- a stripped SSH `PATH` can look like a missing installation
- an old label or hard-coded port can make a healthy gateway look stopped
- always establish the command path, label, and port before running update or repair commands

## 25. Update stop phase may need launchd bootout even after clean gateway SIGTERM

Symptom:
- `openclaw update --channel <channel>` prints that `launchctl stop` did not fully stop the service
- updater reports it used a `bootout` fallback and left the service unloaded before continuing
- gateway logs may show a clean `SIGTERM` shutdown, followed by a short-lived restart that is immediately terminated before the package swap completes

What to inspect:
- update command stdout/stderr
- `launchctl print gui/$(id -u)/ai.openclaw.gateway` before and after the update
- recent gateway logs around the stop/restart window
- final `openclaw status --deep`, `/health`, and LaunchAgent PID

Why it matters:
- this is a lifecycle hiccup, not necessarily an update failure
- do not rerun the update just because `launchctl stop` needed fallback; first verify the final LaunchAgent is loaded/running and the gateway responds on its configured port
- include the stop/fallback lines in maintainer reports, because they show launchd semantics the updater had to recover from

## 26. Selected update channel has no matching external plugin release

Symptom:
- host updates successfully to a selected channel such as beta
- an external plugin cannot be found on npm for `<package>@<channel>`
- updater falls back to another tag such as `@latest`
- post-update `plugins doctor` can still be clean, but the host is now running a mixed channel cohort

Observed example:
- host updated from `2026.5.7` to `2026.5.10-beta.3`
- one third-party plugin had no `@beta` release and fell back to `@latest`
- a globally installed official channel plugin did update to the matching beta version
- plugin peer dependency links were repaired during the update

What to inspect:
- update output for `Package not found on npm` and `falling back` lines
- `openclaw plugins list --json` for each enabled plugin's `origin`, `source`, and `version`
- `~/.openclaw/plugins/installs.json` to see whether the recorded spec now points at the intended version/tag

Why it matters:
- "host on beta" does not imply every external plugin is on beta
- a clean `plugins doctor` proves loadability, not cohort consistency
- maintainer-facing reports should separate OpenClaw-bundled plugin behavior from external plugin publishing gaps

## 27. Transient post-restart scope and pricing warnings can coexist with healthy channels

Symptom:
- immediately after restart, gateway logs show websocket responses like `missing scope: operator.read`
- `status --deep` reports `Model pricing` degraded because an external pricing fetch failed
- channels remain connected and `/health` reports live

What to inspect:
- whether the scope errors are confined to the seconds after restart
- whether later `channels status --deep` is connected
- whether `/health` returns `{"ok":true,"status":"live"}`
- whether the pricing warning is tied to a third-party pricing fetch rather than local gateway startup

Why it matters:
- these warnings are useful to report, but they should not be conflated with a failed package update
- stale Control UI/websocket clients can race the new gateway during restart
- external pricing fetch failures may degrade status without affecting channel delivery or plugin loading
