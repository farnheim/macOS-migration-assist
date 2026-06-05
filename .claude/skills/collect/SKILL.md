---
name: collect
description: Run on the SOURCE Mac. Reconcile the source→target macOS version and architecture, get explicit permission, then inventory system settings, apps, Homebrew and VST/AU plugins into ./manifests/. This is the FIRST step, before /verify and /migrate. Use when the user wants to start a migration, take inventory, or "collect" the old machine.
---

# /collect — inventory the source Mac

You orchestrate the first migration step on the **source** machine. Obey `.claude/rules.md` strictly: explicit permission, no user documents, no secrets, read-only on the source.

## 1. Permission (rule 1) — do this before any system-reading command
State plainly what WILL be read (installed `.app`s, Homebrew formulae/casks/taps, VST/AU plugins, `defaults`, security state) and what will NOT be touched (documents, Keychain, `~/.ssh`, `.env`, password databases, browser history). Get an explicit yes. Do not proceed otherwise.

## 2. Detect the source system
Run: `sw_vers; uname -m; system_profiler SPHardwareDataType | grep -E "Model Name|Chip|Memory"`. Keep the `grep` — raw `system_profiler` leaks the serial number / Hardware UUID, which must never be stored (rule 5).

## 3. Reconcile source → target (the core of /collect)
We are on the source; we cannot inspect the target. Ask the user with **AskUserQuestion**:
- **Target macOS version** (same as source / a specific newer version).
- **Target architecture**: Apple Silicon (arm64) or Intel (x86_64).
- (optional) Target model, if known.

Then compute and surface compatibility notes:
- **Arch mismatch** (e.g. source arm64 → target x86_64): native binaries (VST/AU `copy`, some casks) may not transfer; flag this prominently — `/migrate` will hard-check it in preflight.
- **Target OS older than source**: some app/cask versions may be unavailable; note affected items.
- **Deprecated casks** (e.g. `chromium-gost`, `moebius` — disabled 2026-09-01): warn if the target will be set up past the cutoff.

## 4. Write migration-context.json
Write `manifests/migration-context.json`:
```json
{
  "schema_version": "1.1",
  "generated_at": "<ISO-8601>",
  "source": { "os": "...", "build": "...", "arch": "...", "model": "..." },
  "target": { "os": "...", "arch": "...", "model": "...|null" },
  "reconciliation": ["<warning strings: arch mismatch, OS gap, deprecated casks, ...>"]
}
```

## 5. Collect (spawn system-analyst)
Launch the **`system-analyst`** agent (Agent tool) to inventory the four groups and write `macos-settings.json`, `homebrew.json`, `applications.json`, `vst.json` + `Brewfile`, each item carrying `apply`/`verify`/`expected`/`install_method`/`dependencies`/`requires_runtime` and a heuristic `selected`. If the agent isn't loaded yet (created/edited this session — needs a restart), execute its documented procedure inline instead.

> `system-analyst` emits only *heuristic* `selected` (subagents can't prompt — see CLAUDE.md design note); the human confirms in **/verify** next.

## 6. Hand off
Summarize counts per category, list what needs manual binaries/licenses (vst-payload, ICC, proprietary installers), and tell the user the next step is **/verify** to confirm exactly what migrates.
