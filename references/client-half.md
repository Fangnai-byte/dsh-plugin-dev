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

The host owns the shared module base; a bundle only receives a `require` that resolves what the host is willing to hand over. Anything else must be bundled into your client file, or declared in `dsh.client.external` so another loaded package can answer it.

Two rules that `dsh-client-modules` enforces at composition time, both throwing with `client-modules:` prefix:

- An `external` specifier must name a package row that is actually present in the same composition.
- A row must not list itself in `external` (`"<id>" requests module … that it answers itself`).

The exact list of base specifiers is not documented in the package; read the host's own client entry (`dsh-web-frontend` / `dsh-web-app`) if you need to know, and when in doubt ship zero-dependency DOM code.

## URL limits

A single combo URL must not exceed 3 KiB (`MAX_COMBO_URL_BYTES = 3 * 1024`). Expose multiple small routes, for example one metadata route listing asset names plus one route per asset file.

## Build failures you will actually see

- A missing client entry fails with `client-modules: client bundle not found; run \`pnpm run build\` before launch`, plus the package name and the path it looked for. Missing bundles across packages are aggregated into one `N client packages failed to compose` error. Ship the built artifact with the package; never build at load time.
- A client half that omits `dsh.client.platform` fails during parsing, before any UI appears.
