# manifests/templates

Reference skeletons for the five manifest files the agents read and write. **These are tracked in git;
the real generated manifests in `manifests/` are git-ignored** because they contain a personal software
inventory (your apps, versions, paths, machine model).

You normally do **not** copy these by hand — `/collect` + `/verify` generate the real files for you. The
templates exist so you can:

- see the full schema and every field before running anything (the authoritative field list is in the root
  `README.md` and `.claude/rules.md` §4);
- hand-author or patch a manifest if needed;
- understand the contract when working on the agents/skills.

Each `*.template.json` carries `schema_version` `1.1`, placeholder `source`/`generated_at`, and one
representative `item` per relevant `install_method`. Replace `<…>` placeholders with real values, or just
let the agents produce the real manifests.

| Template | Real file (generated, git-ignored) |
|----------|------------------------------------|
| `migration-context.template.json` | `manifests/migration-context.json` |
| `macos-settings.template.json` | `manifests/macos-settings.json` |
| `homebrew.template.json` | `manifests/homebrew.json` |
| `applications.template.json` | `manifests/applications.json` |
| `vst.template.json` | `manifests/vst.json` |
| `Brewfile.template` | `manifests/Brewfile` |
