# rules.md — Data Collection & Privacy Rules

These rules are mandatory for the three skills (`/collect`, `/verify`, `/migrate`), both agents
(`system-analyst`, `migration-assist`), and any work touching the user's system in this project.
They take priority over convenience and speed.

## 0. Workflow & enforcement points

- **`/collect`** (source) requests permission (rule 1) before reading anything, reconciles source→target
  OS/arch, and inventories — strictly read-only (rule 5).
- **`/verify`** (source) is the human-in-the-loop gate: the user confirms the selection there, and
  dependency consistency (rule 4) is enforced. It runs in the main conversation because `AskUserQuestion`
  does not work inside subagents.
- **`/migrate`** (target) applies only confirmed items, never with `sudo` / overwrite / security toggles
  without explicit consent (rule 5), and installs respecting `dependencies` order.

## 1. Explicit permission to collect and analyze

- Before any inventory or system analysis, the agent **must obtain the user's explicit permission**.
- Permission is requested **before** running commands that read the system (`defaults read`,
  `system_profiler`, `brew`, scanning `/Applications`, plugins, etc.).
- Permission applies only to the session and scope described to the user. Expanding the scope
  (new directories, new data types) requires fresh confirmation.
- Any action that modifies the system is requested separately — permission to **read** does not grant
  permission to **write**.

### Permission model (settings.local.json)
To avoid prompt fatigue, the harness grants a standing allow-list once (in `.claude/settings.local.json`,
which is git-ignored — personal to each machine) instead of asking per action. This is a *tooling* convenience and does NOT replace the *conversational*
consent in this rule — `/collect` still asks the user once, in plain language, before it starts reading.
- **Pre-allowed (no prompt):** file `Read`/`Edit`/`Write` and read-only inventory commands
  (`sw_vers`, `uname`, `system_profiler`, `defaults read`, `brew list/info/leaves/tap/bundle dump`,
  `mas list`, `ls`, `test`, `csrutil status`, `spctl`, `fdesetup status`, `cp`, `mkdir`).
- **Always prompt (`ask`):** `sudo`, `brew install`, `mas install` — these stay interactive so rule 5
  (no silent installs, no overwrites, no security toggles) holds. `defaults write` is deliberately NOT
  pre-allowed, so applying macOS settings on the target still surfaces.

## 2. User documents are not analyzed

- Only **installed applications and components** are analyzed: `.app` bundles, Homebrew
  formulae/casks/taps, VST/VST3/AU plugins, system settings (`defaults`), profiles, and security state.
- It is **forbidden** to read, index, or include in manifests the contents of user data:
  `~/Documents`, `~/Desktop`, `~/Downloads`, `~/Pictures`, `~/Movies`, `~/Music` (media files),
  iCloud Drive, mail, messages, notes, browser history, cookies.
- Only **installation metadata** may be read (name, version, bundle path, identifier) — never the
  contents of data files those applications open.

## 3. No passwords, keys, or secrets

- Passwords, tokens, private keys, license keys, certificates, seed phrases, and any secrets
  **must never** enter the analysis or the manifests.
- Forbidden to read: Keychain, `~/.ssh`, `~/.gnupg`, `*.p12`/`*.pem`/`*.key`, `.env` files,
  `~/.aws/credentials`, password-manager files (KeePass `.kdbx`, etc.), config files containing tokens.
- Licenses are recorded **only as a fact of being required** (`notes: "requires vendor license/account"`),
  without transferring the keys themselves. The user performs activation manually on the target.
- If a collection command risks printing a secret — do not run it; record the item as `manual`.

## 4. Versions and dependencies in manifests (schema structure)

Every item must carry enough information for a reproducible install:

- `version` — exact version on the source (or `null` if unavailable).
- `expected` — expected value for verification on the target.
- **`dependencies`** — array of dependencies: `id`s of other items in this/another manifest, external
  applications (e.g. UAD → `ua-connect`), or runtimes (e.g. Universal Gcode Sender → Java).
  Use an empty array `[]` when there are no dependencies.
- **`requires_runtime`** — runtime/version critical for reproducibility (e.g. `node@24.10.0`,
  `java>=17`), or `null`.
- `install_method`, `apply`, `verify`, `post_apply` — self-contained, per the schema contract.

> Schema changes (`dependencies`, `requires_runtime`) **must be reflected in the contract block in both
> agent files** (`system-analyst.md`, `migration-assist.md`), otherwise they will drift.
> `migration-assist` must install respecting dependency order (dependencies before dependents) and must
> not install an item whose `dependencies` are not selected/satisfied. `/verify` enforces the inverse at
> selection time: a `selected` item may not have an unselected dependency (auto-enable it or ask).

## 5. Other privacy and security rules

- **Read-only on the source.** `system-analyst` installs and changes nothing — it only writes JSON to
  the project directory.
- **No `sudo` without explicit consent.** Collection must not require elevated privileges; if it does,
  the item is marked `manual`/`system` with an explanation rather than run silently.
- **Never touch system security automatically.** SIP, Gatekeeper, FileVault, and the firewall are
  recorded informationally; enabling/disabling is a manual checklist requiring user confirmation.
- **Data minimization.** Collect only what is needed to migrate apps and settings. Do not include extra
  identifiers, serial numbers, MAC addresses, UUIDs, or hardware bindings unless strictly required for
  verification (and then with a note in `notes`).
- **Manifests contain no PII.** No e-mail, account names, or login-bearing paths where avoidable
  (use `$HOME` instead of an absolute `/Users/<name>`).
- **Transparency.** Before writing manifests, show the user a compact summary: what was collected,
  what is marked `manual`, and what requires manual transfer of binaries/licenses.
- **User control.** The user can edit the final migration set (`selected`) manually; disputed or large
  decisions are confirmed with an explicit question, not chosen silently.
- **Payload directories** (`manifests/vst-payload/`, `manifests/colorsync/`) are created on demand only
  when there is something to stage, and contain only plugin/profile binaries — no config files with
  secrets and no user presets containing personal data.
