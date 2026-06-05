---
name: migrate
description: Run on the TARGET (new) Mac after copying the manifests/ directory over. Preflight-checks architecture and OS against the migration context, then installs and verifies only the confirmed (selected) items in dependency order via the migration-assist agent, producing migration-report.md. Use when the user is on the new machine and wants to apply, install, or run the migration.
---

# /migrate — apply the plan on the new Mac

Runs on the **target** machine. Obey `.claude/rules.md`: never run `sudo` or overwrite existing files/settings without explicit confirmation, never auto-toggle security (SIP/FileVault/firewall), and install respecting `dependencies` order.

## 1. Preconditions
Require `manifests/` with the four JSON files (plus payload dirs if any `copy` items are selected). If absent, tell the user to run **/collect** and **/verify** on the source and copy the whole `manifests/` dir here, then stop.

## 2. Preflight reconciliation
Read `manifests/migration-context.json` (if present) and each manifest's `source`. Run `sw_vers; uname -m`. Compare the current machine to:
- **`source.arch`** — a mismatch blocks native binaries (VST/AU `copy`, some casks); warn hard before doing anything.
- **`context.target`** — confirm we are on the intended target; note any deviation.
Report mismatches up front.

## 3. Run migration-assist
Launch the **`migration-assist`** agent (Agent tool). It will:
- check `brew` / `mas` availability;
- install in **dependency order** — taps → formulae → casks → mas → manual checklist → macOS settings → VST/AU — skipping any item whose `dependencies` are not satisfied and checking `requires_runtime` first;
- run each selected item's `verify` and diff against `expected` (OK / DIFF / MISSING / SKIP);
- write `manifests/migration-report.md`.

**Mode choice happens here, in the main thread** (the agent is a subagent and cannot prompt — see CLAUDE.md design note). Ask the user with AskUserQuestion:
- **Generate scripts (default, safe):** spawn `migration-assist` to assemble `install.sh` + `verify.sh` (dependency-ordered, idempotent, non-zero exit on diffs) for the user to run. Nothing on the system is changed.
- **Drive interactively:** you (the skill) walk the phases in the main thread, asking confirmation before any `sudo` / overwrite / security toggle, using the per-item commands from the manifest (and `migration-assist` for script/verify generation).
Pass the chosen mode to the agent in its prompt. If the agent isn't loaded yet (created/edited this session — needs a restart), execute its documented procedure inline.

## 4. Wrap up
Relay the report summary: installed / verified / diffs / left-manual, plus the checklist of licenses, accounts, hardware tokens and copy-payloads that still need human action.
