# macOS Migration Assist

> 🚧 **Under construction.** This project is a work in progress — skills, agents and the manifest schema
> may still change. Use with care and review what it does before running it on a real machine.

A **selective, verified, consent-gated macOS migration toolkit**. It moves the parts of an old Mac that
actually matter — system settings, applications, Homebrew packages and VST/AU plugins — onto a new Mac,
with every item's version pinned and verified on arrival.

It is *not* a "dump everything / restore everything" tool. Unlike `brew bundle dump`, Mackup or
nix-darwin, every item is curated: you confirm exactly what migrates, dependencies are installed in order,
and each install is checked against an expected version. The niche it's built for: curating *which* of a
musician's licensed audio apps/plugins plus a developer's toolchain should follow them to a new machine.

> This is not a conventional codebase — there is no build/lint/test step. The "programs" are **three skills**
> (slash commands), **two subagents**, and the **JSON manifests** they read and write.

---

## Running the workflow

Three skills orchestrate the two agents. Run them as slash commands in Claude Code.

### 1. `/collect` — inventory the source Mac
On the **old** machine. Asks explicit permission, detects the source OS/arch, then **reconciles
source → target** by asking you the target's macOS version and architecture (you can't inspect the new Mac
from the old one). Writes `migration-context.json`, then spawns `system-analyst` to produce the four
manifests + `Brewfile` with a heuristic `selected`.

### 2. `/verify` — confirm what migrates
On the **old** machine, in the main conversation (where prompts work). Walks each category
(`macos-settings → homebrew → applications → vst`), showing every item grouped by vendor with badges for
dependencies (⛓), runtimes (⚙), manual installs (✍) and copy-payloads (📦). You pick **Select all /
Keep current / Customize / Skip all** per category. It enforces dependency consistency (a selected item
can't have an unselected dependency), writes `selected` back, and **regenerates the `Brewfile`** to match.

### 3. Copy `manifests/` to the new Mac
Including any payload subdirs you staged (`vst-payload/`, `colorsync/`).

### 4. `/migrate` — apply on the target Mac
On the **new** machine. Preflights arch/OS against `migration-context.json` (a native-binary arch mismatch
warns hard). Then spawns `migration-assist`, which:

1. installs all Homebrew + Mac App Store items in **one** idempotent pass: `brew bundle --file=manifests/Brewfile`;
2. emits a **manual checklist** for proprietary apps (no automation possible);
3. applies `defaults` settings, copies ICC profiles / VST payloads;
4. runs each selected item's `verify`, diffs against `expected`, and writes `manifests/migration-report.md`.

Its **default mode is safe**: it assembles `install.sh` + `verify.sh` for you to run rather than touching
the system. Interactive driving (with `sudo`/overwrite confirmations) only happens when the `/migrate`
skill explicitly drives it from the main thread.

---

## The manifest contract

Four files in `./manifests/`, one per migration group, each shaped:

```json
{
  "schema_version": "1.1",
  "category": "macos-settings | homebrew | applications | vst",
  "generated_at": "<ISO-8601>",
  "source": { "os": "macOS <x.y.z>", "build": "<build>", "arch": "arm64", "model": "<Mac model>" },
  "items": [ /* self-describing items, see below */ ]
}
```

The load-bearing idea: **each item carries its own install and verify commands.** `system-analyst` decides
*how*; `migration-assist` just runs `apply` then `verify` and diffs against `expected`.

| Field | Meaning |
|-------|---------|
| `id` | unique kebab-case id (referenced by `dependencies`) |
| `name` | human-readable name |
| `version` | exact version on the source, or `null` |
| `selected` | whether this item migrates (heuristic from `/collect`, finalized in `/verify`) |
| `install_method` | `cask` · `mas` · `formula` · `tap` · `manual` · `copy` · `defaults` · `system` |
| `apply` | shell command that installs/applies it, or `null` (for `manual`/`system`) |
| `verify` | shell command printing the actual installed version/state, or `null` |
| `expected` | the value `verify` should print on success |
| `post_apply` | command to run after `apply` (e.g. `killall Dock`), or `null` |
| `manual_url` | installer/login link for `manual` items |
| `notes` | license, warnings, caveats |
| `dependencies` | array of `id`s that must be installed **before** this item (`[]` if none) |
| `requires_runtime` | a critical runtime/version the target must provide (e.g. `node@24.10.0`, `java>=17`), or `null` |
| `group` | logical group (vendor/set), e.g. a plugin vendor or app category |
| `size_hint` | reserved, usually `null` |

> The authoritative field list lives here, in `rules.md` §4, and in the schema block of **both** agent
> files (each agent restates it because they load independently). Changing a field name or `install_method`
> value means updating all of those.

### `install_method` decides what happens on the target

| Method | On the target |
|--------|---------------|
| `formula` / `cask` / `mas` / `tap` | installed via the **Brewfile** in one `brew bundle` pass |
| `manual` | no `apply` — becomes a human checklist (proprietary installer / license required) |
| `defaults` | `defaults write …` then `post_apply` |
| `copy` | binaries copied from a staged payload dir (`vst-payload/`, `colorsync/`) |
| `system` | informational only (SIP/Gatekeeper/FileVault/firewall) — never toggled automatically |

---

## Manifest examples

Illustrative items in the shape `system-analyst` produces (notes trimmed, machine-specific values genericized — the real manifests are generated locally and git-ignored).

### `homebrew.json` — a formula and a cask
```json
{
  "id": "formula-node",
  "name": "node",
  "version": "24.10.0",
  "selected": true,
  "install_method": "formula",
  "apply": "brew install node",
  "verify": "brew list --versions node",
  "expected": "node 24.10.0",
  "requires_runtime": "node@24.10.0",
  "notes": "Pinned 24.10.0 for reproducibility; Homebrew may install a newer major. PROVIDES a runtime other tooling may depend on.",
  "dependencies": [],
  "group": "Homebrew Formulae"
}
```
```json
{
  "id": "cask-vlc",
  "name": "vlc",
  "version": "3.0.20",
  "selected": true,
  "install_method": "cask",
  "apply": "brew install --cask vlc",
  "verify": "brew list --cask --versions vlc",
  "expected": "vlc 3.0.20",
  "notes": "Some casks need system-extension approval after install (reboot + allow in Privacy & Security).",
  "group": "Homebrew Casks"
}
```

### `applications.json` — a proprietary app (manual)
A cask exists but installs the wrong major version, so it's kept `manual`. There is no `apply`; `verify`
still checks the app's bundle version once you've installed it by hand.
```json
{
  "id": "app-example-daw",
  "name": "ExampleDAW Pro 3",
  "version": "3.4.1",
  "selected": true,
  "install_method": "manual",
  "apply": null,
  "verify": "defaults read \"/Applications/ExampleDAW Pro 3.app/Contents/Info.plist\" CFBundleShortVersionString",
  "expected": "3.4.1",
  "manual_url": "https://vendor.example/login",
  "notes": "A cask exists but installs the wrong major version (4.x), so this pinned 3.x is kept manual. Requires a vendor account + license authorization.",
  "dependencies": [],
  "group": "Audio / DAW"
}
```

### `macos-settings.json` — a `defaults` key and a `system` (security) item
```json
{
  "id": "keyboard-key-repeat",
  "name": "Keyboard: KeyRepeat",
  "selected": true,
  "install_method": "defaults",
  "apply": "defaults write -g KeyRepeat -int 2",
  "verify": "defaults read -g KeyRepeat",
  "expected": "2",
  "notes": "Fast key repeat rate. Takes effect after logout/login.",
  "group": "Keyboard"
}
```
```json
{
  "id": "security-sip",
  "name": "System Integrity Protection (SIP)",
  "selected": false,
  "install_method": "system",
  "apply": null,
  "verify": "csrutil status",
  "expected": "System Integrity Protection status: enabled.",
  "notes": "Informational. SIP is enabled by default on the target; changing it requires Recovery Mode. Never toggled automatically.",
  "group": "Security"
}
```

### `vst.json` — a copyable plugin bundle
Self-contained plugins are `copy` (binaries staged in `vst-payload/`):
```json
{
  "id": "vst-examplefx-bundle",
  "name": "ExampleFX reverb bundle",
  "selected": true,
  "install_method": "copy",
  "apply": "for f in ExampleVerb ExampleDelay; do cp -R \"$HOME/migration/vst-payload/VST3/$f.vst3\" \"/Library/Audio/Plug-Ins/VST3/\"; done",
  "verify": "test -e \"/Library/Audio/Plug-Ins/VST3/ExampleVerb.vst3\" && echo present",
  "expected": "present",
  "manual_url": "https://vendor.example/",
  "notes": "Self-contained, copyable. Stage binaries into manifests/vst-payload/{VST,VST3,Components}/. Native arm64 — arch must match target. Paid plugins may still need license entry on first launch.",
  "dependencies": [],
  "group": "ExampleFX"
}
```
Installer-based or license-gated plugins instead use `install_method: manual` (no `apply`) and can declare a
cross-manifest `dependencies` entry pointing at the companion app that installs/licenses them.

---

## `Brewfile` — the verified install set

The `Brewfile` is **not** a blind `brew bundle dump`. It is the *projection of the `selected: true` items*
whose `install_method ∈ {tap, formula, cask, mas}`, aggregated across **both** `homebrew.json` and
`applications.json` (app casks live in the latter). `/verify` regenerates it whenever the selection changes,
and `/migrate` installs everything with one idempotent `brew bundle --file=manifests/Brewfile`. Per-item
`apply` (`brew install …`) stays in the manifests only as the fallback/individual path.

```ruby
# Brewfile — VERIFIED projection of selected items (install_method ∈ tap|formula|cask|mas)
# Install on the target with:  brew bundle --file=manifests/Brewfile

brew "node"
brew "ffmpeg"

cask "visual-studio-code"
cask "vlc"
# mas "App Name", id: 123456789   # Mac App Store items, when present
```

---

## `migration-context.json`

Written by `/collect`, checked by `/migrate` in preflight. It records the reconciled source and the
intended target plus compatibility warnings (arch match/mismatch, OS gaps, deprecated casks, runtimes the
target must provide):

```json
{
  "schema_version": "1.1",
  "source": { "os": "macOS <x.y.z>", "arch": "arm64", "model": "<Mac model>" },
  "target": { "os": "macOS 15 (Sequoia)", "arch": "arm64", "model": null },
  "reconciliation": [
    "ARCH MATCH: source arm64 -> target arm64. Native binaries transfer without recompilation.",
    "OS FORWARD: target newer than source — generally forward-compatible.",
    "RUNTIMES TARGET MUST PROVIDE: java>=17, plus any vendor runtimes/drivers/hardware tokens your manual apps need."
  ]
}
```

---

## Privacy & safety (`.claude/rules.md`)

These rules are mandatory for both agents and all three skills, and take priority over convenience:

- **Explicit permission** before any system read; permission to *read* never implies permission to *write*.
- **No user documents.** Only installed apps/components and their metadata (name, version, bundle path) are
  read — never `~/Documents`, mail, messages, browser history, iCloud, media files.
- **No secrets.** Keychain, `~/.ssh`, `~/.gnupg`, `.env`, `*.p12/.pem/.key`, password databases are
  off-limits. Licenses are recorded only as *a fact of being required* — keys never travel; you activate
  manually on the target.
- **No serial number / Hardware UUID** stored (`system_profiler` is always filtered to Model/Chip/Memory);
  user paths normalized to `$HOME`, not `/Users/<name>`.
- **Read-only on the source.** `system-analyst` writes nothing but JSON into the project dir.
- **No `sudo`, no overwrites, no security toggles** without explicit consent. SIP/Gatekeeper/FileVault/
  firewall are recorded informationally and left as a manual checklist.
- **No symlinking of preferences** — macOS 14+ breaks symlinked prefs (this is why Mackup's "link mode" is
  unsafe). Per-key `defaults write` + `verify` is the correct modern approach.

To reduce prompt fatigue you can keep a standing allow-list in `.claude/settings.local.json` (personal and
git-ignored) that pre-approves the read-only inventory commands while leaving `sudo` / `brew install` /
`mas install` / `defaults write` interactive. The recommended allow/ask split is spelled out in
`.claude/rules.md` §1.

---

## Project layout

```
.
├── README.md                         ← you are here
├── .claude/
│   ├── CLAUDE.md                     architecture overview for Claude Code
│   ├── rules.md                      data-collection & privacy rules (authoritative)
│   ├── settings.local.json           personal permission allow-list (git-ignored)
│   ├── agents/
│   │   ├── system-analyst.md         source: inventory → manifests
│   │   └── migration-assist.md       target: install + verify
│   └── skills/
│       ├── collect/SKILL.md          /collect
│       ├── verify/SKILL.md           /verify
│       └── migrate/SKILL.md          /migrate
└── manifests/
    ├── templates/                    tracked — schema skeletons to author/inspect from
    │   ├── *.template.json
    │   └── Brewfile.template
    ├── migration-context.json        ┐
    ├── macos-settings.json           │
    ├── homebrew.json                 │  generated by /collect + /verify,
    ├── applications.json             │  git-ignored (personal inventory)
    ├── vst.json                      │
    ├── Brewfile                      ┘
    ├── vst-payload/{VST,VST3,Components}/   staged plugin binaries (on demand, ignored)
    └── colorsync/                           staged custom ICC profiles (on demand, ignored)
```

> **Git:** the real manifests are a personal software inventory of one machine, so `.gitignore` excludes
> them (and the payload dirs + generated scripts/report). Only `manifests/templates/` is committed — that's
> the scaffold for anyone using the agents. `/collect` + `/verify` regenerate the real files locally.

## Prior art & positioning

Built on standard primitives — Homebrew `brew bundle` + `Brewfile`, the `mas` Mac App Store CLI, per-key
`defaults` — but deliberately the *opposite* of all-or-nothing tools like Mackup, chezmoi, GNU Stow or
nix-darwin. Those dump and restore everything; this toolkit is **selective, verified and consent-gated**:
per-item `selected`, `verify` vs `expected`, dependency ordering, and a human `/verify` gate.
