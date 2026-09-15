---
name: dsh-plugin-dev
description: "Develop, install, and debug DeepSeek Harness (dsh) Cordis plugins with a host half plus an optional web client half: package.json dsh.bundle.patch and dsh.client declarations, cordis.patch.yml row insertion, profile bundle wiring, ctx.webServer routes, llm/stream hooks, and post-install verification. Use when asked to create a dsh plugin, make a dsh plugin load, mount a plugin into a dsh profile, add a web UI panel to dsh, or diagnose why a dsh plugin has no effect."
---

# Develop DeepSeek Harness (dsh) Plugins

A dsh plugin is a normal npm package that dsh loads in two independent halves. Do not assume it is installed just because files were copied: a working install needs the plugin directory, the profile link, and the profile bundle entry all present.

## Follow this workflow

1. Read the local dsh sources before writing code. The installed packages ship `README.zh.md` with `description`/`kind` frontmatter; read those first, then grep `lib/index.js` for the exact exported function or hook you intend to use. Never invent APIs from memory.
2. Decide which halves you need:
   - Host half: always. Runs in the dsh/node process, registers routes, hooks, and effects.
   - Web client half: only when a browser UI is required. Served by the host through `/plugins`.
3. Write `package.json` with the dsh declarations (see "Declare the plugin").
4. Write `cordis.patch.yml` to insert your plugin row into the Cordis config tree.
5. Implement the host half with `apply`, `inject`, and `ctx.effect`.
6. Implement the client half with `window.__ModuleLoader__.load(...)` when needed.
7. Install and verify with the three checks in `references/install-and-verify.md`. Verification is part of the task, not optional.
8. Only restart the user's harness after saying what you are about to start, and use the launch script the user names.

## Declare the plugin

`package.json` carries the whole contract:

```json
{
  "name": "dsh-my-plugin",
  "exports": { ".": "./bundle/host.js", "./client": "./bundle/client.js" },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web", "inject": [], "external": [] }
  }
}
```

- `dsh.bundle.patch` points at the patch file dsh applies when the package is listed in `dsh.profile.bundles`.
- `dsh.client.platform` is a required string when a client half exists. `inject` and `external` are optional string arrays, `immediately` is an optional boolean. `parseDshClient` rejects a missing `platform`, so a client half without it fails loudly at load time.
- Keep the contract details in `references/client-half.md` in view while editing these fields.

## Mount with cordis.patch.yml

Insert a row rather than editing existing rows:

```yaml
- insert:
    - id: my-plugin
      name: dsh-my-plugin
```

Patch semantics matter: a patch replaces the target row's `config` entirely instead of merging field by field. Write the complete config you want. See `references/profile-and-patch.md` for resolution order and live reload behavior.

## Implement the host half

```js
export function apply(ctx, config) {
  ctx.inject(['webServer'], (ctx) => {
    try {
      ctx.webServer.register({ kind: 'prefix', path: '/dsh-my-plugin', handler });
    } catch {
      // duplicate (kind, path) throws; degrade to no-op instead of killing the host
    }
  });
  ctx.on('llm/stream', (options, next) => next());
}
```

- Register every side effect (timers, listeners, files, routes) through `ctx.effect` so unloading restores state.
- Guard `ctx.webServer.register` with try/catch: duplicate `(kind, path)` pairs throw.
- Use `internals.logger ?? ctx.logger` and a stable `[plugin-name]` prefix, so logs are greppable in the harness output.

## Implement the web client half

Bundle the client as a lazy CJS factory; running the file must only register the factory. Module side effects belong inside `factory`.

```js
window.__ModuleLoader__.load({
  id: 'dsh-my-plugin',
  factory: (require) => { /* DOM work here */ },
});
```

Do not depend on arbitrary packages: only the shared platform module base is available. Read `references/client-half.md` before choosing dependencies.

## Guard against the usual failures

- Missing `lib/client.js` or an equivalent client entry fails the build with an explicit package list; ship the built file, do not rely on a build step at load time.
- A single combo URL must stay under 3 KiB; split assets into separate routes instead of one giant URL.
- `llm/stream` is a waterfall hook. Preserve the protocol invariant that final usage arrives before the terminating finish; do not emit a finish early when filtering chunks.
- Do not report success from the installer's exit code alone. Always re-check the three install judgments.

## Read the references

- `references/host-half.md` — host APIs: webServer registration kinds and precedence, `llm/stream` semantics, effect and logging conventions.
- `references/client-half.md` — `parseDshClient` field contract, module loader bundle format, shared module base, combo URL limits.
- `references/profile-and-patch.md` — profile layout, bundle resolution order, patch layering, `dsh plugin` pnpm forwarding, live reload.
- `references/install-and-verify.md` — install steps, the three success judgments, and HTTP verification commands.
