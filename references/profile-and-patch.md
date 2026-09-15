# Profiles and Patch Layering Reference

Verified against the local `@deepseek-ai/dsh` CLI package.

## Layout

- `$DSH_HOME` (usually `%USERPROFILE%\.dsh`): `plugins/`, `storages/`, `profiles/<name>/`, and a home-level `cordis.patch.yml`.
- `profiles/<name>/package.json` holds `dsh.profile.bundles` (the combo list) and `dependencies`.
- `profiles/<name>/cordis.patch.yml` is the profile-level patch file.
- `profiles/<name>/node_modules/` holds plugin links.

## Config overlay order

From an empty root, applied in this order, later layers winning:

1. Each bundle listed in `dsh.profile.bundles`, in array order.
2. The profile's own `cordis.patch.yml`.
3. `$DSH_HOME/cordis.patch.yml`.
4. `--patch` overlays from the command line.

## Bundle resolution

Bundles resolve first from the dsh installation directory, then from the profile's `node_modules`. A plugin must therefore be reachable from the profile to be loadable.

## Patch file mechanics

- Rows are inserted with an array form: `- insert:` followed by the new row objects.
- A patch replaces the target row's `config` wholesale rather than merging keys. Always write the full config.

## Managing plugin packages

`dsh plugin --profile <name> <pnpm args>` forwards straight to pnpm inside the profile, so `dsh plugin --profile web add dsh-my-plugin` is the package-manager path. Local development usually links instead: a `link:` dependency in `package.json` plus a junction in the profile's `node_modules`.

## Reload behavior

`patchReload: live` watches the profile-level and home-level patch files, so config edits apply without a restart. Code changes to a plugin still need a process restart unless the half is reloaded by the loader.
