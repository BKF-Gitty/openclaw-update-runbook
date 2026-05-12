# OpenClaw Update Runbook

An operator-focused skill and reference pack for updating OpenClaw, debugging post-update regressions, and proving which layer is actually broken before making changes.

This is designed for people using Codex or Claude-style agent workflows to maintain OpenClaw hosts. It turns the update/debug process into a repeatable audit instead of a guessy rescue mission.

Important: the repository is only the distribution point. To use this as a real skill, install or copy the whole folder locally so `SKILL.md` and `references/failure-patterns.md` stay together. The skill expects those files to exist side by side on disk.

## What it includes

- `SKILL.md`: the main update runbook skill
- `references/failure-patterns.md`: concrete regression patterns seen across multiple hosts
- `agents/openai.yaml`: lightweight agent metadata for Codex skill use

## What it helps with

- confirming the real service state after an update
- separating service-manager issues from detached-process issues
- spotting bundled-vs-global plugin drift
- finding stale config that survives upgrades
- checking whether plugin install records match disk reality
- reading the right logs before changing too much
- cleaning up task ledger problems that keep a host noisy or half-broken

## Who this is for

- operators maintaining one or more OpenClaw hosts
- people helping friends or teams recover after an update
- anyone who wants a structured checklist for "OpenClaw feels broken after upgrade"

## Install

Clone or download this repository, then place the folder where your Codex skills live, or install it from the repo if your Codex setup supports repo-based skill installs.

Do not copy only `SKILL.md`. The skill refers to local companion files under `references/` and `agents/`.

The main entry point is:

- `SKILL.md`

## How to use

Invoke the skill when you are:

- upgrading OpenClaw
- checking health right after an upgrade
- debugging a host that became slow, disconnected, or inconsistent after update

The agent should start with `SKILL.md` and open `references/failure-patterns.md` only when the main workflow points to a known regression pattern or contradictory runtime symptoms.

The runbook is intentionally conservative: verify service reality first, inspect plugin and config drift second, then apply the smallest fix that makes the host consistent again.

## Maintenance model

This runbook is cumulative. New host-specific lessons should usually be added to `references/failure-patterns.md` rather than replacing existing guidance.

Contribution style:

- additive over destructive
- preserve older patterns unless they are clearly wrong
- keep examples generic and free of secrets or personal host details

## Suggested repository description

Structured OpenClaw update runbook for Codex/Claude operators: service checks, plugin drift, config regressions, task cleanup, and post-upgrade debugging.
