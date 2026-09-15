# Host Half Reference

Verified against `@deepseek-ai/dsh-host-webserver` and `@deepseek-ai/dsh-llm` in the local install.

## webServer registration

`register({ kind, path, handler })`:

- `kind` is `"exact"` or `"prefix"`.
- Matching order: exact match first, then longest prefix, then the fallback handler.
- A duplicate `(kind, path)` pair throws immediately. Wrap registration in try/catch and degrade to a no-op so one bad plugin cannot take down the host.

Handlers serve whatever the plugin needs: JSON status endpoints, images, per-file asset routes. Prefer one route per file over packing many assets into a query string.

## llm/stream hook

`llm/stream` is a waterfall hook. Signature:

```js
ctx.on('llm/stream', (options, next) => {
  // read the request: it is deep-frozen, so inspect it and decide, do not rewrite it
  return next();
});
```

- The request object handed to the hook is deep-frozen. Reading it (routing, gating, accounting) works; assigning fields does not. Still requires actual verification against the serialized request in your pipeline.
- Chunks arrive as token deltas.
- Protocol invariant: the final `usage` arrives before the terminating `finish`. Any wrapper must keep that order.
- Pre-request rejection is legitimate: throwing from the hook blocks the call, which is how a quota or budget gate refuses a request.
- Accounting happens after the response resolves, keyed on the reported usage.

## Effect and logging conventions

- Register timers, listeners, file handles, and routes through `ctx.effect` so plugin unload restores the prior state instead of leaking.
- Log through `ctx.logger` with a `[plugin-name]` prefix.

## Injection

`ctx.inject(['webServer'], (ctx) => { ... })` defers plugin setup until the named services exist. Always wrap service-dependent setup this way instead of reading services at top level.

## Two loading paths, two contexts

Do not conflate them. The context a half receives depends on which path mounted it.

- **Plain profile plugin** — a package listed in `dsh.profile.bundles`. `cordis-plugin-loader` mounts it through `registry.plugin`, and `apply` receives the ordinary Cordis fiber context: `ctx.logger`, `ctx.inject`, `ctx.effect`, `ctx.on` and the rest are all present, and any service registered on the root is reachable once declared.
- **Dynamic Cordis package** — code created at runtime via `cordis_define` / `cordis_run`. Only these run inside `dsh-cordis-host-runner` (service `dynamicCordisRunner`, mounted by `dsh-web-app`'s `cordis.patch.yml`), which hands each half a sandbox context facade: a whitelist `Proxy` exposing a fixed verb set (`CTX_VERBS`, `TIMER_VERBS`).

Consequences:

- The sandbox facade is a dynamic-package restriction only. It does not expose `logger` or `inject`, so a dynamic half cannot use them — a normally bundled plugin uses both freely. Everything in this reference that mentions sandboxing applies to the dynamic path alone.
- Declaring services in `inject` still matters everywhere, but the mechanism is Cordis' own proxy rule: touching an undeclared service throws `cannot get property without inject`. That is not a sandbox behavior.
- Framework internals (`root`, `fiber`, `registry`, `extend`, `plugin`) are withheld inside the sandbox by design. There is no supported `internals.logger` on any path. A handful of first-party packages accept a third `internals` argument as a test probe (`dsh-llm-retry`, `dsh-web-app`, `dsh-headless`); `dsh-budget` even reads `internals.logger ?? ctx.logger`, where the fallback is what actually runs. Use `ctx.logger`.
