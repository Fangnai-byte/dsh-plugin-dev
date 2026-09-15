# Install and Verify Reference

## The three success judgments

A dsh plugin is installed only when all three hold. Check all three; a copied directory alone means nothing.

1. The plugin directory exists: `$DSH_HOME/plugins/<name>`.
2. The profile links it: `$DSH_HOME/profiles/<profile>/node_modules/<name>`, usually a junction pointing back at the plugin directory.
3. The profile declares it in `package.json`: `<name>` appears in `dsh.profile.bundles`, and a matching `link:` entry appears in `dependencies`.

## Typical install steps

1. Copy the plugin into `$DSH_HOME/plugins/<name>`.
2. Create the link under `profiles/<profile>/node_modules/`.
3. Add the `link:` dependency and the bundle entry in `profiles/<profile>/package.json`.
4. Restart the profile, or rely on live patch reload when only config changed.

Make installers idempotent: re-running must not duplicate bundle entries or fail on an existing link.

## Verify at runtime, not by exit code

After restarting the harness:

- Boot completion is the gate. A bundled plugin whose entry fails to activate aborts boot outright (`assertEntriesActivated` in `dsh-app-boot`), so a harness reaching normal startup has activated every entry, including yours. A plugin missing from `dsh.profile.bundles` produces no output at all — silence is not success.
- Do not judge by grepping stdout. The install ships no console exporter, `ctx.logger` writes only to a 1000-entry in-memory buffer, and loader-level load logs require `enableLogs`, which defaults to off.
- Request the plugin's routes on the harness port — `3080` is the default, configurable via `ctx.webStartup.port` — for example `http://127.0.0.1:3080/<plugin-route>`, and confirm a real payload rather than a 404 from the fallback handler.
- Confirm first-run state was created. Check the artifacts your own plugin declares (the files or directories it writes), not a path you assume the host provides: the shared storage root is `$DSH_HOME/storages`, and a JSON storage unit lands as `<root>/<unit>.json` for single-record mode or `<root>/<unit>/<table>/<key>.json` plus optional `global.json` for per-record mode. A plugin that keeps its config under `$DSH_HOME/storages/<name>/config.json` chose that layout itself, so it is not a general first-run probe. Absence after a restart still means the plugin never ran.
- Check that no duplicate-route throw appeared. An unguarded duplicate `(kind, path)` throws during registration; if boot dies while activating your entry, suspect this first.

## Worked example: dsh-budget

- Host routes: `/__budget-status`, `/dsh-budget/mea.png`, `/dsh-budget/voice-list`, and one route per voice file.
- The status route returning JSON with `day`, `budget`, `spent`, `remaining`, and `configPath` proves both the host half and its storage layer are live.
- The voice-list route proves asset routes registered without exceeding the per-URL limit.
- Its only side effect on the harness is the `[dsh-budget]` log prefix plus the storage directory; anything else means the wrong plugin loaded.

## Common failure signatures

| Symptom | Likely cause |
| --- | --- |
| No effect at all, harness starts fine | Not in `dsh.profile.bundles`, or the profile was not restarted |
| Route falls through to the default handler | Route never registered, or the try/catch swallowed a duplicate |
| Client half missing in the browser | No `dsh.client.platform`, or the client entry file was not shipped |
| Config edits ignored | Edited the wrong patch layer, or `patchReload` is not live and no restart happened |
