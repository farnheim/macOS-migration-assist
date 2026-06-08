---
name: verify
description: Run after /collect on the SOURCE Mac. Walk every manifest category-by-category and confirm exactly what to migrate, with a "select all" shortcut and per-group customization, then write the chosen selection back to the manifests. Enforces dependency consistency. Use when the user wants to review, confirm, curate, or approve what gets migrated.
---

# /verify — confirm the migration selection

Interactive, human-in-the-loop review of the collected manifests. **Runs in the main conversation** (the one place `AskUserQuestion` works) — this is where the user approves the migration set.

## 0. Preconditions
Require `manifests/{macos-settings,homebrew,applications,vst}.json`. If any is missing, tell the user to run **/collect** first, then stop.

## 1. Walk each category, in this order: macos-settings → homebrew → applications → vst
For each manifest:
  a. **Print the full list**, grouped by `group`. Per item show: name, version, `install_method`, current selected (✓/✗), and badges for `dependencies` (⛓), `requires_runtime` (⚙), `manual` (✍), and copy-payload (📦 needs staged binaries). Show the running selected/total for the category.
  b. **Ask one decision** with AskUserQuestion for the whole category:
     - **Select all** — set every item `selected: true`.
     - **Keep current** — keep the heuristic selection from /collect.
     - **Customize** — drill into the category's groups.
     - **Skip all** — set every item `selected: false`.
  c. If **Customize**: present **every item individually** as AskUserQuestion options (multiSelect, ≤4 options per question, page through with multiple questions). Do **not** collapse items into pre-made bundles or a per-`group` all-or-nothing toggle — `group` is only a display heading for organizing/ordering the list, not a selectable unit (see rules.md §0). The user checks the items to migrate; translate checks into `selected` flags. The sole exception is a manifest item that is *itself* an intentional bundle (e.g. the UAD suite as one `vst-uad-suite` item) — that toggles as the single item it already is.

## 2. Enforce dependency consistency (rule 4)
After applying the user's choices, for every item now `selected: true` with a non-empty `dependencies`, ensure each dependency `id` is also selected. If a dependency is off, auto-enable it and tell the user why (or ask if it's a large/cost decision). Never leave a selected item with an unmet dependency. Known chains:
- `vst-uad-bundle` → `app-ua-connect`
- `app-cryptopro-ecp` → `app-cptools`
- `app-totalmix` → `app-fireface-usb-settings`

Also surface every selected item's `requires_runtime` (e.g. `java>=17`, CryptoPro CSP, Elektron Overbridge, Rutoken hardware token) so the user knows what the target must provide.

## 3. Write back & summarize
Edit only the `selected` field in the JSON files — change nothing else.

Then **regenerate `manifests/Brewfile`** as the projection of the now-final selection (see CLAUDE.md "Brewfile = the verified install set"): every `selected: true` item with `install_method ∈ {tap, formula, cask, mas}` across `homebrew.json` **and** `applications.json`, emitted `tap "x"` / `brew "x"` / `cask "x"` / `mas "Name", id: N` (taps → brews → casks → mas). It must match the confirmed selection exactly — a cask you just deselected must disappear from the Brewfile.

Then print a final table:
- per category: selected / total;
- the **manual / licensed** items the user must install by hand;
- the **copy-payload** binaries to stage before migrating — create `manifests/vst-payload/{VST,VST3,Components}/` on demand and drop the plugin binaries in (e.g. Valhalla, Sonic Charge, Degrader). `manifests/colorsync/` is only needed if there are custom ICC profiles (none on this source). These dirs are not pre-created.

End by telling the user to copy the whole `manifests/` dir (including payload subdirs) to the new Mac and run **/migrate** there.
