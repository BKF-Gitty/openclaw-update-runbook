---
name: openclaw-update-runbook
description: Use when updating OpenClaw or debugging an OpenClaw instance after an update. This skill acts as a structured update runbook with emphasis on gateway startup, launchd state, plugin registry and install drift, bundled-vs-npm/clawhub plugin confusion, stale config carried across upgrades, channel health, task ledger corruption, and logs that explain why the updated system is slow, disconnected, or half-broken.
---

# OpenClaw Update Runbook

Use this skill when an OpenClaw host was just updated, is about to be updated, or is behaving strangely after an update.

The goal is not only to get it running, but to prove which layer is broken:

- service lifecycle
- host package version
- plugin/package compatibility
- config drift
- channel health
- task ledger health
- runtime performance

## Quick workflow

1. Establish the real starting state.
   Check:
   - `openclaw --version`
   - `openclaw status --deep`
   - `openclaw doctor --non-interactive --no-workspace-suggestions`
   - `openclaw channels status --deep`
   - `openclaw tasks audit`

2. Verify the gateway is actually managed correctly.
   Look at launchd state, running PID, and `/health`.
   Do not trust only one of:
   - `launchctl`
   - process list
   - health endpoint

   It is common to have:
   - a plist present but not loaded
   - a detached gateway process still serving traffic
   - launchd and the live process disagreeing

3. Separate bundled plugins from globally installed plugins.
   First inspect plugin health:
   - `openclaw plugins doctor`
   - `openclaw plugins list --json`
   - `openclaw plugins inspect <id>`

   Important rule:
   - If a capability is supposed to be bundled, verify whether a stale global npm install is shadowing it.
   - If a capability is not bundled, check npm and ClawHub before assuming config is wrong.

4. Check for config carried across the upgrade that no longer validates.
   Pay attention to:
   - `tools.web.search.provider`
   - `plugins.allow`
   - `plugins.entries.*`
   - model aliases and fallback chains
   - update channel metadata

   If doctor says a provider or plugin is unknown, inspect the actual config file and do not assume `doctor --fix` fully cleaned it.

5. Compare plugin install records to what exists on disk.
   Inspect:
   - `~/.openclaw/plugins/installs.json`
   - `~/.openclaw/npm/node_modules/@openclaw/...`
   - `~/.openclaw/extensions/...`

   Look for:
   - recorded install paths that do not exist
   - recorded versions drifting from installed versions
   - source-only TypeScript plugin packages with no compiled `dist/`
   - plugin runtime deps removed from third-party plugin directories

6. Inspect recent gateway logs before changing too much.
   Read:
   - `~/.openclaw/logs/gateway.log`
   - `~/.openclaw/logs/gateway.err.log`
   - `/tmp/openclaw/openclaw-YYYY-MM-DD.log`

   Prioritize recent startup lines and warnings involving:
   - plugin load failures
   - config validation
   - channel auth
   - context-engine fallback
   - active-memory timeouts
   - event loop degradation
   - task restart blocking

7. Audit runtime/task health after the upgrade.
   Check for:
   - stale running tasks
   - lost tasks
   - delivery failures
   - timestamp inconsistencies

   A successful package update can still leave the system unhealthy if stale tasks block restarts or keep the audit red.

8. Re-run the narrowest fix, then verify again.
   Common fix sequence:
   - stop gateway cleanly
   - update host package
   - refresh plugin registry if needed
   - repair or update broken plugin installs
   - restart gateway
   - re-run `doctor`, `plugins doctor`, `status --deep`, `channels status --deep`, and `tasks audit`

## Where to look first

Use this order when diagnosing post-update failures:

- Service state: launchd, PID, `/health`
- Host version: `openclaw --version`
- Plugin mismatch: `openclaw plugins doctor`
- Config drift: `openclaw doctor`
- Channel reality: `openclaw channels status --deep`
- Task ledger: `openclaw tasks audit`
- Runtime symptoms: gateway logs

## When to open references

Start with this file first.

Open [references/failure-patterns.md](references/failure-patterns.md) when:

- `doctor` or `plugins doctor` points to a known-looking regression
- `channels status` or logs disagree with the apparent service health
- plugin installs, install records, or config state do not match what is on disk
- the update completed, but the host is still slow, disconnected, noisy, or half-broken

Use the reference file for symptom matching and concrete examples after the main workflow has narrowed the likely failure area.

## Bundled vs external plugin rule

Do not assume a broken plugin means "plugin missing."

There are three common cases:

- Bundled plugin exists in the host package, but stale config still points at an old provider/plugin id.
- Bundled plugin exists, but a globally installed npm plugin shadows it and is on the wrong version.
- Plugin is not bundled, so the fix is to inspect npm or ClawHub and reconcile install records.

Discord is a good example of the second case: a host can upgrade correctly while still loading an older globally installed `@openclaw/discord` plugin.

If the feature is not bundled, check npm and ClawHub before rewriting config.

## Fixing mindset

Prefer the smallest fix that makes state consistent again:

- refresh registry before reinstalling everything
- update one stale plugin before removing all plugins
- inspect the actual config file when helper commands appear to succeed but warnings remain
- verify whether a third-party plugin needs local runtime deps before deleting plugin-side `node_modules`

Do not stop at "service is up." A good finish means:

- the right version is installed
- the gateway is managed correctly
- channels are connected
- plugin doctor is clean or explained
- task audit is not carrying a fresh blocking error

## Maintainer notes

If the upgrade exposed an OpenClaw bug rather than local drift, collect:

- exact version before and after
- relevant config keys
- plugin source path actually loaded
- whether the plugin was bundled or globally installed
- `doctor`/`plugins doctor` warning text
- the specific log lines around startup failure or restart

For concrete regression patterns and example symptoms, read [references/failure-patterns.md](references/failure-patterns.md).

## Updating this skill

When another operator or agent learns something new from a different OpenClaw host:

- do not delete existing workflow steps unless they are clearly wrong
- do not replace an existing failure pattern with a narrower one
- prefer additive updates over rewrites
- add new regression patterns to `references/failure-patterns.md`
- only tighten the main workflow in this file if the new lesson changes the recommended audit order for most hosts

If a new issue is host-specific or uncertain, add it as a new failure pattern with:

- symptom
- what to inspect
- why it matters

Do not silently erase older patterns just because the current host did not hit them.
