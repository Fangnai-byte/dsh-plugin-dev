# Web Client Half Reference

Verified against `@deepseek-ai/dsh-client-modules`.

## Declaration contract

`parseDshClient` validates the `dsh.client` object in `package.json`:

- `platform` — required string. Missing or non-string causes a loud failure.
- `inject` — optional string array.
- `external` — optional string array.
- `immediately` — optional boolean.

## How the browser gets the bundle

- The host serves client bundles through the `/plugins` route.
- The browser loads them as lazy CJS: executing the bundle file only registers a factory; module side effects run when the module is materialized.
- Bundle format:

```js
window.__ModuleLoader__.load({
  id: 'dsh-my-plugin',
  factory: (require) => {
    // real work here: DOM, polling, event listeners
  },
});
```

## Available modules

The shared base covers React, Cordis, and static UI libraries. `PLATFORM_MODULES` is the README's label for it; the implementation field the client loader reads is `staticModules`. Anything else must either be bundled into the client file or declared as an external the host can provide. When in doubt, ship zero-dependency DOM code.

## URL limits

A single combo URL must not exceed 3 KiB (`MAX_COMBO_URL_BYTES = 3 * 1024`). Expose multiple small routes, for example one metadata route listing asset names plus one route per asset file.

## Build failures you will actually see

- A missing client entry (for example `lib/client.js`) fails loudly with build instructions and the list of expected packages. Ship the built artifact with the package; do not build at load time.
- A client half that omits `dsh.client.platform` fails during parsing, before any UI appears.
