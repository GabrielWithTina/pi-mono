# http-proxy.ts — HTTP Proxy Setup

**Source:** `packages/ai/src/utils/http-proxy.ts`

## Purpose

Side-effect module that configures global HTTP proxy for `fetch`-based SDKs in Node.js. Bun has built-in proxy support.

## Setup (lines 8–13)

```typescript
if (typeof process !== "undefined" && process.versions?.node) {
    import("undici").then((m) => {
        const { EnvHttpProxyAgent, setGlobalDispatcher } = m;
        setGlobalDispatcher(new EnvHttpProxyAgent());
    });
}
```

- **Line 8**: Guards to Node.js only (not Bun, not browser)
- **Line 9**: Dynamic import of `undici` — Node's HTTP agent library
- **Line 11**: `EnvHttpProxyAgent` reads `HTTP_PROXY` and `HTTPS_PROXY` env vars
- **Line 11**: `setGlobalDispatcher` makes all `fetch()` calls use the proxy agent

No explicit exports. ES module caching ensures setup runs exactly once regardless of import count. Imported at `stream.ts` line 2 and `oauth/index.ts`.
