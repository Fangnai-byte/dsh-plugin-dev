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
  // inspect or rewrite options before the request
  return next();
});
```

- Chunks arrive as token deltas.
- Protocol invariant: the final `usage` arrives before the terminating `finish`. Any wrapper must keep that order.
- Pre-request rejection is legitimate: throwing from the hook blocks the call, which is how a quota or budget gate refuses a request.
- Accounting happens after the response resolves, keyed on the reported usage.

## Effect and logging conventions

- Register timers, listeners, file handles, and routes through `ctx.effect` so plugin unload restores the prior state instead of leaking.
- Log through `ctx.logger` with a `[plugin-name]` prefix.

## Injection

`ctx.inject(['webServer'], (ctx) => { ... })` defers plugin setup until the named services exist. Always wrap service-dependent setup this way instead of reading services at top level.

## Sandbox context: declare before you touch

Dynamic plugin halves never receive the raw Cordis root context. `dsh-cordis-host-runner` hands each half a sandbox context facade: a whitelist `Proxy` that only exposes a fixed verb set (`CTX_VERBS`, `TIMER_VERBS`). Two consequences:

- Touch only services you declared in `inject`. Reaching for an undeclared service hits an internal guard and rejects.
- Framework internals (`root`, `fiber`, `registry`, `extend`, `plugin`, and the private `internals` bag) are withheld by design. There is no supported `internals.logger`; use `ctx.logger`.

So the plugin `apply` is not called with the module's own context either — it runs as `objectPlugin.apply(sandboxContext(ctx, ...), config)`. Anything absent from that facade is unavailable, not merely undocumented.
