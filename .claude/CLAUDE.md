# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

Not a software codebase — a **macOS migration toolkit** built from three user-facing skills that orchestrate two Claude Code subagents, all handing off through a shared file contract. The goal: selectively move settings, apps, Homebrew packages and VST/AU plugins from an old Mac to a new one, with versions pinned and verified.

There is no build/lint/test step. The "programs" are the three skills in `.claude/skills/`, the two agent definitions in `.claude/agents/`, and the JSON manifests they read and write.

## Architecture: two agents, one manifest contract

The entire design hinges on a single decoupling point — the `./manifests/` directory:

- **`system-analyst`** (`.claude/agents/system-analyst.md`) runs on the **source** Mac. Read-only against the system; it only writes files. It inventories four groups, applies heuristic `selected` flags, and emits the manifests.
- **`migration-assist`** (`.claude/agents/migration-assist.md`) runs on the **target** Mac. It reads the manifests and installs/verifies only items with `"selected": true`.

The two agents never call each other. They communicate solely through the manifests, which are copied (along with payload binaries) from the old machine to the new one.

### The manifest contract (keep both agents in sync with it)

Four files in `./manifests/`, one per migration group:

| File | Group |
|------|-------|
| `macos-settings.json` | `defaults`, display/ICC profiles, security state |
| `homebrew.json` | formulae / casks / taps (mirrored by `Brewfile`) |
| `applications.json` | `.app` bundles, Mac App Store apps |
| `vst.json` | VST / VST3 / AU plugins |

Every file is `{ schema_version, category, generated_at, source, items[] }` (schema_version is now `1.1`). The load-bearing idea is that **each item is self-describing** — it carries its own `apply`/`verify`/`expected`/`install_method`/`dependencies`/`requires_runtime` etc. — so `system-analyst` decides *how* to install and verify and `migration-assist` just runs `apply` then `verify` (in dependency order) and diffs against `expected`. The authoritative field list lives in **`rules.md` §4** and the schema block of **both agent files** (each agent restates it because they load independently — accepted drift risk). **If you change a field name or `install_method` value, update both agent files and rules.md §4** — do not re-enumerate the fields here.

A fifth file, `manifests/migration-context.json`, is written by `/collect`: it records the reconciled `source` and intended `target` (os/arch/model) plus compatibility warnings, and `/migrate` checks the target against it in preflight.

**Brewfile = the verified install set.** `manifests/Brewfile` is not a blind `brew bundle dump`; it is the *projection of the `selected: true` items* whose `install_method ∈ {tap, formula, cask, mas}`, aggregated across **both** `homebrew.json` and `applications.json` (app casks live in the latter). `/verify` regenerates it whenever the selection changes, and `/migrate` installs all of Homebrew + MAS with one idempotent `brew bundle --file=manifests/Brewfile` (Homebrew resolves its own internal deps/order) instead of looping per-item. Per-item `apply` (`brew install …`) stays in the manifests as the fallback/individual path; the per-item `verify`/`expected` still drives the verification report. `manual`/`copy`/`defaults`/`system` items are never in the Brewfile — they stay per-item or checklist.

`install_method: manual` means there is no `apply` command (proprietary installer / license required); those become a human checklist, not automation. `copy` items need their binaries staged in payload dirs **created on demand** — `manifests/vst-payload/{VST,VST3,Components}/` for plugins, and `manifests/colorsync/` only when custom ICC profiles exist (this source has none). The dirs are not pre-created or kept empty; the JSON references files that must travel with the manifest.

## Running the workflow (three skills orchestrate the two agents)

User-facing entry points live in `.claude/skills/` and are invoked as slash commands. They are the orchestration layer; the two agents are the workers they spawn.

1. **`/collect`** (source Mac) — gets permission, reconciles source→target macOS version/arch, writes `manifests/migration-context.json`, then spawns `system-analyst` to produce the four manifests + `Brewfile` with heuristic `selected`.
2. **`/verify`** (source Mac) — runs in the main conversation and walks each manifest category with the user, confirming exactly what migrates ("select all" / customize per group), enforcing dependency consistency, and writing `selected` back.
3. Copy the whole `manifests/` dir (including any payload subdirs you staged) to the new Mac.
4. **`/migrate`** (target Mac) — preflights arch/OS against `migration-context.json`, then spawns `migration-assist` (script-gen or interactive) which installs Homebrew + MAS in one `brew bundle --file=manifests/Brewfile` pass, then manual/settings/VST in dependency order, verifies each selected item against `expected`, and writes `manifests/migration-report.md`.

**Design note — subagents are non-interactive:** `AskUserQuestion` is unavailable inside subagents, so **all** human-in-the-loop prompts live in the main-thread skills, never in the agents. `system-analyst` only emits heuristic `selected`; `/verify` does the reconciliation. `migration-assist` likewise does not prompt — `/migrate` collects the mode choice and any `sudo`/overwrite confirmations in the main thread and passes them in; the agent's default, safe mode is to generate `install.sh`/`verify.sh` (changes nothing). Neither agent declares `AskUserQuestion` in its `tools`.

Both skills (`.claude/skills/`) and agents (`.claude/agents/`) are loaded at session start, so **after creating or editing one you may need to restart Claude Code before the Skill/Agent tool can invoke it.** Until then, the fallback is to execute the documented procedure inline.

## Prior art & positioning

Existing tools and the primitives we reuse:
- **Homebrew `brew bundle` + `Brewfile`** — the standard for packages/casks/MAS; our `homebrew.json`/`Brewfile` is exactly this slice.
- **`mas`** — Mac App Store CLI; our `install_method: mas`.
- **Mackup** (`lra/mackup`) syncs app *settings* but its symlink "link mode" **breaks preferences on macOS 14+** (only copy mode is safe). We never symlink — selective per-key `defaults write` + `verify` is the correct modern approach.
- **chezmoi / GNU Stow / nix-darwin** — dotfile/declarative end; all-or-nothing reproducibility.

Those tools are broad and automatic (dump everything, restore everything). **This toolkit is deliberately selective, verified, and consent-gated**: per-item `selected`, `verify` vs `expected`, dependency ordering, and a human `/verify` gate. That niche — curating *which* of a musician's licensed audio apps/plugins + dev tools move — is the reason to build rather than just `brew bundle`.

## Environment gotchas (this project hit them)

- The shell is **zsh**, which does **not** word-split unquoted variables. Iterating a space-separated list with `for x in $var` runs the loop once with the whole string. Use arrays: `arr=(a b c); for x in "${arr[@]}"`.
- App versions come from `defaults read "<App>.app/Contents/Info.plist" CFBundleShortVersionString`; formula versions from `brew list --versions <name>`; cask existence from `brew info --cask <slug>` (exit 0 = exists).
- Cask name ≠ app name, and some apps are lookalikes that must stay `manual`: e.g. **Chromium-Gost** is a GOST-TLS fork, not the `chromium` cask.
- The target architecture (`uname -m`) must match `source.arch` before installing native binaries (VST/AU); `migration-assist` checks this in preflight.
- UAD plugins (`uaudio_*`) depend on the **UA Connect** app plus licenses — their `selected` state should track UA Connect, and they can't be plain-copied.
- `migration-assist` must never run `sudo`, overwrite existing settings/files, or auto-toggle security (SIP/FileVault/firewall) without explicit confirmation; those are checklist items.
