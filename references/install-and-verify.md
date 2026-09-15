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

- Grep the harness stdout for the plugin's `[name]` log prefix; a host half that loaded always logs.
- Request the plugin's routes on the harness port, for example `http://127.0.0.1:3080/<plugin-route>`, and confirm a real payload rather than a 404 from the fallback handler.
- Confirm first-run state was created, such as `$DSH_HOME/storages/<name>/config.json`. Its absence after a restart means the plugin never ran.
- Check that no duplicate-route throw appeared in the logs.

## Worked example: dsh-budget

- Host routes: `/__budget-status`, `/dsh-budget/mea.png`, `/dsh-budget/voice-list`, and one route per voice file.
- The status route returning JSON with `day`, `budget`, `spent`, `remaining`, and `configPath` proves both the host half and its storage layer are live.
- The voice-list route proves asset routes registered without exceeding the per-URL limit.
- Its only side effect on the harness is the `[dsh-budget]` log prefix plus the storage directory; anything else means the wrong plugin loaded.

## Common failure signatures

| Symptom | Likely cause |
| --- | --- |
| No `[name]` log at all | Not in `dsh.profile.bundles`, or the profile was not restarted |
| Route falls through to the default handler | Route never registered, or the try/catch swallowed a duplicate |
| Client half missing in the browser | No `dsh.client.platform`, or the client entry file was not shipped |
| Config edits ignored | Edited the wrong patch layer, or `patchReload` is not live and no restart happened |
